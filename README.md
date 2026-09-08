# 앨범 초동 판매량 예측 프로젝트 (K-pop Album First-Week Sales Prediction)

K-pop 아이돌/밴드/솔로 아티스트 앨범의 **첫 주(초동) 판매량**을 예측하는 프로젝트입니다.
직접 수집·구축한 253개 앨범 × 27개 피처(39개 그룹) 데이터셋을 기반으로 EDA와 CatBoost
회귀 모델링을 진행했습니다.

## 구성

```
.
├── data/
│   └── fnc_final_target.xlsx     # 최종 데이터셋 (253개 앨범 × 27개 피처, 39개 그룹)
├── eda.ipynb                     # 탐색적 데이터 분석 (EDA)
├── modeling.ipynb                # CatBoost 모델링 (실험 비교 + 최종 모델 + SHAP/PDP 해석)
└── data_collection/               # 데이터 수집 자동화 (n8n)
    ├── README.md
    ├── workflow/
    ├── data/
    └── .env.example
```

## 1. 데이터셋

- 253개 앨범(2019.10 ~ 2026.06 발매), 39개 그룹 대상
- 그룹 SNS 지표(Spotify/TikTok/Instagram/유튜브 구독자), 유튜브 티저 조회수·좋아요·댓글 수,
  이전 컴백 이력(콘서트, 뮤직비디오, 빌보드 순위, 직전 초동 판매량 등), 앨범 메타데이터(트랙 수,
  발매월, 앨범 타입 등)로 구성
- 이 중 **유튜브 티저 조회수 / 댓글 수 / 좋아요 수**는 원래 앨범마다 수작업으로 유튜브에서
  검색해 기록했던 항목이며, 이 과정을 [`data_collection/`](data_collection/)에서 n8n +
  YouTube Data API v3로 자동화했습니다.

## 2. EDA — [`eda.ipynb`](eda.ipynb)

- 결측치 구조 파악 (구조적 결측 vs 임의 결측)
- 타겟(실제 초동 판매량) 분포 및 로그 변환
- 범주형 변수(국내외, 앨범 종류, 성별, 그룹 타입)별 판매량 비교
- 수치형 변수와 타겟의 스피어만 상관관계
- SNS 지표 간 다중공선성(VIF) 진단
- 발매 시점(월/연차), 트랙 수, 콘서트 이력·컴백 주기에 따른 판매량 패턴 분석

## 3. 모델링 — [`modeling.ipynb`](modeling.ipynb)

두 단계로 구성되어 있습니다.

1. **실험 비교**: 결측 대체 방식(원본/의미적 대체/PCA) × fold 수(5/10)를 조합한 CatBoost
   회귀 실험. 각 조합마다 Optuna로 하이퍼파라미터를 튜닝하고 K-Fold CV로 RMSE/MAE/R²를
   평균 ± 표준편차로 비교.
2. **최종 모델**: 원본(대체 없음) 변형 + CatBoost 네이티브 결측 처리 조합의 튜닝된
   하이퍼파라미터로 전체 데이터를 학습. RepeatedKFold(10-fold × 10회 반복)로 표준오차와
   95% 신뢰구간을 계산하고, Feature Importance / Partial Dependence Plot / SHAP
   (beeswarm, bar)로 모델을 해석.

> 원래 실험 비교는 로컬 노트북, 최종 모델 해석은 Google Colab에서 별도로 작성했던
> 두 노트북(`modeling.ipynb`, `catboost_final_model.ipynb`)을 이 저장소에서 하나로
> 병합했습니다. Colab 전용 코드(Google Drive 마운트, 파일 업로드, `apt-get`을 통한
> 한글 폰트 설치 등)와 두 노트북에서 중복되던 설정 코드는 제거하고, 최종 모델 학습에
> 사용한 하이퍼파라미터(원래는 Drive의 `best_numbers.json`에서 로드)는 재현성을 위해
> 코드에 직접 명시했습니다.

## 4. 데이터 수집 자동화 — [`data_collection/`](data_collection/)

앨범별 유튜브 티저 영상의 조회수/댓글 수/좋아요 수를 수집하는, 원래 수작업이었던 과정을
n8n 워크플로우로 자동화한 결과물입니다. 그룹명·앨범명·발매일만 주어지면 YouTube Data
API v3로 공식 티저 영상을 검색해 가장 적합한 영상을 선택하고 통계를 수집합니다. 자세한
내용과 실행 방법은 [`data_collection/README.md`](data_collection/README.md)를 참고하세요.

## 사용 기술

- Python (pandas, numpy, scikit-learn, statsmodels)
- CatBoost, Optuna(하이퍼파라미터 튜닝), SHAP
- n8n, YouTube Data API v3
