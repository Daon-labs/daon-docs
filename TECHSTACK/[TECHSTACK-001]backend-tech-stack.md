# 백엔드 기술 스택

- 작성일: 2026-05-11
- 문서 상태: 초안
- 범위: 국내주식 LLM 분석 MVP 백엔드
- 관련 PRD: `daon-docs/PRD/[PRD-001]domestic-stock-llm-analysis.md`
- 관련 ADR: `daon-docs/ADR/[ADR-001]backend-stack-python-vs-spring-boot.md`

## 1. 백엔드 목표

Daon 백엔드는 국내주식 종목에 대한 자연어 질문을 받아 KIS Open API의 실전 REST/WebSocket 데이터, 내부 캐시/DB 데이터, LLM 분석 결과를 조합해 한국어 답변을 제공한다.

MVP에서 백엔드는 다음 성격을 우선한다.

- 장중 실시간 데이터와 REST 조회 데이터를 안정적으로 수집한다.
- KIS API 호출 제한과 인기 종목 쏠림을 캐시, 큐, rate limit으로 제어한다.
- 웹 UI 또는 내부 클라이언트와 카카오톡 1:1 Agent가 같은 분석 API를 사용하게 한다.
- 답변 기준 시각, 사용 데이터, 추정/확정 구분, 오류 상황을 추적 가능하게 남긴다.
- 모의투자 API 또는 모의 fallback 없이 실전 API 기준으로 설계한다.

## 2. 결정된 주 스택

| 영역 | 선택 |
| --- | --- |
| 주 언어 | Java 21 |
| 애플리케이션 프레임워크 | Spring Boot 3.x |
| API | Spring Web MVC 또는 WebFlux는 구현 시점 부하 모델에 맞춰 선택하되, 외부 공개 API는 REST 중심 |
| KIS REST 연동 | Spring HTTP client 계층으로 캡슐화 |
| KIS WebSocket 연동 | Spring 기반 WebSocket client/worker 프로세스 |
| DB | PostgreSQL |
| 캐시/분산 락/rate limit 보조 | Redis |
| 비동기 작업 | 초기: Spring scheduling + 내부 작업 큐, 확장 시 Redis Streams 또는 Kafka 검토 |
| LLM 연동 | 외부 LLM API adapter 계층으로 분리 |
| 인증/비밀값 | 환경 변수 또는 secret manager 기반 구성 |
| 관측성 | structured logging, metrics, tracing |
| 배포 단위 | Spring Boot 단일 백엔드 애플리케이션에서 시작하고, 수집 worker는 필요 시 분리 |

## 3. KIS Open API 기준

프로젝트 지침에 따라 KIS Open API 판단은 `데이터/한국투자증권_오픈API_전체문서_20260504_030007.xlsx`를 우선 참고한다.

실전 도메인은 REST 기준 `https://openapi.koreainvestment.com:9443`, WebSocket 기준 `ws://ops.koreainvestment.com:21000`을 사용한다.

## 4. 주요 백엔드 모듈

| 모듈 | 책임 |
| --- | --- |
| API server | 사용자 질문, 종목 조회, 분석 요청, 관리자 기능을 제공한다. |
| Kakao webhook adapter | 카카오톡 1:1 Agent 이벤트를 내부 분석 요청으로 변환한다. |
| Conversation context | 사용자별 최근 대화 맥락과 지시어 참조를 관리한다. |
| Stock symbol resolver | 종목명, 종목코드, 별칭을 국내주식 종목으로 해석한다. |
| KIS REST adapter | 현재가, 분봉, 기간별 시세, 투자자, 재무 데이터를 조회한다. |
| KIS realtime collector | 관심 종목의 실시간 체결/호가/장운영 이벤트를 수집한다. |
| Market data cache | 최신 시세와 자주 묻는 분석 재료를 Redis에 저장한다. |
| Market data store | 분석 재현과 장중 복기를 위해 필요한 시계열/요약 데이터를 PostgreSQL에 저장한다. |
| Analysis orchestrator | 질문 의도에 맞춰 필요한 데이터 조회, 요약, LLM 호출을 조합한다. |
| LLM adapter | 모델 호출, 프롬프트 구성, 응답 파싱, 비용/지연 기록을 담당한다. |
| Observability | API 호출량, KIS 오류, 캐시 적중률, LLM 지연, 사용자 응답 지연을 기록한다. |

## 5. Python 사용 범위

Python은 주 백엔드 스택으로 채택하지 않는다.

다만 다음 용도는 허용한다.

- 분석 로직 또는 프롬프트 실험용 notebook/script
- 과거 데이터 샘플링과 지표 검증용 일회성 도구
- 향후 Java 서비스와 분리된 비동기 분석 worker가 필요하다고 검증된 경우

운영 요청 처리, KIS 토큰/호출 제한 관리, 카카오톡 Webhook, 사용자 세션, 관리자 기능은 Spring Boot 백엔드가 소유한다.

## 6. 후속 결정 필요 항목

- Spring MVC와 WebFlux 중 초기 구현 모델
- Redis Streams와 Kafka 중 비동기 작업 확장 방식
- 시계열 데이터를 PostgreSQL 기본 테이블로 유지할지 TimescaleDB 확장을 사용할지
- LLM provider와 모델 선택
- 카카오톡 1:1 Agent Webhook 인증 및 재시도 정책
- KIS API별 TTL, 구독 수 제한, 장애 시 degrade 정책
