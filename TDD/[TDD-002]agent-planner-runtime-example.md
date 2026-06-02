# TDD-002. Agent Planner Runtime Example

- 작성일: 2026-05-20
- 문서 상태: 초안
- 범위: 사용자 질문부터 Agent Planner, domain tool 실행, agentic loop, 최종 답변 생성까지의 예시 흐름
- 관련 문서:
  - `daon-docs/TDD/[TDD-001]backend-tdd.md`
  - `daon-docs/PRD/[PRD-001]domestic-stock-llm-analysis.md`
  - `daon-docs/PRD/[PRD-002]kis-api-data-strategy.md`

## 1. 목적

이 문서는 `TDD-001`의 Agent Planner 기반 런타임 구조를 실제 사용자 질문 예시로 풀어 설명한다.

핵심 목적은 다음과 같다.

- `planner.model`의 각 모델이 언제 생성되고 어떻게 사용되는지 설명한다.
- `AgentPlan`이 tool 실행 계획이 아니라, 질문을 분석 가능한 context로 정리한 결과임을 명확히 한다.
- Spring AI tool calling과 Spring Boot의 user-controlled execution이 어떻게 결합되는지 설명한다.
- 여러 domain tool을 호출하고, agentic loop를 반복하며, 최종 답변을 생성하는 흐름을 예시로 남긴다.

## 2. 예시 질문

```text
사용자:
"삼성전자 오늘 10시쯤 갑자기 왜 빠졌어?"
```

이 질문은 다음 성격을 가진다.

- 국내주식 단일 종목 질문이다.
- 장중 특정 시점의 가격 변동 원인을 묻는다.
- 가격/거래량 흐름, 이벤트 후보, 시장/업종 맥락을 함께 확인해야 한다.

## 3. 전체 흐름 요약

```text
사용자 질문
  -> AgentPlanningRequest 생성
  -> LLM Agent Planner 호출
  -> AgentPlan 생성
      -> scope / needsClarification / analysisTargets 정리
  -> needsClarification 이면 clarificationQuestion 반환
  -> Stock Resolver / Time Market Resolver
      -> analysisTargets의 stockRefs, timeRef 확정
  -> Tool-enabled Analyst LLM 호출
      -> Spring AI domain tool 정의를 보고 필요한 tool 선택
  -> User-Controlled Tool Execution
      -> Spring Boot가 tool call 검증, loop 예산 확인, 실행
  -> ToolResult 수집
  -> Evidence Packet 생성
  -> Agentic Loop 반복
      -> 추가 tool 필요 판단
      -> 허용 tool 실행 / 미허용 tool 거절
      -> Evidence Packet 갱신
  -> Answer Composer
  -> Answer Guard
  -> 최종 응답
  -> Trace 저장
```

## 4. AgentPlanningRequest 생성

`AgentChatService`는 사용자 요청을 Planner 입력으로 변환한다.

```java
AgentPlanningRequest request = new AgentPlanningRequest(
    "삼성전자 오늘 10시쯤 갑자기 왜 빠졌어?",
    OffsetDateTime.now(ZoneId.of("Asia/Seoul")),
    "user-123",
    "session-456"
);
```

`AgentPlanningRequest`는 LLM에게 넘길 입력이다. 초기에는 원문 질문, 요청 시각, 사용자 ID, 세션 ID 정도만 포함해도 된다. 이후 conversation context가 구현되면 최근 언급 종목, 직전 시간 구간, pending clarification 상태 등을 추가할 수 있다.

## 5. AgentPlan 생성

LLM Agent Planner는 최종 답변을 생성하지 않고, 호출할 tool도 직접 확정하지 않는다. Planner는 사용자 질문을 지원 범위, 명확화 필요 여부, 분석 대상 후보로 구조화한다.

```java
AgentPlan plan = agentPlanner.plan(request);
```

예상 `AgentPlan`은 다음과 같다.

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

이 단계에서 모델은 다음 역할을 한다.

| 모델 | 역할 |
| --- | --- |
| `AgentPlan` | Planner 출력 전체. 실행 계획이 아니라 분석 context이다. |
| `ScopeAssessment` | 지원 가능 여부와 제외 사유 판단 |
| `AnalysisTarget` | 분석 대상 후보. 종목 후보와 시간 표현 후보를 함께 담는다. |

