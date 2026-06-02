# TDD-003. KIS Domain Tool DTO 설계

- 작성일: 2026-05-31
- 문서 상태: 초안
- 범위: KIS Open API 사용 결정 결과를 기반으로 한 LLM-facing domain tool 및 DTO 설계
- 관련 문서:
  - `daon-docs/PRD/[PRD-002]kis-api-data-strategy.md`
  - `daon-docs/TDD/[TDD-001]backend-tdd.md`
  - `daon-docs/TDD/[TDD-002]agent-planner-runtime-example.md`
  - `데이터/한국투자증권_오픈API_전체문서_20260504_030007.xlsx`
  - `데이터/idxcode.txt`
  - `데이터/kospi_code.txt`
  - `데이터/kosdaq_code.txt`

## 1. 목적

이 문서는 `PRD-002`에서 `o`, `x`로 사용 여부가 결정된 KIS Open API를 기준으로, Daon Agent가 LLM에게 노출할 domain tool과 각 tool의 반환 DTO를 정의한다.

핵심 방향은 KIS raw API 하나를 LLM tool 하나로 노출하지 않는 것이다. KIS API는 기능 단위로 쪼개져 있고, 사용자 질문은 여러 데이터 축을 함께 요구한다. 따라서 LLM-facing tool은 "API 호출기"가 아니라 "분석에 필요한 의미 있는 근거 묶음"을 반환해야 한다.

예를 들어 사용자가 "삼성전자 지금 왜 빠져?"라고 물으면 현재가 API 하나만으로는 답하기 어렵다. 현재 가격, 거래량, 호가 잔량, 장운영 상태, 업종 동조 여부, VI, 뉴스/공시 제목, 장중 수급 추정이 함께 필요하다. 이런 묶음을 Spring Boot domain tool executor가 구성하고, LLM은 정규화된 DTO만 받는다.

## 2. 설계 원칙

- LLM에게 KIS raw API 이름, TR ID, URL, KIS 원문 필드명을 직접 노출하지 않는다.
- 하나의 domain tool은 사용자 질문에 답하는 데 필요한 evidence bundle을 반환한다.
- domain tool 내부에서는 여러 KIS raw API를 호출할 수 있다.
- 캐시와 DB가 없는 초기 단계에서도 tool 경계는 유지한다. 초기 구현은 각 tool 실행 시 필요한 KIS API를 즉시 호출한다.
- 모든 DTO는 데이터 기준 시각, 호출한 KIS API 목록, 데이터 제한 사항을 포함한다.
- 추정 데이터, 확정 데이터, 이벤트 후보, 계산 지표를 DTO 안에서 분리한다.
- `x`로 결정된 API는 기본 tool 실행 경로에 포함하지 않는다. 단, 향후 고도화 후보로 문서상 이유를 남긴다.
- P2 표에서 사용 여부가 비어 있는 일정성 이벤트와 ETF/ETN API는 이번 문서의 기본 tool catalog에는 포함하지 않고 후속 확장 후보로 둔다.
- KIS raw request/response DTO는 KIS 문서와의 추적성을 최우선으로 한다. Java 파일명에는 TR ID를 포함하고, 주석과 endpoint metadata에는 한국어 API명을 그대로 남긴다.
- LLM-facing domain DTO와 KIS raw DTO는 서로 다른 패키지에 둔다. raw DTO는 KIS 응답 구조를 수용하고, domain DTO는 답변 생성에 적합한 의미 구조를 제공한다.

## 3. 계층 구조

```text
LLM / Analyst
  -> LLM-facing Domain Tool
      -> Domain Tool Executor
          -> KIS Raw API Client
          -> KIS Raw Request DTO
          -> KIS Raw Response DTO
          -> Normalizer / Calculator
          -> Domain Tool DTO
  -> Evidence Packet Builder
  -> Answer Composer
```

각 계층의 책임은 다음과 같다.

| 계층 | 책임 |
| --- | --- |
| KIS Raw API Client | KIS 요청 파라미터, TR ID, URL, 인증 헤더, 연속조회, HTTP 오류 처리 |
| Raw Request DTO | KIS query parameter와 header 중 API별로 달라지는 요청값을 표현 |
| Raw Response DTO | KIS 원문 응답 구조 수용. 필드명은 KIS 문서 기준을 따른다. |
| Normalizer | 문자열 숫자, 코드값, 시간값을 Daon 도메인 타입으로 변환 |
| Domain Tool Executor | 여러 raw API 호출을 조합하고 계산 지표를 생성 |
| Domain Tool DTO | LLM이 이해할 수 있는 분석 의미 단위의 근거 묶음 |
| Evidence Packet Builder | 여러 tool DTO를 최종 답변 근거 패킷으로 병합 |

## 4. KIS Raw Request/Response DTO 설계

### 4.1 명명 규칙

KIS raw DTO 파일명은 다음 규칙을 사용한다.

```text
Kis{TR_ID}{ShortEnglishAlias}Request
Kis{TR_ID}{ShortEnglishAlias}Response
```

예시는 다음과 같다.

| KIS API명 | TR ID | Raw DTO prefix |
| --- | --- | --- |
| 주식현재가 시세 | `FHKST01010100` | `KisFhkst01010100InquirePrice` |
| 주식현재가 호가/예상체결 | `FHKST01010200` | `KisFhkst01010200AskingPrice` |
| 주식당일분봉조회 | `FHKST03010200` | `KisFhkst03010200TodayMinuteChart` |
| 변동성완화장치(VI) 현황 | `FHPST01390000` | `KisFhpst01390000ViStatus` |
| 종목별 투자자매매동향(일별) | `FHPTJ04160001` | `KisFhptj04160001InvestorTradeByStockDaily` |

이 규칙을 쓰는 이유는 다음과 같다.

- 한국어 API명은 PRD와 KIS 문서에서 가장 읽기 쉽지만 Java 파일명으로 쓰기 어렵다.
- 영어 별칭만 쓰면 KIS 문서의 한국어 API명과 연결이 약하다.
- TR ID를 파일명에 넣으면 KIS 엑셀 문서, PRD, 코드 검색이 모두 같은 식별자로 연결된다.
- 한국어 API명은 파일 상단 주석과 `KisEndpoint` enum에 반드시 남긴다.

각 raw DTO 파일 상단에는 다음 주석을 둔다.

```java
/**
 * KIS API: 주식현재가 시세
 * TR ID: FHKST01010100
 * URL: /uapi/domestic-stock/v1/quotations/inquire-price
 * PRD-002 사용 여부: o
 */
public record KisFhkst01010100InquirePriceRequest(...) {
}
```

### 4.2 Endpoint Metadata

KIS API 식별 정보는 `KisEndpoint`를 source of truth로 둔다. request/response DTO, client method, domain tool mapping은 모두 이 enum의 TR ID와 한국어 API명을 기준으로 맞춘다.

```java
public enum KisEndpoint {
    STOCK_CURRENT_PRICE(
            "주식현재가 시세",
            "FHKST01010100",
            "/uapi/domestic-stock/v1/quotations/inquire-price",
            true
    );

    private final String apiKoreanName;
    private final String trId;
    private final String url;
    private final boolean usedByPrd002;
}
```

