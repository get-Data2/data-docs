---
description: "과제 5 상태 추정 작업 목록"
---

# Tasks: 상태 추정 (과제 5)

**Input**: `specs/005-state-estimation/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다(헌법 IV). 확인 기준 계산(재현율·macro-F1·비어있음→사용중 오류율)과 누수·선택 고정을 자동 시험으로 검증한다. 마커는 `unit`이다.

**Organization**
- US1(정답)·US2(관찰 기록)·US3(사용 여부)·US4(환기) = P1, US5(오류 분석) = P2, US6(심화) = P3
- 실행 순서: 기반 → US2(관찰 기록을 먼저 열어 사람이 기록을 시작할 수 있게) → US1 → US4 → US3 → US5 → US6 → 마무리
- **관찰은 사람이 한다.** US2를 먼저 끝내 관찰을 시작하고, 관찰이 모이는 동안 나머지를 진행한다.
- 학습 실행은 과제 3 실험이 끝난 뒤 한다(보드 메모리 공유).

## Format: `[ID] [P?] [Story] Description`

- 경로는 저장소 루트 기준. 소스 `src/state/`, 시험 `tests/state/`, 설정 `config/state.yaml`, 작업 `var/state/`, 관찰 `data/observations.csv`

---

## Phase 1: Setup

- [ ] T001 (`.gitignore`의 `data/`는 완료, pyproject 수정은 상위 작업자가 모아서) `pyproject.toml`에 extra `state = ["scikit-learn>=1.9"]`를 추가하고 `-e '.[test,analysis,forecast,state]'`로 설치한다(torch를 바꾸지 않는지 확인). `.gitignore`에 `data/`를 추가한다
- [X] T002 [P] `config/state.yaml`을 contracts/config.md 그대로 작성한다
- [X] T003 [P] `tools/state` 래퍼(`tools/forecast`와 같은 GPU 그룹 처리·환경 변수)를 만들고 `~/.local/bin/state`에 링크한다. 이름 충돌을 확인한다(`which state`)
- [X] T004 [P] `tests/state/__init__.py`, `conftest.py`를 작성한다
  - 고정물 `scfg`: 짧은 합성 분할
  - `synth_space()`: 공간 3개, 평일 일과 CO₂ 상승과 1,000ppm 넘는 날·안 넘는 날, 회의실 10분 주기
  - `synth_obs()`: 관찰 CSV 예시

## Phase 2: Foundational

- [X] T005 `src/state/config.py`: 로드·검증(contracts/config.md 규칙, 특히 occupancy 특징에 time·schedule 금지), 해시, `analysis` 분할·일과표 연결
- [X] T006 `src/state/features.py`
  - 공통 전처리 위의 인과적 특징 묶음(level·delta·shape·obs·time·schedule)
  - 관측 칸만, 앞으로 채우기, 입력 관측 비율
  - `src/preprocess/`가 아직 없으면 `forecast.data`·`analysis` 함수를 직접 부르고, 생기면 import만 바꾼다(과제 8 T006과 맞춤)
- [X] T007 [P] `tests/state/test_features.py` (unit)
  - **인과성**: t 이후 값을 바꿔도 t의 특징이 같다
  - occupancy 묶음에 시각·일정 열이 없다
  - 10분 주기 공간의 변화량은 관측 칸 기준이다
- [X] T008 `src/state/models.py`: HGB·로지스틱·MLP 공통 인터페이스(fit·predict_proba·save·load), 클래스 가중, MLP는 PyTorch(결정성 설정 `forecast.env`)
- [X] T009 `src/state/select.py`
  - 검증 선택(1표준편차 안 단순한 쪽)
  - 임계값: 환기는 재현율 ≥ 0.85 중 정밀도 최대(미달 표시), 사용 여부는 macro-F1 최대
  - `freeze_selection()`, `require_selection()`
- [X] T010 [P] `tests/state/test_select.py` (unit)
  - 임계값 규칙을 손으로 만든 점수로 확인한다
  - 선택 전 평가 거부, 변조 거부
  - **누수**: 시험 기간 값을 바꿔도 선택·임계값·모델 해시가 같다
- [X] T011 `src/state/__main__.py`: 하위 명령 뼈대(contracts/cli.md), `check`, `info`

---

## Phase 3: User Story 2 — 직접 관찰 기록 (P1) 🎯 사람이 먼저 시작할 수 있게

- [X] T012 [US2] `src/state/observe.py`
  - CSV 읽기·검증(겹침·모르는 공간·끝 ≤ 시작, 줄 번호 오류)
  - 덧붙이기(덮어쓰기 없음): 열린 구간 닫기·새로 열기, `stop`, `--at`
  - `--status`(공간별 시간·비율·경계 포함 횟수·목표 진행률)
  - `--check`(문 열림과 어긋나는 구간 표시, research R3)
- [X] T013 [P] [US2] `tests/state/test_observe.py` (unit): 전환 기록 → 구간 닫힘, 겹침 거부, 진행률 계산, 경계 포함 판정
- [X] T014 [US2] 관찰 안내 문서 `docs/deliverables/task5/observation-guide.md`
  - 목표량, 경계 시각, 명령 예시, 기록 시 주의(문 열어 둠·창문 열림은 note)
  - 이 문서를 사용자에게 전달하고 관찰 시작을 요청한다

---

## Phase 4: User Story 1 — 정답 정의 (P1)

- [X] T015 [US1] `src/state/labels.py`
  - 사용 여부 일정 기반 정답(일과표·추정 예약·실제 예약 CSV, 출처 열)
  - 관찰 정답 붙이기(칸 안 다수 상태)
  - 환기 라벨(양성·음성·이미 초과·판단 불가)
- [X] T016 [P] [US1] `tests/state/test_labels.py` (unit)
  - 999 → 30분 안 1,001: 양성
  - 1,000에서 시작: 이미 초과(제외)
  - 30분 뒤 31분에 초과: 음성
  - 미래 관측 부족: 판단 불가
  - 공휴일 일정 = 비어있음
  - 관찰 칸 다수 상태
- [ ] T017 [US1] `state labels`를 실데이터로 실행한다. 양성 비율(공간·분할별), 이미 초과 비율, 관찰과 일정 정답 불일치 표(관찰이 있으면)를 기록한다

---

## Phase 5: User Story 4 — 환기 선제 경보 (P1)

- [X] T018 [US4] `src/state/train.py`: 후보 × 격자 × 씨앗 학습, 검증 예측·지표 저장, 끝난 것 건너뜀, 실패 기록 후 계속
- [ ] T019 [US4] `state features && state train vent`를 실데이터로 실행한다(과제 3 실험 종료 뒤). 학습 시간·메모리를 기록한다
- [X] T020 [US4] (코드 `src/state/pipeline.py`·`forecast_vent.py`, 실데이터 실행은 T025) 환기 평가
  - 재현율·정밀도·F1·macro-F1·PR 곡선·AP·하루당 헛경보·선행 시간, "항상 음성" 정확도
  - 단순 규칙(CO₂ ≥ 850)
  - 과제 3 판정(예측 30분 최댓값 ≥ 1,000 − δ, δ는 검증 재현율 기준) — 과제 3 `forecast test` 뒤

---

## Phase 6: User Story 3 — 사용 여부 (P1)

- [ ] T021 [US3] `state train occupancy leak`: 사용 여부 후보와 누수 비교 실험(일정 입력 포함)
- [ ] T022 [US3] `state select`로 두 문제의 선택을 고정한다(누수 실험은 선택 대상 아님)
- [X] T023 [US3] (코드 `src/state/pipeline.py`, 실데이터 실행은 T025) 사용 여부 평가
  - 정답 기준 두 가지(관찰·일정)별 정확도·F1·macro-F1·비어있음→사용중 오류율·혼동 행렬
  - 단순 규칙(15분 +30ppm), 누수 실험 비교(SC-006)
  - 관찰이 목표 미달이면 "판단 보류" + 진행률
- [X] T024 [P] [US3] `tests/state/test_select_pipeline.py` (unit): 손으로 만든 예측·정답으로 지표·오류율·판정표를 대조한다. 관찰 부족 시 판단 보류
- [ ] T025 [US3] `state evaluate`를 실행한다. 판정표를 확인하고 `var/state/bundle/`(bundle.json: 모델 종류·파일, 입력 규약 버전, 특징 순서, 정규화, 임계값, 해시)을 저장한다

---

## Phase 7: User Story 5 — 오류 분석 (P2)

- [X] T026 [US5] `src/state/analyze.py`(코드, 실행은 T027)
  - 전환 뒤 경과별 오류율(Q1)
  - 인원별 재현율(Q2, 관찰에 인원이 있을 때)
  - 시간대 × 공간 × 상황 상위 혼동 3개와 사례
  - 환기 헛경보 하루당 수·시각대(Q3)
  - 그래프
- [ ] T027 [US5] `state analyze` 실행, 해석 메모

---

## Phase 8: User Story 6 — 심화 (P3)

- [X] T028 [US6] `src/state/analyze.py`(unseen_space) 처음 보는 공간: 공간 하나를 학습·검증·정규화에서 빼고 학습해 평가한다. 세 공간을 모두 돌리고 포함 학습과 비교한다
- [X] T029 [US6] 노쇼·예약 없는 사용 규칙
  - 예약 시작 뒤 15분 비어있음, 예약 밖 15분 이상 사용중
  - 관찰·실제 예약과 대조한 결과만 성능으로 보고하고, 추정 예약 결과는 동작 예시(순환)로만 쓴다
- [X] T030 [P] [US6] `tests/state/test_advanced.py` (unit): 합성 예약·사용 여부로 노쇼·예약 없는 사용 판정

---

## Phase 9: Polish & 산출물

- [X] T031 `src/state/report.py` + `state report`: 결과 표(`tables/`)·그래프(혼동 행렬, PR 곡선, 전환 뒤 오류, 사례)
- [ ] T032 산출물 본문을 쓴다
  - `docs/deliverables/task5/01-scorecards.md`(두 분류기 성적표·혼동 행렬·판정)
  - `02-label-vs-observation.md`(일정 정답과 관찰의 차이)
  - `03-error-cases.md`(가장 많이 틀리는 상황)
  - 측정 조건과 해석을 단다
- [ ] T033 심화 문서 `docs/advanced/task5/01-unseen-space.md`, `02-noshow-unbooked.md`
- [ ] T034 점검 문서 `docs/tasks/task5-state-estimation.md`
  - 해결할 것 5, 확인 기준 3, 산출물 3, 생각해 볼 점 4, 심화 2를 대응시킨다
  - **확인 명령**은 실행해 맞춘 뒤 적는다
- [ ] T035 [P] `docs/deliverables/README.md`, `docs/README.md`, `.github/README.md`, `CLAUDE.md`를 갱신한다
- [ ] T036 전체 시험을 실행하고 두 저장소를 커밋·푸시한다. 관찰 CSV(`data/`)가 커밋되지 않았는지 확인한다

---

## Dependencies & Execution Order

- Setup(T001~T004) → Foundational(T005~T011) → **US2(T012~T014)를 먼저 끝내 관찰 시작**
- US1(T015~T017) → US4(T018~T020) → US3(T021~T025) → US5(T026~T027) → US6(T028~T030) → Polish(T031~T036)
- 외부 의존
  - T019·T021 학습은 과제 3 실험 종료 뒤
  - T020의 과제 3 판정은 `forecast test`(모델 묶음) 뒤
  - 관찰 기준 판정(T023)은 관찰 목표량이 모인 뒤. 그 전에는 일정 기반 결과와 "판단 보류"로 기록
- 과제 8은 T025의 `var/state/bundle/`, 과제 7은 최종 분류기가 MLP일 때 그 묶음을 쓴다

## Parallel Examples

- T002 · T003 · T004 동시
- T007 · T010 · T013 · T016 시험은 각 구현 직후 병렬
- 과제 3 실험이 도는 동안 T005~T018 코드와 합성 시험을 진행

## Implementation Strategy

1. **먼저 관찰 기록 도구(US2)**: 사람이 기록을 시작해야 확인 기준 1을 판정할 수 있으므로 가장 먼저 연다.
2. **MVP**: 정답 + 환기 분류기 + 규칙 비교. 관찰 없이도 확인 기준 2(재현율 85%, macro-F1)를 판정할 수 있다.
3. 사용 여부 분류기와 누수 실험(일정 기반 결과), 관찰이 모이면 관찰 기준 판정을 추가한다.
4. 오류 분석 → 심화 → 문서.
