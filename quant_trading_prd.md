자동매매 시스템 제품 요구사항 정의서 (PRD)
본 문서는 AI 코딩 에이전트(Antigravity)가 즉시 파이썬 코드로 구현할 수 있도록 설계된 **'테마 및 120일 이평선 기반 자동매매 시스템'**의 제품 요구사항 정의서입니다. 본 PRD는 펀더멘털 분석, 기술적 분석, 엄격한 리스크 관리, 그리고 클라우드 기반 모니터링을 결합한 퀀트 투자 시스템 구축을 목표로 합니다
,
.

--------------------------------------------------------------------------------
1. System Overview (시스템 개요 및 목표)
시스템 명: Theme-based 120MA Quant Auto-Trader
목표: '원전 및 반도체 테마' 중 '영업이익 흑자'를 내는 기업만을 선별하여, 해당 주가가 120일 장기 이동평균선을 강하게 돌파하는 시점에 매수하여 추세 상승의 수익을 극대화하는 자동화 시스템입니다
,
.
핵심 원칙:
철저한 손절(Risk Management): 매수 단가 대비 -10% 하락 시 월간 리밸런싱을 기다리지 않고 장중 즉각 시장가로 매도하여 계좌의 치명적 손실을 방지합니다.
정기적 포트폴리오 교체: 매월 1회 펀더멘털 데이터 및 기술적 이탈 여부를 재평가하여 포트폴리오를 교체(Rebalancing)합니다
.

--------------------------------------------------------------------------------
2. Data Requirements (필요 데이터 명세)
시스템 구현을 위해 수집 및 가공해야 할 데이터 소스와 스키마는 다음과 같습니다.
2.1 펀더멘털 데이터 (유니버스 필터링용)
데이터 소스: 전자공시시스템 Open DART API (또는 OpenDartReader 파이썬 라이브러리 활용)
,
.
필요 파라미터:
crtfc_key: DART API 인증키
.
corp_code: 상장 기업 고유 번호
,
.
bgn_de: 공시 검색 시작일 (최근 1년).
pblntf_detail_ty: 사업보고서, 분기보고서, 반기보고서 코드
.
추출 및 판단 항목:
영업이익 흑자: 재무제표(손익계산서) 상 '영업이익(Operating Profit)' 값이 '0' 초과인지 판별
,
.
테마 분류: 네이버 금융 크롤링 또는 사업보고서 '사업의 내용' 텍스트 마이닝(키워드 기반 리트리벌)을 통해 '원자력', '원전', '반도체', 'DRAM' 등의 키워드가 포함된 종목 추출
,
.
2.2 가격 데이터 (매매 타이밍 및 손절 모니터링용)
데이터 소스: 증권사 API (예: 한국투자증권 KIS Developers API) 연동을 통한 일봉 및 실시간 장중 현재가 수집
,
.
필요 항목 및 기간:
120일 이동평균선 계산을 위해 최근 120영업일 이상의 일봉 OHLCV(시가, 고가, 저가, 종가, 거래량) 데이터 필요
,
.
매일 실시간 체결가 데이터 수신(WebSocket 또는 REST 방식 활용)
.

--------------------------------------------------------------------------------
3. Algorithmic Flowchart (알고리즘 플로우차트)
매일 장중에 도는 모니터링 루프와 매월 실행되는 리밸런싱 루프를 명확하게 분리하여 API 과부하를 막고 효율성을 높입니다.
graph TD
    Start([시스템 시작]) --> IsMonthly{매월 1회<br>리밸런싱 일자인가?}
    
    %% Monthly Rebalancing Loop
    subgraph Monthly_Loop [매월 Monthly 리밸런싱 및 종목 발굴 루프]
        IsMonthly -- Yes --> DART_Call[DART API 호출 & 뉴스 크롤링]
        DART_Call --> FilterUniverse[원전/반도체 테마 + 영업이익 흑자 필터링]
        FilterUniverse --> EvalHoldings[기존 보유 종목 재평가<br>펀더멘털 및 이평선 훼손 여부]
        EvalHoldings --> SellInvalid[조건 미달 보유 종목 전량 매도]
        SellInvalid --> UpdateWatchlist[신규 투자 유니버스 확정 및 DB 저장]
    end

    %% Daily Intraday Loop
    subgraph Daily_Loop [매일 Daily 장중 모니터링 루프]
        IsMonthly -- No --> FetchIntraday[실시간 현재가 및 120MA 조회]
        UpdateWatchlist --> FetchIntraday
        
        FetchIntraday --> CheckStopLoss{보유종목 수익률<br><= -10%?}
        
        CheckStopLoss -- Yes --> ExecStopLoss[즉각 시장가 손절 매도]
        CheckStopLoss -- No --> CheckBuyCond{유니버스 내 종목 중<br>120MA 상향 돌파 발생?}
        
        CheckBuyCond -- Yes --> ExecBuy[해당 종목 매수]
        CheckBuyCond -- No --> Hold[대기 및 모니터링 지속]
    end

    %% Output & Logging
    ExecStopLoss --> LogSheet[Google Sheets API<br>매매 내역 실시간 기록]
    ExecBuy --> LogSheet
    Hold -. 주기적 상태 로깅 .-> LogSheet
    SellInvalid --> LogSheet
    
    LogSheet --> Wait([장 마감 후 대기])

