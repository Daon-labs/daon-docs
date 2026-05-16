# TDD-001. Agent Chat Runtime Flow 기술 설계

- 작성일: 2026-05-15
- 문서 상태: 초안
- 범위: 카카오톡 사용자 질문 입력부터 Daon Agent 답변 제공까지의 백엔드 런타임 흐름
- 관련 문서:
  - `daon-docs/PRD/[PRD-001]domestic-stock-llm-analysis.md`
  - `daon-docs/PRD/[PRD-002]kis-api-data-strategy.md`
  - `daon-docs/ADR/[ADR-001]backend-stack-python-vs-spring-boot.md`

## 1. 목적

이 문서는 Daon의 대화형 국내주식 분석 Agent가 카카오톡 메시지를 수신한 뒤, 사용자 의도를 구조화하고, KIS Open API 기반 데이터 tool을 호출하며, 근거 기반 답변을 생성해 다시 카카오톡으로 반환하는 전체 런타임 흐름을 정의한다.

Daon의 핵심 방향은 단순히 LLM이 자유롭게 금융 API를 호출하는 챗봇이 아니다. Spring Boot 백엔드가 금융 데이터 수집, 캐시, 호출 제한, workflow, trace, 안전장치를 소유하고, LLM은 자연어 이해, 근거 부족 판단, 추가 데이터 요청 제안, 근거 기반 답변 생성을 담당한다.

## 2. 설계 원칙

- 카카오톡은 입출력 채널이며, 분석 로직은 내부 Agent API가 담당한다.
- LLM은 의도분류를 수행할 수 있지만, workflow 실행 권한은 Spring Boot가 가진다.
- LLM은 분석 중 추가 데이터가 필요하다고 판단할 수 있지만, 실제 tool 실행 권한은 Spring Boot가 가진다.
- rule 기반 의도분류는 MVP 기본 전략에서 제외한다.
- 의도분류 결과는 structured output으로 받고, Spring Boot가 schema validation과 업무 규칙 검증을 수행한다.
- 기본 필수 데이터는 Spring Boot가 먼저 수집하고, 이후 bounded agentic loop에서 LLM이 추가 tool request를 제안한다.
- Agentic loop는 허용 tool, 최대 반복 횟수, 호출 예산, 중복 호출 방지 정책 안에서만 동작한다.
- KIS API 응답 원문을 LLM에 그대로 전달하지 않고, Spring Boot가 분석용 evidence packet으로 정리한 뒤 전달한다.
- 답변은 확인된 사실, 추정, 불확실성을 분리한다.
- 모든 답변에는 데이터 기준 시각과 데이터 제한 사항을 포함한다.
- 모든 Agent 실행은 추적 가능해야 한다. 어떤 tool을 호출했고, cache hit 여부와 데이터 기준 시각이 무엇인지 저장한다.

## 3. 전체 요청 흐름

