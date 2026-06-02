# TDD-001. Agent Chat Runtime Flow 기술 설계

- 작성일: 2026-05-15
- 문서 상태: 초안
- 범위: 카카오톡 사용자 질문 입력부터 Daon Agent 답변 제공까지의 백엔드 런타임 흐름
- 관련 문서:
  - `daon-docs/PRD/[PRD-001]domestic-stock-llm-analysis.md`
  - `daon-docs/PRD/[PRD-002]kis-api-data-strategy.md`
  - `daon-docs/ADR/[ADR-001]backend-stack-python-vs-spring-boot.md`
  - 카카오 챗봇 스킬 응답 포맷: https://kakaobusiness.gitbook.io/main/tool/chatbot/skill_guide/answer_json_format
  - 카카오 AI 챗봇 콜백 개발 가이드: https://kakaobusiness.gitbook.io/main/tool/chatbot/skill_guide/ai_chatbot_callback_guide

## 1. 목적

이 문서는 Daon의 대화형 국내주식 분석 Agent가 카카오톡 메시지를 수신한 뒤, 사용자 질문을 분석 가능한 context로 구조화하고, Spring AI tool calling을 통해 필요한 분석용 domain tool을 선택하며, Spring Boot가 KIS Open API 호출과 정책 검증을 통제한 뒤 근거 기반 답변을 생성해 다시 카카오톡으로 반환하는 전체 런타임 흐름을 정의한다.

Daon의 핵심 방향은 단순히 LLM이 자유롭게 금융 API를 호출하는 챗봇이 아니다. Spring Boot 백엔드가 금융 데이터 수집, 캐시, 호출 제한, tool 실행, trace, 안전장치를 소유하고, LLM은 자연어 이해, 분석 context 생성, tool 선택, 근거 부족 판단, 근거 기반 답변 생성을 담당한다.

## 2. 설계 원칙

- 카카오톡은 입출력 채널이며, 분석 로직은 내부 Agent API가 담당한다.
- PRD-002의 데이터 범위가 넓으므로 질문 유형을 세밀한 intent enum으로 모두 분류하지 않는다.
- Agent Planner는 질문을 분석해 scope, 명확화 필요 여부, 분석 대상 후보를 structured output으로 생성한다.
- domain tool 선택은 Spring AI tool calling을 사용하는 Analyst 단계가 수행한다.
- LLM은 종목명, 별칭, 시간 표현, 비교 대상, 분석 초점의 후보를 추출할 수 있지만, 종목코드, 시장, 시간 범위의 최종 확정은 Spring Boot가 수행한다.
- LLM에게 KIS raw API를 직접 노출하지 않는다. LLM-facing tool은 분석 의미 단위의 domain tool로 제한한다.
- KIS raw API 호출, 파라미터 구성, 연속조회, 캐시/TTL, rate limit, 응답 정규화는 Spring Boot 내부 adapter와 domain tool executor가 담당한다.
- LLM은 분석 중 필요한 domain tool을 선택할 수 있지만, 실제 tool 실행 권한은 Spring Boot가 가진다.
- Agentic loop는 허용 tool, 최대 반복 횟수, 호출 예산, 중복 호출 방지 정책 안에서만 동작한다.
- KIS API 응답 원문을 LLM에 그대로 전달하지 않고, Spring Boot가 분석용 evidence packet으로 정리한 뒤 전달한다.
- 답변은 확인된 사실, 추정, 불확실성을 분리한다.
- 모든 답변에는 데이터 기준 시각과 데이터 제한 사항을 포함한다.
- 모든 Agent 실행은 추적 가능해야 한다. 어떤 tool을 호출했고, cache hit 여부와 데이터 기준 시각이 무엇인지 저장한다.
- 초기 Kakao Webhook 구현은 동기 응답만 지원한다. 카카오 callback 응답(`useCallback`, `callbackUrl`)은 LLM/KIS 연동 후 응답 지연 문제가 실제로 확인되는 후속 단계에서 도입한다.

## 3. 전체 요청 흐름

```text
Kakao 사용자 메시지
  -> Kakao Webhook Adapter
  -> Internal Agent API
  -> Conversation Session 로딩
  -> LLM Agent Planner
      -> scope, 명확화 필요 여부, 분석 대상 후보 추출
  -> Scope/Entity/Time Resolver
      -> analysisTargets의 종목코드, 국내시장 범위, 조회 시간 범위 확정
  -> Tool-enabled Analyst LLM
      -> Spring AI domain tool schema를 보고 필요한 tool 선택
  -> User-Controlled Tool Execution
      -> Tool Policy Validation
      -> 캐시 확인
      -> 필요한 KIS raw API 호출
      -> 응답 정규화
  -> Evidence Packet 생성
  -> Agentic Analysis Loop
      -> LLM Analyst가 evidence 충분성 판단
      -> 필요 시 Spring AI tool call 생성
      -> Spring Boot Policy Validation
      -> 허용된 Domain Tool 실행
      -> Evidence Packet 갱신
      -> 반복 또는 최종 답변 결정
  -> LLM Answer Composer
  -> 답변 후처리/안전장치
  -> Kakao 응답 포맷 변환
  -> 사용자 답변
  -> Trace/Observability 저장
```

## 4. 컴포넌트 책임

