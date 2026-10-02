# ESS 배터리 수명 예측

초기 충·방전 데이터로 리튬이온 배터리의 최종 수명(`cycle_life`)을 예측하는 회귀 프로젝트입니다. `cycle_life`는 방전용량이 초기 기준의 80%에 도달할 때까지의 사이클 수로, ESS의 점검·교체 시점을 계획하는 기준으로 해석합니다.

## 프로젝트 개요

- **데이터셋:** MIT-Stanford Battery Dataset 계열의 과제 제공 `.mat` 파일
- **학습 데이터:** Batch 1 (`2017-05-12`)
- **최종 평가 데이터:** Batch 2 (`2018-02-20`)
- **추가 분석 데이터:** Batch 3 (`2018-04-12`)
- **태스크:** Regression — Cycle Life 예측
- **예측 시점:** 초기 100사이클 이내

ESS 배터리의 예상보다 이른 수명 종료는 교체 비용과 전력 공급 안정성에 영향을 준다. 초기 100사이클 안에서 수명 신호를 발견해 조기 불량 셀을 선별하고 예방 교체 계획을 지원하는 것이 목표다.

## 데이터 사용 기준

과제에서 제공한 Batch 1·2·3 파일을 그대로 사용하고 `cycle_life`가 없는 셀만 지도학습에서 제외했다. 분석 표본은 Batch 1 46개, Batch 2 39개, Batch 3 44개다. 원논문의 41·43·40개 전처리 구성을 재현한 실험이 아니므로 원논문 MAPE 9.1%는 참고 목표로만 비교한다.

| 데이터 | Day 1 | Day 2 |
|---|---|---|
| Batch 1 | EDA 및 배치 비교 | 학습, 정책 그룹 CV, 정책 그룹 Hold-out |
| Batch 2 | EDA 및 배치 비교 | 최종 테스트 1회 |
| Batch 3 | EDA 및 배치 비교 | 모델 성능 평가 미수행(선택사항) |
| `2018-04-03 varcharge` | 제외 | 별도 충전 최적화 실험 데이터 |

원본 `.mat` 파일은 용량이 크므로 Git 저장소에 포함하지 않는다.

## 파일 구조

```text
.
├── archive/                         # 원본 데이터, Git 제외
├── figures/day2/
│   ├── 01_model_selection.png
│   ├── 02_batch2_predictions.png
│   └── 03_batch_shift.png
├── results/
│   ├── model_performance.csv
│   ├── candidate_model_results.csv
│   ├── batch2_predictions.csv
│   ├── batch2_error_summary.csv
│   └── batch_shift_summary.csv
├── 01_EDA.ipynb
├── 02_Modeling.ipynb
├── requirements.txt
└── README.md
```

## 환경 설정

```bash
git clone https://github.com/RoyaPark12/ess-battery-cycle-life.git
cd ess-battery-cycle-life
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook 01_EDA.ipynb 02_Modeling.ipynb
```

실행 전 프로젝트 루트에 `archive/` 폴더를 만들고 다음 과제 제공 파일을 배치한다.

```text
archive/
├── 2017-05-12_batchdata_updated_struct_errorcorrect.mat
├── 2018-02-20_batchdata_updated_struct_errorcorrect.mat
└── 2018-04-12_batchdata_updated_struct_errorcorrect.mat
```

## EDA

### Cycle Life 분포

| Batch | 유효 셀 | 중앙값 | 단수명 `<500` | 장수명 `>1,000` |
|---|---:|---:|---:|---:|
| Batch 1 | 46 | 858.5 | 0개 (0.0%) | 10개 (21.7%) |
| Batch 2 | 39 | 472.0 | 28개 (71.8%) | 3개 (7.7%) |
| Batch 3 | 44 | 1,005.5 | 0개 (0.0%) | 23개 (52.3%) |

**핵심 발견:** Batch 2는 단수명 셀이 집중되어 있고 Batch 3은 장수명 셀이 많다. 배치별 목표분포 차이가 커서 Batch 1 내부 성능만으로 Batch 2 일반화를 보장할 수 없다.

### 열화곡선과 Knee Point