--------------------------------------------------------------------------------
4. Pseudocode & Logic Flow (의사코드 및 상세 로직)
개발 시 즉각 구현이 가능하도록, Python에 가까운 의사코드로 작성합니다.
4.1 월간 리밸런싱 로직 (Monthly Loop)
def monthly_rebalance():
    # 1. 전체 상장사 코드 목록 가져오기
    tickers = get_kospi_kosdaq_tickers()
    
    universe = []
    for ticker in tickers:
        # 2. DART API 연동: 최근 분기/사업보고서 영업이익 확인
        operating_profit = get_dart_operating_profit(ticker)
        
        # 3. 크롤링 기반 테마 확인 (원전, 반도체 키워드 존재 여부)
        is_target_theme = check_theme_keyword(ticker, keywords=["원전", "원자력", "반도체", "DRAM"])
        
        # 4. 필터링 통과 시 유니버스 편입
        if operating_profit > 0 and is_target_theme:
            universe.append(ticker)
            
    # 5. 기존 보유 종목 재평가 (유니버스 이탈 또는 120MA 크게 이탈 시 매도)
    for holding in current_portfolio:
        if holding not in universe or is_downtrend(holding):
            execute_sell(holding, order_type="MARKET")
            
    # 6. 신규 유니버스 업데이트
    save_universe_to_db(universe)
4.2 장중 매매 및 손절 로직 (Daily Loop)
120일 이동평균선 상향 돌파(Golden Cross)는 전일 종가가 120일 이동평균선 아래에 있었으나, 당일 현재가가 120일 이동평균선을 뚫고 올라간 상태로 정의합니다
,
.
def daily_intraday_monitor():
    universe = load_universe_from_db()
    
    while is_market_open():
        for ticker in current_portfolio:
            current_price = get_realtime_price(ticker)
            buy_price = get_average_buy_price(ticker)
            
            # 리스크 관리 (손절): 매수 단가 대비 -10% 이하 도달 시 무조건 즉각 시장가 매도
            profit_rate = (current_price - buy_price) / buy_price
            if profit_rate <= -0.10:
                execute_sell(ticker, order_type="MARKET", reason="STOP_LOSS")
                log_to_google_sheets(ticker, "SELL(Stop-Loss)", current_price, profit_rate)
                
        for ticker in universe:
            if ticker not in current_portfolio:
                # 120일 이동평균선 및 전일/당일 가격 로드
                prev_close = get_previous_close(ticker)
                prev_120MA = calculate_120MA(ticker, day="yesterday")
                current_price = get_realtime_price(ticker)
                current_120MA = calculate_120MA(ticker, day="today")
                
                # 매수 조건 (타이밍): 120일 이평선 상향 돌파 정의
                cross_up = (prev_close < prev_120MA) and (current_price > current_120MA)
                
                if cross_up:
                    execute_buy(ticker, order_type="MARKET")
                    log_to_google_sheets(ticker, "BUY(120MA Cross-up)", current_price, 0.0)
                    
        time.sleep(1) # 과도한 API 호출 방지

--------------------------------------------------------------------------------
5. Exception Handling & Edge Cases (예외 처리)
안정적인 자동매매를 위해 다음과 같은 에러 대응 로직을 반드시 포함해야 합니다.
DART 및 증권사 API Rate Limit (호출 제한) 대응:
문제: 증권사 API(초당 N회 제한) 및 DART API의 일일 한도 초과
,
.
대응 로직: API Request 함수를 래핑하여 에러 코드(예: HTTP 429 Too Many Requests) 반환 시 백오프(Exponential Backoff) 및 time.sleep()을 적용하여 재시도 하도록 구현합니다
. Access Token 갱신 로직을 24시간마다 스케줄러로 구동합니다
.
상장 폐지, 거래 정지, 단기 과열 종목 처리:
문제: 거래 정지된 종목에 주문을 넣으면 시스템이 패닉을 일으키거나 크래시될 수 있습니다.
대응 로직: 매수/매도 전 get_stock_status() 함수를 통해 거래 정지, 관리 종목, 단기 과열 여부를 체크합니다. 상태가 비정상인 경우 로깅만 남기고 해당 턴의 주문은 스킵(pass)합니다.
데이터 무결성 문제:
문제: 액면분할 등으로 인해 과거 주가가 급변하여 120MA 계산에 오류가 생길 수 있습니다
.
대응 로직: 주가 데이터 수집 시 반드시 '수정 주가(Adjusted Price)'를 사용하여 이평선 왜곡을 방지합니다.

--------------------------------------------------------------------------------
6. Output & Monitoring (출력 및 모니터링)
자동매매 시스템이 수집하는 매매 내역, 포트폴리오 상태, 수익률은 Google Sheets API를 통해 실시간 대시보드 형식으로 전송하여 스마트폰 등에서 즉시 확인할 수 있도록 합니다
,
.
연동 방식: gspread 라이브러리와 Google Cloud의 서비스 계정(Service Account) JSON 키를 사용하여 시트 업데이트
.
Google Sheets 데이터 스키마 (컬럼 정의):
열 이름 (Column Name)
데이터 타입 (Data Type)
설명 (Description)
Timestamp
DATETIME (YYYY-MM-DD HH:MM:SS)
이벤트 발생 및 로깅 시간
Ticker_Code
STRING
6자리 종목 코드
Company_Name
STRING
종목명 (예: 삼성전자)
Action
STRING (Enum)
수행 동작 (BUY, SELL, STOP_LOSS, HOLD)
Price
INTEGER
매수/매도/현재 체결 단가 (KRW)
Quantity
INTEGER
체결 수량
Return_Rate
FLOAT (Percentage)
매도 시 실현 수익률, HOLD 시 미실현 수익률 (%)
Account_Balance
INTEGER
거래 직후의 계좌 잔고 현황
Trigger_Reason
STRING
매매 원인 (예: 120MA Cross-up, Monthly Rebalance, Risk -10% Limit)
이를 통해 Google Sheets 내장 함수 =GOOGLEFINANCE() 등과 결합하여, 수익률 그래프, MDD, 샤프지수 등의 자동 시각화 대시보드를 구축합니다