| 컴포넌트 | 책임 | 비고 |
| --- | --- | --- |
| Kakao Webhook Adapter | 카카오톡 요청 검증, 메시지 추출, 중복 요청 방지, 내부 Agent 요청 변환 | 분석 로직을 포함하지 않는다. |
| Internal Agent API | 채널 독립적인 Agent 요청/응답 계약 제공 | 카카오톡, 웹 UI, 테스트 클라이언트가 동일 API를 사용한다. |
| Conversation Service | 사용자, 세션, 최근 대화 turn, 이전 종목/시간/비교 맥락 관리 | 후속 질문 처리에 필요하다. |
| Agent Planner | LLM structured output 기반 scope, 명확화 필요 여부, 분석 대상 후보 추출 | domain tool 이름은 선택하지 않는다. |
| Scope/Entity/Time Resolver | LLM 후보를 검증해 국내주식 종목코드, 시장 범위, 시간 범위를 최종 확정 | LLM에 최종 식별자 확정을 맡기지 않는다. |
| Tool-enabled Analyst | Evidence와 분석 context를 보고 Spring AI tool calling으로 필요한 domain tool 선택 | KIS raw API는 알지 못한다. |
| Tool Execution Controller | LLM tool call, 종목, 시간 범위, 호출 예산, 중복 호출, 금지 범위 검증 | user-controlled execution으로 Spring Boot가 소유한다. |
| Domain Tool Catalog | LLM에게 노출되는 분석 의미 단위의 tool 계약 제공 | KIS raw API를 직접 노출하지 않는다. |
| Domain Tool Executor | domain tool을 내부 KIS adapter, 캐시, 정규화 로직으로 실행 | Spring Boot가 명시적으로 실행한다. |
| Agentic Analysis Loop | LLM이 evidence 충분성을 판단하고 필요 시 추가 tool call을 생성하는 반복 루프 | Spring Boot의 policy validation을 통과한 tool만 실행한다. |
| KIS Adapter | KIS REST/WebSocket raw API 호출, 토큰, 재시도, rate limit, TTL 처리 | 실전 API 기준. 모의투자 API나 모의 데이터 대체 없음. |
| Evidence Packet Builder | KIS 응답을 LLM 입력용 근거 패킷으로 변환 | 원문 응답을 직접 전달하지 않는다. |
| Answer Composer | evidence 기반 한국어 답변 생성 | Spring AI ChatClient 사용 후보. |
| Answer Guard | 금지 표현, 과도한 단정, 기준 시각 누락, 근거 외 주장 검사 | 필요 시 재생성 또는 보정한다. |
| Channel Response Adapter | 내부 Agent 응답을 카카오톡 응답 형식으로 변환 | 채널별 표현 제약만 처리한다. |
| Trace Logger | agent run, tool call, evidence, LLM latency/cost 저장 | 관리자 화면과 운영 분석에 사용한다. |

## 5. Kakao Webhook Adapter

카카오톡 Webhook 계층은 메시지 채널 어댑터다. 이 계층은 사용자의 자연어 질문을 분석하지 않고, 카카오톡 이벤트를 내부 Agent 요청으로 변환하는 일만 수행한다.

초기 구현은 카카오 챗봇 Skill 서버의 기본 동기 응답 흐름만 대상으로 한다. 스킬 서버는 카카오 요청을 받은 HTTP 요청 안에서 내부 Agent 응답을 받아 `SkillResponse` 형식으로 반환한다. callback 흐름은 이 단계에서 구현하지 않는다.

주요 책임은 다음과 같다.

- 카카오 요청 서명 또는 인증 정보 검증
- 카카오 사용자 식별자 추출
- 메시지 ID와 텍스트 추출
- 중복 메시지 처리 방지
- 내부 사용자와 카카오 사용자 매핑
- 내부 Agent API 요청 생성
- Agent 응답을 카카오톡 응답 형식으로 변환

초기 구현에서 제외하는 책임은 다음과 같다.

- `userRequest.callbackUrl` 기반 비동기 최종 응답 전송
- `useCallback: true` 응답 반환
- callback token 만료, 재전송, 실패 응답 처리

위 항목은 LLM/KIS 호출 시간이 카카오 동기 응답 시간 한계를 초과하는 시점에 별도 설계로 도입한다.

내부 요청 예시는 다음과 같다.

```json
{
  "channel": "KAKAO",
  "externalUserId": "kakao-user-123",
  "messageId": "kakao-message-789",
  "message": "삼성전자 지금 왜 떨어지는 거야?",
  "receivedAt": "2026-05-15T10:12:30+09:00"
}
```

## 6. Internal Agent API

내부 Agent API는 Daon 분석 기능의 채널 독립 진입점이다. 카카오톡 Webhook, 웹 채팅 UI, Swagger 또는 내부 테스트 클라이언트는 모두 동일한 Agent API를 호출한다.

초기 API 후보는 다음과 같다.

```http
POST /api/agent/chat
```

요청 예시는 다음과 같다.

```json
{
  "channel": "KAKAO",
  "userId": "user-123",
  "sessionId": "session-456",
  "message": "삼성전자 지금 왜 떨어지는 거야?",
  "requestedAt": "2026-05-15T10:12:30+09:00"
}
```

응답 예시는 다음과 같다.

```json
{
  "answer": "기준 시각: 10:12\n삼성전자는 현재 전일 대비 하락 중이고...",
  "analysisTargets": [
    {
      "stockRefs": [
        "삼성전자"
      ],
      "timeRef": "지금"
    }
  ],
  "symbols": [
    {
      "code": "005930",
      "name": "삼성전자"
    }
  ],
  "dataAsOf": "2026-05-15T10:12:28+09:00",
  "warnings": [
    "뉴스/공시 제목은 원인 후보이며 확정 근거가 아닙니다."
  ],
  "traceId": "agent-run-123"
}
```

## 7. Conversation Session 관리

Conversation Service는 사용자의 최근 대화 맥락을 관리한다. PRD-001은 "그 종목", "아까 말한 구간", "다른 종목이랑 비교해줘" 같은 후속 질문을 요구한다. 이를 위해 최소한 다음 정보를 저장한다.

