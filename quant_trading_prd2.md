자동매매 시스템 백테스팅 검증 로직 제품 요구사항 정의서 (PRD)
1. System Overview (시스템 개요 및 목표)
시스템 명: Robust Quant Backtest Validation Pipeline
목표: 금융 시계열 데이터의 **자기 상관성(Autocorrelation)**으로 인한 **정보 누출(Information Leakage)**을 원천 차단하고, 모형 성능이 과도하게 긍정적으로 평가되는 심각한 오류를 방지하기 위한 파이썬 기반 백테스팅 검증 아키텍처 구축.
핵심 원칙:
전통적인 무작위 교차 검증을 배제하고, 시간의 흐름을 엄격히 준수하는 시계열 기반 검증 로직 적용.
과적합(Overfitting) 여부를 정량적으로 모니터링하여, 완전히 독립된 OOS(Out-of-Sample) 구간에서도 시장 변화에 무너지지 않는 강건한 퀀트 시스템 완성.

--------------------------------------------------------------------------------
2. Core Validation Architecture (핵심 검증 아키텍처)
2.1 갭(Gap)이 포함된 시계열 교차 검증 (TSCV)
기반 모듈: sklearn.model_selection.TimeSeriesSplit 활용.
시계열 순서 유지: 훈련 데이터 셋이 검증 데이터 셋보다 시간상 앞선 데이터로 구성되도록 순서를 고정하며, 훈련 셋을 점진적으로 확장(Expanding Window)하는 방식으로 진행.
블라인드 구간(Gap) 부여: 가장 핵심적인 방어 로직으로, Train 셋의 종료일과 Test 셋의 시작일 사이에 의도적으로 일정 기간의 공백(Gap/Buffer)을 삽입하여 자기 상관성에 의한 미래 참조 왜곡을 방지.
2.2 OOS(Out-of-Sample) 전진 분석 (Walk-Forward Analysis)
독립 검증: TSCV 과정에서 특정 기간에 하이퍼파라미터가 과최적화되는 현상을 막기 위해, 학습 및 교차 검증 파이프라인에 전혀 노출되지 않은 독립적인 미래 구간을 '전진 분석 데이터'로 별도 분리하여 최종 백테스트에 활용.
일반화 성능 평가: 이를 통해 모형이 학습에 사용되지 않은 완전히 새로운 시장 상황에서도 안정적으로 작동하는지, 즉 일반화 성능(Generalization Performance)을 정밀하게 평가합니다
.

--------------------------------------------------------------------------------
3. Algorithmic Flowchart (검증 파이프라인 플로우차트)
graph TD
    Data[금융 시계열 데이터 로드] --> SplitOOS{OOS 데이터 분리}
    SplitOOS -- Train/Validation 용도 --> TSCV[TimeSeriesSplit 기반 분할]
    SplitOOS -- 최종 테스트 용도 --> OOS[OOS 전진 분석 데이터 보관]
    
    subgraph Cross_Validation_Loop [갭(Gap) 포함 시계열 교차 검증]
        TSCV --> AddGap[Train 종료일과 Test 시작일 사이 Gap 부여]
        AddGap --> TrainModel[모델 학습 진행]
        TrainModel --> EvalModel[Validation 셋 기반 평가]
        EvalModel --> CalcMetrics[과적합 정량 평가 지표 산출]
    end
    
    CalcMetrics --> IsOverfit{과적합 편차/비율<br>임계치 통과 여부}
    
    IsOverfit -- Fail --> Tune[하이퍼파라미터 조정 및 재학습]
    Tune --> TSCV
    
    IsOverfit -- Pass --> WalkForward[OOS 전진 분석 데이터로 최종 검증]
    WalkForward --> FinalEval[최종 일반화 성능 및 수익률 평가]
    FinalEval --> Deploy([실전 자동매매 시스템 배포])