```text
Kakao 사용자 메시지
  -> Kakao Webhook Adapter
  -> Internal Agent API
  -> Conversation Session 로딩
  -> LLM Intent Classifier
  -> Intent 결과 검증
  -> Stock Resolver
  -> Baseline Workflow 선택
  -> Baseline KIS Data Tool 호출
  -> Baseline Evidence Packet 생성
  -> Agentic Analysis Loop
      -> LLM Analyst가 evidence 충분성 판단
      -> 추가 Tool Request Plan 생성
      -> Spring Boot Policy Validation
      -> 허용된 KIS Data Tool 실행
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
| Intent Classifier | LLM structured output 기반 의도분류 | rule 기반 분류는 MVP 기본 전략에서 제외한다. |
| Intent Validator | intent, 종목 수, 시간 표현, confidence, 명확화 필요 여부 검증 | Spring Boot가 소유한다. |
| Stock Resolver | 종목명/코드/별칭을 내부 종목 코드로 해석 | LLM에 종목코드 확정을 맡기지 않는다. |
| Baseline Analysis Workflow | intent별 최소 필수 데이터 묶음과 최초 호출 순서 결정 | Spring Boot가 명시적으로 실행한다. |
| Agentic Analysis Loop | LLM이 evidence 충분성을 판단하고 추가 tool request를 제안하는 반복 루프 | Spring Boot의 policy validation을 통과한 tool만 실행한다. |
| Tool Request Policy | intent별 허용 tool, 최대 반복 횟수, 호출 예산, 중복 호출 여부 검증 | 비용, 지연, KIS 호출 제한을 통제한다. |
| KIS Adapter/Tools | KIS REST/WebSocket API 호출, 토큰, 재시도, rate limit, TTL 처리 | 실전 API 기준. 모의투자 API나 모의 데이터 대체 없음. |
| Evidence Packet Builder | KIS 응답을 LLM 입력용 근거 패킷으로 변환 | 원문 응답을 직접 전달하지 않는다. |
| Answer Composer | evidence 기반 한국어 답변 생성 | Spring AI ChatClient 사용 후보. |
| Answer Guard | 금지 표현, 과도한 단정, 기준 시각 누락, 근거 외 주장 검사 | 필요 시 재생성 또는 보정한다. |
| Channel Response Adapter | 내부 Agent 응답을 카카오톡 응답 형식으로 변환 | 채널별 표현 제약만 처리한다. |
| Trace Logger | agent run, tool call, evidence, LLM latency/cost 저장 | 관리자 화면과 운영 분석에 사용한다. |

## 5. Kakao Webhook Adapter

카카오톡 Webhook 계층은 메시지 채널 어댑터다. 이 계층은 사용자의 자연어 질문을 분석하지 않고, 카카오톡 이벤트를 내부 Agent 요청으로 변환하는 일만 수행한다.

주요 책임은 다음과 같다.

- 카카오 요청 서명 또는 인증 정보 검증
- 카카오 사용자 식별자 추출
- 메시지 ID와 텍스트 추출
- 중복 메시지 처리 방지
- 내부 사용자와 카카오 사용자 매핑
- 내부 Agent API 요청 생성
- Agent 응답을 카카오톡 응답 형식으로 변환

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
  "intent": "CURRENT_MOVE_REASON",
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
- 최근 분석 intent
- 최근 시간 구간
- 최근 비교 대상 종목
- 직전 Agent run trace

후속 질문 처리 예시는 다음과 같다.

| 사용자 질문 | 필요한 맥락 | 처리 |
| --- | --- | --- |
| "그럼 SK하이닉스는?" | 직전 intent | 같은 intent를 SK하이닉스에 적용 |
| "아까 그 구간 다시 설명해줘" | 직전 시간 구간 | 같은 종목과 시간 구간으로 복기 workflow 실행 |
| "다른 종목이랑 비교해줘" | 직전 종목 | 비교 대상이 없으면 명확화 질문 |

## 8. LLM Intent Classification

MVP에서는 rule 기반 의도분류를 기본 전략으로 사용하지 않는다. 사용자의 자연어 표현 다양성을 처리하기 위해 LLM structured output 기반 의도분류를 사용한다.

의도분류 LLM의 역할은 답변 생성이 아니다. 의도분류 모델은 사용자 메시지를 정해진 schema로 구조화한다.

초기 intent 후보는 다음과 같다.

| Intent | 설명 |
| --- | --- |
| `CURRENT_MOVE_REASON` | "지금 왜 올라/떨어져?" 같은 현재 급등락 원인 질문 |
| `PAST_MOVE_REASON` | "4시간 전에 왜 빠졌어?" 같은 특정 시점 변동 복기 |
| `INTRADAY_SUMMARY` | "오늘 장중 특이사항 정리해줘" 같은 당일 흐름 요약 |
| `LONG_TERM_TREND` | "요즘 추세 어때?" 같은 장기 추세 요약 |
| `MULTI_STOCK_COMPARE` | "A, B, C 중 뭐가 나아?" 같은 복수 종목 비교 |
| `CLARIFICATION_REQUIRED` | 종목, 시간, 비교 대상이 불명확해 추가 질문이 필요한 경우 |
| `UNSUPPORTED` | MVP 범위를 벗어난 질문 |

Structured output 예시는 다음과 같다.

```json
{
  "intent": "CURRENT_MOVE_REASON",
  "stockMentions": [
    {
      "rawText": "삼성전자",
      "kind": "NAME"
    }
  ],
  "timeExpression": {
    "type": "NOW",
    "rawText": "지금"
  },
  "comparisonCriteria": [],
  "needsClarification": false,
  "clarificationQuestion": null,
  "confidence": 0.93
}
```

Java record 후보는 다음과 같다.

```java
public record IntentClassificationResult(
    IntentType intent,
    List<StockMention> stockMentions,
    TimeExpression timeExpression,
    List<String> comparisonCriteria,
    boolean needsClarification,
    String clarificationQuestion,
    double confidence
) {}
```

## 9. Intent 결과 검증

LLM이 의도분류를 수행하더라도, Spring Boot는 결과를 그대로 신뢰하지 않는다. Intent Validator는 다음 검증을 수행한다.

- 지원하는 intent인지 확인한다.
- 종목이 필요한 intent에서 종목 언급이 없으면 명확화 질문으로 전환한다.
- 복수 종목 비교 intent에서 종목이 2개 미만이면 명확화 질문으로 전환한다.
- 시간 표현이 필요한 intent에서 시간 구간을 계산할 수 없으면 명확화 질문으로 전환한다.
- confidence가 기준값보다 낮으면 명확화 질문으로 전환한다.
- 후속 질문이면 conversation context로 누락 정보를 보완한다.
- MVP 제외 범위인 주문, 잔고, 매수/매도 지시, 해외주식 질문은 unsupported로 처리한다.

검증 실패 시 KIS API를 호출하지 않고 사용자에게 짧은 확인 질문을 반환한다.

## 10. Stock Resolver

Stock Resolver는 사용자 입력의 종목명, 종목코드, 별칭을 내부 표준 종목으로 해석한다.

이 책임은 LLM에 맡기지 않는다. LLM은 "삼성전자"라는 언급을 추출할 수 있지만, 최종 종목코드 확정은 종목 마스터 DB 또는 로컬 종목 코드 데이터가 담당한다.

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

## 11. Baseline Workflow 선택

Spring Boot는 검증된 intent를 기준으로 baseline workflow를 선택한다. Baseline workflow는 해당 intent를 분석하기 위해 항상 필요한 최소 데이터 묶음을 먼저 수집한다.

```text
CURRENT_MOVE_REASON -> CurrentMoveAnalysisWorkflow
PAST_MOVE_REASON -> PastMoveAnalysisWorkflow
INTRADAY_SUMMARY -> IntradaySummaryWorkflow
LONG_TERM_TREND -> LongTermTrendWorkflow
MULTI_STOCK_COMPARE -> MultiStockCompareWorkflow
```

이 단계의 핵심은 시작점을 안정적으로 만드는 것이다. LLM에게 빈 컨텍스트에서 "필요한 tool을 알아서 호출하라"고 맡기지 않고, Spring Boot가 PRD-002 기준의 기본 근거 데이터를 먼저 준비한다.

이후 추가 데이터가 필요한지는 Agentic Analysis Loop에서 LLM Analyst가 판단한다. 즉 workflow는 완전히 고정된 최종 경로가 아니라, 안전한 기본 근거를 만드는 출발점이다.

## 12. Baseline KIS Data Tool 호출

KIS Data Tool은 내부 application service로 구현한다. Spring AI tool calling은 일부 도구를 LLM에 노출할 수 있지만, tool 실행 자체는 Spring Boot의 policy validation과 service orchestration을 거친다.

`CURRENT_MOVE_REASON` workflow의 초기 필수 데이터는 PRD-002 기준으로 다음과 같다.

| 데이터 | KIS API | TR ID | 사용 목적 |
| --- | --- | --- | --- |
| 현재가 시세 | 주식현재가 시세 | `FHKST01010100` | 현재가, 등락률, 거래량, 거래대금, 상태 확인 |
| 호가 스냅샷 | 주식현재가 호가/예상체결 | `FHKST01010200` | 매도/매수 잔량, 호가 불균형 확인 |
| 당일 분봉 | 주식당일분봉조회 | `FHKST03010200` | 장중 변곡점, 거래량 급증 구간 확인 |
| 업종 현재지수 | 국내업종 현재지수 | `FHPUP02100000` | 시장/업종 동조 여부 확인 |
| 업종 분봉 | 업종 분봉조회 | `FHKUP03500200` | 특정 시점의 업종 흐름 비교 |
| VI 현황 | 변동성완화장치 현황 | `FHPST01390000` | 급변 이벤트 후보 확인 |
| 뉴스/공시 제목 | 종합 시황/공시 제목 | `FHKST01011800` | 원인 후보 확인 |

도구 호출 전에는 Redis 또는 DB 캐시를 먼저 확인한다.

- 현재가/호가 스냅샷 TTL
- 1분봉 최신성
- 뉴스/공시 제목 TTL
- 업종 지수 TTL
- KIS API 호출 제한과 큐 대기 상태

도구 호출 trace 예시는 다음과 같다.

```json
{
  "toolName": "getCurrentPrice",
  "stockCode": "005930",
  "source": "KIS",
  "cacheHit": false,
  "dataAsOf": "2026-05-15T10:12:28+09:00",
  "latencyMs": 184,
  "status": "SUCCESS"
}
```

## 13. Baseline Evidence Packet 생성

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
  "intent": "CURRENT_MOVE_REASON",
  "stock": {
    "code": "005930",
    "name": "삼성전자"
  },
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

## 14. Agentic Analysis Loop

Agentic Analysis Loop는 Daon이 더 agentic하게 동작하는 핵심 구간이다. LLM은 현재 evidence packet을 검토한 뒤, 근거가 충분한지 또는 추가 데이터가 필요한지 structured output으로 판단한다.

루프의 기본 흐름은 다음과 같다.

```text
Baseline Evidence Packet
  -> LLM Analyst Review
  -> SUFFICIENT_EVIDENCE 이면 최종 답변 생성으로 이동
  -> NEED_MORE_DATA 이면 Tool Request Plan 생성
  -> Spring Boot Tool Request Policy 검증
  -> 허용된 tool만 실행
  -> Evidence Packet 갱신
  -> LLM Analyst Review 반복
