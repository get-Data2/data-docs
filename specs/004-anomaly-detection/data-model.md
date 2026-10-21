# Data Model: 이상 탐지 (과제 4)

## 입력(과제 2 산출, 읽기만)

| 파일 | 쓰는 열 | 규칙 |
|---|---|---|
| `var/series/sensor_5min.parquet` | device_id, space_id, item, bin, value, filled | 관측(`filled=False`)만 사용 |
| `var/cache/events.parquet` | kind, space_id, start, end | 학습 제외 구간(B0), 헛경보 중 실제 이상 후보 판별(B0·B3) |
| `var/cache/stuck.parquet` | space_id, device_id, item, start, end | 학습 제외 구간(값 고정 후보) |
| `config/analysis.yaml` | split, schedule, baseline.usage_hours | 분할·일과표·사용 시간대 |
| `var/forecast/bundle/` | model.pt, bundle.json | 심화 1 |

## 센서 넓은 표 (`data.py`)

공간마다 5분 칸 × 열(장치, 항목)의 표를 만든다.

| 필드 | 형 | 설명 |
|---|---|---|
| `t` | (N,) UTC | 칸 |
| `cols` | [(device_id, item)] | 열 순서(설정의 공간 대상 장치, 정렬) |
| `x` | float32 (N, C) | 관측값, 빈 칸 NaN |
| `stats` | (C, 4) | 학습 기간 평균·표준편차·이웃 차이 MAD·값 범위(5~95%) |

## 고장(정답) — `var/anomaly/faults.parquet`

| 열 | 형 | 설명 |
|---|---|---|
| fault_id | int | 1부터 |
| kind | F1~F5 | 종류 |
| space_id | str | 공간 |
| device_id | str · NaN | 센서 고장(F1~F3)만 |
| items | str | 영향 항목(쉼표) |
| start, end | UTC | 주입 구간 |
| match_end | UTC | 탐지 인정 끝(end + 30분, F3은 start + 24시간) |
| magnitude | float | 크기(ppm·℃·배율) |
| seed | int | 주입 씨앗 |

검증
- 같은 공간 고장 사이 간격 ≥ 12시간
- F4·F5 시작은 사용 시간대 수업일
- 구간이 시험 기간 안

## 평가판 — `var/anomaly/board.parquet`

`sensor_5min`의 시험 기간 행에 열 `value_injected`, `fault_id`(없으면 0)를 더한다. `fault_id == 0`인 행의 `value_injected`는 `value`와 같다(자동 시험).

## 점수 — `var/anomaly/scores/<탐지기>__<기간>.parquet`

| 열 | 설명 |
|---|---|
| detector | `ae` · `rule_spike` · `rule_stuck` · `forecast_err` · `drift` |
| period | `val` · `test` |
| space_id, t | 공간, 칸 |
| score | 공간 점수(탐지기 원 단위) |
| q | 그 공간 검증 기간 점수 분위수(0~1) |
| cause1~3 | 기여 상위 (device_id/item) |

규칙 탐지기는 0/1 점수와 원인 장치를 낸다(q = score).

## 임계값 — `var/anomaly/thresholds.json`

| 필드 | 설명 |
|---|---|
| detector → {threshold_q, val_alarms_per_day, candidates[]} | 후보별 검증 하루당 경보, 고른 값과 이유 |
| target_per_day | 설정 목표(기본 0.5) |
| inputs_sha | 검증 점수 파일 해시 |
| frozen_at, sha256 | 고정 시각·해시 |

규칙: `evaluate`는 이 파일이 없거나 해시가 맞지 않으면 거부한다. 임계값은 검증 점수에서만 계산한다(평가판을 바꿔도 같음 — 자동 시험).

## 경보 — `var/anomaly/alarms/<탐지기>.parquet`

| 열 | 설명 |
|---|---|
| detector | 탐지기(결합은 `combined`) |
| space_id, start, end | 경보 구간(30분 안 병합) |
| peak_q | 최고 분위수 |
| cause | 1순위 원인 (device_id/item) |
| fault_id | 매칭된 고장(없으면 0 = 헛경보) |
| delay_min | 매칭 시 지연 |
| real_event | 헛경보 중 과제 2 사건 후보(B0·B3)와 겹치는지 |

## 결과 — `var/anomaly/results.json`

탐지기 × 고장 종류: 건수, 잡은 수, 탐지율, 지연(중앙·최대), 원인 정확도. 탐지기별 하루당 헛경보(전체·실제 이상 후보 제외). 확인 기준 판정. 임계값 곡선(후보별 탐지율·헛경보).

## 탐지기 묶음 — `var/anomaly/bundle/`

| 파일 | 내용 |
|---|---|
| `ae_<공간>.pt` | 오토인코더 가중치 |
| `bundle.json` | 열 순서, 센서 정규화 통계, 창 길이, 검증 분위수 표(점수 → q), 임계값, 규칙 파라미터, 알림 대상 표, 데이터 해시 |
