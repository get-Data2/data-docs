---
description: "과제 3 60분 예측 작업 목록"
---

# Tasks: 60분 예측 (과제 3)

**Input**: `specs/003-forecast-60min/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다. 헌법 IV에 따라 확인 기준 3(시험 기간 미사용)을 자동 시험으로 검증한다. 인과적 입력·마스크 손실·평가 행 일치·재개도 시험한다.
- 합성 시계열 시험: 마커 `unit`
- 실데이터 소규모 학습 시험(GPU 있으면 GPU): 새 마커 `forecast`(느림, 기본 실행에서 제외)

**Organization**
- 스토리 우선순위: US1·US2·US3 = P1, US4·US5 = P2, US6 = P3
- 실행 의존 관계 때문에 순서는 **기반 → US2(누수 장치) → US3(설계 실험·선택) → US6-B1(최종 모델 선택) → US1(시험 평가·확인 기준) → US5 → US4 → US6-A2·A3 → 마무리**다.
  - 선택이 끝나야 시험 평가를 할 수 있다(research R9).
- 실험 실행(긴 작업)은 백그라운드로 돌리고, 끝난 실행은 건너뛰며 이어서 돈다.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 병렬 가능(다른 파일, 미완료 작업 의존 없음)
- 경로는 저장소 루트 기준. 소스 `src/forecast/`, 시험 `tests/forecast/`, 설정 `config/forecast.yaml`, 작업 파일 `var/forecast/`
- GPU 실행은 `tools/forecast`(그룹 미반영 세션이면 `sg video`·`sg render`로 다시 실행)

---

## Phase 1: Setup

- [X] T001 `pyproject.toml`에 extra `forecast = ["torch>=2.14"]`를 추가하고, 패키지 탐색에 `forecast`가 포함되는지 확인한다.
  pytest 마커 `forecast`(실데이터 학습 시험, 기본 제외)를 등록한다. torch는 이미 cu130 index로 설치돼 있으므로 `-e '.[test,analysis,forecast]'` 설치가 torch를 바꾸지 않는지 확인한다
- [X] T002 [P] `tools/forecast` 실행 래퍼를 작성한다.
  - 프로젝트 폴더로 이동해 `.venv/bin/python -m forecast "$@"`를 실행한다.
  - `id -nG`(현재 프로세스)에 `video`가 없고 `id -nG "$USER"`(계정)에는 있으면 `sg video -c "sg render -c '<자기 자신 + 인자>'"`로 다시 실행한다. 인자는 따옴표를 안전하게 처리한다.
  - `CUBLAS_WORKSPACE_CONFIG=:4096:8`을 내보낸다.
  - `~/.local/bin/forecast`에 링크한다
- [X] T003 [P] `config/forecast.yaml`을 contracts/config.md 예시 그대로 작성한다
- [X] T004 [P] `tests/forecast/__init__.py`, `tests/forecast/conftest.py`를 작성한다. 고정물은 다음과 같다.
  - `fcfg`: 예측 설정. 작은 크기로 덮어쓰기(hidden 8, max_epochs 2, seeds [0])
  - `synth_series(days, spaces, gaps)`: 과제 2 `space_5min` 형식 합성 시계열. 평일 09:00 상승, 점심 하락, 회의실은 10분 주기 관측
  - `synth_eval_points`: 합성 시계열에서 과제 2 `build`와 같은 규칙으로 만든 평가 시점

---

## Phase 2: Foundational

**⚠️ 스토리 작업 전 필수**

- [X] T005 `src/forecast/config.py`
  - `config/forecast.yaml`을 로드·검증한다(contracts/config.md 검증 규칙).
  - `analysis.config.load_analysis_config`로 분할·일과표·사용 시간대를 읽는다.
  - `ExperimentConfig`(data-model "실험 설정")를 만들고, 설정 해시(seeds·inputs 제외, 정렬 JSON sha256 앞 12자리)를 계산한다.
  - `stage_candidates(stage, chosen)`: 앞 단계 선택을 누적 적용한 후보 목록. B1은 곱집합
- [X] T006 [P] `src/forecast/env.py`
  - `pick_device(pref)`: auto는 cuda 가능 시 cuda
  - `set_determinism(seed)`: random·numpy·torch 씨앗, TF32 끔, cudnn deterministic, `use_deterministic_algorithms(True, warn_only=True)`
  - `env_info()`: torch·CUDA 버전, GPU 이름, 결정성 설정
  - `peak_memory()`: 프로세스 RSS 최대값, `torch.cuda.max_memory_allocated`
- [X] T007 `src/forecast/data.py`
  - `load_inputs(cfg)`: 과제 2 파일을 읽고, 평가 시점 sha256을 확인한다(불일치 시 `InputHashError`).
  - `build_space_arrays(series, norm, spaces, t_start, t_end)`: data-model "공간 배열"을 만든다.
    - 관측(`filled=False`)만 쓰고, 빈 칸은 앞으로만 채운다.
    - `x_obs`, `x_age`(상한 `age_cap_h`)를 만들고, `y_raw`에는 관측만 둔다.
  - `split_t0(arr, split, kind, min_input_obs, lookback)`: data-model "표본 선정 규칙"
  - `pooled_norm(norm, spaces)`: 처음 보는 공간용 합친 통계(칸 수 가중)
- [X] T008 `src/forecast/features.py`
  - `time_features(t)`: 하루 시각·요일 sin·cos, 주말·공휴일(`holidays.KR`)
  - `schedule_features(t, space, acfg)`: `analysis.schedule`의 일과표 행으로 상태 4종 원-핫과 다음 변화까지 남은 시간(시간, 상한 3)
  - `booking_features(t, space, bookings)`: 예약 중 여부, 시작·끝까지 남은 시간
  - `estimate_bookings(series, space, rule)`: research R10 규칙으로 추정 예약 구간 표(source=`estimated`)
  - `covariates(arr_t, space, features, acfg, bookings)`: 묶음 순서가 고정된 특징 행렬과 열 이름 목록
- [X] T009 `src/forecast/dataset.py`
  - `WindowDataset(arrays, index, lookback_bins, horizon_bins, features, target)`: 인덱스로 잘라 `past`, `future`, `space`, `target`, `target_mask`, `current`를 낸다(data-model "표본").
  - 변화량 목표는 (y(t0+h) − y(t0)) / σ, 값 목표는 (y − μ) / σ다. 정답이 관측이 아닌 칸은 mask 0, target 0으로 둔다.
  - `to_original(pred, current, norm, target)`: 원 단위로 복원한다
- [X] T010 `src/forecast/models.py`
  - `RNNEncoder(kind=gru|lstm)`
  - `TCNEncoder`: 인과적 팽창 합성곱, 잔차 연결. 수용 영역이 과거 길이 이상이 되도록 층 수를 자동으로 정한다
  - `Head`: [인코더 출력, 공간, 미래 펼침] → MLP → 12×2
  - `ForecastModel(direct)`
  - `RecursiveForecaster`: 한 칸 모델을 12번 반복한다. 예측을 다음 입력으로 넣을 때 관측 표시 0, 경과 시간 증가, 미래 특징은 해당 칸 값
  - `count_params()`
- [X] T011 `src/forecast/train.py`
  - `masked_l1(pred, target, mask)`
  - `train_one(exp, seed, arrays, device)`: AdamW, 배치 512, 학습 표본만 섞는다. 검증 손실로 ReduceLROnPlateau(patience_lr)와 조기 종료(patience_stop)를 하고, 최고 가중치를 복원한다. NaN이면 `failed`
  - `predict(model, dataset)`: 원 단위 예측
  - 검증 MAE: 공간 × 항목 × 지평(5·15·30·60)과 정규화 평균(선택 지표 `val_norm_mae`)
  - 출력: `run.json`(data-model "실행 기록"), `model.pt`, `val_pred.parquet`를 `var/forecast/runs/<name>/<hash>-s<seed>/`에 쓴다
- [X] T012 `src/forecast/__main__.py` 뼈대 (배열은 캐시하지 않고 매번 만든다 — 약 11초라 캐시 이득이 작다)
  - 하위 명령 `check · build · train · run · select · test · report · info`(contracts/cli.md)와 공통 옵션 `--config`, `--device`
  - `check`: 장치·입력 해시·설정 결과표
  - `build`: 배열·추정 예약을 `var/forecast/arrays.npz`, `booking_estimated.parquet`에 캐시하고 표본 수 표를 낸다
- [X] T013 [P] `tests/forecast/test_data.py` (unit)
  - **인과성**: 칸 i 이후 값을 바꿔도 `x_val[:i+1]`, `x_obs`, `x_age`가 같다
  - 보간 칸(`filled=True`)은 입력·정답에 쓰이지 않는다
  - 10분 주기 공간의 관측 표시가 번갈아 나온다
  - 표본 선정이 분할 경계(t0+60분 포함)를 지킨다
  - 입력 관측 비율 미만 표본은 test에서 `unpredictable`이 된다
  - `pooled_norm`이 뺀 공간 통계를 쓰지 않는다
- [X] T014 [P] `tests/forecast/test_features.py` (unit)
  - 공휴일(예: 2026-05-05)은 비수업일이다
  - 09:00 직전 칸의 "다음 변화까지 남은 시간"이 맞다
  - 미래 12칸 특징이 t0 이후 센서값에 의존하지 않는다(센서값을 바꿔도 같음)
  - 추정 예약: 합성 회의 1건(+600ppm 상승 → 하락)에서 시작·끝이 15분 단위로 맞게 나온다
- [X] T015 [P] `tests/forecast/test_models.py` (unit)
  - GRU·LSTM·TCN × direct·recursive 출력 형상이 (배치, 12, 2)다
  - TCN 수용 영역이 과거 길이 이상이다
  - 변화량 목표에서 출력 0이 원 단위로 현재값 유지와 같다
  - 마스크 손실은 마스크 0 칸의 값과 무관하다
- [X] T016 `forecast check`, `forecast build`를 실데이터로 실행한다. 공간별 학습·검증·시험 표본 수, 예측 불가 수, 추정 예약 건수, GPU 인식을 기록한다(T040 점검 문서에 반영)

**Checkpoint**: 실데이터 배열·표본이 만들어지고 합성 시험이 통과한다

---

## Phase 3: User Story 2 — 시험 기간을 학습·선택에 쓰지 않음 (P1)

**Goal**: 확인 기준 3을 장치(코드)와 시험으로 보장한다. 실험을 시작하기 전에 둔다.

**Independent Test**: 시험 기간 값을 바꾼 데이터로 학습·선택을 다시 해도 결과가 같다.

- [X] T017 [US2] `src/forecast/experiments.py`의 선택 고정 부분
  - `freeze_selection(stages, final, test_set)`: `selection.json`(data-model)을 쓰고 sha256을 기록한다
  - `require_selection()`: 파일이 없거나, 기록된 실행 해시와 맞지 않으면 `SelectionMissingError`
  - 시험 평가 함수는 이 검사를 반드시 거친다
- [X] T018 [P] [US2] `tests/forecast/test_leakage.py` (unit, 합성 데이터)
  - 같은 씨앗으로 두 번 학습한다. 두 번째는 시험 기간의 입력·정답 값을 ×10, −999로 바꾼다
  - 학습 손실 기록, 검증 손실 기록, 최고 에폭, 가중치(텐서 동일), 단계 선택 결과가 같아야 한다
- [X] T019 [P] [US2] `tests/forecast/test_selection_gate.py` (unit): `selection.json` 없이, 또는 해시가 변조된 상태로 시험 평가를 부르면 거부된다. 고정 시각이 시험 평가 시각보다 앞선다
- [X] T020 [US2] 정규화 확인 시험: 학습에 쓰는 통계가 과제 2 `norm_v1-provisional.json`과 같다(실데이터, 마커 `forecast`) — `tests/forecast/test_norm_real.py`

**Checkpoint**: 누수 시험이 통과한 뒤에만 실험을 시작한다

---

## Phase 4: User Story 3 — 입력·출력 설계 (P1) 🎯

**Goal**: D0~D4 단계 실험으로 설계를 검증 기준으로 고르고 이유를 남긴다.

**Independent Test**: 단계마다 검증 MAE 표(평균 ± 표준편차)와 고른 값·이유가 나온다.

- [X] T021 [US3] `src/forecast/experiments.py`의 실행부
  - `run_stage(stage, chosen)`: 후보 × 씨앗을 학습하고, `run.json`이 `done`이면 건너뛴다
  - `pick(stage)`: `val_norm_mae` 평균 최소. 단, 1표준편차 안이면 단순한 쪽(tie_rule)
  - `stage_table(stage)`: 후보별 공간 × 항목 × 지평 검증 MAE
  - `run all` 순서는 D0 → D1 → D2 → D3 → D4 → B1 → S → A2 → A3이다. 실패한 실행은 기록하고 계속한다(`--retry-failed`)
- [X] T022 [P] [US3] `tests/forecast/test_experiments.py` (unit)
  - 재개: `run.json`이 있으면 다시 학습하지 않는다
  - 단계 누적: D2 후보가 D1 선택값을 쓴다
  - tie_rule이 동률 근처에서 단순한 쪽을 고른다
  - 실패 실행이 다음 실행을 막지 않는다
- [X] T023 [US3] `forecast train D0 --seed 0`을 실데이터로 실행한다. 실제 학습 시간, 에폭, 최대 메모리(목표 ≤ 2GB), 검증 MAE를 확인한다.
  - 시간 추정(research R14)을 실측으로 고친다
  - 문제가 있으면(메모리·발산·시간) 크기·배치를 설정에서 조정하고 그 이유를 research에 기록한다
- [X] T024 [US3] 백그라운드에서 `forecast run D0 D1 D2 D3 D4` 실행한다. 끝나면 단계별 표를 확인한다
- [X] T025 [US3] 설계 결과 해석: 과거 길이, 입력 묶음별 기여, direct vs recursive의 지평별 오차 증가 모양,
  값 vs 변화량(변화가 없을 때 현재값을 지키는 정도)을 정리한다. 생각해 볼 점 Q2의 근거로 쓴다

**Checkpoint**: D1~D4 선택값이 정해진다

---

## Phase 5: User Story 6 (일부) — 심화 1: 과거 길이 × 모델 (P3, 최종 모델 선택에 필요)

- [X] T026 [US6] 백그라운드에서 `forecast run B1`(1·2·4시간 × GRU·LSTM·TCN, 씨앗 3)을 실행한다. 검증 MAE, 학습 시간, 파라미터 수를 표로 만든다
- [X] T027 [US6] `forecast select`: D1~D4·B1로 최종 설정을 고정한다. 시험 대상은 final, S(final − schedule), B1 최적, A2·A3 실험이다

**Checkpoint**: `selection.json` 고정. 이후에만 시험 평가

---

## Phase 6: User Story 1 — 기준선을 이기는 60분 예측 (P1)

**Goal**: 시험 평가와 확인 기준 판정, 모델 비교표.

**Independent Test**: 평가 시점 행 수가 기준선과 같고, 판정표가 자동으로 나온다.

- [X] T028 [US1] `src/forecast/evaluate.py`
  - `test_predict(run)`: 평가 시점 파일의 고유 (공간, t0)를 예측한다. 예측 불가 표본은 현재값 유지로 대체하고 표시한다
  - `join_baseline()`: 기준선 parquet과 키로 결합한다(행 수 일치를 검사하고, 다르면 오류)
  - `comparison_table(subset=all|usage)`: 공간 × 항목 × 지평, 기준선 3종 + 모델(씨앗 평균 ± 표준편차), 개선율. `min_points` 미만은 "평가 불가"
  - `criteria_table()`: CO₂ 60분 30%, 온도 60분 개선, 누수 시험 결과 링크
- [X] T029 [P] [US1] `tests/forecast/test_evaluate.py` (unit): 합성 평가 시점에서 예측 행 키 집합 = 평가 시점 키 집합. 예측 불가 대체 표시, 개선율 계산, `min_points` 처리를 확인한다
- [X] T030 [US1] `forecast test` 구현과 실행. 모델 묶음 `var/forecast/bundle/`(data-model)을 저장하고, `run.json`에 `test_evaluated_at`을 기록한다
- [X] T031 [US1] 확인 기준 판정 확인. 못 미치는 항목이 있으면 원인을 분석한다(사용 시간대 오차 구성, 공간 특성). **시험을 본 뒤 설정을 바꾸지 않는다** — 바꾸면 새 선택·새 시험으로 기록하고 그 사실을 문서에 적는다

---

## Phase 7: User Story 5 — 일정 정보의 가치 (P2)

- [ ] T032 [US5] `src/forecast/analysis.py`의 `schedule_value()`
  - final vs final − schedule의 시험 MAE 차이를 공간 × 지평 × (전체·사용 시간대·일정 사건 직후 30분)으로 계산하고, 씨앗 변동과 함께 낸다
  - 결론 문장을 만든다: "○ppm(○%) 줄어든다", 또는 변동 안이면 "구별할 수 없다"

---

## Phase 8: User Story 4 — 공간별 오차 분석 (P2)

- [ ] T033 [US4] `src/forecast/analysis.py`
  - `phase_errors()`: 일과 단계별 MAE(수업 시작 전후 30분, 점심, 퇴실, 야간, 주말)
  - `top_errors(n=20)`
  - `window_open_segments()`: 창문 개방 추정(설정 `window_open`)과 그 구간 MAE
  - `event_errors()`: 과제 2 `events.parquet` 시험 기간 사건 구간 MAE
  - 공간 성질 표: 센서 수, 주기, 일정 반응률 — 과제 2 진단서 값을 인용
- [ ] T034 [P] [US4] 그래프(`docs/deliverables/task3/figures/`)
  - 지평별 MAE 곡선(모델 + 기준선 3종, 공간별)
  - 하루 시각별 오차(모델 vs 현재값 유지)
  - 대표 하루(수업일 1일, 회의실 회의일 1일)의 예측 궤적. 한글 폰트는 과제 2와 같다

---

## Phase 9: User Story 6 (나머지) — 심화 2·3 (P3)

- [X] T035 [US6] 백그라운드에서 `forecast run S A2 A3`을 실행한다(S는 선택 전에 돌아도 되지만 시험 평가는 선택 뒤). 이어서 `forecast test`로 시험 대상에 포함한다
- [ ] T036 [US6] 심화 2 분석: `booking` 유무에 따른 회의실 30·60분 MAE와 실습실·사무실 영향. **추정 예약 = 상한** 표시. 추정 예약 건수·평균 길이도 정리
- [ ] T037 [US6] 심화 3 분석: 공간별 "빼고 학습" vs "포함 학습(`space` 끔)" vs 현재값 유지 시험 MAE와 해석

---

## Phase 10: Polish & 산출물

- [ ] T038 `src/forecast/report.py` + `forecast report`
  - `docs/deliverables/task3/01-model-comparison.md`: 비교표, 판정표, 측정 조건
  - `docs/deliverables/task3/02-space-error-analysis.md`: 공간별 분석, 그래프
  - `docs/deliverables/task3/03-schedule-value.md`: 일정 포함·제외 비교와 결론
  - `docs/advanced/task3/01-lookback-model.md`, `02-future-info.md`, `03-unseen-space.md`
  - 각 문서에 측정 조건(데이터 기간, 실데이터, 분할·평가 시점 해시, 장치, 전력 모드 25W, 씨앗 수)과 해석 자리를 둔다
- [ ] T039 산출물·심화 문서의 해석 절을 작성한다(헌법 VII)
  - 왜 이 모델·설계가 이겼나
  - 회의실이 어려운 이유와 필요한 외부 정보(Q1)
  - 값 vs 변화량(Q2)
  - 오차가 큰 순간과 맞힐 수 있는지(Q3)
- [ ] T040 `docs/tasks/task3-forecast-60min.md` 점검 문서를 완성한다
  - 해결할 것 5, 확인 기준 3, 산출물 3, 생각해 볼 점 3, 심화 3을 항목별로 대응시키고 산출물에 링크한다
  - **확인 명령 표**는 실제로 실행해 맞춘 뒤 적는다(`forecast check/info`, pytest, 판정표 위치)
- [ ] T041 [P] `docs/deliverables/README.md`(과제 3 행 상태·링크), `docs/README.md`, `.github/README.md`(빠른 사용에 `forecast` 추가), `CLAUDE.md` 진행 상황을 갱신한다
- [ ] T042 전체 시험을 실행한다. `pytest -m "unit or broker or mysql"`, `pytest tests/forecast`, 그리고 `-m forecast` 실데이터 시험이다. 결과 수를 점검 문서에 기록한다
- [ ] T043 두 저장소를 커밋·푸시한다. 비밀 값·내부 주소가 없는지 확인한다. 가중치·예측은 `var/`라 커밋하지 않는다

---

## Dependencies & Execution Order

- Setup(T001~T004) → Foundational(T005~T016) → US2(T017~T020)
- US3(T021~T025) → B1(T026) → 선택(T027)
- US1(T028~T031) → US5(T032) · US4(T033~T034) · 심화 2·3(T035~T037) → Polish(T038~T043)
- 긴 실행: T024(설계, 약 2시간), T026(B1, 약 2시간), T035(S·A2·A3, 약 1시간). 백그라운드로 돌리고 그동안 T028~T029, T032~T034 코드를 작성한다(시험 평가는 선택 뒤)

## Parallel Examples

- T002 · T003 · T004 동시
- T006은 T005와 별개 파일. T013 · T014 · T015 시험은 각 구현 직후 병렬
- T024 실행 중에 T028(evaluate.py) · T029 · T033(analysis.py) 코드 작성

## Implementation Strategy

1. **MVP**: 기반 + 누수 장치 + D0 한 실행. **검증 기간** MAE로 현재값 유지 대비 개선 정도를 가장 빨리 본다.
   시험 평가는 하지 않는다. 시험 성적을 본 뒤 설계를 고르면 누수가 되기 때문이다.
2. 설계 단계 D1~D4와 B1을 돌려 정식 선택을 고정한 뒤, 시험 평가를 한 번 한다.
3. 일정 가치, 공간 분석, 심화 2·3 → 산출물 문서.