`KisEndpoint`에는 `PRD-002`에서 `o` 처리한 API를 우선 등록한다. `x` 처리한 API는 기본 구현 대상이 아니므로 같은 enum에 넣지 않거나, 넣더라도 `usedByPrd002=false`와 제외 사유를 명시해야 한다.

### 4.3 패키지 구조

raw DTO는 `daon.be.agent.data.source.kis` 아래에 둔다. 패키지는 KIS 문서와 URL의 큰 분류를 기준으로 나누고, 파일명에서 TR ID로 세부 API를 식별한다.

```text
daon/be/agent/data/source/kis/
  common/
    KisEndpoint.java
    KisMarketCode.java
    KisPeriodCode.java
    KisTrCont.java
    KisResponseValidator.java

  oauth/
    KisTokenService.java
    dto/
      KisTokenRequest.java
      KisTokenResponse.java

  quotation/
    price/
      KisFhkst01010100InquirePriceRequest.java
      KisFhkst01010100InquirePriceResponse.java
      KisFhkst01010200AskingPriceRequest.java
      KisFhkst01010200AskingPriceResponse.java
      KisFhkst11300006IntstockMultpriceRequest.java
      KisFhkst11300006IntstockMultpriceResponse.java

    intraday/
      KisFhkst03010200TodayMinuteChartRequest.java
      KisFhkst03010200TodayMinuteChartResponse.java
      KisFhkst03010230DailyMinuteChartRequest.java
      KisFhkst03010230DailyMinuteChartResponse.java
      KisFhpst01060000TimeItemConclusionRequest.java
      KisFhpst01060000TimeItemConclusionResponse.java

    daily/
      KisFhkst03010100DailyItemChartPriceRequest.java
      KisFhkst03010100DailyItemChartPriceResponse.java

    index/
      KisFhpup02100000IndexPriceRequest.java
      KisFhpup02100000IndexPriceResponse.java
      KisFhkup03500200IndexMinuteChartRequest.java
      KisFhkup03500200IndexMinuteChartResponse.java
      KisFhpup02120000IndexDailyPriceRequest.java
      KisFhpup02120000IndexDailyPriceResponse.java

    calendar/
      KisCtca0903rCheckHolidayRequest.java
      KisCtca0903rCheckHolidayResponse.java

    event/
      KisFhpst01390000ViStatusRequest.java
      KisFhpst01390000ViStatusResponse.java
      KisFhkst01011800NewsTitleRequest.java
      KisFhkst01011800NewsTitleResponse.java
      KisFhkst130000C0CaptureUpLowPriceRequest.java
      KisFhkst130000C0CaptureUpLowPriceResponse.java

    investor/
      KisFhptj04030000InvestorTimeByMarketRequest.java
      KisFhptj04030000InvestorTimeByMarketResponse.java
      KisFhptj04040000InvestorDailyByMarketRequest.java
      KisFhptj04040000InvestorDailyByMarketResponse.java
      KisFhptj04160001InvestorTradeByStockDailyRequest.java
      KisFhptj04160001InvestorTradeByStockDailyResponse.java
      KisHhptj04160200InvestorTrendEstimateRequest.java
      KisHhptj04160200InvestorTrendEstimateResponse.java
      KisFhptj04400000ForeignInstitutionTotalRequest.java
      KisFhptj04400000ForeignInstitutionTotalResponse.java

    program/
      KisFhppg04650101ProgramTradeByStockRequest.java
      KisFhppg04650101ProgramTradeByStockResponse.java
      KisFhppg04650201ProgramTradeByStockDailyRequest.java
      KisFhppg04650201ProgramTradeByStockDailyResponse.java
      KisFhppg04600101CompProgramTradeTodayRequest.java
      KisFhppg04600101CompProgramTradeTodayResponse.java
      KisFhppg04600001CompProgramTradeDailyRequest.java
      KisFhppg04600001CompProgramTradeDailyResponse.java
      KisHhppg046600C1InvestorProgramTradeTodayRequest.java
      KisHhppg046600C1InvestorProgramTradeTodayResponse.java

    risk/
      KisFhpst04830000DailyShortSaleRequest.java
      KisFhpst04830000DailyShortSaleResponse.java
      KisFhpst04760000DailyCreditBalanceRequest.java
      KisFhpst04760000DailyCreditBalanceResponse.java
      KisHhpst074500C0DailyLoanTransRequest.java
      KisHhpst074500C0DailyLoanTransResponse.java

    stockinfo/
      KisCtpf1002rSearchStockInfoRequest.java
      KisCtpf1002rSearchStockInfoResponse.java

  finance/
    ratio/
      KisFhkst66430300FinancialRatioRequest.java
      KisFhkst66430300FinancialRatioResponse.java
      KisFhkst66430600StabilityRatioRequest.java
      KisFhkst66430600StabilityRatioResponse.java

    statement/
      KisFhkst66430200IncomeStatementRequest.java
      KisFhkst66430200IncomeStatementResponse.java
      KisFhkst66430100BalanceSheetRequest.java
      KisFhkst66430100BalanceSheetResponse.java

    expectation/
      KisFhkst663300C0InvestOpinionRequest.java
      KisFhkst663300C0InvestOpinionResponse.java
      KisHhkst668300C0EstimatePerformRequest.java
      KisHhkst668300C0EstimatePerformResponse.java

  ranking/
    volume/
      KisFhpst01710000VolumeRankRequest.java
      KisFhpst01710000VolumeRankResponse.java

    fluctuation/
      KisFhpst01700000FluctuationRankRequest.java
      KisFhpst01700000FluctuationRankResponse.java

    tradepower/
      KisFhpst01680000VolumePowerRequest.java
      KisFhpst01680000VolumePowerResponse.java
      KisFhkst190900C0BulkTransNumRequest.java
      KisFhkst190900C0BulkTransNumResponse.java

    attention/
      KisHhmcm000100C0HtsTopViewRequest.java
      KisHhmcm000100C0HtsTopViewResponse.java

    marketcap/
      KisFhpst01740000MarketCapRequest.java
      KisFhpst01740000MarketCapResponse.java

    trend/
      KisFhpst01870000NearNewHighLowRequest.java
      KisFhpst01870000NearNewHighLowResponse.java

    risk/
      KisFhpst04820000ShortSaleRankRequest.java
      KisFhpst04820000ShortSaleRankResponse.java
      KisFhkst17010000CreditBalanceRankRequest.java
      KisFhkst17010000CreditBalanceRankResponse.java

    valuation/
      KisFhpst01750000FinanceRatioRankRequest.java
      KisFhpst01750000FinanceRatioRankResponse.java
      KisFhpst01790000MarketValueRankRequest.java
      KisFhpst01790000MarketValueRankResponse.java

    afterhours/
      KisFhpst02350000OvertimeVolumeRequest.java
      KisFhpst02350000OvertimeVolumeResponse.java
      KisFhpst02340000OvertimeFluctuationRequest.java
      KisFhpst02340000OvertimeFluctuationResponse.java

  websocket/
    marketstatus/
      KisH0unmko0MarketOperationInfoMessage.java
```

