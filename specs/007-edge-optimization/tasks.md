---
description: "과제 7 엣지 최적화 작업 목록"
---

# Tasks: 엣지 최적화 (과제 7)

**Input**: `specs/007-edge-optimization/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다(헌법 IV). 확인 기준 1(원 단위 차이 판정)과 지연 통계 계산을 자동 시험으로 검증한다.
- 기본 마커: `unit`(합성 작은 모델, TensorRT 불필요)
- 실제 TensorRT가 필요한 시험: 마커 `trt`

**Organization**
- 우선순위: US1(변환)·US2(지연)·US3(결과 차이)·US4(자원) = P1, US5(보고서) = P2, US6(심화) = P3
- 실행 순서: **동작 확인 → 기반 → US1 → US3 → US2·US4 → US5 → US6 → 마무리**
- 결과 차이(US3)는 GPU 경합 영향이 없어 먼저 한다. 지연·자원 측정(US2·US4)은 과제 3·4·5 학습이 끝난 뒤 보드가 조용할 때 한다.
- **선행 조건**: 과제 3 모델 묶음(`forecast test` 뒤), 과제 4 묶음, 과제 5 묶음. 없는 묶음은 건너뛰고 나중에 같은 명령으로 채운다.

## Format: `[ID] [P?] [Story] Description`

- 경로는 저장소 루트 기준. 소스 `src/edgeopt/`, 시험 `tests/edgeopt/`, 설정 `config/edgeopt.yaml`, 작업 `var/edgeopt/`

---

## Phase 0: 동작 확인 (먼저)

- [ ] T001 패키지를 설치한다: `uv pip install onnx onnxruntime-gpu==1.31.0 tensorrt-cu13==11.4.0.106`(`.venv`). 설치 전후 `torch` 판이 바뀌지 않았는지 확인한다(cu130 유지)
- [ ] T002 `src/edgeopt/__main__.py`의 `probe`와 `src/edgeopt/runners.py` 최소판
  - 합성 Linear·Conv1d·GRU를 ONNX로 내보낸다
  - ORT CPU·CUDA EP, TensorRT FP16 엔진 생성·실행, ORT TensorRT EP를 실행한다
  - 결과 일치(FP32 1e-5, FP16 1e-2)와 실패 사유를 `var/edgeopt/probe.json`에 쓴다
- [ ] T003 `edgeopt probe`를 실행한다(GPU 그룹 래퍼 경유)
  - 실패하면 research R2의 대안 순서대로 시도한다: lib 경로 → ORT TensorRT EP → torch-tensorrt
  - 결과를 research에 기록한다
  - **모두 실패하면 사용자에게 관리자 요청(JetPack TensorRT) 여부를 묻는다**

---

## Phase 1: Setup

- [ ] T004 [P] `pyproject.toml`에 extra `edgeopt = ["onnx", "onnxruntime-gpu", "tensorrt-cu13"]`를 넣고, pytest 마커 `trt`를 등록한다
- [ ] T005 [P] `tools/edgeopt` 래퍼를 만든다
  - `tools/forecast`와 같은 GPU 그룹 처리
  - pip TensorRT lib 경로를 `LD_LIBRARY_PATH`에 넣는다
  - `~/.local/bin/edgeopt`에 링크한다
- [ ] T006 [P] `config/edgeopt.yaml`을 contracts/config.md 그대로 작성한다
- [ ] T007 [P] `tests/edgeopt/__init__.py`, `conftest.py`
  - 합성 작은 예측 모델(과제 3 `ForecastModel`, hidden 8)과 오토인코더(과제 4 `ConvAE`)
  - 가짜 묶음 디렉터리

---

## Phase 2: Foundational

- [ ] T008 `src/edgeopt/config.py`: 로드·검증(contracts/config.md 검증 규칙), 설정 해시, 대상 모델 목록 해석
  - 묶음이 없으면 `waiting`
  - `forecast_b1`은 `selection.json`의 B1 단계 후보 중 최종과 같은 과거 길이의 GRU·LSTM·TCN(씨앗 0)
- [ ] T009 `src/edgeopt/models.py`: 묶음 → (torch 모듈, 예시 입력, 원 단위 복원 함수, 입력 규약)
  - 예측: `forecast.models.ForecastModel` + `bundle.json`
  - 반복 예측이면 반복 안의 정규화 값을 입력으로 받는 래퍼(research R3)
  - 오토인코더: `anomaly.autoencoder.ConvAE`
  - 분류기: 신경망 여부 판별
- [ ] T010 `src/edgeopt/tegra.py`
  - `tegrastats` 시작·중지(로그 파일)
  - 줄 해석: 시각, VDD_IN·VDD_CPU_GPU_CV·VDD_SOC mW, GR3D_FREQ, RAM, 온도
  - 구간 평균, 대기 대비 증가분, 1회당 에너지
- [ ] T011 [P] `tests/edgeopt/test_tegra.py` (unit): 2026-10-21 실제 `tegrastats` 줄 2개(research 기록)를 해석해 값이 맞다. 구간 평균·에너지 계산을 손 계산과 맞춘다
- [ ] T012 `src/edgeopt/bench.py`
  - 지연 통계(평균·중앙값·p95·p99·최대, 나눈 측정의 p95)
  - 동기화를 포함한 측정 루프
  - 측정 전 검사
    - 우리 학습 프로세스 감지 → 거부
    - 대기 GR3D 평균 → 경합 표시·재시도
    - 온도 상한
  - 조합마다 새 프로세스로 실행한다(`python -m edgeopt _bench_one ...`)
- [ ] T013 [P] `tests/edgeopt/test_bench_stats.py` (unit)
  - 알려진 지연 배열로 p95 계산
  - 학습 프로세스가 있으면 거부(가짜 프로세스 목록 주입)
  - 경합 판정

---

## Phase 3: User Story 1 — 변환 (P1)

- [ ] T014 [US1] `src/edgeopt/export.py`
  - ONNX 내보내기: 연산 집합 17, 배치 축 동적, 시간 축 고정
  - 검증: ORT CPU vs PyTorch CPU 최대 차이 ≤ 1e-5
  - 실패 시 단계·오류를 기록하고 계속한다
- [ ] T015 [US1] `src/edgeopt/trt.py`
  - FP16 엔진 생성: 프로필 1/3/4, 작업 공간, 선택적 FP32 마지막 층
  - 실행 래퍼(`execute_async_v3` + 스트림 동기화)
  - 생성 환경 기록
- [ ] T016 [P] [US1] `tests/edgeopt/test_export.py` (unit): 합성 예측·오토인코더 ONNX 내보내기 → ORT CPU 결과가 PyTorch와 같다. 배치 1·3·4 동작
- [ ] T017 [P] [US1] `tests/edgeopt/test_trt.py` (마커 trt): 합성 모델 FP16 엔진 결과가 1e-2 이내다
- [ ] T018 [US1] `edgeopt export && edgeopt build`를 실데이터 묶음으로 실행하고, 성공·실패와 빌드 시간을 기록한다

---

## Phase 4: User Story 3 — 결과 차이 (P1)

- [ ] T019 [US3] `src/edgeopt/diff.py`
  - 과제 3 시험 표본 전체(`forecast` 배열·`eval_samples`)로 입력을 만든다
  - 방식마다 예측 → 원 단위 복원(FP32, 엔진 밖) → 기준 대비 CO₂·온도 차이(최대·평균·99%)와 시험 MAE 변화
  - 오토인코더: 점수 차이와 과제 4 임계값 기준 경보 판정 바뀐 비율
  - 분류기: 바뀐 비율
- [ ] T020 [P] [US3] `tests/edgeopt/test_diff.py` (unit): 일부러 0.5ppm·1.5ppm 차이를 넣은 출력으로 판정 ✅/❌, 바뀐 비율 계산
- [ ] T021 [US3] `edgeopt diff`를 실행하고 판정표를 확인한다
  - CO₂ 최대 차이가 1ppm을 넘으면 그 표본(공간·시각·입력 범위)을 본다
  - `trt.fp32_last_layer`를 켜고 다시 만든 뒤 비교한다(research R4·R7)

---

## Phase 5: User Story 2·4 — 지연과 자원 (P1, 보드가 조용할 때)

- [ ] T022 [US2] `edgeopt bench` 전체(방식 × 모델 × 배치 1·3·4, CPU 스레드 6·1)
  - 측정 조건·`tegrastats`·메모리를 기록한다
  - 경합으로 표시된 측정은 다시 잰다
- [ ] T023 [US2] 재현성: 같은 조합 일부(예측 최종 × 4방식 × 배치 1)를 다른 시각에 다시 재서 p95가 ±20% 안인지 확인한다
- [ ] T024 [US4] 전처리 시간(과제 3 입력 만들기 1회)을 재고, 서비스 한 주기 예산(60초) 대비 비중을 계산한다(과제 8 입력)

---

## Phase 6: User Story 5 — 보고서 (P2)

- [ ] T025 [US5] `src/edgeopt/report.py` + `edgeopt report`
  - 표: 방식 × 모델 × 배치 지연(평균·p95)·결과 차이·메모리·전력·에너지, 판정표
  - 그래프: 방식별 지연 분포(상자 그림), 배치별 지연, 1회당 에너지
- [ ] T026 [US5] `edgeopt runtime --runner <결정>`: 과제 8용 실행 묶음을 고정한다
- [ ] T027 [US5] `docs/deliverables/task7/01-optimization-report.md` 본문
  - 측정 조건, 표, 결론(과제 8 실행 방식과 이유)
  - Q1~Q3의 답을 측정 숫자로 쓴다(research R10)

---

## Phase 7: User Story 6 — 심화 (P3)

- [ ] T028 [US6] INT8: 학습 기간 표본 2,000개(씨앗 0)로 보정해 엔진을 만들고 `bench`·`diff`를 돌린다. 순환 신경망 미지원이면 기록한다
- [ ] T029 [US6] `src/edgeopt/colocate.py` + `edgeopt colocate`
  - 서비스 구성 동시 적재(방식별) 총 메모리·지연
  - LLM 크기 가정(1B·3B·7B, 4비트) 조합 표
- [ ] T030 [US6] `docs/advanced/task7/01-int8.md`, `02-colocation-llm.md` 본문과 해석

---

## Phase 8: Polish

- [ ] T031 점검 문서 `docs/tasks/task7-edge-optimization.md`
  - 해결할 것 4, 확인 기준 2, 산출물, 생각해 볼 점 3, 심화 2를 대응시킨다
  - **확인 명령**은 실행해 맞춘 뒤 적는다
- [ ] T032 [P] `docs/deliverables/README.md`, `docs/README.md`, `.github/README.md`, `CLAUDE.md`를 갱신한다
- [ ] T033 전체 시험(`unit`, 그리고 `trt`)을 실행하고 두 저장소를 커밋·푸시한다. 엔진·ONNX는 `var/`라 커밋하지 않는다

---

## Dependencies & Execution Order

- Phase 0(T001~T003) → Setup(T004~T007) → Foundational(T008~T013) → US1(T014~T018) → US3(T019~T021) → US2·US4(T022~T024) → US5(T025~T027) → US6(T028~T030) → Polish
- T018 이후는 각 과제 묶음이 있어야 한다. 과제 3 묶음만 있어도 예측 모델로 확인 기준 1을 먼저 판정할 수 있다
- T022~T023은 **과제 3·4·5 학습이 끝난 뒤**, 다른 계정 GPU 경합을 기록하며 한다

## Implementation Strategy

1. **MVP**: probe → 과제 3 최종 모델 하나를 ONNX·FP16으로 바꾸고 → diff(확인 기준 1) → 배치 1 지연 4방식
2. 나머지 모델(비교 3종·오토인코더·분류기)과 배치 3·4, 전력·메모리
3. 보고서·과제 8 실행 묶음 → 심화(INT8, 동시 적재)