`AnalysisTarget.stockRefs`는 단일 종목과 복수 종목을 모두 `List<String>`으로 표현한다.

```json
{
  "stockRefs": [
    "삼성전자",
    "SK하이닉스"
  ],
  "timeRef": "2026-05-12",
  "reason": "두 종목이 같은 날짜에 하락한 원인을 비교하기 위한 대상입니다."
}
```

중요한 점은 `AgentPlan`이 domain tool 이름을 포함하지 않는다는 것이다. 실제 tool 선택은 Spring AI tool calling을 사용하는 Analyst 단계에서 수행한다. Planner가 tool을 미리 정하면 tool 설명과 schema를 보고 자율적으로 선택하는 tool calling의 장점과 책임이 겹친다.

## 6. Clarification 선처리

`AgentPlan.needsClarification == true`이면 tool 호출 이전에 `clarificationQuestion`을 사용자에게 반환한다.

```text
"삼성"으로 검색되는 종목이 여러 개 있습니다.
삼성전자(005930), 삼성SDI(006400), 삼성전기(009150) 중 어떤 종목인가요?
```

이 경우 KIS API 또는 domain tool을 호출하지 않는다.

현재 구조에서는 clarification 답변을 받았을 때 이전 질문과 이전 plan을 이어서 보완 planning하는 기능이 필요하다. 이는 multi-turn conversation, pending clarification, follow-up question 처리 범위이며 본 예시에서는 다루지 않는다.

## 7. Resolver 단계

LLM이 만든 값은 아직 후보이므로 Spring Boot가 검증 가능한 값으로 확정한다.

```text
"삼성전자" -> 삼성전자 / 005930 / KOSPI
"오늘 10시쯤" -> 2026-05-20 09:50~10:10 KST
```

확정 결과 예시는 다음과 같다.

```json
{
  "targets": [
    {
      "stocks": [
        {
          "code": "005930",
          "name": "삼성전자",
          "market": "KOSPI"
        }
      ],
      "timeRange": {
        "from": "2026-05-20T09:50:00+09:00",
        "to": "2026-05-20T10:10:00+09:00"
      }
    }
  ]
}
```

Resolver는 종목을 임의로 확정하지 않는다. 종목이 모호하거나 시간 표현을 조회 가능한 범위로 계산할 수 없으면 clarification으로 전환한다.

## 8. Tool-Enabled Analyst 호출

Planner가 tool 이름을 정하지 않으므로, 다음 단계에서 Analyst LLM이 Spring AI domain tool 정의를 보고 필요한 tool을 선택한다.

Analyst LLM에게 제공되는 입력은 다음과 같다.

- 원본 사용자 질문
- `AgentPlan`
- Resolver가 확정한 종목과 시간 범위
- 이미 확보한 `EvidencePacket`
- 사용 가능한 domain tool schema와 설명

예시 tool 정의는 다음과 같은 의미 단위로 제한한다.

```text
get_current_market_snapshot(stockCode)
get_intraday_price_flow(stockCode, timeRange)
get_market_and_industry_context(stockCode, timeRange)
get_event_candidates(stockCode, timeRange)
```

LLM은 KIS raw API 이름이나 TR ID를 알지 못한다. LLM은 domain tool 이름과 설명만 보고 필요한 tool call을 만든다.

예상 tool call은 다음과 같다.

```json
[
  {
    "name": "get_intraday_price_flow",
    "arguments": {
      "stockCode": "005930",
      "timeRange": "2026-05-20T09:50:00+09:00/2026-05-20T10:10:00+09:00"
    }
  },
  {
    "name": "get_current_market_snapshot",
    "arguments": {
      "stockCode": "005930"
    }
  },
  {
    "name": "get_event_candidates",
    "arguments": {
      "stockCode": "005930",
      "timeRange": "2026-05-20T09:50:00+09:00/2026-05-20T10:10:00+09:00"
    }
  }
]
```

## 9. User-Controlled Tool Execution

Spring AI의 자동 tool execution에 모든 권한을 맡기지 않는다. Daon은 금융 데이터 호출을 통제해야 하므로 user-controlled execution을 사용한다.

Spring Boot는 LLM이 만든 tool call을 실행하기 전에 다음을 확인한다.