방전용량은 처음부터 일정한 속도로 감소하지 않고, 초기에는 완만하다가 Knee Point 이후 감소 속도가 커졌다. 전체 셀의 Knee Cycle 중앙값은 Batch 1 573.5회, Batch 2 317.0회, Batch 3 764.5회였다.

**핵심 발견:** Knee가 늦을수록 Cycle Life도 긴 경향이 있지만 전체 미래곡선이 필요하므로 초기 예측 피처에서는 제외했다.

### ΔQ(V) 곡선

Cycle 100 Q(V)와 Cycle 10 Q(V)의 차이 분산에 `log10`을 적용했다. 이 값과 Cycle Life의 Spearman 상관계수는 Batch 1 -0.871, Batch 2 -0.709, Batch 3 -0.797이었다.

**핵심 발견:** 세 배치에서 방향과 강도가 일관된 음의 관계를 보여 최종 모델의 핵심 피처로 선택했다.

### 충전 속도와 수명

C-rate와 Cycle Life의 Spearman 상관계수는 Batch 1 -0.634, Batch 2 +0.321, Batch 3 -0.132였다. Batch 2의 관계는 `p=0.0465`로 유의하지만 C-rate 범위가 약 0.017C에 불과해 충전 정책 자체보다 다른 조건이 반영됐을 가능성이 크다.

**핵심 발견:** C-rate는 Batch 1에서는 유효하지만 배치 전반에서 관계가 일관되지 않아 보조 후보로만 비교했다.

### 추가 데이터 품질 확인

Batch 2의 내부저항 0값은 4,286/22,064행(19.43%)이었다. 0을 실제 저항으로 해석하지 않았으며, 배치별 수명 관계도 일관되지 않아 초기 핵심 피처에서 제외했다.

## Modeling

### 피처 엔지니어링 전략

초기 100사이클 이내에서 계산할 수 있는 값만 사용했다.

| 구성 | 피처 | 목적 |
|---|---|---|
| A | `log10_deltaq_variance` | 세 배치에 공통인 핵심 신호 |
| B | A + `effective_c_rate_to_80pct` | 충전 조건의 추가 효과 확인 |
| C | A + `mean_Tavg_first100` | 온도 정보의 추가 효과 확인 |
| D | A + C-rate + 온도 | 전체 조합의 과적합 여부 확인 |

`cell_id`는 식별자, `charging_policy`는 분할 그룹, `batch`는 데이터 구분에만 사용했다. Knee Point는 미래정보 누출을 막기 위해 제외했다.

### 데이터 분할과 검증

- **Train:** Batch 1의 35개 셀, 18개 충전 정책
- **Cross-Validation:** Train 영역에서 정책 그룹 기반 5-fold `GroupKFold`
- **Valid:** Batch 1의 11개 셀, 5개 충전 정책
- **Test:** 모델과 피처를 확정한 뒤 Batch 2에서 1회 평가
- **정책 중복:** Train과 Valid 사이 0개

Train과 Valid의 Cycle Life 중앙값은 각각 857회와 860회다. 같은 충전 정책이 양쪽에 섞이지 않도록 분리해 정책 정보 누출을 막았다.

### 후보 모델과 최종 선택

Dummy Regressor, Linear Regression, Ridge, ElasticNet, Random Forest, Gradient Boosting을 비교했다. 최종 모델은 **`log10_deltaq_variance` 단독 Ridge Regression**이다.

Gradient Boosting은 Batch 1 Hold-out MAPE가 더 낮았지만 교차검증 편차와 CV-Valid 차이가 컸다. C-rate를 포함한 조합은 Batch 2에서 관측 범위가 거의 사라져 배치 이동에 취약하다. Ridge 단독 모델은 CV 8.72%, Hold-out 9.77%, Gap 1.05%p로 안정적이고 구조가 단순해 최종 선택했다.

## 성능 결과