### 4.4 Raw DTO와 Domain DTO의 분리

raw request/response DTO는 KIS API 하나와 1:1로 대응한다. 반면 domain tool DTO는 사용자 질문의 의미 단위와 대응한다.

```text
KisFhkst01010100InquirePriceResponse
KisFhkst01010200AskingPriceResponse
KisCtpf1002rSearchStockInfoResponse
  -> StockNowContextDto

KisFhkst03010200TodayMinuteChartResponse
KisFhkst03010230DailyMinuteChartResponse
KisFhpst01060000TimeItemConclusionResponse
  -> IntradayMoveContextDto
```

즉 KIS raw DTO는 API 추적성을 위해 잘게 만들고, LLM-facing DTO는 분석 의미를 위해 묶어서 만든다.

## 5. Domain Tool 공통 DTO

모든 domain tool DTO는 다음 공통 메타데이터를 포함한다.

```java
public record ToolEvidenceMeta(
        String toolName,
        OffsetDateTime requestedAt,
        OffsetDateTime dataTimestamp,
        List<KisApiCallSummary> kisApiCalls,
        DataFreshness freshness,
        List<String> limitations
) {
}

public record KisApiCallSummary(
        String apiName,
        String trId,
        String url,
        OffsetDateTime calledAt,
        boolean success,
        String failureReason
) {
}

public enum DataFreshness {
    REALTIME,
    INTRADAY_SNAPSHOT,
    INTRADAY_ESTIMATED,
    DAILY_CONFIRMED,
    HISTORICAL_CONFIRMED,
    STALE,
    UNAVAILABLE
}
```

공통 값 객체는 다음처럼 둔다.

```java
public record StockIdentityDto(
        String stockCode,
        String stockName,
        String market,
        String exchange,
        String industryCode,
        String industryName,
        Boolean etf,
        Boolean etn,
        Boolean managementIssue,
        Boolean tradingSuspended,
        Boolean nxtAvailable
) {
}

public record PricePointDto(
        BigDecimal price,
        BigDecimal change,
        BigDecimal changeRate,
        String changeSign,
        Long volume,
        BigDecimal tradingValue,
        OffsetDateTime observedAt
) {
}

public record OhlcvDto(
        LocalDate date,
        LocalTime time,
        BigDecimal open,
        BigDecimal high,
        BigDecimal low,
        BigDecimal close,
        Long volume,
        BigDecimal tradingValue,
        boolean provisional
) {
}

public record RankedStockDto(
        int rank,
        String stockCode,
        String stockName,
        BigDecimal price,
        BigDecimal changeRate,
        Long volume,
        BigDecimal tradingValue,
        List<String> reasons
) {
}
```

## 6. 권장 Domain Tool Catalog

### 6.1 GET_STOCK_NOW_CONTEXT

현재 종목의 가격, 거래량, 상태, 호가 균형을 한 번에 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `StockNowContextDto` |
| 주요 질문 | "지금 왜 올라?", "현재 어떤 상태야?", "호가가 어떤데?" |
| 내부 API | 주식현재가 시세, 주식현재가 호가/예상체결, 국내주식 장운영정보, 주식기본조회 |
| 제외 API | 주식현재가 체결. PRD-002에서 1분봉 중심 분석으로 대체하기로 결정 |

```java
public record StockNowContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        PricePointDto currentPrice,
        DailyPriceRangeDto dailyRange,
        OrderbookSnapshotDto orderbook,
        TradingStatusDto tradingStatus,
        ValuationSnapshotDto valuationFromQuote,
        List<String> interpretationHints
) {
}
```

이 tool은 단일 API로 분리하면 의미가 약하다. 현재가 시세는 가격과 일부 상태를 제공하지만, 호가 잔량과 매수/매도 대기 물량은 호가 API가 필요하다. 종목 정체성, 시장, ETF/ETN 여부, 관리종목 여부는 주식기본조회가 더 적합하다.

### 6.2 GET_INTRADAY_MOVE_CONTEXT

당일 또는 특정 거래일의 장중 가격 흐름과 변곡점을 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `IntradayMoveContextDto` |
| 주요 질문 | "오늘 장중 특이사항", "10시쯤 왜 빠졌어?", "아까 튄 구간 설명해줘" |
| 내부 API | 주식당일분봉조회, 주식일별분봉조회, 주식현재가 당일시간대별체결, 주식현재가 시세 |
| 제외 API | 실시간체결가 WebSocket, 주식현재가 체결. MVP는 1분봉 중심 |

```java
public record IntradayMoveContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        TimeRangeDto requestedRange,
        PricePointDto currentPrice,
        List<OhlcvDto> minuteBars,
        List<IntradayTurningPointDto> turningPoints,
        List<TimeAndSalesSummaryDto> executionSummaries,
        List<String> limitations
) {
}
```

`주식당일분봉조회`는 1회 최대 30건이고 당일 데이터만 제공한다. 과거 특정 거래일 복기는 `주식일별분봉조회`를 사용한다. 분봉만으로 부족한 특정 시점 복기는 `주식현재가 당일시간대별체결`을 제한적으로 붙인다.

### 6.3 GET_PRICE_TREND_CONTEXT

일/주/월/년 단위 가격 추세와 현재 위치를 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `PriceTrendContextDto` |
| 주요 질문 | "요즘 추세 어때?", "52주 고점 근처야?", "최근 한 달 흐름 정리해줘" |
| 내부 API | 국내주식기간별시세, 주식현재가 시세, 주식기본조회 |
| 제외 API | 주식현재가 일자별. 기간별시세로 대체 |

```java
public record PriceTrendContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        String periodType,
        List<OhlcvDto> bars,
        TrendSummaryDto trendSummary,
        VolatilitySummaryDto volatility,
        BreakoutContextDto breakoutContext,
        List<String> limitations
) {
}
```

이 tool은 장중 가격 원인보다 중장기 맥락을 제공한다. 복수 종목 비교에서는 각 종목별로 fan-out 한 뒤 수익률, 변동성, 거래대금, 돌파 여부를 동일 기준으로 비교한다.

### 6.4 GET_MARKET_INDUSTRY_CONTEXT

개별 종목 움직임이 시장/업종 동조인지, 종목 단독 이슈인지 판단할 근거를 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `MarketIndustryContextDto` |
| 주요 질문 | "시장 때문이야?", "업종 전체가 빠지는 거야?", "코스닥 수급이 어때?" |
| 내부 API | 국내업종 현재지수, 업종 분봉조회, 국내업종 일자별지수, 시장별 투자자매매동향(시세), 시장별 투자자매매동향(일별) |
| 제외 API | 국내업종 시간별지수(분). 업종 분봉조회가 더 적합 |