- Domain Tool Catalog에 등록된 read-only 분석 tool인가
- 확정된 종목이 국내주식인가
- 시간 범위가 조회 가능한 장중 구간인가
- 같은 agent run에서 동일 대상/시간 범위로 이미 호출한 tool인가
- 이번 agent run의 tool 호출 예산을 넘지 않는가
- 주문, 계좌, 해외주식, 모의투자 API 요구가 섞이지 않았는가

이번 단계에서는 실제 KIS API 호출 없이 placeholder tool 결과를 반환할 수 있다. 단, placeholder 결과에는 실제 KIS 데이터가 아님을 `limitations`에 명시한다.

각 tool executor는 다음 정보를 남긴다.

- domain tool 이름
- tool argument
- 내부 KIS API 후보
- cache hit/miss
- dataAsOf
- latency
- 성공/실패/거절 상태
- 실패 또는 거절 시 제한 답변에 사용할 오류 요약

## 10. 복수 종목 fan-out

`AnalysisTarget.stockRefs`는 여러 종목을 담을 수 있지만, 대부분의 KIS API와 domain tool은 단일 종목 기준이다. 따라서 복수 종목 질문에서는 실행 계층이 fan-out을 수행한다.

```text
stockRefs = ["삼성전자", "SK하이닉스"]
requested tool = get_current_market_snapshot

실제 실행:
  -> get_current_market_snapshot("005930")
  -> get_current_market_snapshot("000660")
```

Evidence Packet Builder는 종목별 tool 결과를 다시 fan-in하여 비교 가능한 근거로 묶는다.

## 11. Evidence Packet 생성

Domain tool 실행 결과는 LLM에 그대로 전달하지 않는다. Evidence Packet Builder가 답변 생성에 필요한 근거 구조로 정리한다.

```json
{
  "question": "삼성전자 오늘 10시쯤 갑자기 왜 빠졌어?",
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
      "timeRange": {
        "from": "2026-05-20T09:50:00+09:00",
        "to": "2026-05-20T10:10:00+09:00"
      },
      "reason": "장중 특정 시점의 하락 원인과 이후 흐름을 분석하기 위한 대상입니다."
    }
  ],
  "toolEvidence": [
    {
      "tool": "GET_INTRADAY_PRICE_FLOW",
      "status": "SUCCESS",
      "summary": "09:58~10:03 구간에서 가격 하락폭이 확대되고 거래량이 증가했습니다.",
      "dataAsOf": "2026-05-20T10:12:00+09:00"
    },
    {
      "tool": "GET_CURRENT_MARKET_SNAPSHOT",
      "status": "SUCCESS",
      "summary": "현재가는 하락 구간 이후 일부 회복했지만 전일 대비 약세 상태입니다.",
      "dataAsOf": "2026-05-20T10:12:30+09:00"
    },
    {
      "tool": "GET_EVENT_CANDIDATES",
      "status": "SUCCESS",
      "summary": "10시 전후 VI 발동은 확인되지 않았고, 관련 뉴스/공시 제목 1건이 09:55에 확인됐습니다.",
      "dataAsOf": "2026-05-20T10:11:30+09:00"
    }
  ],
  "confirmedFacts": [
    "09:58~10:03 구간에서 가격 하락폭이 확대됨",
    "해당 구간 거래량이 직전 10분 평균보다 증가함",
    "현재가는 하락 구간 이후 일부 회복했지만 전일 대비 약세 상태임"
  ],
  "eventCandidates": [
    "10시 전후 VI 발동은 확인되지 않음",
    "관련 뉴스/공시 제목 1건이 09:55에 확인됨"
  ],
  "limitations": [
    "뉴스/공시 제목만 확인했으며 원문은 확인하지 않음",
    "분봉 데이터는 장중 미확정 데이터일 수 있음",
    "수급 데이터는 아직 확인하지 않음"
  ]
}
```

Evidence Packet은 확인된 사실, 이벤트 후보, 제한 사항을 분리한다. LLM Answer Composer는 이 근거 밖의 금융 사실을 새로 만들면 안 된다.

## 12. Agentic Loop 1회차

LLM Analyst는 Evidence Packet을 보고 추가 데이터가 필요한지 판단한다. 이 판단은 structured output으로 받을 수도 있고, Spring AI tool calling 응답의 tool call 유무로 판단할 수도 있다.

