# Data Model: 상시 추론 서비스 (과제 8)

## 입력

| 원천 | 쓰는 것 | 규칙 |
|---|---|---|
| MySQL `measurement` | ts, space_id, device_id, item, value | 최근 `window_h`만 증분으로 읽는다(`ts > 마지막 − late_min`) |
| `config/collector.yaml` | 장치 기대 주기, 사건형 여부, 공간 별칭 | 끊김 판정(과제 1 기준) |
| `config/analysis.yaml` | 축 A 규칙, 분석 간격, 공간 제외 장치, 일과표 | 공통 전처리 |
| `var/forecast/bundle/` | model.pt, bundle.json(입력 규약·정규화·시험 MAE) | 예측 |
| `var/anomaly/bundle/` | 공간별 가중치, 분위수 표, 임계값, 규칙 파라미터, 알림 대상 | 이상 |
| `var/state/bundle/` | 사용 여부·환기 경보 분류기, 임계값 | 상태 |
| 과제 7 결과(예: `var/edgeopt/runtime/runtime.json`) | 모델별 실행 방식, 엔진 경로, 검증 오차 | 실행 방식 선택 |

## 메모리 창 (`window.py`)

| 필드 | 설명 |
|---|---|
| raw | 최근 `window_h` 1분 원본(DataFrame) |
| last_ts | 마지막으로 읽은 ts |
| sensor_bins | 닫힌 5분 칸 센서 평균(공통 전처리) |
| space_bins | 공간 대표값(공통 전처리) |

규칙: 칸은 닫힌 칸까지만 만든다(research R3). 늦게 도착한 행은 그 칸을 다시 계산한다.

## 공간 결과 (`store.py`, 발행·API 공통)

| 필드 | 형 | 설명 |
|---|---|---|
| schema | str | `team2.inference.v1` |
| space_id, space_name | str | `meeting`, `회의실` |
| t0 | ISO UTC | 마지막 닫힌 칸 끝(예측 기준 시각) |
| t0_kst | ISO KST | 사람용 |
| computed_at | ISO UTC | 계산 끝 시각 |
| freshness_min | float | 공간 대표값 마지막 관측 칸 나이 |
| stale | bool | 신선도 한도(15분) 초과 |
| forecast | object | `{unit: {co2: ppm, temperature: °C}, step_min: 5, horizons_min: [5..60], co2: [...], temperature: [...], interval: {...}|null}` |
| co2_over | object | `{threshold_ppm: 1000, first_over_min: 23|null, current_ppm: 812}` |
| anomaly | object | `{is_anomaly, score_q, threshold_q, causes: [{device_id, item, contribution}], since}` |
| rules | object | `{data_gap: [...장치], stuck: [...], out_of_range: [...]}` |
| state | object | `{occupied: bool, p_occupied, ventilation_soon: bool, p_vent}` |
| availability | object | 판단별 `ok` · `unavailable:<이유>` · `error:<이유>` |
| models | object | 묶음별 `{version, sha256, runtime: tensorrt|torch-cuda|torch-cpu}` |
| summary_ko | str | 한 줄 요약 |

## 경보 (`alerts.py`, 표 `svc_alert`)

| 필드 | 설명 |
|---|---|
| alert_id | (kind, space_id, device_id) 해시 |
| kind | co2_over_forecast · ventilation_soon · anomaly · data_gap · stuck · out_of_range · model_drift · service_degraded |
| space_id, device_id | 대상(device는 없을 수 있음) |
| status | active · cleared |
| event | raised · renotified · cleared |
| started_at, last_notified_at, cleared_at | 시각 |
| evidence | 근거 값(JSON) |
| action_ko | 권장 행동 문장 |
| notify_role | 센서 관리 담당 · 시설 담당 · 공간 사용자(과제 4 운영 제안) |

상태 전이: none →(판정)→ active[raised] →(renotify_min 경과, 계속)→ active[renotified] →(clear_after 주기 연속 정상)→ cleared[cleared] → none

`var/service/state.json`에 active 경보와 정상 주기 수를 저장해 재시작 뒤 복원한다(재시작으로 경보가 다시 raised되지 않게).

## 주기 기록 (표 `svc_cycle`)

| 열 | 설명 |
|---|---|
| cycle_id | 예정 시각(분) 기준 정수 |
| scheduled_at, started_at, finished_at | 시각 |
| status | ok · partial · failed · skipped |
| read_ms, prep_ms, infer_ms, publish_ms, total_ms | 단계별 시간 |
| rows_read | 읽은 원본 행 수 |
| error | 실패 이유(요약) |
| rss_mb, gpu_mb | 메모리 |
| runtime | 실행 방식 요약 |

## 예측 기록 (표 `svc_forecast_log`, 심화 2)

| 열 | 설명 |
|---|---|
| t0, space_id, item, horizon_min | 키 |
| y_pred | 예측 |
| y_true | 정답 칸이 닫히면 채움 |
| abs_err | 채점 |

## 장애 시험 기록 (표 `svc_event`)

수집기 정지·재시작, 브로커 재시작, 서비스 재시작 등 시험 단계와 관찰 시각. `infer report`가 장애 시험 절을 만든다.

## DDL 규칙

- 표 이름 접두사는 `svc_`다(팀 DB 안).
- 기록 간격은 과제 1과 같이 일괄(2초)이고, 스풀은 `var/service/spool.db`다.
- 보관: `svc_forecast_log`는 30일, 나머지는 90일(설정)이다.