```java
public record MarketIndustryContextDto(
        ToolEvidenceMeta meta,
        String market,
        IndustryIndexSnapshotDto industrySnapshot,
        List<OhlcvDto> industryMinuteBars,
        List<OhlcvDto> industryDailyBars,
        MarketInvestorFlowDto intradayMarketInvestorFlow,
        List<MarketInvestorFlowDto> dailyMarketInvestorFlows,
        RelativeStrengthDto relativeStrength,
        List<String> interpretationHints
) {
}
```

`idxcode.txt`는 업종 코드 resolver의 기준 데이터로 사용한다. 개별 종목의 업종 코드는 현재가 시세 또는 주식기본조회에서 보강한다.

### 6.5 GET_EVENT_TIMELINE_CONTEXT

가격 변동 주변의 이벤트 후보를 시간순으로 묶는다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `EventTimelineContextDto` |
| 주요 질문 | "뉴스 때문이야?", "VI 걸렸어?", "오늘 무슨 이슈 있었어?" |
| 내부 API | 변동성완화장치(VI) 현황, 종합 시황/공시(제목), 국내주식 상하한가 포착 |
| 제외 API | 예상체결가 추이, 장마감 예상체결가. 예상체결 계열은 제외 결정 |

```java
public record EventTimelineContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        TimeRangeDto requestedRange,
        List<ViEventDto> viEvents,
        List<NewsTitleEventDto> newsTitleEvents,
        List<LimitPriceCaptureDto> limitPriceCaptures,
        List<EventCorrelationHintDto> correlationHints,
        List<String> cautions
) {
}
```

뉴스/공시 제목은 원인 확정 근거가 아니다. DTO에서도 `correlationHints`와 `cautions`를 분리해 "동시간대 후보"로만 다룬다.

### 6.6 GET_SUPPLY_DEMAND_CONTEXT

외국인, 기관, 개인의 수급 흐름을 추정/확정 데이터로 분리해 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `SupplyDemandContextDto` |
| 주요 질문 | "외국인이 사는 거야?", "기관 수급 때문이야?", "최근 수급 어때?" |
| 내부 API | 종목별 투자자매매동향(일별), 종목별 외인기관 추정가집계, 국내기관/외국인 매매종목가집계, 시장별 투자자매매동향 |
| 제외 API | 주식현재가 투자자, 외국계 매매종목 가집계, 종목별 외국계 순매수추이, 회원사 실시간 매매동향(틱) |

```java
public record SupplyDemandContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        IntradayEstimatedInvestorFlowDto intradayEstimate,
        List<DailyInvestorFlowDto> dailyInvestorFlows,
        List<InstitutionForeignRankDto> marketInstitutionForeignRanks,
        MarketInvestorFlowDto marketFlow,
        List<String> cautions
) {
}
```

`종목별 외인기관 추정가집계`는 직원 입력 기반 누계 자료이므로 확정 수급으로 표현하면 안 된다. `종목별 투자자매매동향(일별)`은 당일 15:40 이후 조회 제한과 산출 시간 변동이 있으므로 DTO 제한 사항에 반영한다.

### 6.7 GET_PROGRAM_TRADING_CONTEXT

프로그램매매가 개별 종목 또는 시장 전체 가격 흐름에 미친 가능성을 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `ProgramTradingContextDto` |
| 주요 질문 | "프로그램 매매 영향이야?", "시장 전체 프로그램 수급이 어때?" |
| 내부 API | 종목별 프로그램매매추이(체결), 종목별 프로그램매매추이(일별), 프로그램매매 종합현황(시간), 프로그램매매 종합현황(일별), 프로그램매매 투자자매매동향(당일) |
| 제외 API | 실시간프로그램매매 WebSocket. 초 단위 추적 제외 |

```java
public record ProgramTradingContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        List<ProgramTradePointDto> stockIntradayProgramTrades,
        List<ProgramTradeDailyDto> stockDailyProgramTrades,
        List<ProgramTradePointDto> marketIntradayProgramTrades,
        List<ProgramTradeDailyDto> marketDailyProgramTrades,
        ProgramTradeInvestorBreakdownDto investorBreakdown,
        List<String> interpretationHints
) {
}
```

시장 전체 프로그램매매 시간 API는 최근 30분 중심으로 사용한다. 장기 비교 기준은 일별 API로 보완한다.

### 6.8 GET_SHORT_CREDIT_LOAN_CONTEXT

하락 압력, 반대매매 부담, 공매도 가능 물량 같은 수급 리스크를 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `ShortCreditLoanContextDto` |
| 주요 질문 | "공매도 부담 있어?", "신용이 과열이야?", "대차잔고 늘었어?" |
| 내부 API | 국내주식 공매도 일별추이, 국내주식 신용잔고 일별추이, 종목별 일별 대차거래추이, 국내주식 공매도 상위종목, 국내주식 신용잔고 상위 |

```java
public record ShortCreditLoanContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        List<ShortSaleDailyDto> shortSaleTrend,
        List<CreditBalanceDailyDto> creditBalanceTrend,
        List<LoanTransactionDailyDto> loanTransactionTrend,
        List<RankedStockDto> shortSaleRankCandidates,
        List<RankedStockDto> creditBalanceRankCandidates,
        RiskSummaryDto riskSummary,
        List<String> cautions
) {
}
```

공매도, 신용, 대차는 단독으로 가격 방향을 설명하면 안 된다. DTO의 `cautions`에는 "공매도 증가가 가격 하락의 확정 원인은 아님", "대차잔고 증가는 실제 공매도 체결과 다름" 같은 제한을 포함한다.

### 6.9 GET_FUNDAMENTAL_CONTEXT

종목의 정적 정보, 실적 규모, 재무비율, 안정성 지표를 묶는다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `FundamentalContextDto` |
| 주요 질문 | "재무는 어때?", "장기 투자 관점에서 체력이 좋아?", "A/B 중 재무가 나은 건?" |
| 내부 API | 주식기본조회, 국내주식 재무비율, 국내주식 손익계산서, 국내주식 대차대조표, 국내주식 안정성비율 |
| 제외 API | 수익성비율, 성장성비율, 기타주요비율. 중복 또는 데이터 품질 이슈로 제외 |

```java
public record FundamentalContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        List<FinancialRatioPeriodDto> financialRatios,
        List<IncomeStatementPeriodDto> incomeStatements,
        List<BalanceSheetPeriodDto> balanceSheets,
        List<StabilityRatioPeriodDto> stabilityRatios,
        FundamentalSummaryDto summary,
        List<String> dataQualityNotes
) {
}
```

KIS 재무 API는 일부 항목이 `99.99` 등 비정상 값으로 표시될 수 있다. Normalizer는 이를 숫자 99.99로 확정하지 말고 `dataQualityNotes`에 남기거나 null 처리한다.

### 6.10 GET_EXPECTATION_CONTEXT

증권사 의견과 애널리스트 추정 실적을 별도 묶음으로 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `ExpectationContextDto` |
| 주요 질문 | "목표가가 어떻게 변했어?", "실적 전망은 어때?", "컨센서스가 좋아졌어?" |
| 내부 API | 국내주식 종목투자의견, 국내주식 종목추정실적 |
| 제외 API | 국내주식 증권사별 투자의견. 종목 기준 분석에는 직접성이 낮음 |

