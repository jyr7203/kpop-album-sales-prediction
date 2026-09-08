# 유튜브 티저 데이터 수집 자동화 (n8n)

피처 데이터 중, **앨범별 유튜브 티저 영상의 조회수 / 댓글 수 / 좋아요 수**를 수집하는 과정을
n8n 워크플로우로 자동화하였습니다.


## 배경 및 목적

- 대상 데이터: `그룹명` 컬럼에 있는 39개 그룹 전체의 앨범 중 **2019.12 ~ 2026.06**에
  발매된 앨범 (총 252건, `data/input.csv` 참고)
- 목적: 각 앨범의 공식 티저 영상을 찾아 조회수 / 댓글 수 / 좋아요 수를
  자동으로 수집하여, 초동 판매량 예측 모델의 입력 피처로 사용

## 워크플로우 구조

```
수동 실행
  └─ 입력 CSV (로컬 데이터) 읽기
      └─ CSV 파싱
          └─ 앨범 발매일 필터 (2019-12-01~2026-06-30)
              └─ 1건씩 처리 (Loop Over Items)
                  ├─ 유튜브 티저 영상 검색 (YouTube Data API search.list)
                  │    └─ 가장 적합한 영상 선택 (제목에 teaser/티저 포함 + 발매일과
                  │         가장 가까운 공개일 기준으로 매칭, Code 노드)
                  │        └─ 영상 발견
                  │             ├─ Yes → 유튜브 통계 조회 (videos.list statistics)
                  │             │           └─ 조회수/좋아요/댓글수 정리
                  │             └─ No  → 영상 없음 처리 (결측값 처리)
                  │                        └─ 호출 간격 대기 (API 쿼터 보호) → 다음 앨범으로 반복
                  └─ (전체 처리 완료) → 결과 CSV 변환 → 결과 CSV 저장
```

두 가지 버전으로 제공되며, 예시 데이터는 로컬 csv 파일로 입출력되는 버전으로 진행되었습니다.

- [`workflow/fnc_youtube_teaser_collector_csv.json`](workflow/fnc_youtube_teaser_collector_csv.json) — **로컬 CSV 파일**로 입출력하는 버전
- [`workflow/fnc_youtube_teaser_collector.json`](workflow/fnc_youtube_teaser_collector.json) — 입출력을 **Google Sheets**로 하는 버전 (구글 계정 OAuth 연결 필요)

## 핵심 로직

1. **입력**: 로컬 CSV 읽어서 파싱
2. **필터링**: 발매일이 2019-12-01 ~ 2026-06-30 범위인 행만 통과
3. **영상 검색**: `{그룹명} {앨범명} teaser`로 YouTube 검색 API를 호출하고,
   `publishedBefore`를 발매일 기준 ISO 8601 형식(`YYYY-MM-DDT00:00:00.000Z`)으로
   제한해 실제 발매 전 공개된 티저만 대상으로 함
4. **영상 선택**: 검색 결과 중 제목에 `teaser`/`티저`가 포함된 영상을 우선
   후보로 삼고, 그중 발매일과 공개일 차이가 가장 작은 영상 선택
5. **통계 수집**: 선택된 영상의 `viewCount`, `likeCount`, `commentCount`를
   `videos.list`로 조회 (좋아요/댓글이 비공개인 영상은 null 처리)
6. **결과 저장**: 전체 앨범 처리가 끝난 뒤 결과를 한 번에 CSV로 저장
   (워크플로우 진행 중 오류 발생 시 진행된 과정까지의 결과는 저장되지 않으므로,
   배치를 API 일일 쿼터 제한 하에 나눠 수집해야 합니다)
8. **속도 제어**: 각 앨범 처리 후 Wait 노드로 대기하여 rate limit 피함

## 사용 방법 (로컬 CSV 버전)

1. n8n에서 `workflow/fnc_youtube_teaser_collector_csv.json` Import
2. `data/input.csv`를 `~/.n8n-files/input.csv` 위치에 저장 (또는 "입력 CSV
   (로컬 데이터) 읽기" 노드의 `fileSelector` 값을 원하는 경로로 수정)
3. **HTTP Query Auth** Credential 연결 (Name: `key`, Value: YouTube Data API v3 키) —
   "유튜브 티저 영상 검색", "유튜브 통계 조회" 두 노드에 동일하게 연결
4. "결과 CSV 저장" 노드의 저장 경로도 `~/.n8n-files/output.csv`로 확인
5. 하루 처리량을 80건 이내로 맞춰 `input.csv`를 나눠 여러 날에 걸쳐 실행하여 수집

## 사용 기술

- [n8n](https://n8n.io/) — 워크플로우 오케스트레이션 (로컬 self-hosted, Node.js 20 LTS)
- YouTube Data API v3 (`search.list`, `videos.list`)