예상 판단은 다음과 같다.

```json
{
  "status": "NEED_MORE_DATA",
  "reason": "가격 하락과 거래량 증가는 확인됐지만 개별 수급 압력인지 시장/업종 동반 약세인지 구분할 근거가 부족합니다.",
  "answerReadiness": {
    "canAnswerWithCurrentEvidence": true,
    "riskIfAnswerNow": "시장/업종 동조 여부와 수급 근거가 없어 원인 설명이 제한적입니다."
  }
}
```

이후 Analyst LLM은 추가 tool call을 요청할 수 있다.

```json
[
  {
    "name": "get_market_and_industry_context",
    "arguments": {
      "stockCode": "005930",
      "timeRange": "2026-05-20T09:50:00+09:00/2026-05-20T10:10:00+09:00"
    }
  },
  {
    "name": "get_supply_demand_context",
    "arguments": {
      "stockCode": "005930",
      "timeRange": "2026-05-20T09:50:00+09:00/2026-05-20T10:10:00+09:00"
    }
  }
]
```

Spring Boot는 이 tool call을 다시 검증한다.

```text
실행:
- get_market_and_industry_context

거절:
- get_supply_demand_context
  reason = "MVP 1차에서 실구현되지 않은 P1 tool"
```

거절된 tool은 실행하지 않지만, 거절 사유는 trace와 evidence 제한 사항에 남긴다.

## 13. 추가 Domain Tool 실행과 Evidence 갱신

허용된 tool만 실행한다.

```text
get_market_and_industry_context(005930, 09:50~10:10)
  -> FHPUP02100000 국내업종 현재지수
  -> FHKUP03500200 업종 분봉조회
```

갱신된 Evidence Packet에는 시장/업종 맥락과 policy note가 추가된다.

```json
{
  "marketContext": [
    "09:58~10:03 구간에 반도체 업종 지수도 약세였음",
    "다만 삼성전자의 하락폭이 업종 평균보다 더 컸음"
  ],
  "policyNotes": [
    "GET_SUPPLY_DEMAND_CONTEXT는 MVP 1차 미구현으로 실행하지 않음"
  ],
  "limitations": [
    "외국인/기관/프로그램 수급은 아직 확인하지 못함",
    "뉴스 제목은 원인 후보일 뿐 확정 근거가 아님"
  ]
}
```

## 14. Agentic Loop 2회차

LLM Analyst는 갱신된 evidence를 다시 검토한다.

```json
{
  "status": "SUFFICIENT_EVIDENCE",
  "reason": "가격/거래량 변화, 업종 동조 여부, VI/뉴스 후보를 확인했으므로 제한 사항을 명시하면 답변 가능함.",
  "answerReadiness": {
    "canAnswerWithCurrentEvidence": true,
    "riskIfAnswerNow": "수급 데이터가 없어 매도 주체를 특정할 수 없음."
  }
}
```

이 시점에서 loop를 종료한다. 최대 loop 횟수에 도달한 경우에도 현재 확보한 evidence와 제한 사항을 기준으로 답변한다.

## 15. 최종 답변 생성

Answer Composer는 Evidence Packet 안에서만 답변한다.

```text
기준 시각: 2026-05-20 10:12

삼성전자는 오늘 09:58~10:03 전후로 하락폭이 커졌고, 이 구간에서 거래량도 직전 10분 평균보다 늘었습니다. 단순한 얇은 거래의 흔들림이라기보다는 거래가 동반된 하락으로 보는 편이 맞습니다.

확인되는 근거는 세 가지입니다.

1. 10시 전후 분봉에서 가격 하락과 거래량 증가가 같이 나타났습니다.
2. 같은 시간 반도체 업종도 약세였지만, 삼성전자의 하락폭이 업종 평균보다 더 컸습니다.
3. VI 발동은 확인되지 않았고, 09:55 전후 관련 뉴스/공시 제목 1건이 확인됐습니다.

다만 뉴스 원문과 외국인/기관/프로그램 수급은 아직 확인하지 못했습니다. 그래서 현재 근거만 보면 "업종 약세가 깔린 상태에서 삼성전자에 개별 매도 압력이 더해진 흐름" 정도로 보는 것이 안전합니다. 특정 뉴스 하나를 하락 원인으로 단정하기는 어렵습니다.
```

