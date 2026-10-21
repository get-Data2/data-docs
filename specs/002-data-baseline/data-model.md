# Data Model: 데이터 이해와 기준선 (과제 2)

모든 파일은 보드 로컬 `var/`(git 제외)에 둔다. 시각은 UTC로 저장하고, 평일·시간대 판정과 표시는 KST로 한다.

## 원본 사본 — `var/cache/raw/YYYY-MM.parquet`

| 열 | 형식 | 설명 |
|---|---|---|
| ts | timestamp[ms, UTC] | 기준 시각(과제 1 `measurement.ts`) |
| space_id | category | 수신 시점 공간 |
| device_id | category | |
| item | category | co2·temperature·humidity·battery |
| value | float64 | |
| source | category | chirpstack·xflow·influx |

`var/cache/manifest.json`: 달마다 `{rows, sha256, extracted_at, closed}`. 닫힌 달은 다시 읽지 않는다.

## 품질 표시 — `var/cache/quality/YYYY-MM.parquet` (축 A)

원본 행과 1:1이다(원본은 바꾸지 않음). 열: ts, device_id, item, `quality` ∈ {`ok`, `saturated`, `error`}, `rule` ∈ {A1, A2, A3, A4, ''}.

## 사건 후보 — `var/cache/events.parquet` (축 B)

| 열 | 설명 |
|---|---|
| space_id | |
| kind | B0(의심)·B1·B2·B3 |
| start, end | UTC |
| devices | 근거 센서 목록 |
| evidence | 판정 근거 요약(최대값, 변화율 등) |
| rule_version | 설정 해시 |

## 공간 시계열 — `var/series/space_5min.parquet`

| 열 | 설명 |
|---|---|
| bin | 5분 칸 시작(UTC) |
| space_id, item | |
| value | 센서 칸 평균의 중앙값(축 A `error` 제외, 짧은 결측 보간 후) |
| n_sensors | 그 칸에 값을 낸 센서 수 |
| filled | 보간 칸을 포함하면 True |
| event | 그 칸이 축 B 사건 후보 구간이면 종류(없으면 '') |

긴 결측 칸은 행이 없거나 value가 NaN이다.

## 분할 — `config/analysis.yaml split` + `var/eval/norm_<split>.json`

- 분할: 버전 이름, 학습·검증·시험 [시작, 끝](KST 날짜).
- 정규화 통계: 공간 × 항목의 학습 기간 평균·표준편차와 계산에 쓴 칸 수, 그리고 학습 기간 사본 해시. 시험 기간 데이터는 읽지 않는다(FR-007, SC-002).

## 평가 시점 — `var/eval/eval_points_<split>.parquet` (과제 2~6 공유)

| 열 | 설명 |
|---|---|
| space_id, item | lab·office·meeting × co2·temperature |
| t0 | 예측 시점(5분 칸, UTC) |
| horizon_min | 5·15·30·60 |
| y_true | t0 + horizon 칸의 공간 값(보간 칸 제외) |
| usage_hours | 평일·비공휴일·KST 08~19시면 True |

- 메타데이터(`eval_points_<split>.json`): 분할 버전, 생성 시각, 행 수, SHA256, 설정 해시.
- 규칙: 과제 3~6 모델은 **이 파일의 행만** 예측하고 같은 y_true로 채점한다.

## 기준선 결과 — `var/eval/baseline_<split>.parquet`, `docs/reports/task2/baseline_<split>.md`

평가 시점 행 + `persistence`, `mean_1h`, `yesterday` 예측 열. 집계 표: 공간 × 항목 × 지평 × 방법 MAE(전체/사용 시간대) + 평가 시점 수.

## 일과표 — `config/analysis.yaml schedule`

행 목록: `{space, days: [mon..fri], start: "09:00", end: "18:00", kind: class|lunch}`, 공휴일 제외 플래그.
실제 자료로 바꿀 때는 `schedule.file: data/schedule.csv`(열: space_id, date 또는 weekday, start, end, kind)를 쓴다.