- 사용자별 active conversation
- conversation turn별 원문 질문과 최종 답변
- 최근 언급 종목
- 최근 분석 초점
- 최근 시간 구간
- 최근 비교 대상 종목
- 최근 사용한 domain tool과 evidence 요약
- 직전 Agent run trace

후속 질문 처리 예시는 다음과 같다.

| 사용자 질문 | 필요한 맥락 | 처리 |
| --- | --- | --- |
| "그럼 SK하이닉스는?" | 직전 분석 초점 | 같은 분석 초점으로 SK하이닉스에 대한 domain tool plan 생성 |
| "아까 그 구간 다시 설명해줘" | 직전 시간 구간 | 같은 종목과 시간 구간으로 장중 흐름 domain tool 실행 |
| "다른 종목이랑 비교해줘" | 직전 종목 | 비교 대상이 없으면 명확화 질문 |

## 8. LLM Agent Planner

MVP에서는 PRD-002의 넓은 데이터 범위를 질문 유형별 intent enum으로 모두 분류하지 않는다. 사용자의 질문은 "현재 급락", "특정 시점", "장기 추세", "수급", "재무", "이벤트"처럼 여러 분석 축이 섞일 수 있으므로, LLM은 세밀한 의도분류기보다 분석 context 생성자 역할을 한다.

Agent Planner의 역할은 답변 생성이나 tool 선택이 아니다. Planner는 사용자 메시지를 정해진 schema로 구조화하고, 이후 Resolver와 Analyst가 사용할 분석 대상 후보를 만든다.

Planner가 추출해야 하는 정보는 다음과 같다.

| 항목 | 설명 |
| --- | --- |
| 범위 판단 | 국내주식 분석 범위인지, 주문/계좌/해외주식 등 제외 범위인지 |
| 명확화 필요 여부 | 종목, 시간, 비교 대상, 질문 범위가 불충분한지 |
| 분석 대상 | 사용자가 언급한 종목 후보 목록, 시간 표현 후보, 해당 대상이 필요한 이유 |

Planner schema에는 별도의 요청 action, analysis focus enum, domain tool plan을 두지 않는다. 실제 tool 선택은 Spring AI tool calling을 사용하는 Analyst 단계가 domain tool schema와 설명을 보고 수행한다. Planner가 tool 이름을 미리 정하면 tool calling의 자율 선택 책임과 겹치고, Spring AI의 tool 설명 기반 선택 기능을 충분히 활용하지 못한다.

시간 후보는 `analysisTargets[].timeRef`에 둔다. 하나의 질문에 "오늘 10시쯤"과 "지금"처럼 여러 시간 표현이 섞일 수 있으므로, target을 여러 개 만들 수 있다. 단일 종목과 복수 종목은 모두 `analysisTargets[].stockRefs` 목록으로 표현한다.

Structured output 예시는 다음과 같다.

```json
{
  "scope": {
    "status": "SUPPORTED",
    "reason": null
  },
  "needsClarification": false,
  "clarificationQuestion": null,
  "analysisTargets": [
    {
      "stockRefs": [
        "삼성전자"
      ],
      "timeRef": "오늘 10시쯤",
      "reason": "장중 특정 시점의 하락 원인과 이후 흐름을 분석하기 위한 대상입니다."
    },
    {
      "stockRefs": [
        "삼성전자"
      ],
      "timeRef": "지금",
      "reason": "하락 이후 현재 상태를 함께 확인하기 위한 대상입니다."
    }
  ]
}
```

복수 종목 비교 질문은 다음처럼 표현한다.

```json
{
  "scope": {
    "status": "SUPPORTED",
    "reason": null
  },
  "needsClarification": false,
  "clarificationQuestion": null,
  "analysisTargets": [
    {
      "stockRefs": [
        "삼성전자",
        "SK하이닉스"
      ],
      "timeRef": "2026-05-12",
      "reason": "두 종목이 같은 날짜에 하락한 원인을 비교하기 위한 대상입니다."
    }
  ]
}
```

Java record 후보는 다음과 같다.

```java
public record AgentPlan(
    ScopeAssessment scope,
    boolean needsClarification,
    String clarificationQuestion,
    List<AnalysisTarget> analysisTargets
) {}

public record AnalysisTarget(
    List<String> stockRefs,
    String timeRef,
    String reason
) {}

public record ScopeAssessment(
    ScopeStatus status,
    String reason
) {}

public enum ScopeStatus {
    SUPPORTED,
    UNSUPPORTED
}
```

Planner는 종목코드, 시장, 시간 범위를 최종 확정하지 않는다. "삼전", "삼성전자", "4시간 전", "아까" 같은 자연어 후보를 추출하고, 최종 식별자와 조회 범위는 Spring Boot resolver가 확정한다.

`needsClarification == true`이면 domain tool 호출 이전에 `clarificationQuestion`을 사용자에게 반환한다. clarification 답변을 받아 이전 plan을 보완하는 multi-turn conversation은 후속 구현 범위로 분리한다.

## 9. Scope/Target/Tool 실행 검증

LLM이 만든 `AgentPlan`과 Spring AI tool call을 그대로 신뢰하지 않는다. Spring Boot는 다음 검증을 수행한다.