```java
public record ExpectationContextDto(
        ToolEvidenceMeta meta,
        StockIdentityDto identity,
        List<InvestmentOpinionDto> investmentOpinions,
        EstimatedPerformanceDto estimatedPerformance,
        List<String> cautions
) {
}
```

투자의견과 추정실적은 후행적이거나 월중 변동 가능성이 있다. 가격 변동의 직접 원인으로 단정하지 않고 장기 기대치 맥락으로만 사용한다.

### 6.11 GET_MARKET_MOVER_CANDIDATES

오늘 시장에서 먼저 볼 만한 이상 흐름 후보를 만든다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `MarketMoverCandidatesDto` |
| 주요 질문 | "오늘 뭐가 특이해?", "급등주/급락주 찾아줘", "거래량 터진 종목 있어?" |
| 내부 API | 거래량순위, 등락률 순위, 체결강도 상위, 대량체결건수 상위, HTS조회상위20종목, 신고/신저근접종목 상위, 상하한가 포착 |
| 제외 API | 호가잔량 순위. 허수/취소 주문 영향과 초단위 호가 추적 제외 결정 |

```java
public record MarketMoverCandidatesDto(
        ToolEvidenceMeta meta,
        String market,
        List<RankedStockDto> volumeRankCandidates,
        List<RankedStockDto> fluctuationRankCandidates,
        List<RankedStockDto> volumePowerCandidates,
        List<RankedStockDto> bulkTradeCandidates,
        List<RankedStockDto> htsTopViewedCandidates,
        List<RankedStockDto> nearHighLowCandidates,
        List<RankedStockDto> limitPriceCandidates,
        List<MarketMoverCandidateSummaryDto> mergedCandidates
) {
}
```

순위 API는 "원인"이 아니라 "후보 생성" 도구다. `mergedCandidates`는 여러 순위 API에 중복 등장한 종목을 통합하고, 이후 상세 분석 tool 호출 대상으로 넘긴다.

### 6.12 GET_AFTER_HOURS_CONTEXT

시간외 단일가 구간의 이상 거래 후보와 장후 이벤트 후보를 제공한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `AfterHoursContextDto` |
| 주요 질문 | "시간외에서 왜 올랐어?", "내일 볼 종목 있어?", "장후에 움직인 종목은?" |
| 내부 API | 시간외거래량순위, 시간외등락율순위, 종합 시황/공시(제목) |
| 제외 API | 시간외잔량 순위, 시간외예상체결등락률. 잔량/예상체결 계열 제외 |

```java
public record AfterHoursContextDto(
        ToolEvidenceMeta meta,
        LocalDate tradingDate,
        List<RankedStockDto> afterHoursVolumeCandidates,
        List<RankedStockDto> afterHoursFluctuationCandidates,
        List<NewsTitleEventDto> afterHoursNewsTitleEvents,
        List<AfterHoursCandidateSummaryDto> mergedCandidates,
        List<String> cautions
) {
}
```

시간외 가격은 거래량이 얇으면 왜곡될 수 있으므로 거래량, 정규장 거래대금, 뉴스/공시 제목을 함께 붙인다.

### 6.13 GET_SCREENED_STOCK_CANDIDATES

재무/시장가치 순위, 시총 상위 API를 조합해 추천 후보군을 만든다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `ScreenedStockCandidatesDto` |
| 주요 질문 | "조건에 맞는 종목 찾아줘", "재무 좋은 후보 있어?", "밸류에이션 싼 종목은?" |
| 내부 API | 국내주식 재무비율 순위, 국내주식 시장가치 순위, 국내주식 시가총액 상위 |

```java
public record ScreenedStockCandidatesDto(
        ToolEvidenceMeta meta,
        List<RankedStockDto> financialRatioRankCandidates,
        List<RankedStockDto> marketValueRankCandidates,
        List<RankedStockDto> marketCapRankCandidates,
        List<ScreenedStockCandidateDto> mergedCandidates,
        List<String> filteringRules
) {
}
```

이 tool은 API 결과를 그대로 추천하지 않는다. 최소한 관리/주의 종목 제외, 거래대금 조건, 최근 급등락 여부, 재무 데이터 품질 확인을 거친 후보로만 사용한다.

### 6.14 GET_WATCHLIST_SNAPSHOT

관심/트레이싱 종목 다수를 빠르게 스캔한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `WatchlistSnapshotDto` |
| 주요 질문 | "관심 종목 중 특이한 거 있어?", "내 리스트에서 오늘 강한 종목은?" |
| 내부 API | 관심종목(멀티종목) 시세조회, 주식현재가 시세, 거래량순위, 등락률순위 |

```java
public record WatchlistSnapshotDto(
        ToolEvidenceMeta meta,
        String watchlistId,
        List<WatchlistStockSnapshotDto> stocks,
        List<WatchlistAlertCandidateDto> alertCandidates,
        List<String> limitations
) {
}
```

초기에는 DB가 없으므로 watchlist 입력을 요청 파라미터의 종목코드 목록으로 받을 수 있다. 이후 DB가 생기면 사용자 관심종목 저장소와 연결한다.

### 6.15 COMPARE_STOCKS_CONTEXT

복수 종목을 동일 기준으로 비교할 수 있게 여러 domain tool 결과를 fan-in 한다.

| 항목 | 내용 |
| --- | --- |
| 반환 DTO | `CompareStocksContextDto` |
| 주요 질문 | "삼성전자랑 SK하이닉스 비교해줘", "A/B/C 중 뭐가 상대적으로 강해?" |
| 내부 API | 단일 raw API 직접 호출보다 위 domain tool들을 종목별 fan-out 후 병합 |

```java
public record CompareStocksContextDto(
        ToolEvidenceMeta meta,
        List<String> stockCodes,
        ComparisonScopeDto scope,
        List<ComparableStockSnapshotDto> stocks,
        List<ComparisonMetricDto> metrics,
        List<String> missingDataNotes,
        List<String> cautions
) {
}
```

비교 tool은 모든 데이터를 항상 호출하지 않는다. 질문 초점에 따라 `StockNow`, `PriceTrend`, `SupplyDemand`, `Fundamental`, `Expectation` 중 필요한 묶음만 fan-out 한다.

## 7. Domain Tool 구현 현황

이 표는 LLM-facing domain tool 구현 상태를 추적한다. KIS raw API client, raw request/response DTO, endpoint metadata가 구현되어 있어도 domain DTO 정규화, Spring AI `@Tool` 노출, 단위 테스트가 없으면 `미구현`으로 본다.