--------------------------------------------------------------------------------
4. Quantitative Evaluation Metrics (과적합 정량 평가 지표)
백테스트 검증 모듈 내에 학습 성과와 검증 성과의 차이를 계량화하는 함수를 구현하여, 모형의 안정성을 시스템적으로 판단하도록 로직을 설계합니다
.
과적합 편차 (Overfitting Gap):
공식: 학습 구간 연간 실현 수익률(ARP_Train) - 검증 구간 연간 실현 수익률(ARP_Validation)
.
목적: 검증 구간에서의 성과 하락폭이 비정상적으로 크지 않은지 절대적인 수치로 확인.
과적합률 (Overfitting Rate):
공식: (ARP_Train - ARP_Validation) / ARP_Train
.
목적: 학습 구간 대비 검증 구간의 성과 하락을 비율로 파악하여, 과도하게 높은 불균형 상태(시장 환경 등 외부 요인에 의한 왜곡 가능성)를 모니터링
.

--------------------------------------------------------------------------------
5. Pseudocode & Logic Flow (의사코드 및 상세 로직)
개발자가 즉시 파이썬 환경에서 구현할 수 있도록 커스텀 TSCV와 평가 로직을 정의합니다.
import numpy as np
from sklearn.model_selection import TimeSeriesSplit

class GapTimeSeriesSplit(TimeSeriesSplit):
    """
    Train 셋과 Test 셋 사이에 의도적인 버퍼(Gap)를 두어 정보 누출을 방지하는 교차 검증 클래스
    """
    def __init__(self, n_splits=5, gap_size=10):
        super().__init__(n_splits=n_splits)
        self.gap_size = gap_size

    def split(self, X, y=None, groups=None):
        for train_index, test_index in super().split(X, y, groups):
            # Train 데이터의 끝에서 gap_size 만큼 제외하여 자기상관성 차단
            train_end = train_index[-1] - self.gap_size
            if train_end <= train_index:
                continue # Gap이 너무 커서 Train 데이터가 없어지는 경우 방지
                
            purged_train_index = train_index[train_index <= train_end]
            yield purged_train_index, test_index

def evaluate_overfitting(arp_train, arp_validation):
    """
    학습 성과와 검증 성과의 차이를 계량화하여 과적합 정량 지표를 산출
    """
    # 과적합 편차 산출 [1]
    overfitting_gap = arp_train - arp_validation
    
    # 과적합률 산출 [1, 3]
    overfitting_rate = (arp_train - arp_validation) / arp_train
    
    return overfitting_gap, overfitting_rate

def backtest_pipeline(data, model):
    # 1. OOS 데이터 분리 (Walk-Forward Analysis용)
    train_val_data, oos_data = split_out_of_sample(data)
    
    # 2. 갭이 포함된 교차 검증 (TSCV with Gap)
    tscv = GapTimeSeriesSplit(n_splits=5, gap_size=10) # 10 거래일 블라인드 구간 부여
    
    for train_idx, val_idx in tscv.split(train_val_data):
        train_set, val_set = train_val_data[train_idx], train_val_data[val_idx]
        
        model.fit(train_set)
        arp_train = calculate_annual_realized_profit(model, train_set)
        arp_val = calculate_annual_realized_profit(model, val_set)
        
        # 3. 과적합 정량 지표 모니터링
        gap, rate = evaluate_overfitting(arp_train, arp_val)
        
        if gap > THRESHOLD_GAP or rate > THRESHOLD_RATE:
            raise OverfittingError("과적합 편차/비율이 허용 임계치를 초과했습니다.")
            
    # 4. 최종 OOS 전진 분석 (Walk-Forward)
    final_performance = walk_forward_test(model, oos_data)
    return final_performance

--------------------------------------------------------------------------------
6. Advanced Strategy (고도화: 조합적 제거 교차검증)
시스템을 향후 더욱 고도화하기 위해 조합적 제거 교차검증(Combinatorial Purged Cross Validation, CPCV) 도입을 검토해야 합니다.
Purging (강제 제거): 훈련 데이터 세트에서 테스트 세트와 겹칠 우려가 있거나 정보 누출이 의심되는 기간을 알고리즘적으로 강제 제거합니다.
경로 대폭 확대: 남은 데이터 블록들을 단순히 선형적으로 검증하는 것을 넘어 다수의 조합(Combinations)으로 엮어 시뮬레이션 경로를 기하급수적으로 늘립니다.
기대 효과: 백테스팅 과정에서 발생할 수 있는 과최적화(Overfitting) 위험을 근본적으로 크게 줄일 수 있는 가장 발전된 금융 검증 로직으로 기능합니다.
