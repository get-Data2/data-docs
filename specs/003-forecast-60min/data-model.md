# Data Model: 60분 예측 (과제 3)

## 입력(과제 2 산출, 읽기만)

| 파일 | 쓰는 열 | 규칙 |
|---|---|---|
| `var/series/space_5min.parquet` | space_id, item, bin(UTC), value, filled | `filled=False`(관측)만 입력·정답에 쓴다 |
| `var/eval/eval_points_v1-provisional.parquet` | space_id, item, t0, horizon_min, y_true, usage_hours | 시험 평가 대상 행. sha256 `2f878c11d09ab68d…`가 다르면 실행 중단 |
| `var/eval/baseline_v1-provisional.parquet` | 위 키 + persistence, mean_1h, yesterday | 같은 키로 모델 예측과 결합 |
| `var/eval/norm_v1-provisional.json` | spaces.<공간>.<항목>.mean/std | 입력·목표 정규화 |
| `var/cache/events.parquet` | kind, space_id, start, end | 시험 기간 사건 구간 오차(분석만) |
| `config/analysis.yaml` | split, schedule, baseline.usage_hours | 분할·일과표·사용 시간대 정의 공유 |

## 공간 배열 (`data.py`)

공간마다 5분 칸의 연속 배열(학습 시작 − 최대 과거 길이 ~ 시험 끝)을 만든다.

| 필드 | 형 | 설명 |
|---|---|---|
| `t` | datetime64[UTC] (N) | 5분 칸 |
| `x_val` | float32 (N, 3) | CO₂·온도·습도 정규화 값. 빈 칸은 직전 관측으로 앞으로 채움(처음 관측 전은 0) |
| `x_obs` | uint8 (N, 3) | 관측 표시 |
| `x_age` | float32 (N, 3) | 마지막 관측 뒤 지난 시간(시간 단위, 상한 6) |
| `y_raw` | float32 (N, 2) | CO₂·온도 원값(관측만, 빈 칸 NaN) — 정답·현재값 |
| `cov` | float32 (N, F) | 시간·일정·예약 특징(R6). t0 이후 칸도 미리 알 수 있는 값만 |

검증 규칙
- `x_val[i]`는 `t ≤ t[i]`의 관측만으로 만든다. 자동 시험으로 확인한다: 칸 i 이후 값을 바꿔도 `x_val[:i+1]`이 같아야 한다.
- 정규화 통계는 과제 2 파일에서만 읽는다. 처음 보는 공간 실험에서는 학습 공간들의 통계를 칸 수로 가중 평균한 값을 쓴다.

## 표본 (`dataset.py`)

표본은 (공간, t0 인덱스)로 식별한다. 실제 텐서는 배치를 만들 때 잘라 만든다.

| 텐서 | 형 | 내용 |
|---|---|---|
| `past` | (L, P) | 칸 t0−L+1 … t0의 [x_val, x_obs, x_age, cov(과거용)] |
| `future` | (12, Fu) | 칸 t0+1 … t0+12의 cov(시간·일정·예약) |
| `space` | (S,) | 공간 원-핫(`space` 묶음이 켜졌을 때) |
| `target` | (12, 2) | 목표(변화량 또는 값, 정규화) |
| `target_mask` | (12, 2) | 정답 관측 표시 |
| `current` | (2,) | y(t0) 원값(변화량 → 원값 복원용) |

선정 규칙

| 분할 | t0 조건 |
|---|---|
| train | t0와 t0+60분이 학습 기간 안, y(t0) CO₂·온도 관측, 입력 관측 비율 ≥ `min_input_obs`(기본 0.1) |
| val | 같은 조건, 검증 기간 |
| test | 평가 시점 파일의 고유 (공간, t0). 입력 관측 비율이 부족하면 `unpredictable=True` |

## 실험 설정 (`config.py`)