현재 구현 완료된 LLM-facing domain tool은 11개다. `GET_STOCK_NOW_CONTEXT`, `GET_INTRADAY_MOVE_CONTEXT`, `GET_EVENT_TIMELINE_CONTEXT`, `GET_MARKET_INDUSTRY_CONTEXT`, `GET_PRICE_TREND_CONTEXT`, `GET_SUPPLY_DEMAND_CONTEXT`, `GET_FUNDAMENTAL_CONTEXT`, `COMPARE_STOCKS_CONTEXT`, `GET_MARKET_MOVER_CANDIDATES`, `GET_PROGRAM_TRADING_CONTEXT`, `GET_SHORT_CREDIT_LOAN_CONTEXT`는 domain service, domain DTO, `KISTools`의 Spring AI `@Tool` 메서드, 단위 테스트를 기준으로 구현 완료로 본다.

| 순서 | Domain Tool | 상태 | 코드 기준 | 다음 작업 |
| --- | --- | --- | --- | --- |
| 1 | `GET_STOCK_NOW_CONTEXT` | 구현완료 | `KisStockNowContextService`, `StockNowContextDto`, `KISTools.getStockNowContext`, `KisStockNowContextServiceTest` | 국내주식 장운영정보 WebSocket 스냅샷 결합은 후속 보강 |
| 2 | `GET_INTRADAY_MOVE_CONTEXT` | 구현완료 | `KisIntradayMoveContextService`, `IntradayMoveContextDto`, `KISTools.getIntradayMoveContext`, `KisIntradayMoveContextServiceTest` | 과거 거래일 조회의 시간대별체결 보강과 다회 연속조회는 후속 보강 |
| 3 | `GET_EVENT_TIMELINE_CONTEXT` | 구현완료 | `KisEventTimelineContextService`, `EventTimelineContextDto`, `KISTools.getEventTimelineContext`, `KisEventTimelineContextServiceTest` | 뉴스/공시 원문 조회와 상하한가 포착 시각 보강은 후속 보강 |
| 4 | `GET_MARKET_INDUSTRY_CONTEXT` | 구현완료 | `KisMarketIndustryContextService`, `MarketIndustryContextDto`, `KISTools.getMarketIndustryContext`, `KisMarketIndustryContextServiceTest` | 종목코드 기반 업종 resolver와 종목 대비 상대강도 계산은 후속 보강 |
| 5 | `GET_PRICE_TREND_CONTEXT` | 구현완료 | `KisPriceTrendContextService`, `PriceTrendContextDto`, `KISTools.getPriceTrendContext`, `KisPriceTrendContextServiceTest` | 이동평균, 장기 변동성, 복수 종목 비교용 표준 지표는 후속 보강 |
| 6 | `GET_SUPPLY_DEMAND_CONTEXT` | 구현완료 | `KisSupplyDemandContextService`, `SupplyDemandContextDto`, `KISTools.getSupplyDemandContext`, `KisSupplyDemandContextServiceTest` | 연속조회, 시장별 rank 복수 결과, 당일 산출 시각 검증은 후속 보강 |
| 7 | `GET_FUNDAMENTAL_CONTEXT` | 구현완료 | `KisFundamentalContextService`, `FundamentalContextDto`, `KISTools.getFundamentalContext`, `KisFundamentalContextServiceTest` | 복수 종목 비교용 표준 점수화와 재무 API 비정상 표시값 세부 룰은 후속 보강 |
| 8 | `COMPARE_STOCKS_CONTEXT` | 구현완료 | `KisCompareStocksContextService`, `CompareStocksContextDto`, `KISTools.getCompareStocksContext`, `KisCompareStocksContextServiceTest` | focus별 하위 tool 선택 정책과 expectation 컨텍스트 결합은 후속 보강 |
| 9 | `GET_MARKET_MOVER_CANDIDATES` | 구현완료 | `KisMarketMoverCandidatesService`, `MarketMoverCandidatesDto`, `KISTools.getMarketMoverCandidates`, `KisMarketMoverCandidatesServiceTest` | 후보별 상세 domain tool 후속 fan-out과 우선주/관리·주의 종목 필터는 후속 보강 |
| 10 | `GET_PROGRAM_TRADING_CONTEXT` | 구현완료 | `KisProgramTradingContextService`, `ProgramTradingContextDto`, `KISTools.getProgramTradingContext`, `KisProgramTradingContextServiceTest` | 시장구분 코드 세분화와 프로그램매매 금액 단위 검증은 후속 보강 |
| 11 | `GET_SHORT_CREDIT_LOAN_CONTEXT` | 구현완료 | `KisShortCreditLoanContextService`, `ShortCreditLoanContextDto`, `KISTools.getShortCreditLoanContext`, `KisShortCreditLoanContextServiceTest` | 공매도·신용·대차 rank 필터링, 잔고 단위 검증, 대차 연속조회는 후속 보강 |
| 12 | `GET_EXPECTATION_CONTEXT` | 미구현 | catalog tool 없음 | 종목투자의견과 추정실적 기반 기대치 DTO 구현 |
| 13 | `GET_AFTER_HOURS_CONTEXT` | 미구현 | catalog tool 없음 | 시간외 거래량/등락률 순위와 장후 뉴스 후보 DTO 구현 |
| 14 | `GET_SCREENED_STOCK_CANDIDATES` | 미구현 | catalog tool 없음 | 재무비율 순위, 시장가치 순위, 시총 상위 기반 후보 DTO 구현 |
| 15 | `GET_WATCHLIST_SNAPSHOT` | 미구현 | catalog tool 없음 | 관심종목 멀티시세와 순위 API 기반 watchlist snapshot DTO 구현 |

상태 값은 다음 기준으로 갱신한다.

| 상태 | 기준 |
| --- | --- |
| 미구현 | catalog에만 있거나 placeholder만 존재 |
| 진행중 | domain DTO 또는 executor 일부가 있으나 Spring AI tool 노출 또는 테스트가 없음 |
| 구현완료 | domain DTO, executor, Spring AI `@Tool`, 단위 테스트가 있고 기본 검증을 통과 |
| 보류 | API 제한, 정책 결정, 데이터 품질 문제로 의도적으로 구현을 미룬 상태 |

## 8. 사용 결정 API와 Tool 매핑

이 표는 `PRD-002`에서 `o` 처리한 API만 기본 구현 대상으로 정리한다. `Raw DTO prefix`는 실제 파일명이 아니라 request/response 공통 접두어다. 예를 들어 `KisFhkst01010100InquirePrice`는 `KisFhkst01010100InquirePriceRequest`, `KisFhkst01010100InquirePriceResponse` 두 파일로 나뉜다.