- 국내주식 분석 범위인지 확인한다.
- 주문, 정정/취소, 잔고, 계좌, 매수 가능 금액, 매수/매도 지시, 해외주식, 모의투자 API 요구를 unsupported로 처리한다.
- `analysisTargets`에서 종목 후보가 없으면 명확화 질문으로 전환한다.
- 복수 종목 비교 문맥에서 종목 후보가 2개 미만이면 명확화 질문으로 전환한다.
- 시간 표현을 계산할 수 없으면 명확화 질문으로 전환한다.
- Spring AI가 요청한 tool call이 Domain Tool Catalog에 등록된 read-only 분석 tool인지 확인한다.
- 같은 agent run에서 이미 호출한 tool과 같은 대상/시간 범위인지 확인한다.
- tool 호출 예산과 최대 loop 횟수를 초과하지 않는지 확인한다.
- KIS API 호출 제한 또는 큐 대기 상태가 허용 범위인지 확인한다.
- 캐시된 데이터로 충분히 대체 가능한지 확인한다.
- 후속 질문이면 conversation context로 누락 정보를 보완한다.

검증 실패 시 KIS API를 호출하지 않고 사용자에게 짧은 확인 질문 또는 제한 답변을 반환한다.

## 10. Stock Resolver

Stock Resolver는 사용자 입력의 종목명, 종목코드, 별칭을 내부 표준 종목으로 해석한다.

이 책임은 LLM에 맡기지 않는다. LLM은 "삼성전자", "삼전", "005930" 같은 언급 후보를 추출할 수 있지만, 최종 종목코드와 시장 확정은 종목 마스터 DB 또는 로컬 종목 코드 데이터가 담당한다.

초기 데이터 소스 후보는 다음과 같다.

- `데이터/kospi_code.txt`
- `데이터/kosdaq_code.txt`
- 향후 KIS `주식기본조회(CTPF1002R)` 기반 종목 마스터 테이블

해석 결과 예시는 다음과 같다.

```json
{
  "rawText": "삼성전자",
  "code": "005930",
  "name": "삼성전자",
  "market": "KOSPI",
  "confidence": 1.0
}
```

종목명이 모호하면 명확화 질문을 반환한다.

```text
"삼성"으로 검색되는 종목이 여러 개 있습니다.
삼성전자(005930), 삼성SDI(006400), 삼성전기(009150) 중 어떤 종목인가요?
```

## 11. Time/Market Resolver

Time/Market Resolver는 LLM이 추출한 시간 표현과 시장 범위 후보를 실제 조회 가능한 범위로 변환한다.

주요 책임은 다음과 같다.

- "지금", "오늘", "4시간 전", "요즘", "그때" 같은 표현을 기준 시각과 conversation context로 해석한다.
- 장운영 시간, 휴장일, 장 시작 전/후, 데이터 보관 범위를 고려해 조회 가능한 시간 범위를 만든다.
- "4시간 전"이 장 시작 전이거나 휴장일이면 KIS 조회를 강행하지 않고 명확화 또는 보정 질문을 반환한다.
- 국내주식 범위만 허용하고 해외주식, 국내선물옵션, 계좌/주문 범위는 unsupported로 처리한다.
- LLM이 제안한 시간 범위가 과도하게 넓으면 tool별 최대 범위로 축소하거나 확인 질문을 반환한다.

예시는 다음과 같다.

```text
현재 시각: 2026-05-20 10:30 KST
사용자 표현: "4시간 전에 왜 빠졌어?"
계산 결과: 2026-05-20 06:30 KST
판단: 장 시작 전이므로 장중 거래 데이터 없음
응답: "4시간 전은 장 시작 전이라 장중 거래 데이터가 없습니다. 오늘 장 시작 이후 흐름을 볼까요?"
```

## 12. Domain Tool Catalog

LLM에게는 KIS raw API를 직접 노출하지 않는다. 대신 질문 해결에 필요한 분석 의미 단위의 domain tool만 제공한다.

raw KIS API를 LLM-facing tool로 노출하지 않는 이유는 다음과 같다.

- KIS API 이름과 사용자 분석 의도가 1:1로 대응하지 않는다.
- TR ID, 시장 구분 코드, 날짜 형식, 연속조회, 조회 한도, 보관 기간, 장중/장후 의미 차이를 LLM이 매번 정확히 처리하기 어렵다.
- 질문 하나에서 과도한 API 호출이나 중복 호출이 발생할 수 있다.
- 복수 종목 비교에서 `종목 수 x raw API 수`로 호출량이 폭증할 수 있다.
- KIS raw 응답은 필드명, 코드값, 숫자 문자열이 많아 LLM 입력으로 부적합하며 evidence 정규화가 필요하다.
- 주문, 계좌, 모의투자 등 MVP 제외 API가 tool catalog에 섞이면 planner가 잘못된 방향으로 갈 수 있다.
- KIS 스펙 변경이 LLM-facing tool schema 변경으로 직접 전파되는 것을 피해야 한다.

초기 domain tool 후보는 다음과 같다.

| Domain Tool | 역할 | 내부 KIS API 후보 |
| --- | --- | --- |
| `GET_CURRENT_MARKET_SNAPSHOT` | 현재가, 등락률, 거래량/대금, 상태, 기본 시세 스냅샷 | 주식현재가 시세 `FHKST01010100`, 호가/예상체결 `FHKST01010200` |
| `GET_INTRADAY_PRICE_FLOW` | 당일 또는 특정 거래일 장중 분봉 흐름, 변곡점, 거래량 급증 구간 | 주식당일분봉조회 `FHKST03010200`, 주식일별분봉조회 `FHKST03010230` |
| `GET_HISTORICAL_PRICE_TREND` | 일/주/월/년 단위 중장기 가격 흐름 | 국내주식기간별시세 `FHKST03010100` |
| `GET_MARKET_AND_INDUSTRY_CONTEXT` | 시장/업종 동조 여부, 상대강도 | 국내업종 현재지수 `FHPUP02100000`, 업종 분봉조회 `FHKUP03500200`, 국내업종 일자별지수 `FHPUP02120000` |
| `GET_EVENT_CANDIDATES` | VI, 뉴스/공시 제목, 일정성 이벤트 후보 | VI 현황 `FHPST01390000`, 종합 시황/공시 제목 `FHKST01011800` |
| `GET_SUPPLY_DEMAND_CONTEXT` | 외국인/기관/프로그램/투자자 수급 맥락 | 투자자/프로그램매매 관련 P1 API |
| `GET_FUNDAMENTAL_CONTEXT` | 재무비율, 재무제표, 추정실적, 투자의견 | 재무비율, 손익계산서, 대차대조표, 안정성비율, 종목투자의견, 종목추정실적 |
| `COMPARE_STOCKS` | 복수 종목을 동일 기준으로 비교 | 위 domain tool의 종목별 조합 및 비교 요약 |