| 구분 | MAPE 또는 Gap | 해석 |
|---|---:|---|
| Train (Batch 1 CV) | 8.719% | 정책 그룹 기반 5-fold 평균 |
| Valid (Batch 1 Hold-out) | 9.768% | 학습과 정책이 겹치지 않는 검증 셀 |
| Test (Batch 2) | 38.392% | 최종 모델 고정 후 1회 평가 |
| Gap (Train-Valid) | +1.049%p | Batch 1 내부 과적합은 크지 않음 |
| Gap (Valid-Test) | +28.624%p | 배치 간 일반화 성능 저하 |
| Gap (Target-Test) | +29.292%p | 원논문 참고 목표 9.1% 대비 차이 |

Batch 2 보조지표는 MAE 186.53사이클, RMSE 197.52사이클, R² 0.189다. 세부 결과는 `results/model_performance.csv`와 `results/batch2_predictions.csv`에 저장했다.

## 오류 분석

Batch 2 셀의 89.74%를 실제보다 높게 예측했고 평균 과대 예측은 162.75사이클이었다. 오차가 큰 셀은 실제 수명이 약 392~449회인 단수명 셀에 집중되었다. Batch 1 수명 중앙값은 858.5회지만 Batch 2는 472.0회이므로, Batch 1 관계를 학습한 모델이 Batch 2의 짧은 수명을 충분히 낮게 예측하지 못했다.

**원인 가설:** 배치별 목표분포와 ΔQ(V) 분포가 이동했고, 초기 열화를 설명하는 변수를 ΔQ(V) 하나로 요약하면서 Batch 2 특유의 단수명 원인을 놓쳤다.

**개선 방향:** 초기 용량, 충전 시간, 온도 변화량 등의 피처를 사전에 정의하고 배치 보정을 적용한 뒤 새로운 외부 배치에서 평가한다. Batch 2 결과를 반복해서 보며 모델을 다시 선택하지 않는다.

## ESS 도메인 해석

현재 모델은 초기 100사이클에서 열화 위험이 큰 셀을 선별하고 추가 점검 대상을 정하는 보조 지표로 활용할 수 있다. 실제 BESS 운영에서는 셀별 예상 수명과 불확실성을 함께 제시해 점검 우선순위, 예방 정비, 교체 재고 계획에 연결할 수 있다.

Batch 2 일반화 성능이 충분하지 않으므로 자동 교체 결정에는 사용할 수 없다. 실 배포를 위해서는 다양한 제조 로트·운영 온도·충전 정책 데이터, 센서 결측 처리, 예측 불확실성, 시간에 따른 성능 감시와 재학습 기준이 추가로 필요하다.

## 최종 결론

Day 1 EDA에서 세 배치에 공통으로 수명과 강한 음의 관계를 보인 ΔQ(V) 분산을 핵심 피처로 선정하였다. Batch 1의 후보 조합을 비교한 결과, ΔQ(V) 분산 단독 Ridge 모델은 교차검증 MAPE 8.72%, Hold-out MAPE 9.77%를 기록했으며 Train–Valid Gap은 1.05%p로 작아 Batch 1 내부에서 뚜렷한 과적합은 확인되지 않았다.

Batch 2 Test MAPE는 38.39%로 증가했고, 셀의 89.74%를 실제보다 길게 예측하였다. Batch 1에는 500회 미만의 단수명 셀이 없지만 Batch 2에는 28개(71.8%)가 포함되어 있어, 학습 데이터에서 경험하지 못한 단수명 영역에 대한 일반화 성능이 저하된 것으로 판단된다.

ΔQ(V) 분산은 초기 열화 위험을 판단하는 유효한 신호지만 현재 모델을 실제 ESS 교체 시점 결정에 바로 사용하기는 어렵다. 향후에는 단수명 셀이 포함된 학습 데이터와 추가 초기 피처를 확보하고, 배치 차이를 반영한 보정과 새로운 외부 데이터 검증이 필요하다. 원논문 MAPE 9.1%는 데이터와 전처리 구성이 다르므로 참고 목표로 해석한다.

## 참고문헌

Severson, K. A. et al. (2019). Data-driven prediction of battery cycle life before capacity degradation. *Nature Energy*, 4, 383–391.

## 팀 구성

- 박소정: EDA, 피처 엔지니어링, 모델 개발, Batch 2 성능 평가, 오류 분석 및 문서화