| KIS API명 | TR ID | Raw DTO prefix | 권장 Tool |
| --- | --- | --- | --- |
| 접근토큰발급(P) | - | `KisToken` | LLM-facing tool 아님. KIS client 인프라 |
| 실시간 웹소켓 접속키 발급 | - | `KisWebsocketApproval` | LLM-facing tool 아님. WebSocket 인프라 |
| 국내주식 장운영정보 (통합) | `H0UNMKO0` | `KisH0unmko0MarketOperationInfo` | `GET_STOCK_NOW_CONTEXT` 또는 WebSocket 인프라 |
| 주식현재가 시세 | `FHKST01010100` | `KisFhkst01010100InquirePrice` | `GET_STOCK_NOW_CONTEXT`, `GET_INTRADAY_MOVE_CONTEXT`, `GET_PRICE_TREND_CONTEXT` |
| 주식현재가 호가/예상체결 | `FHKST01010200` | `KisFhkst01010200AskingPrice` | `GET_STOCK_NOW_CONTEXT` |
| 주식현재가 당일시간대별체결 | `FHPST01060000` | `KisFhpst01060000TimeItemConclusion` | `GET_INTRADAY_MOVE_CONTEXT` |
| 관심종목(멀티종목) 시세조회 | `FHKST11300006` | `KisFhkst11300006IntstockMultprice` | `GET_WATCHLIST_SNAPSHOT` |
| 주식당일분봉조회 | `FHKST03010200` | `KisFhkst03010200TodayMinuteChart` | `GET_INTRADAY_MOVE_CONTEXT` |
| 주식일별분봉조회 | `FHKST03010230` | `KisFhkst03010230DailyMinuteChart` | `GET_INTRADAY_MOVE_CONTEXT` |
| 국내주식기간별시세(일/주/월/년) | `FHKST03010100` | `KisFhkst03010100DailyItemChartPrice` | `GET_PRICE_TREND_CONTEXT`, `COMPARE_STOCKS_CONTEXT` |
| 국내업종 현재지수 | `FHPUP02100000` | `KisFhpup02100000IndexPrice` | `GET_MARKET_INDUSTRY_CONTEXT` |
| 업종 분봉조회 | `FHKUP03500200` | `KisFhkup03500200IndexMinuteChart` | `GET_MARKET_INDUSTRY_CONTEXT` |
| 국내업종 일자별지수 | `FHPUP02120000` | `KisFhpup02120000IndexDailyPrice` | `GET_MARKET_INDUSTRY_CONTEXT` |
| 국내휴장일조회 | `CTCA0903R` | `KisCtca0903rCheckHoliday` | Resolver/Trading calendar 인프라 |
| 변동성완화장치(VI) 현황 | `FHPST01390000` | `KisFhpst01390000ViStatus` | `GET_EVENT_TIMELINE_CONTEXT` |
| 종합 시황/공시(제목) | `FHKST01011800` | `KisFhkst01011800NewsTitle` | `GET_EVENT_TIMELINE_CONTEXT`, `GET_AFTER_HOURS_CONTEXT` |
| 국내주식 상하한가 포착 | `FHKST130000C0` | `KisFhkst130000C0CaptureUpLowPrice` | `GET_EVENT_TIMELINE_CONTEXT`, `GET_MARKET_MOVER_CANDIDATES` |
| 시장별 투자자매매동향(시세) | `FHPTJ04030000` | `KisFhptj04030000InvestorTimeByMarket` | `GET_MARKET_INDUSTRY_CONTEXT`, `GET_SUPPLY_DEMAND_CONTEXT` |
| 시장별 투자자매매동향(일별) | `FHPTJ04040000` | `KisFhptj04040000InvestorDailyByMarket` | `GET_MARKET_INDUSTRY_CONTEXT`, `GET_SUPPLY_DEMAND_CONTEXT` |
| 종목별 투자자매매동향(일별) | `FHPTJ04160001` | `KisFhptj04160001InvestorTradeByStockDaily` | `GET_SUPPLY_DEMAND_CONTEXT` |
| 종목별 외인기관 추정가집계 | `HHPTJ04160200` | `KisHhptj04160200InvestorTrendEstimate` | `GET_SUPPLY_DEMAND_CONTEXT` |
| 국내기관/외국인 매매종목가집계 | `FHPTJ04400000` | `KisFhptj04400000ForeignInstitutionTotal` | `GET_SUPPLY_DEMAND_CONTEXT` |
| 종목별 프로그램매매추이(체결) | `FHPPG04650101` | `KisFhppg04650101ProgramTradeByStock` | `GET_PROGRAM_TRADING_CONTEXT` |
| 종목별 프로그램매매추이(일별) | `FHPPG04650201` | `KisFhppg04650201ProgramTradeByStockDaily` | `GET_PROGRAM_TRADING_CONTEXT` |
| 프로그램매매 종합현황(시간) | `FHPPG04600101` | `KisFhppg04600101CompProgramTradeToday` | `GET_PROGRAM_TRADING_CONTEXT` |
| 프로그램매매 종합현황(일별) | `FHPPG04600001` | `KisFhppg04600001CompProgramTradeDaily` | `GET_PROGRAM_TRADING_CONTEXT` |
| 프로그램매매 투자자매매동향(당일) | `HHPPG046600C1` | `KisHhppg046600C1InvestorProgramTradeToday` | `GET_PROGRAM_TRADING_CONTEXT` |
| 국내주식 공매도 일별추이 | `FHPST04830000` | `KisFhpst04830000DailyShortSale` | `GET_SHORT_CREDIT_LOAN_CONTEXT` |
| 국내주식 신용잔고 일별추이 | `FHPST04760000` | `KisFhpst04760000DailyCreditBalance` | `GET_SHORT_CREDIT_LOAN_CONTEXT` |
| 종목별 일별 대차거래추이 | `HHPST074500C0` | `KisHhpst074500C0DailyLoanTrans` | `GET_SHORT_CREDIT_LOAN_CONTEXT` |
| 주식기본조회 | `CTPF1002R` | `KisCtpf1002rSearchStockInfo` | `GET_STOCK_NOW_CONTEXT`, `GET_FUNDAMENTAL_CONTEXT`, resolver |
| 국내주식 재무비율 | `FHKST66430300` | `KisFhkst66430300FinancialRatio` | `GET_FUNDAMENTAL_CONTEXT` |
| 국내주식 손익계산서 | `FHKST66430200` | `KisFhkst66430200IncomeStatement` | `GET_FUNDAMENTAL_CONTEXT` |
| 국내주식 대차대조표 | `FHKST66430100` | `KisFhkst66430100BalanceSheet` | `GET_FUNDAMENTAL_CONTEXT` |
| 국내주식 안정성비율 | `FHKST66430600` | `KisFhkst66430600StabilityRatio` | `GET_FUNDAMENTAL_CONTEXT` |
| 국내주식 종목투자의견 | `FHKST663300C0` | `KisFhkst663300C0InvestOpinion` | `GET_EXPECTATION_CONTEXT` |
| 국내주식 종목추정실적 | `HHKST668300C0` | `KisHhkst668300C0EstimatePerform` | `GET_EXPECTATION_CONTEXT` |
| 거래량순위 | `FHPST01710000` | `KisFhpst01710000VolumeRank` | `GET_MARKET_MOVER_CANDIDATES`, `GET_WATCHLIST_SNAPSHOT` |
| 국내주식 등락률 순위 | `FHPST01700000` | `KisFhpst01700000FluctuationRank` | `GET_MARKET_MOVER_CANDIDATES`, `GET_WATCHLIST_SNAPSHOT` |
| 국내주식 체결강도 상위 | `FHPST01680000` | `KisFhpst01680000VolumePower` | `GET_MARKET_MOVER_CANDIDATES` |
| 국내주식 대량체결건수 상위 | `FHKST190900C0` | `KisFhkst190900C0BulkTransNum` | `GET_MARKET_MOVER_CANDIDATES` |
| HTS조회상위20종목 | `HHMCM000100C0` | `KisHhmcm000100C0HtsTopView` | `GET_MARKET_MOVER_CANDIDATES` |
| 국내주식 시가총액 상위 | `FHPST01740000` | `KisFhpst01740000MarketCap` | `GET_SCREENED_STOCK_CANDIDATES` |
| 국내주식 신고/신저근접종목 상위 | `FHPST01870000` | `KisFhpst01870000NearNewHighLow` | `GET_MARKET_MOVER_CANDIDATES` |
| 국내주식 공매도 상위종목 | `FHPST04820000` | `KisFhpst04820000ShortSaleRank` | `GET_SHORT_CREDIT_LOAN_CONTEXT` |
| 국내주식 신용잔고 상위 | `FHKST17010000` | `KisFhkst17010000CreditBalanceRank` | `GET_SHORT_CREDIT_LOAN_CONTEXT` |
| 국내주식 재무비율 순위 | `FHPST01750000` | `KisFhpst01750000FinanceRatioRank` | `GET_SCREENED_STOCK_CANDIDATES` |
| 국내주식 시장가치 순위 | `FHPST01790000` | `KisFhpst01790000MarketValueRank` | `GET_SCREENED_STOCK_CANDIDATES` |
| 국내주식 시간외거래량순위 | `FHPST02350000` | `KisFhpst02350000OvertimeVolume` | `GET_AFTER_HOURS_CONTEXT` |
| 국내주식 시간외등락율순위 | `FHPST02340000` | `KisFhpst02340000OvertimeFluctuation` | `GET_AFTER_HOURS_CONTEXT` |

