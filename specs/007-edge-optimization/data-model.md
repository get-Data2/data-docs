# Data Model: 엣지 최적화 (과제 7)

## 입력(읽기만)

| 출처 | 파일 | 쓰는 것 |
|---|---|---|
| 과제 3 | `var/forecast/bundle/model.pt`, `bundle.json` | 최종 예측 모델, 입력 규약(과거 칸·특징 순서), 정규화 |
| 과제 3 | `var/forecast/runs/<해시>-s<씨앗>/` + `selection.json` | 심화 1(B1)의 GRU·LSTM·TCN — 같은 과거 길이 비교용(Q2) |
| 과제 3 | 평가 시점 파일 + `forecast` 배열 | 결과 차이 표본(시험 (공간, t0) 전체) |
| 과제 4 | `var/anomaly/bundle/ae_<공간>.pt`, `bundle.json` | 오토인코더 3개, 열 순서, 정규화, 임계값(경보 판정 변화) |
| 과제 5 | `var/state/bundle/` | 분류기(신경망이면 변환, 트리·선형이면 CPU만) |

## 변환 대상 모델 (`config.models` 해석 결과)

| 필드 | 설명 |
|---|---|
| name | `forecast_final` · `forecast_gru` · `forecast_lstm` · `forecast_tcn` · `ae_lab` · `ae_office` · `ae_meeting` · `state_occupancy` · `state_vent` |
| source | 묶음 경로, 원본 가중치 sha256 |
| inputs | 이름·형상(배치 축 동적)·자료형 |
| outputs | 이름·형상 |
| restore | 원 단위 복원 방법(예측: 현재값 + 출력 × σ, 오토인코더: 열 오차, 분류: argmax) |
| convertible | 신경망 여부(트리·선형 모델은 false) |

## 변환 기록 — `var/edgeopt/convert.json`

| 필드 | 설명 |
|---|---|
| model → onnx | 경로, sha256, 연산 집합 버전, 내보내기 검증(ORT CPU vs PyTorch 최대 차이) |
| model → fp16 · int8 | 엔진 경로, sha256, 성공 여부, 오류(단계·메시지), 빌드 시간, 작업 공간, 최적화 프로필 |
| env | TensorRT·ORT·torch·CUDA·드라이버 버전, GPU 이름, L4T 버전, 전력 모드 |
| int8_calibration | 표본 수, 씨앗, 기간(학습), 표본 키 해시 |

## 측정 기록 — `var/edgeopt/bench/<방식>__<모델>__b<배치>.json`

| 필드 | 설명 |
|---|---|
| runner, model, batch | 식별 |
| warmup, repeat, chunks | 워밍업·반복·나눈 측정 수(3) |
| latency_ms | mean, median, p95, p99, max, chunk_p95[] |
| preprocess_ms | 전처리 1회(따로) |
| threads | CPU 스레드 수(PyTorch·ORT CPU) |
| memory | rss_peak_mb, gpu_alloc_mb(가능할 때), board_ram_before_mb, board_ram_during_mb |
| power | idle_mw{in, cpu_gpu_cv, soc}, run_mw{…}, delta_mw, energy_mj_per_call, gr3d_mean, temp_start/end{cpu, gpu, tj} |
| conditions | power_mode, clocks_locked=false, contention(대기 GR3D 평균), other_gpu_processes, started_at |
| status | ok · unsupported(사유) · failed(오류) |

## 결과 차이 — `var/edgeopt/diff/<방식>__<모델>.parquet` + 요약

| 열 | 설명 |
|---|---|
| space_id, t0 | 표본 키(예측·오토인코더) |
| ref, out | 기준(PyTorch CPU FP32)과 이 방식의 원 단위 출력(예측: 지평 12 × CO₂·온도) |
| abs_diff | 원 단위 차이 |

요약(`results.json`): 방식 × 모델마다 다음을 둔다.
- 예측: CO₂·온도 최대·평균·99% 차이, 시험 MAE(원본 대비 변화)
- 분류: 바뀐 비율
- 오토인코더: 점수 차이와 경보 판정 바뀐 비율
- 판정(✅/❌)

## 동시 적재 — `var/edgeopt/colocate.json`

| 필드 | 설명 |
|---|---|
| runner | 실행 방식 |
| models | 함께 올린 모델 |
| rss_mb, board_ram_delta_mb | 총 메모리 |
| latency_ms_batch1 | 모델별·한 주기 합 |
| llm_table | 크기 가정(1B·3B·7B, 4비트) × 남는 메모리 × 가능 여부 |

## 실행 묶음(과제 8용) — `var/edgeopt/runtime/`

| 파일 | 내용 |
|---|---|
| `<모델>.plan` 또는 `.onnx` | 고른 실행 방식의 파일 |
| `runtime.json` | 고른 방식·이유, 원본 묶음 해시, 입력 규약 버전, 생성 환경(버전이 다르면 다시 생성), 결과 차이 판정 |