Answer Guard는 다음 항목을 확인한다.

- 기준 시각 포함 여부
- 확인된 사실과 추정의 구분
- 매수/매도 권유 표현 여부
- evidence에 없는 원인 단정 여부
- 카카오톡 응답 길이 제한

## 16. Trace 저장

최종적으로 Agent run trace를 저장한다.

```json
{
  "traceId": "agent-run-123",
  "planner": {
    "scope": {
      "status": "SUPPORTED",
      "reason": null
    },
    "analysisTargets": [
      {
        "stockRefs": [
          "삼성전자"
        ],
        "timeRef": "오늘 10시쯤",
        "reason": "장중 특정 시점의 하락 원인과 이후 흐름을 분석하기 위한 대상입니다."
      }
    ]
  },
  "resolved": {
    "targets": [
      {
        "stockCode": "005930",
        "timeRange": "2026-05-20T09:50:00+09:00/2026-05-20T10:10:00+09:00"
      }
    ]
  },
  "toolCalls": [
    {
      "tool": "GET_INTRADAY_PRICE_FLOW",
      "status": "SUCCESS",
      "internalKisApis": [
        "FHKST03010200"
      ]
    },
    {
      "tool": "GET_CURRENT_MARKET_SNAPSHOT",
      "status": "SUCCESS",
      "internalKisApis": [
        "FHKST01010100",
        "FHKST01010200"
      ]
    },
    {
      "tool": "GET_EVENT_CANDIDATES",
      "status": "SUCCESS",
      "internalKisApis": [
        "FHPST01390000",
        "FHKST01011800"
      ]
    },
    {
      "tool": "GET_MARKET_AND_INDUSTRY_CONTEXT",
      "status": "SUCCESS",
      "internalKisApis": [
        "FHPUP02100000",
        "FHKUP03500200"
      ]
    },
    {
      "tool": "GET_SUPPLY_DEMAND_CONTEXT",
      "status": "REJECTED",
      "reason": "MVP 1차 미구현 P1 tool"
    }
  ],
  "loopCount": 2,
  "finishReason": "SUFFICIENT_EVIDENCE"
}
```

Trace는 관리자 화면과 품질 개선에 사용한다. 같은 질문에 대해 어떤 tool이 실행됐고, 어떤 tool이 정책상 거절됐는지 확인할 수 있어야 한다.

## 17. 모델 사용 관계 정리

| 모델 | 생성 시점 | 소비하는 컴포넌트 | 역할 |
| --- | --- | --- | --- |
| `AgentPlanningRequest` | Agent API 요청 수신 후 | `AgentPlanner` | Planner 입력 |
| `AgentPlan` | LLM Planner 응답 후 | Clarification Handler, Resolver, Analyst, Evidence Builder, Trace Logger | 질문 분석 context |
| `ScopeAssessment` | Planner 응답 내부 | Clarification Handler, Resolver, Trace Logger | 지원 가능 여부와 제외 사유 판단 |
| `AnalysisTarget` | Planner 응답 내부 | Resolver, Analyst, Evidence Builder, Trace Logger | 종목 후보와 시간 후보를 묶은 분석 대상 |
| `ToolCall` | Analyst LLM 응답 내부 | Tool Execution Controller | Spring AI tool calling이 만든 실행 요청 |
| `ToolResult` | Domain tool 실행 후 | Evidence Builder, Trace Logger | tool별 실행 결과 |
| `EvidencePacket` | ToolResult 수집 후 | Analyst, Answer Composer, Trace Logger | 답변 생성 근거 |

## 18. 구현상 중요한 경계

- Planner는 질문을 분석 가능한 context로 정리한다.
- Planner는 domain tool 이름을 직접 선택하지 않는다.
- Analyst LLM은 Spring AI tool schema와 설명을 보고 필요한 tool을 선택한다.
- LLM은 종목코드, 시장, 시간 범위를 최종 확정하지 않는다.
- LLM은 KIS raw API를 직접 호출하지 않는다.
- Spring Boot는 Resolver, Tool Execution Controller, Domain Tool Executor를 통해 실행 권한을 통제한다.
- user-controlled tool execution으로 loop 제한, 중복 호출 방지, 예산, trace를 관리한다.
- Evidence Packet은 LLM 답변의 유일한 금융 근거가 되어야 한다.