## 9. 제외 API 처리 원칙

`x` API는 다음 유형으로 나뉜다.

| 제외 유형 | API 예시 | 처리 |
| --- | --- | --- |
| MVP 해상도 초과 | 실시간체결가, 실시간호가, 최근 체결, 회원사 실시간 매매동향 | 1분봉/REST 스냅샷 기반 tool로 대체 |
| 예상체결 제외 | 실시간예상체결, 예상체결가 추이, 장마감 예상체결가, 예상체결 상승/하락상위 | 실제 체결과 확정 데이터 중심으로 답변 |
| 중복 API | 주식현재가 일자별, 주식현재가 투자자, 국내업종 시간별지수 | 더 범용적이거나 상세한 API로 대체 |
| 데이터 품질/직접성 낮음 | 기타주요비율, 배당률 상위, 시간외잔량 순위 | 후속 기능에서 재검증 |
| 과도하게 미시적 | 외국계 순매수추이, 회원사 틱 | 고급 분석 단계에서 재검토 |

제외 API는 구현에서 완전히 잊는 것이 아니라, 각 domain tool의 `limitations`와 후속 개선 후보에 남긴다.

## 10. Tool 선택 기준

질문 유형별 1차 추천 tool은 다음과 같다.

| 사용자 질문 유형 | 1차 Tool | 보강 Tool |
| --- | --- | --- |
| 지금 급등/급락 원인 | `GET_STOCK_NOW_CONTEXT`, `GET_INTRADAY_MOVE_CONTEXT` | `GET_MARKET_INDUSTRY_CONTEXT`, `GET_EVENT_TIMELINE_CONTEXT`, `GET_SUPPLY_DEMAND_CONTEXT` |
| 특정 시점 복기 | `GET_INTRADAY_MOVE_CONTEXT` | `GET_EVENT_TIMELINE_CONTEXT`, `GET_MARKET_INDUSTRY_CONTEXT` |
| 오늘 장중 특이사항 | `GET_INTRADAY_MOVE_CONTEXT`, `GET_MARKET_MOVER_CANDIDATES` | `GET_EVENT_TIMELINE_CONTEXT`, `GET_PROGRAM_TRADING_CONTEXT` |
| 중장기 추세 | `GET_PRICE_TREND_CONTEXT` | `GET_FUNDAMENTAL_CONTEXT`, `GET_SUPPLY_DEMAND_CONTEXT` |
| 수급 원인 | `GET_SUPPLY_DEMAND_CONTEXT` | `GET_PROGRAM_TRADING_CONTEXT`, `GET_SHORT_CREDIT_LOAN_CONTEXT` |
| 재무/실적 | `GET_FUNDAMENTAL_CONTEXT` | `GET_EXPECTATION_CONTEXT` |
| 복수 종목 비교 | `COMPARE_STOCKS_CONTEXT` | 질문 초점별 하위 tool fan-out |
| 오늘 볼 종목 탐색 | `GET_MARKET_MOVER_CANDIDATES` | `GET_SCREENED_STOCK_CANDIDATES`, `GET_AFTER_HOURS_CONTEXT` |
| 관심 종목 모니터링 | `GET_WATCHLIST_SNAPSHOT` | 이상 종목만 상세 tool 호출 |

## 11. 초기 구현 권장 순서

캐시와 DB가 없는 초기 구현에서는 호출 비용과 답변 가치의 균형을 기준으로 다음 순서를 권장한다.

1. `GET_STOCK_NOW_CONTEXT`
2. `GET_INTRADAY_MOVE_CONTEXT`
3. `GET_EVENT_TIMELINE_CONTEXT`
4. `GET_MARKET_INDUSTRY_CONTEXT`
5. `GET_PRICE_TREND_CONTEXT`
6. `GET_SUPPLY_DEMAND_CONTEXT`
7. `GET_FUNDAMENTAL_CONTEXT`
8. `COMPARE_STOCKS_CONTEXT`
9. `GET_MARKET_MOVER_CANDIDATES`
10. `GET_PROGRAM_TRADING_CONTEXT`
11. `GET_SHORT_CREDIT_LOAN_CONTEXT`
12. `GET_EXPECTATION_CONTEXT`
13. `GET_AFTER_HOURS_CONTEXT`
14. `GET_SCREENED_STOCK_CANDIDATES`
15. `GET_WATCHLIST_SNAPSHOT`

초기 1~4번만으로도 단일 종목 현재 흐름, 특정 시점 복기, 이벤트 후보, 시장/업종 동조 여부를 설명할 수 있다.

## 12. 남은 설계 과제

- KIS raw response DTO의 필드 단위 설계
- 종목명/종목코드 resolver와 `kospi_code.txt`, `kosdaq_code.txt` 정제 방식
- 업종 코드 resolver와 `idxcode.txt` 정제 방식
- tool별 호출 예산, timeout, retry, rate limit 정책
- 캐시/DB 도입 후 freshness 판정 기준
- Spring AI tool schema에 노출할 argument 설계
- Evidence Packet에 tool DTO를 병합하는 규칙
- API 장애 시 partial DTO 반환 정책
- 숫자 문자열, 부호 코드, 시간 문자열, 단위 변환 표준화