| 필드 | 값 | 기본 |
|---|---|---|
| `name` | 실험 이름(예: `D1-lookback-2h`) | — |
| `model` | `gru` · `lstm` · `tcn` | `gru` |
| `size` | hidden / layers / channels / head | 64 / 2 / 64 / 128 |
| `lookback_min` | 30 · 60 · 120 · 240 · 1440 | 120 |
| `features` | `sensors`, `time`, `space`, `schedule`, `booking` 부분집합 | 앞 4개 |
| `output` | `direct` · `recursive` | `direct` |
| `target` | `delta` · `value` | `delta` |
| `train_spaces` | 공간 목록 | lab, office, meeting |
| `seeds` | 정수 목록 | [0, 1, 2] |
| `train` | lr, weight_decay, batch, max_epochs, patience_lr, patience_stop | 1e-3, 1e-4, 512, 40, 5, 8 |

- **설정 해시**: `seeds`를 뺀 설정의 정렬 JSON sha256 앞 12자리. 같은 해시면 같은 실험이다.

## 실행 기록 (`var/forecast/runs/<name>/<hash>-s<seed>/run.json`)

| 필드 | 설명 |
|---|---|
| name, config_hash, seed | 식별 |
| status | `done` · `failed`(+error) |
| device, torch, cuda, deterministic | 실행 환경 |
| data_hash | 공간 시계열 sha256 앞 16자리, 평가 시점 sha256, 정규화 파일 sha256 |
| params, train_seconds, epochs, best_epoch | 학습 정보 |
| val_mae | 공간 × 항목 × 지평(5·15·30·60) 검증 MAE(원 단위)와 정규화 평균(선택 지표) |
| model_sha256 | `model.pt` 해시 |
| peak_mem_mb | 프로세스 최대 메모리(RSS)·GPU 최대 할당 |
| test_evaluated_at | 시험 평가를 했으면 그 시각(선택 고정 뒤여야 함) |

상태 전이: (없음) → `done` | `failed`. `failed`는 `run --retry-failed`로만 다시 돈다. `done`이면 건너뛴다.

## 선택 파일 (`var/forecast/selection.json`)

| 필드 | 설명 |
|---|---|
| stages | 단계(D1~D4, B1)마다 후보별 검증 지표(평균 ± 표준편차)와 고른 값, 이유(최소 / 변동 안이라 단순한 쪽) |
| final | 최종 실험 설정(+해시) |
| test_set | 시험 평가할 실험 목록: final, final−schedule, B1 최적, 심화 2·3 |
| frozen_at, sha256 | 고정 시각과 파일 해시 |

규칙: `test` 명령은 이 파일이 없거나, 실행 기록의 해시와 맞지 않으면 거부한다.

## 시험 예측 (`test_pred.parquet`)

| 열 | 설명 |
|---|---|
| space_id, item, t0, horizon_min | 평가 시점 파일 키 |
| y_pred | 원 단위 예측(씨앗별) |
| seed | 씨앗 |
| unpredictable | 입력 부족으로 현재값 유지로 대체했는지 |

평가 시점 파일과 키 집합이 같아야 한다(회의실 5·15분 포함 — 기준선과 같은 `min_points` 규칙으로 표에서 "평가 불가"). 자동 시험으로 확인한다.

## 예약(일정) 자료

| 필드 | 설명 |
|---|---|
| space_id, start, end (KST) | 구간 |
| kind | `class` · `lunch` · `work` · `booking` · `booking_estimated` |
| source | `timetable`(과제 2 설정) · `file`(실제 예약 CSV) · `estimated`(R10) |

추정 예약 파일은 `var/forecast/booking_estimated.parquet`에 둔다. 규칙 파라미터와 해시를 함께 기록한다.

## 모델 묶음 (`var/forecast/bundle/`)

| 파일 | 내용 |
|---|---|
| `model.pt` | 최종 모델 가중치(검증 최고 씨앗 기준, 다른 씨앗은 실행 기록에) |
| `bundle.json` | 실험 설정, 입력 규약 버전, 특징 순서, 정규화 통계, 공간 목록, 데이터 해시, 시험 MAE 요약 |

과제 7(최적화)·과제 8(서비스)은 이 묶음만 읽는다.