각 domain tool은 Spring Boot 내부 application service로 구현한다. Spring AI tool calling을 사용하더라도 LLM에게 노출되는 것은 domain tool schema이며, tool 실행 자체는 Spring Boot의 policy validation과 service orchestration을 거친다.

## 13. Domain Tool 실행

도구 호출 전에는 Redis 또는 DB 캐시를 먼저 확인한다.

- 현재가/호가 스냅샷 TTL
- 1분봉 최신성
- 뉴스/공시 제목 TTL
- 업종 지수 TTL
- KIS API 호출 제한과 큐 대기 상태

도구 호출 trace 예시는 다음과 같다.

```json
{
  "toolName": "GET_INTRADAY_PRICE_FLOW",
  "stockCode": "005930",
  "source": "DOMAIN_TOOL",
  "internalKisApis": [
    "FHKST03010200"
  ],
  "cacheHit": false,
  "dataAsOf": "2026-05-15T10:12:28+09:00",
  "latencyMs": 184,
  "status": "SUCCESS"
}
```

## 14. Evidence Packet 생성

KIS API 응답 원문은 LLM에 직접 전달하지 않는다. Evidence Packet Builder가 도메인 의미를 가진 구조로 정리한다.

Evidence packet은 다음 목적을 가진다.

- LLM 입력 길이를 통제한다.
- KIS 필드명을 답변 생성에 적합한 도메인 용어로 변환한다.
- 확인된 사실, 신호, 이벤트 후보, 제한 사항을 분리한다.
- 데이터 기준 시각을 명확히 전달한다.
- 뉴스/공시 제목을 원인 확정 근거가 아니라 후보 근거로 제한한다.

예시는 다음과 같다.

```json
{
  "question": "삼성전자 지금 왜 떨어지는 거야?",
  "scope": {
    "status": "SUPPORTED",
    "reason": null
  },
  "analysisTargets": [
    {
      "stocks": [
        {
          "code": "005930",
          "name": "삼성전자",
          "market": "KOSPI"
        }
      ],
      "timeRef": "지금",
      "reason": "현재 하락 원인과 장중 흐름을 분석하기 위한 대상입니다."
    }
  ],
  "dataFreshness": {
    "priceAsOf": "2026-05-15T10:12:28+09:00",
    "minuteChartAsOf": "2026-05-15T10:12:00+09:00",
    "newsAsOf": "2026-05-15T10:11:30+09:00"
  },
  "confirmedFacts": [
    "현재 전일 대비 하락 중",
    "10:05 이후 하락폭 확대",
    "해당 구간 거래량이 직전 10분 평균보다 증가"
  ],
  "marketContext": [
    "동일 시간 업종 지수도 약세",
    "시장 전체도 하락"
  ],
  "eventCandidates": [
    "관련 뉴스/공시 제목 1건 확인",
    "VI 발동은 확인되지 않음"
  ],
  "limitations": [
    "뉴스 제목만 확인했으며 원문 내용은 확인하지 않음",
    "현재 분봉은 장중 미확정 데이터"
  ]
}
```

## 15. Agentic Analysis Loop

Agentic Analysis Loop는 Daon이 더 agentic하게 동작하는 핵심 구간이다. LLM은 현재 evidence packet을 검토한 뒤, 근거가 충분한지 판단하고 필요하면 Spring AI tool calling으로 추가 domain tool을 선택한다.

루프의 기본 흐름은 다음과 같다.

```text
Evidence Packet
  -> LLM Analyst Review
  -> SUFFICIENT_EVIDENCE 이면 최종 답변 생성으로 이동
  -> NEED_MORE_DATA 이면 Spring AI tool call 생성
  -> Spring Boot Tool Execution Policy 검증
  -> 허용된 domain tool만 실행
  -> Evidence Packet 갱신
  -> LLM Analyst Review 반복
```

LLM Analyst의 structured output 후보는 다음과 같다.

```json
{
  "status": "NEED_MORE_DATA",
  "reason": "하락 구간의 거래량 증가는 확인됐지만 개별 수급 이슈인지 시장 동반 약세인지 근거가 부족합니다.",
  "answerReadiness": {
    "canAnswerWithCurrentEvidence": true,
    "riskIfAnswerNow": "수급 원인 가능성을 확인하지 못해 하락 원인 설명이 업종/거래량 중심으로 제한됩니다."
  }
}
```

`NEED_MORE_DATA` 상태에서는 Analyst LLM이 Spring AI tool calling 응답으로 다음과 같은 tool call을 만들 수 있다.

```json
[
  {
    "name": "get_supply_demand_context",
    "arguments": {
      "stockCode": "005930",
      "timeRange": "2026-05-15T09:00:00+09:00/2026-05-15T10:12:00+09:00"
    }
  },
  {
    "name": "get_event_candidates",
    "arguments": {
      "stockCode": "005930",
      "timeRange": "2026-05-15T09:00:00+09:00/2026-05-15T10:12:00+09:00"
    }
  }
]
```

충분한 근거가 있다고 판단한 경우는 다음과 같다.

