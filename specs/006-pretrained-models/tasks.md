---
description: "과제 6 사전학습 모델 비교 작업 목록"
---

# Tasks: 사전학습 모델 비교 (과제 6)

**Input**: `specs/006-pretrained-models/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다(헌법 IV). 같은 시점 비교(행 일치), 인과적 컨텍스트, 선택 고정 전 시험 거부, 앙상블 가중치의 검증 전용, 적중률 계산을 자동 시험으로 검증한다.
- 합성 시계열 + 가짜 모델: 마커 `unit`
- 실제 가중치가 필요한 시험: 마커 `pretrained`(기본 제외)

**Organization**
- US1(비교)·US2(구간)·US3(비용)·US4(채택) = P1, US5(생각해 볼 점) = P2, US6(심화) = P3
- 실행 순서: 기반 → 검증 예측·선택(US1 일부) → 시험 평가(US1·US2) → 비용(US3) → 판단(US4·US5) → 심화(US6) → 마무리
- **의존**: 직접 학습 열은 과제 3 시험 평가 뒤에 채운다. 비용 측정은 과제 3·4 학습이 끝나 보드가 한가할 때 한다.

## Format: `[ID] [P?] [Story] Description`

- 경로는 저장소 루트 기준. 소스 `src/pretrained/`, 시험 `tests/pretrained/`, 설정 `config/pretrained.yaml`, 작업 `var/pretrained/`

---

## Phase 1: Setup

- [X] T001 `pyproject.toml`에 extra `pretrained = ["chronos-forecasting==2.3.2"]`와 pytest 마커 `pretrained`를 추가한다.
  설치(`-e '.[test,analysis,forecast,pretrained]'`) 뒤 torch가 `2.14.1+cu130` 그대로인지 확인한다(바뀌면 되돌리고 의존성 고정)
- [X] T002 [P] `tools/pretrained` 래퍼(`tools/forecast` 방식, `HF_HUB_OFFLINE`은 fetch 외 명령에서 1)를 만들고 `~/.local/bin/pretrained`에 링크한다
- [X] T003 [P] `config/pretrained.yaml`을 contracts/config.md 그대로 작성한다(LLM 가정은 결정 기록대로 1B·3B·7B 4비트, `batch_size`·`size_order`·`tie_rel` 추가)
- [X] T004 [P] `tests/pretrained/__init__.py`, `conftest.py`를 작성한다
  - 과제 3 시험 고정물의 합성 시계열·짧은 분할 재사용
  - 가짜 분위수 모델(입력 마지막 값 ± 고정 폭)

## Phase 2: Foundational

- [X] T005 `src/pretrained/config.py`: 로드·검증(contracts/config.md), 해시, `forecast` 설정 연결
- [X] T006 `src/pretrained/context.py`
  - `context_for(series, space, t0s, L, missing)`: 관측만·t0까지, NaN 유지 또는 앞으로 채움
  - `val_t0()`: 과제 3 규칙(t0 CO₂ 관측, 60분 뒤 기간 안)
  - `test_t0()`: 평가 시점 파일 CO₂ 고유 키
  - 관측 비율·예측 불가 표시
- [X] T007 `src/pretrained/models.py`
  - `load(name, device, dtype)`: 리비전 고정·오프라인
  - `predict_quantiles(model, contexts, H, q)`: 배치, 정렬·0 자르기, 이상 건수 기록
  - 공변량 입력 래퍼(Chronos-2 지원 확인 뒤)
- [X] T008 `src/pretrained/predict.py`: 기간별 예측 → parquet(키·p10·p50·p90·y_pred·unpredictable·y_true). 있으면 건너뜀(이어서 실행)
- [X] T009 `src/pretrained/metrics.py`
  - `comparison_table()`: `forecast.evaluate.baseline_by_cell`·`mae_by_cell` 재사용, 기준선 3 · 직접 학습 · 사전학습
  - `coverage_table()`: 적중률·아래/위 이탈·평균 폭, 공간 × 지평 × 전체/사용 시간대
  - `calibration_curve()`
- [X] T010 `src/pretrained/selection.py`: 크기 × 컨텍스트 선택(검증 정규화 MAE, 동률은 작은 모델·짧은 컨텍스트), 앙상블 w 선택(검증), `freeze`·`require`(해시)
- [X] T011 `src/pretrained/__main__.py`: 하위 명령 뼈대(contracts/cli.md), `check`, `info`
- [X] T012 [P] `tests/pretrained/test_context.py` (unit)
  - t0 이후 값을 바꿔도 컨텍스트가 같다
  - 보간 칸 미사용
  - 관측 비율 미만 → 예측 불가
  - 10분 주기 공간의 NaN 위치
- [X] T013 [P] `tests/pretrained/test_metrics.py` (unit)
  - 손 계산 적중률(정답이 구간 안·아래·위)
  - 평균 폭
  - 비교표 평가 시점 수 = 기준선 표
- [X] T014 [P] `tests/pretrained/test_selection.py` (unit)
  - `selection.json` 없이 시험 거부, 변조 거부
  - 시험 예측을 바꿔도 선택·w가 같다(검증 전용)
  - 동률 규칙

**Checkpoint**: 가짜 모델로 전체 흐름(예측 → 선택 → 시험 → 표)이 합성 데이터에서 돈다

---

## Phase 3: User Story 1·2 — 비교와 예측 구간 (P1)

- [X] T015 [US1] `pretrained fetch`: Chronos-Bolt tiny·mini·small(+ Chronos-2)을 받는다. 리비전 해시를 설정에 고정하고 `models.json`(파라미터 수·라이선스·해시)을 만든다
- [X] T016 [US1] 결측 입력 실측: NaN 컨텍스트를 모델이 받는지, 결과가 앞으로 채우기와 얼마나 다른지 검증 표본 일부로 확인해 `missing` 기본값을 확정한다(research R3)
- [X] T017 [US1] `pretrained predict val`: 크기 3 × 컨텍스트 4(1시간·4시간·1일·1주) 검증 예측. 시간·메모리를 기록한다(과제 3 학습 중이면 CPU 또는 대기)
- [X] T018 [US1] `pretrained select`: 크기·컨텍스트 고정. 앙상블 w는 과제 3 검증 예측이 있을 때 고정한다
- [X] T019 [US1] `pretrained test`(과제 3 `forecast test` 뒤)
  - 선택 조합 + 크기별 비교 열 시험 예측
  - 비교표(전체·사용 시간대)와 행 일치 검사
- [X] T020 [US2] 적중률·이탈·폭·보정 곡선을 계산하고, 80%에서 벗어난 공간·지평·순간(수업 시작·점심)을 분석한다

---

## Phase 4: User Story 3 — 비용 (P1)

- [X] T021 [US3] `src/pretrained/bench.py`
  - 워밍업·반복·동기화, 지연 통계
  - `tegrastats --interval 100` 병행 기록과 파싱(VDD_IN·VDD_CPU_GPU_CV·GR3D·RAM), 유휴 구간
  - 측정 조건 수집: `nvpmodel -q`, 클럭 고정 안 함, 동시 프로세스
  - 과제 7에서 재사용할 수 있는 형태로 만든다
- [X] T022 [P] [US3] `tests/pretrained/test_bench.py` (unit): 가짜 모델로 반복 수·통계 계산, tegrastats 한 줄 파싱(실측 예시 문자열)
- [X] T023 [US3] `pretrained bench`(과제 3·4 학습 종료 뒤): 사전학습 크기 3 + 과제 3 최종 모델 × GPU·CPU × 배치 1·3

---

## Phase 5: User Story 4·5 — 판단과 생각해 볼 점

- [X] T024 [US4] `src/pretrained/budget.py`: 관찰값(`tegrastats` RAM, 수집기·다른 계정 RSS)과 가정 LLM(2B·4B·8B), 측정값으로 예산표·판정
- [X] T025 [US4] 판단표(정확도·적중률·지연·메모리)와 채택 의견 초안
- [X] T026 [US5] Q1 정보 차이 분해: 과제 3 `stages.json` D2 검증 결과 + 사전학습 검증 결과
  - 선택: 과제 3 설정으로 CO₂만 입력 모델 1개 추가 학습(과제 3 실험 종료 뒤, 검증 비교용)
- [X] T027 [US5] Q2 컨텍스트 길이 곡선(검증)과 해석

---

## Phase 6: User Story 6 — 심화 (P3)

- [X] T028 [US6] 공변량 모델: Chronos-2에 온습도(과거)·시간·일과표(과거 + 미래 12칸, `forecast.features`)를 넣는다
  - 검증으로 컨텍스트 선택 → 시험에서 공변량 있음/없음 비교
  - 지원하지 않으면 대체 모델을 research에 기록하고 진행한다
- [X] T029 [US6] 섞은 예측: 검증으로 고정한 w(하나·공간 × 지평별 중 검증 우세)로 시험 평가. 직접·사전학습 단독과 비교한다

---

## Phase 7: Polish & 산출물

- [X] T030 `src/pretrained/report.py` + `pretrained report`: 결과 표(`tables/`)·그래프 — 표 코드 완료, 그래프는 실측 뒤 추가
  - 지평별 MAE
  - 적중률 막대
  - 보정 곡선
  - 지연–정확도 산점도
  - 예산 막대
- [X] T031 산출물 `docs/deliverables/task6/01-comparison-adoption.md`
  - 비교표, 판단표, **채택 의견 반 쪽**
  - 측정 조건: 데이터 기간, 실데이터, 시험 기간, 전력 모드, 클럭 고정 안 함, 반복·워밍업, 동시 작업
- [X] T032 심화 문서 `docs/advanced/task6/01-covariates.md`, `02-ensemble.md`
- [X] T033 점검 문서 `docs/tasks/task6-pretrained-models.md`
  - 해결할 것 5, 확인 기준 2, 산출물, 생각해 볼 점 3, 심화 2를 대응시킨다
  - **확인 명령**은 실행해 맞춘 뒤 적는다
- [X] T034 [P] `docs/deliverables/README.md`, `docs/README.md`, `.github/README.md`, `CLAUDE.md`를 갱신한다
- [X] T035 전체 시험을 실행하고 두 저장소를 커밋·푸시한다(가중치·예측은 `var/`라 제외, 라이선스 표기 확인)

---

## Dependencies & Execution Order

- Setup → Foundational(T005~T014)
- T015~T018(검증 예측·선택)은 언제든 할 수 있다. GPU를 쓰면 과제 3·4 학습과 겹치지 않게 하거나 CPU로 한다.
- T019(시험)는 T018 고정 **그리고** 과제 3 `forecast test` 뒤에 한다(직접 학습 열).
- T023(비용)은 과제 3·4 학습 종료 뒤에 한다(측정 오염 방지).
- T024~T029 → Polish

## Implementation Strategy

1. **MVP**: Bolt small 하나, 컨텍스트 4시간 고정 → 검증 예측 → 선택 고정 → 시험 비교표·적중률. 확인 기준 1번 틀을 먼저 완성한다.
2. 크기 3 × 컨텍스트 4 선택, 비용 측정, 판단표·채택 의견.
3. 심화(공변량·앙상블) → 문서.
