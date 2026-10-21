# Data Model: 상태 추정 (과제 5)

## 입력(읽기만)

| 파일 | 쓰는 열 | 규칙 |
|---|---|---|
| `var/series/space_5min.parquet` | space_id, item, bin, value, filled | 관측(`filled=False`)만. 실시간 구간은 `analysis extract` 뒤 |
| `var/eval/norm_v1-provisional.json` | spaces.<공간>.<항목>.mean/std | 특징 정규화(처음 보는 공간 실험은 합친 통계) |
| `config/analysis.yaml` | split, schedule, baseline.usage_hours | 분할·일과표 |
| `var/forecast/booking_estimated.parquet` | space_id, start, end | 회의실 일정 기반 정답(추정) |
| `var/forecast/bundle/` | model.pt, bundle.json | 과제 3 예측 기반 환기 판정 |
| `data/observations.csv` | 아래 | 직접 관찰(평가 전용) |

## 관찰 기록 — `data/observations.csv`

| 열 | 형 | 규칙 |
|---|---|---|
| space_id | str | 설정의 공간(lab, office, meeting) |
| start, end | KST `YYYY-MM-DD HH:MM` | end > start. 열린 구간은 end 빈칸(명령이 닫음) |
| state | occupied · empty | |
| people | int · 빈칸 | 선택 |
| observer | str | 머리글자 |
| note | str | 자유(예: "창문 열림", "문 열어 둠") |

검증
- 같은 공간에서 구간이 겹치지 않는다.
- 공간·상태 값이 허용 목록 안에 있다.
- 시각은 실시간 수집 기간 안에 있다.

오류는 줄 번호와 이유로 알린다.

## 정답

**사용 여부** — `var/state/labels_occupancy.parquet`

| 열 | 설명 |
|---|---|
| space_id, bin | 공간, 5분 칸(UTC) |
| label | 1 사용중 · 0 비어있음 |
| source | timetable · booking_estimated · booking_file |
| obs_label | 관찰 정답(관찰 칸만, 칸 안 다수 상태) · NaN |
| obs_people | 관찰 인원 · NaN |

**환기 경보** — `var/state/labels_vent.parquet`

| 열 | 설명 |
|---|---|
| space_id, bin | |
| co2_now | 현재 CO₂(관측) |
| max_30 | (t, t+30분] 관측 최댓값 |
| first_exceed | 첫 ≥ 1,000 시각 · NaT |
| label | positive · negative · exceeded · undetermined |

## 특징 — `var/state/features.parquet`

| 열 | 설명 |
|---|---|
| space_id, bin, split | 분할(train·val·test·live) |
| `lvl_*`, `d5_*`, `d15_*`, `d30_*`, `slope30_*`, `sd30_*`, `rise30_*` | 센서 특징(co2·temperature·humidity) |
| `obs_ratio30`, `age_*` | 관측 상태 |
| `t_*`, `sch_*` | 시간·일정(환기·누수 실험 전용) |
| input_ok | 입력 관측 비율 ≥ 기준 |

특징 순서는 설정 해시와 함께 `features.json`에 기록하고, 묶음에 그대로 들어간다.

## 실행 기록 — `var/state/runs/<문제>/<후보 해시>-s<씨앗>/run.json`

문제(occupancy·vent), 후보(모델·하이퍼파라미터·특징 묶음), 씨앗, 장치, 학습 시간, 검증 지표(macro-F1, 재현율, 정밀도, AP), 모델 해시, 데이터 해시.
상태는 `done` · `failed`이고 끝난 것은 건너뛴다.

## 선택 — `var/state/selection.json`

| 필드 | 설명 |
|---|---|
| occupancy | 고른 후보·씨앗들, 임계값(검증 macro-F1 최대), 이유, 후보 표 |
| vent | 고른 후보, 임계값(검증 재현율 ≥ 0.85 중 정밀도 최대, 미달 시 표시), PR 곡선 요약 |
| forecast_vent | 과제 3 판정의 임계값(1,000 − δ)과 검증 재현율 |
| leak_experiment | `with_schedule` 사용 여부 모델(비교 전용, 선택 아님) |
| frozen_at, sha256 | 고정 |

규칙: `evaluate`는 이 파일이 없거나 변조됐거나 모델 해시가 다르면 거부한다.

## 평가 결과 — `var/state/eval/results.json`

문제 × 방법(분류기·단순 규칙·과제 3 판정·누수 실험) × 정답 기준(관찰·일정) × 공간:
- 지표, 혼동 행렬, 칸 수
- 관찰 시간, 판단 불가 수
- 확인 기준 판정

오류 분석 표는 `analysis.json`에 둔다.

## 분류기 묶음 — `var/state/bundle/`

| 파일 | 내용 |
|---|---|
| `occupancy.(joblib|pt)`, `vent.(joblib|pt)` | 최종 분류기 |
| `bundle.json` | 문제별 모델 종류·파일, **입력 규약 버전**, **특징 순서**, 정규화(공간 × 항목 평균·표준편차), **임계값**, 특징 창 길이, 관측 비율 기준, 데이터·선택 해시, 공통 전처리 버전 |

- 과제 8은 `bundle.json`의 입력 규약 버전이 자기 것과 다르면 이 묶음을 쓰지 않고 알린다.
- 과제 7은 모델 종류가 MLP면 변환하고, 트리·선형이면 CPU 측정만 한다.
