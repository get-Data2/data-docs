# Data Model: 사전학습 모델 비교 (과제 6)

## 입력(읽기만)

| 파일 | 쓰는 열·내용 | 규칙 |
|---|---|---|
| `var/series/space_5min.parquet` | space_id, item, bin, value, filled | 관측(`filled=False`)만 컨텍스트에 사용. 공변량은 온도·습도 |
| `var/eval/eval_points_v1-provisional.parquet` | space_id, item, t0, horizon_min, y_true, usage_hours | CO₂ 행이 시험 대상. sha256 불일치 시 중단 |
| `var/eval/baseline_v1-provisional.parquet` | 키 + persistence, mean_1h, yesterday | 비교 열 |
| `var/forecast/selection.json` | final 그룹 실행 id | 직접 학습 모델 식별 |
| `var/forecast/runs/<id>/test_pred.parquet` | 키 + y_pred, seed | 직접 학습 시험 예측(씨앗 평균) |
| `var/forecast/runs/<id>/val_pred.parquet` | space_id, t0, item, horizon_min, y_true, y_pred | 앙상블 가중치용 검증 예측 |
| `var/forecast/stages.json` | D2 후보별 검증 지표 | 정보 차이 분해(Q1) |
| `var/forecast/bundle/` | 모델·입력 규약 | 직접 학습 모델 비용 측정 |

## 모델 기록 — `var/pretrained/models.json`

| 필드 | 설명 |
|---|---|
| name | 설정 이름(예: `bolt_small`) |
| repo, revision | HuggingFace 저장소, 커밋 해시 |
| params | 파라미터 수 |
| files_sha256 | 받은 가중치 파일 해시 |
| license | 라이선스 이름 |
| covariates | 공변량 지원 여부 |
| fetched_at | 받은 시각 |

## 컨텍스트 표본

| 필드 | 설명 |
|---|---|
| space_id, t0 | 표본 키 |
| context | (L,) float32, 관측값 + 빈 칸 NaN(또는 앞으로 채움 — 설정 `missing`) |
| context_obs | 관측 비율 |
| unpredictable | 관측 비율 < `min_context_obs` |
| covariates (심화) | 과거 (L, k), 미래 (12, k′) |

선정 규칙
- `val`: 검증 기간에서 과제 3과 같은 규칙. t0 CO₂ 관측, t0+60분이 기간 안
- `test`: 평가 시점 파일 CO₂ 행의 고유 (space_id, t0)

## 예측 — `var/pretrained/preds/<model>__ctx<분>__<period>[__cov].parquet`

| 열 | 설명 |
|---|---|
| space_id, item(=co2), t0, horizon_min | 평가 키(5·15·30·60) |
| p10, p50, p90 | 분위수 예측(ppm, 정렬·0 미만 자름) |
| y_pred | p50(점 예측) |
| unpredictable | 현재값 유지로 대체했는지 |
| y_true | 정답(관측, test는 평가 시점 파일 값) |

검증: test 파일의 키 집합 = 평가 시점 파일 CO₂ 키 집합(자동 시험).

## 선택 — `var/pretrained/selection.json`

| 필드 | 설명 |
|---|---|
| grid | 크기 × 컨텍스트별 검증 MAE(공간 × 지평, 정규화 평균) |
| chosen | 모델·컨텍스트, 이유 |
| ensemble | 방식(공간·지평별/하나), w 값, 검증 MAE |
| covariate_model | 심화 1 모델·컨텍스트(검증 기준) |
| frozen_at, sha256 | 고정 시각·해시 |

규칙: `test`는 이 파일이 없거나 변조되면 거부한다.

## 비용 측정 — `var/pretrained/bench/<model>__<device>__b<batch>.json`

| 필드 | 설명 |
|---|---|
| model, device, batch, context_len, dtype | 조건 |
| warmup, repeats | 반복 조건 |
| latency_ms | mean, p50, p95, max |
| peak_rss_mb, gpu_alloc_mb | 메모리 |
| power | VDD_IN·VDD_CPU_GPU_CV 평균(측정 중 / 유휴), GR3D 평균 |
| conditions | nvpmodel 모드, clocks_locked=false, 동시 실행 프로세스(사용자·이름·RSS), 측정 시각 |

## 판단표·예산표

- **판단표**: 모델(기준선·직접 학습·사전학습 크기별) × [60분 MAE 전체·사용 시간대, 적중률, 지연 p95(배치 1·3), 최대 RSS]
- **예산표**: 항목(관찰·가정·측정) × MB, 합계, 판정(≤ 85%), LLM 가정 2B·4B·8B 세 경우
