# 앨범 초동 판매량 예측 프로젝트 (K-pop Album First-Week Sales Prediction)

직접 수집, 구축한 253개 앨범 × 27개 피처(39개 그룹) 데이터셋을 기반으로 
K-pop 아이돌/밴드/솔로 아티스트 앨범의 **첫 주(초동) 판매량**을 CatBoost 모델을 통해 예측합니다.

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
- 그룹 SNS 지표(Spotify/TikTok/Instagram/유튜브 구독자),
  유튜브 티저 조회수/좋아요/댓글 수(YouTube Data API v3 이용),
  이전 컴백 이력(콘서트, 뮤직비디오, 빌보드 순위, 직전 초동 판매량 등),
  앨범 메타데이터(트랙 수, 발매월, 앨범 타입 등)로 구성

## 2. EDA — [`eda.ipynb`](eda.ipynb)

- 결측치 구조 및 변수별 타겟(초동 판매량) 분포 및 패턴 분석
- SNS 지표 간 다중공선성 진단
  
## 3. 모델링 — [`modeling.ipynb`](modeling.ipynb)

1. **실험 비교**: 결측 대체 방식(원본/의미적 결측 대체/PCA) × fold 수(5/10)를 조합한 CatBoost
   회귀 실험. 각 조합마다 Optuna로 하이퍼파라미터를 튜닝하고 K-Fold CV로 RMSE/MAE/R²를
   평균 ± 표준오차로 비교.
2. **최종 모델**: CatBoost 네이티브 결측 처리하여 전체 데이터에 대한 학습 진행.
   Feature Importance / Partial Dependence Plot / SHAP으로 모델 해석.


## 4. 데이터 수집 자동화 — [`data_collection/`](data_collection/)

앨범별 유튜브 티저 영상의 조회수/댓글 수/좋아요 수를 직접 수집했던 과정을 n8n 워크플로우로 자동화하였습니다. 
YouTube Data API v3로 그룹명, 앨범명, 발매일이 주어지면 
공식 티저 영상을 검색해 가장 적합한 영상을 선택하고 통계를 수집합니다. 
자세한 내용과 실행 방법은 [`data_collection/README.md`](data_collection/README.md)를 참고하세요.

## 사용 기술

- Python (pandas, numpy, scikit-learn, statsmodels)
- CatBoost, Optuna(하이퍼파라미터 튜닝), SHAP
- n8n, YouTube Data API v3