```json
{
  "status": "SUFFICIENT_EVIDENCE",
  "reason": "현재가, 분봉, 업종 흐름, VI/뉴스 제목으로 현재 질문에 답변 가능한 수준의 근거가 확보됐습니다.",
  "answerReadiness": {
    "canAnswerWithCurrentEvidence": true,
    "riskIfAnswerNow": "뉴스 원문과 확정 수급 데이터는 없으므로 단일 원인 단정은 피해야 합니다."
  }
}
```

### 15.1 Tool Execution Policy

LLM이 만든 tool call은 바로 실행하지 않는다. Spring Boot의 Tool Execution Policy가 다음 항목을 검증한다.

- Domain Tool Catalog에 등록된 read-only 분석 tool인가
- 현재 분석 초점과 질문 범위에서 허용할 수 있는 tool인가
- 해당 tool이 이미 같은 agent run에서 호출됐는가
- tool 호출 예산을 초과하지 않는가
- 최대 loop 횟수를 초과하지 않는가
- KIS API 호출 제한 또는 큐 대기 상태가 허용 범위인가
- 캐시된 데이터로 충분히 대체 가능한가
- 요청한 종목과 시간 구간이 검증된 context 안에 있는가
- MVP 제외 범위인 주문, 계좌, 잔고, 해외주식, 모의투자 API를 요구하지 않는가

초기 MVP에서 허용하는 domain tool은 다음으로 제한한다.

| Tool | 사용 조건 | 비고 |
| --- | --- | --- |
| `GET_CURRENT_MARKET_SNAPSHOT` | 현재 상태, 등락률, 거래량/대금, 호가 스냅샷이 필요한 경우 | P0 |
| `GET_INTRADAY_PRICE_FLOW` | 장중 변곡점, 특정 시점 복기, 당일 특이사항 확인 | P0 |
| `GET_MARKET_AND_INDUSTRY_CONTEXT` | 시장/업종 동조 여부 또는 상대강도 확인 | P0 |
| `GET_EVENT_CANDIDATES` | VI, 뉴스/공시 제목, 이벤트 후보 확인 | P0 |
| `GET_HISTORICAL_PRICE_TREND` | 장기 추세 또는 복수 종목 비교에서 가격 흐름 확인 | P0 |
| `GET_SUPPLY_DEMAND_CONTEXT` | 수급 원인 후보 확인 | P1. MVP 1차에서는 `NOT_IMPLEMENTED` 결과 또는 후속 구현 후보 |
| `GET_FUNDAMENTAL_CONTEXT` | 장기 추세와 복수 종목 비교의 재무 맥락 확인 | P1. MVP 1차에서는 `NOT_IMPLEMENTED` 결과 또는 후속 구현 후보 |
| `COMPARE_STOCKS` | 2개 이상 종목을 동일 기준으로 비교 | 내부적으로 다른 domain tool을 조합 |

MVP 1차에서는 추가 tool call을 최대 2회 loop, 최대 2개 tool per loop로 제한한다.

### 15.2 Loop 종료 조건

Agentic loop는 다음 조건 중 하나를 만족하면 종료한다.

- LLM Analyst가 `SUFFICIENT_EVIDENCE`를 반환한다.
- 최대 loop 횟수에 도달한다.
- 허용된 tool call이 없다.
- tool 호출 예산을 초과한다.
- KIS API 장애 또는 rate limit으로 추가 수집이 불가능하다.
- 추가 tool 결과가 이미 확보한 evidence와 중복된다.

종료 시점에 evidence가 부족하더라도, 현재 확보한 근거와 제한 사항을 명시해 제한 답변을 생성한다.

### 15.3 Trace

Agentic loop는 반복 단계별로 trace를 남긴다.

- loop index
- LLM Analyst status
- 추가 데이터 요청 이유
- 제안된 tool call
- policy validation 결과
- 실행된 tool
- 거절된 tool과 거절 사유
- evidence packet 변경 요약
- loop 종료 사유

이 trace는 포트폴리오와 운영 관점에서 중요하다. 사용자의 질문 하나에 대해 Daon이 왜 추가 데이터를 요청했고, 어떤 정책 때문에 실행하거나 거절했는지 설명할 수 있어야 한다.

## 16. LLM Answer Composer

Answer Composer는 evidence packet을 사용자에게 읽기 쉬운 한국어 답변으로 변환한다. 이 단계에서 LLM은 새로운 금융 사실을 만들지 않고, 제공된 evidence 안에서만 설명한다.

답변 생성 정책은 다음과 같다.

- 첫 문단에 핵심 결론을 제시한다.
- 데이터 기준 시각을 표시한다.
- 확인된 사실과 추정을 분리한다.
- 원인이 확인되지 않으면 확인되지 않았다고 말한다.
- 뉴스/공시 제목은 원인 후보로만 표현한다.
- 투자 조언, 목표가, 매수/매도 신호처럼 보이는 표현을 피한다.
- 카카오톡에서 읽기 좋도록 짧고 구조화한다.

답변 예시는 다음과 같다.

```text
기준 시각: 10:12

삼성전자는 현재 전일 대비 하락 중이고, 하락폭은 10:05 전후부터 커진 흐름입니다.

확인되는 근거는 세 가지입니다.
1. 해당 구간의 1분봉에서 가격 하락과 거래량 증가가 함께 나타났습니다.
2. 같은 시간 업종 지수도 약세라서 개별 종목 단독 이슈보다는 업종/시장 동반 약세 영향이 섞여 보입니다.
3. 현재 확인 가능한 VI 발동은 없습니다.

다만 뉴스/공시 제목만으로 하락 원인을 단정할 만한 이벤트는 확인되지 않았습니다.
따라서 지금은 "업종 약세 속 거래량 동반 하락"으로 보는 편이 안전합니다.
```