```

LLM Analyst의 structured output 후보는 다음과 같다.

```json
{
  "status": "NEED_MORE_DATA",
  "reason": "하락 구간의 거래량 증가는 확인됐지만 개별 수급 이슈인지 시장 동반 약세인지 근거가 부족합니다.",
  "toolRequests": [
    {
      "tool": "GET_INVESTOR_TREND",
      "stockCode": "005930",
      "reason": "외국인/기관 수급 방향 확인 필요",
      "priority": "HIGH"
    },
    {
      "tool": "GET_PROGRAM_TRADE",
      "stockCode": "005930",
      "reason": "프로그램 매도 압력 여부 확인 필요",
      "priority": "MEDIUM"
    }
  ],
  "answerReadiness": {
    "canAnswerWithCurrentEvidence": true,
    "riskIfAnswerNow": "수급 원인 가능성을 확인하지 못해 하락 원인 설명이 업종/거래량 중심으로 제한됩니다."
  }
}
```

충분한 근거가 있다고 판단한 경우는 다음과 같다.

```json
{
  "status": "SUFFICIENT_EVIDENCE",
  "reason": "현재가, 분봉, 업종 흐름, VI/뉴스 제목으로 현재 질문에 답변 가능한 수준의 근거가 확보됐습니다.",
  "toolRequests": [],
  "answerReadiness": {
    "canAnswerWithCurrentEvidence": true,
    "riskIfAnswerNow": "뉴스 원문과 확정 수급 데이터는 없으므로 단일 원인 단정은 피해야 합니다."
  }
}
```

### 14.1 Tool Request Policy

LLM이 제안한 tool request는 바로 실행하지 않는다. Spring Boot의 Tool Request Policy가 다음 항목을 검증한다.

- 현재 intent에서 허용된 tool인가
- 해당 tool이 이미 같은 agent run에서 호출됐는가
- tool 호출 예산을 초과하지 않는가
- 최대 loop 횟수를 초과하지 않는가
- KIS API 호출 제한 또는 큐 대기 상태가 허용 범위인가
- 캐시된 데이터로 충분히 대체 가능한가
- 요청한 종목과 시간 구간이 검증된 context 안에 있는가
- MVP 제외 범위인 주문, 계좌, 잔고, 해외주식, 모의투자 API를 요구하지 않는가

`CURRENT_MOVE_REASON`의 MVP 허용 추가 tool 후보는 다음과 같다.

| Tool | 사용 조건 | 비고 |
| --- | --- | --- |
| `GET_TIME_CONCLUSION` | 분봉만으로 특정 급변 구간의 체결 강도 설명이 부족한 경우 | KIS `주식현재가 당일시간대별체결` |
| `GET_INVESTOR_TREND` | 수급 방향 확인이 필요한 경우 | P1 데이터. MVP에서는 adapter stub 또는 후속 구현 후보로 둘 수 있다. |
| `GET_PROGRAM_TRADE` | 프로그램 매매 압력 확인이 필요한 경우 | P1 데이터. 호출 제한과 데이터 신선도 정책 필요 |
| `GET_ADDITIONAL_NEWS_TITLES` | 이벤트 후보가 불충분하거나 급변 시점 전후 제목 확인이 필요한 경우 | 뉴스/공시 제목 범위 내에서만 사용 |
| `GET_MARKET_INDEX_CONTEXT` | 업종만으로 시장 동조 여부가 부족한 경우 | 코스피/코스닥 지수 맥락 |

MVP 1차에서는 추가 tool request를 최대 2회 loop, 최대 2개 tool per loop로 제한한다.

### 14.2 Loop 종료 조건

Agentic loop는 다음 조건 중 하나를 만족하면 종료한다.

- LLM Analyst가 `SUFFICIENT_EVIDENCE`를 반환한다.
- 최대 loop 횟수에 도달한다.
- 허용된 tool request가 없다.
- tool 호출 예산을 초과한다.
- KIS API 장애 또는 rate limit으로 추가 수집이 불가능하다.
- 추가 tool 결과가 이미 확보한 evidence와 중복된다.

종료 시점에 evidence가 부족하더라도, 현재 확보한 근거와 제한 사항을 명시해 제한 답변을 생성한다.

### 14.3 Trace

Agentic loop는 반복 단계별로 trace를 남긴다.

- loop index
- LLM Analyst status
- 추가 데이터 요청 이유
- 제안된 tool request
- policy validation 결과
- 실행된 tool
- 거절된 tool과 거절 사유
- evidence packet 변경 요약
- loop 종료 사유

이 trace는 포트폴리오와 운영 관점에서 중요하다. 사용자의 질문 하나에 대해 Daon이 왜 추가 데이터를 요청했고, 어떤 정책 때문에 실행하거나 거절했는지 설명할 수 있어야 한다.

## 15. LLM Answer Composer

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

## 16. 답변 후처리와 안전장치

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

## 17. Kakao 응답 변환

Channel Response Adapter는 내부 Agent 응답을 카카오톡 응답 형식으로 변환한다.

초기 MVP에서는 텍스트 응답을 우선한다. 향후 필요하면 빠른 후속 질문 버튼, 자세히 보기 링크, 관리자 trace 링크를 확장할 수 있다.

카카오톡 응답 변환 정책은 다음과 같다.

- 핵심 결론을 먼저 보여준다.
- 너무 긴 근거는 3개 내외로 줄인다.
- 데이터 기준 시각과 제한 사항은 유지한다.
- 내부 trace ID는 사용자에게 기본 노출하지 않는다.
- 장애 메시지는 사용자가 이해할 수 있는 표현으로 변환한다.

## 18. Trace와 Observability

Daon의 포트폴리오 강점은 "AI 답변" 자체보다 답변 생성 과정을 추적할 수 있다는 점이다. 다음 데이터를 저장한다.

| 데이터 | 저장 목적 |
| --- | --- |
| 원본 사용자 질문 | 재현과 품질 평가 |
| intent classification 결과 | 오분류 분석 |
| conversation context 사용 여부 | 후속 질문 품질 분석 |
| 선택된 workflow | 분석 경로 추적 |
| tool call 목록 | 어떤 데이터로 답했는지 확인 |
| agentic loop decision | 추가 데이터 요청 이유와 종료 사유 확인 |
| tool request policy 결과 | 실행/거절된 tool과 정책 근거 확인 |
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

- 어떤 의도로 분류됐는가
- 어떤 종목으로 해석됐는가
- 어떤 KIS API를 호출했는가
- 캐시를 사용했는가
- 데이터 기준 시각은 언제인가
- LLM이 어떤 evidence로 답변했는가
- 어디에서 지연 또는 오류가 발생했는가

## 19. Error/제한 응답 정책

장애 또는 불확실성은 숨기지 않고 사용자에게 제한 사항을 알려야 한다.

| 상황 | 처리 |
| --- | --- |
| 종목을 해석할 수 없음 | 종목명 또는 종목코드 입력 요청 |
| 종목명이 모호함 | 후보 종목 제시 후 선택 요청 |
| intent confidence 낮음 | 명확화 질문 반환 |
| KIS 현재가 API 실패 | 캐시가 있으면 기준 시각과 함께 제한 답변, 없으면 일시 실패 안내 |
| 분봉 데이터 누락 | 현재가/호가/업종 기준 제한 답변 |
| 뉴스/공시 조회 실패 | 뉴스/공시 확인 불가를 명시하고 가격/거래량 중심 답변 |
| LLM 의도분류 실패 | 짧은 재시도 후 실패 시 질문을 다시 요청 |
| LLM Analyst가 허용되지 않은 tool 요청 | tool을 실행하지 않고 현재 evidence 기준 제한 답변 |
| agentic loop 최대 횟수 초과 | 확보된 evidence와 제한 사항을 명시해 답변 |
| LLM 답변 생성 실패 | evidence 기반 템플릿 제한 답변 |
| 카카오톡 응답 제한 초과 | 요약 답변 우선 반환 |

## 20. MVP 구현 범위

1차 MVP는 모든 질문 유형을 동시에 완성하지 않는다. 첫 구현 대상은 `CURRENT_MOVE_REASON`으로 좁힌다.

포함 범위는 다음과 같다.

- Kakao Webhook Adapter 기본 구조
- Internal Agent API
- Conversation Session 최소 저장
- LLM structured intent classification
- Intent validation
- Stock Resolver
- `CURRENT_MOVE_REASON` baseline workflow
- 현재가, 호가, 당일 분봉, 업종, VI/뉴스 제목 baseline tool
- Evidence Packet Builder
- Agentic Analysis Loop
- Tool Request Policy
- 최대 2회 loop, loop당 최대 2개 추가 tool request 제한
- LLM Answer Composer
- Answer Guard 기본 검사
- Agent run/tool call trace 저장

후속 범위는 다음과 같다.

- `PAST_MOVE_REASON`
- `INTRADAY_SUMMARY`
- `LONG_TERM_TREND`
- `MULTI_STOCK_COMPARE`
- 관리자 trace 화면
- 관심 종목/tracing 관리
- Spring AI tool calling 확대
- Web UI 또는 내부 테스트 클라이언트

## 21. Spring AI 사용 범위

MVP에서 Spring AI는 다음 영역에 사용한다.

- ChatClient 기반 LLM 호출
- structured output 기반 intent classification
- structured output 기반 analyst review와 tool request plan 생성
- evidence packet 기반 answer composition
- 필요 시 chat memory/advisor 적용
- LLM 호출 관측성 연동

Spring AI가 소유하지 않는 영역은 다음과 같다.

- KIS API adapter
- KIS 토큰 갱신
- rate limit과 retry 정책
- Redis/DB 캐시 TTL
- intent별 baseline workflow 선택
- tool request policy validation
- 데이터 신선도 판단
- evidence packet 생성 규칙
- 금융 답변 안전 정책
- 카카오톡 Webhook 처리
- trace 저장 스키마

## 22. LangGraph 검토 위치

LangGraph는 MVP 핵심 런타임으로 사용하지 않는다. 현재 Daon MVP는 장기간 상태를 저장하며 재개되는 복잡한 multi-agent graph보다, Spring Boot가 통제하는 bounded agentic loop가 더 중요하다.

LangGraph 또는 별도 graph runtime은 다음 조건에서 재검토한다.

- 장중 이상 종목 자동 감시 후 multi-step 조사 workflow가 필요해진 경우
- 사람이 중간 승인하는 리포트 생성 workflow가 필요해진 경우
- 여러 Agent가 역할을 나누는 장기 실행 작업이 필요해진 경우
- LLM/분석 worker를 Spring Boot 운영 API와 분리해야 하는 경우

## 23. 테스트 전략

초기 테스트는 다음 계층으로 나눈다.

| 테스트 | 검증 대상 |
| --- | --- |
| Intent Classifier schema test | LLM 결과가 Java schema로 안정적으로 변환되는지 |
| Intent Validator unit test | 종목 누락, 낮은 confidence, unsupported intent 처리 |
| Stock Resolver unit test | 종목명/코드/모호한 입력 해석 |
| Baseline Workflow unit test | intent별 필수 tool 호출 계획 |
| Agentic Loop test | 추가 tool request 반복과 종료 조건 |
| Tool Request Policy test | 허용/거절 tool, 중복 호출, 예산 초과 처리 |
| KIS Adapter contract test | KIS 응답을 내부 DTO로 변환 |
| Evidence Packet test | 확인 사실/추정/제한 사항 분리 |
| Answer Guard test | 금지 표현, 기준 시각 누락, 과도한 단정 탐지 |
| Agent API integration test | 질문 입력부터 내부 Agent 응답까지 |
| Kakao Adapter test | 카카오 요청/응답 변환 |

외부 KIS 실전 API와 LLM 호출은 테스트 환경에서 fixture 또는 stub을 우선 사용한다. 실전 API 검증은 별도 수동/운영 검증 시나리오로 분리한다.

## 24. 미결정 사항

- LLM provider와 모델
- Spring AI version
- intent confidence 기준값
- agentic loop 최대 횟수와 tool 호출 예산
- intent별 허용 추가 tool 목록
- 카카오톡 응답 시간 제한 대응 방식
- Redis TTL 상세값
- PostgreSQL trace 스키마 상세
- KIS API 호출 제한 기반 큐 정책
- Agent API 인증 방식
- 관리자 trace 화면 도입 시점

## 25. 후속 문서화 범위

다음 항목은 별도 문서에서 다룬다.

- KIS Adapter 상세 설계
- Redis/DB 캐시 TTL과 스키마
- Agent prompt와 structured output schema
- Agentic loop와 tool request policy 상세 설계
- PostgreSQL trace schema
- 카카오톡 Webhook 상세 연동
- 관리자 trace UI 설계
- API 호출 제한과 큐잉 정책