## 17. 답변 후처리와 안전장치

Answer Guard는 LLM 답변을 후처리한다.

검사 항목은 다음과 같다.

- 기준 시각 포함 여부
- 데이터 제한 사항 포함 여부
- 과도한 단정 표현 포함 여부
- 매수/매도 권유 표현 포함 여부
- evidence에 없는 뉴스, 공시, 수급, 재무 원인 생성 여부
- 카카오톡 응답 길이 초과 여부
- unsupported 질문에 대한 부적절한 답변 여부

문제가 확인되면 다음 중 하나를 수행한다.

- 안전 문구 추가
- 답변 압축
- LLM 재생성
- 사용자에게 명확화 질문 반환
- 장애 시 사용자 안내용 제한 답변 반환

## 18. Kakao 응답 변환

Channel Response Adapter는 내부 Agent 응답을 카카오톡 응답 형식으로 변환한다.

초기 MVP에서는 텍스트 응답을 우선한다. 향후 필요하면 빠른 후속 질문 버튼, 자세히 보기 링크, 관리자 trace 링크를 확장할 수 있다.

카카오톡 응답 변환 정책은 다음과 같다.

- 카카오 챗봇 SkillResponse `version`은 `2.0`으로 고정한다.
- 초기 동기 응답은 `template.outputs[].simpleText.text`를 사용한다.
- callback 응답은 초기 구현 범위에서 제외한다.
- 핵심 결론을 먼저 보여준다.
- 너무 긴 근거는 3개 내외로 줄인다.
- 데이터 기준 시각과 제한 사항은 유지한다.
- 내부 trace ID는 사용자에게 기본 노출하지 않는다.
- 장애 메시지는 사용자가 이해할 수 있는 표현으로 변환한다.

초기 응답 예시는 다음과 같다.

```json
{
  "version": "2.0",
  "template": {
    "outputs": [
      {
        "simpleText": {
          "text": "기준 시각: 10:12\n삼성전자는 현재 전일 대비 하락 중이고, 10:05 전후부터 하락폭이 커진 흐름입니다."
        }
      }
    ]
  }
}
```

## 19. Trace와 Observability

Daon의 포트폴리오 강점은 "AI 답변" 자체보다 답변 생성 과정을 추적할 수 있다는 점이다. 다음 데이터를 저장한다.

| 데이터 | 저장 목적 |
| --- | --- |
| 원본 사용자 질문 | 재현과 품질 평가 |
| agent planner 결과 | scope, 명확화 필요 여부, analysisTargets 품질 분석 |
| conversation context 사용 여부 | 후속 질문 품질 분석 |
| resolver 결과 | 종목코드, 시장, 시간 범위 확정 근거 확인 |
| domain tool call 목록 | 어떤 데이터 묶음으로 답했는지 확인 |
| 내부 KIS API call 목록 | 실제 외부 API 호출량과 장애 분석 |
| agentic loop decision | 추가 데이터 요청 이유와 종료 사유 확인 |
| tool execution policy 결과 | 실행/거절된 tool과 정책 근거 확인 |
| cache hit/miss | 캐시 효율 분석 |
| KIS latency/error | 외부 API 장애 분석 |
| dataAsOf | 데이터 신선도 확인 |
| evidence packet | 답변 근거 확인 |
| LLM prompt/response 메타데이터 | 비용, 지연, 품질 분석 |
| 최종 답변 | 사용자 경험 재현 |

초기 테이블 후보는 다음과 같다.

- `conversation`
- `conversation_turn`
- `agent_run`
- `agent_tool_call`
- `agent_evidence`
- `agent_answer`

관리자 화면에서는 질문 하나를 클릭했을 때 다음을 확인할 수 있어야 한다.

- 어떤 analysisTargets가 생성됐는가
- 어떤 종목으로 해석됐는가
- 어떤 domain tool과 내부 KIS API를 호출했는가
- 캐시를 사용했는가
- 데이터 기준 시각은 언제인가
- LLM이 어떤 evidence로 답변했는가
- 어디에서 지연 또는 오류가 발생했는가

## 20. Error/제한 응답 정책

장애 또는 불확실성은 숨기지 않고 사용자에게 제한 사항을 알려야 한다.

| 상황 | 처리 |
| --- | --- |
| 종목을 해석할 수 없음 | 종목명 또는 종목코드 입력 요청 |
| 종목명이 모호함 | 후보 종목 제시 후 선택 요청 |
| Planner가 핵심 식별자 또는 시간 표현이 불명확하다고 판단함 | 명확화 질문 반환 |
| KIS 현재가 API 실패 | 캐시가 있으면 기준 시각과 함께 제한 답변, 없으면 일시 실패 안내 |
| 분봉 데이터 누락 | 현재가/호가/업종 기준 제한 답변 |
| 뉴스/공시 조회 실패 | 뉴스/공시 확인 불가를 명시하고 가격/거래량 중심 답변 |
| LLM planner 실패 | 짧은 재시도 후 실패 시 질문을 다시 요청 |
| LLM Analyst가 허용되지 않은 domain tool 요청 | tool을 실행하지 않고 현재 evidence 기준 제한 답변 |
| agentic loop 최대 횟수 초과 | 확보된 evidence와 제한 사항을 명시해 답변 |
| LLM 답변 생성 실패 | evidence 기반 템플릿 제한 답변 |
| 카카오톡 응답 제한 초과 | 요약 답변 우선 반환 |

## 21. MVP 구현 범위

1차 MVP는 모든 domain tool을 동시에 완성하지 않는다. 첫 구현 대상은 현재 또는 당일 국내주식 가격 움직임 질문과 복수 종목 비교 질문을 처리할 수 있는 최소 tool set으로 좁힌다.

포함 범위는 다음과 같다.

- Kakao Webhook Adapter 기본 구조
- Internal Agent API
- Conversation Session 최소 저장
- LLM structured Agent Planner
- Scope/Target validation
- Stock Resolver
- Time/Market Resolver
- Domain Tool Catalog 기본 구조
- Spring AI tool calling 기반 Tool-enabled Analyst
- User-controlled tool execution
- `GET_CURRENT_MARKET_SNAPSHOT`
- `GET_INTRADAY_PRICE_FLOW`
- `GET_MARKET_AND_INDUSTRY_CONTEXT`
- `GET_EVENT_CANDIDATES`
- `COMPARE_STOCKS`
- Evidence Packet Builder
- Agentic Analysis Loop
- Tool Execution Policy
- 최대 2회 loop, loop당 최대 2개 추가 tool call 제한
- LLM Answer Composer
- Answer Guard 기본 검사
- Agent run/tool call trace 저장

후속 범위는 다음과 같다.

- `GET_HISTORICAL_PRICE_TREND`
- `GET_SUPPLY_DEMAND_CONTEXT`
- `GET_FUNDAMENTAL_CONTEXT`
- 관리자 trace 화면
- 관심 종목/tracing 관리
- Web UI 또는 내부 테스트 클라이언트
- 카카오 callback 응답(`useCallback`, `callbackUrl`) 기반 장시간 분석 응답

## 22. Spring AI 사용 범위

MVP에서 Spring AI는 다음 영역에 사용한다.

- ChatClient 기반 LLM 호출
- structured output 기반 Agent Planner
- Spring AI tool calling 기반 analyst tool 선택
- user-controlled tool execution
- structured output 기반 analyst review
- evidence packet 기반 answer composition
- 필요 시 chat memory/advisor 적용
- LLM 호출 관측성 연동

Spring AI가 소유하지 않는 영역은 다음과 같다.

- KIS API adapter
- KIS 토큰 갱신
- rate limit과 retry 정책
- Redis/DB 캐시 TTL
- domain tool 실행과 내부 KIS API 조합
- tool execution policy validation
- 데이터 신선도 판단
- evidence packet 생성 규칙
- 금융 답변 안전 정책
- 카카오톡 Webhook 처리
- trace 저장 스키마

## 23. LangGraph 검토 위치

LangGraph는 MVP 핵심 런타임으로 사용하지 않는다. 현재 Daon MVP는 장기간 상태를 저장하며 재개되는 복잡한 multi-agent graph보다, Spring Boot가 통제하는 bounded agentic loop가 더 중요하다.

LangGraph 또는 별도 graph runtime은 다음 조건에서 재검토한다.

- 장중 이상 종목 자동 감시 후 multi-step 조사 workflow가 필요해진 경우
- 사람이 중간 승인하는 리포트 생성 workflow가 필요해진 경우
- 여러 Agent가 역할을 나누는 장기 실행 작업이 필요해진 경우
- LLM/분석 worker를 Spring Boot 운영 API와 분리해야 하는 경우

## 24. 테스트 전략

초기 테스트는 다음 계층으로 나눈다.

| 테스트 | 검증 대상 |
| --- | --- |
| Agent Planner schema test | LLM 결과가 Java schema로 안정적으로 변환되는지 |
| Scope/Target Validator unit test | 종목 누락, unsupported scope, 불명확한 시간/대상 처리 |
| Stock Resolver unit test | 종목명/코드/모호한 입력 해석 |
| Time/Market Resolver unit test | 상대 시간, 휴장일, 장 시작 전/후, 과도한 범위 처리 |
| Domain Tool Catalog test | LLM-facing tool schema와 내부 실행 계약 검증 |
| Agentic Loop test | 추가 tool call 반복과 종료 조건 |
| Tool Execution Policy test | 허용/거절 tool, 중복 호출, 예산 초과 처리 |
| KIS Adapter contract test | KIS 응답을 내부 DTO로 변환 |
| Evidence Packet test | 확인 사실/추정/제한 사항 분리 |
| Answer Guard test | 금지 표현, 기준 시각 누락, 과도한 단정 탐지 |
| Agent API integration test | 질문 입력부터 내부 Agent 응답까지 |
| Kakao Adapter test | 카카오 요청에서 발화/사용자 식별자를 추출하고 동기 `SkillResponse`로 변환 |

외부 KIS 실전 API와 LLM 호출은 테스트 환경에서 fixture 또는 stub을 우선 사용한다. 실전 API 검증은 별도 수동/운영 검증 시나리오로 분리한다.

## 25. 미결정 사항

- LLM provider와 모델
- Spring AI version
- 명확화 질문 전환 기준
- agentic loop 최대 횟수와 tool 호출 예산
- domain tool별 호출 예산, 최대 시간 범위, TTL
- P1 domain tool의 `NOT_IMPLEMENTED` 처리와 실구현 도입 기준
- 카카오톡 동기 응답 시간 제한 초과 시 callback 도입 기준
- Redis TTL 상세값
- PostgreSQL trace 스키마 상세
- KIS API 호출 제한 기반 큐 정책
- Agent API 인증 방식
- 관리자 trace 화면 도입 시점

## 26. 후속 문서화 범위

다음 항목은 별도 문서에서 다룬다.

- KIS Adapter 상세 설계
- Redis/DB 캐시 TTL과 스키마
- Agent Planner prompt와 structured output schema
- Domain Tool Catalog 상세 설계
- Agentic loop와 tool execution policy 상세 설계
- PostgreSQL trace schema
- 카카오톡 Webhook 상세 연동
- 관리자 trace UI 설계
- API 호출 제한과 큐잉 정책
