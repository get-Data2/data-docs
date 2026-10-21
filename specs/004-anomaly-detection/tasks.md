---
description: "과제 4 이상 탐지 작업 목록"
---

# Tasks: 이상 탐지 (과제 4)

**Input**: `specs/004-anomaly-detection/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다(헌법 IV). 확인 기준 계산(탐지율·헛경보·매칭)과 임계값 누수(평가판을 바꿔도 임계값 동일)를 자동 시험으로 검증한다. 마커는 `unit`이다.

**Organization**
- US1(평가판)·US2(모델)·US3(임계값)·US4(규칙)·US5(성적표·운영 제안) = P1, US6(심화) = P3
- 실행 순서: 기반 → US1 → US4 → US2 → US3 → US5 → US6 → 마무리
- 규칙은 GPU가 필요 없어 모델보다 먼저 한다. **오토인코더 학습(GPU)은 과제 3 실험이 끝난 뒤** 한다(research R9).

## Format: `[ID] [P?] [Story] Description`

- 경로는 저장소 루트 기준. 소스 `src/anomaly/`, 시험 `tests/anomaly/`, 설정 `config/anomaly.yaml`, 작업 `var/anomaly/`

---

## Phase 1: Setup

- [X] T001 [P] `config/anomaly.yaml`을 contracts/config.md 그대로 작성한다
- [X] T002 [P] `tools/anomaly` 래퍼(`tools/forecast`와 같은 GPU 그룹 처리·환경 변수)를 만들고 `~/.local/bin/anomaly`에 링크한다
- [X] T003 [P] `tests/anomaly/__init__.py`, `conftest.py`를 작성한다
  - 고정물 `acfg4`: 짧은 합성 분할(과제 3 시험 고정물과 같은 기간)
  - `synth_sensors()`: 공간 3, 실습실 CO₂ 3대·온도 3대, 회의실 10분 주기. 평일 일과 패턴과 잡음이 있다

## Phase 2: Foundational

- [X] T004 `src/anomaly/config.py`: 로드·검증(contracts/config.md 검증 규칙), 설정 해시, `analysis` 분할·일과표 연결
- [X] T005 `src/anomaly/data.py`
  - `wide(sensor, space, cols, t0, t1)`: 공간 넓은 표, 관측만, 빈 칸 NaN
  - `sensor_stats(train)`: 열마다 평균·표준편차·이웃 차이 MAD·5~95% 범위
  - `space_median(board)`: 평가판 → 공간 대표값(과제 2 규칙: 장치 칸 평균의 중앙값, 제외 장치 반영)
  - `usage_mask(t)`: `analysis.baseline.usage_mask` 재사용
- [X] T006 `src/anomaly/alarms.py`
  - `to_quantile(score_test, score_val)`: 검증 분포 대비 분위수
  - `alarms_from(q, thr, merge_min)`: 연속 구간 → 경보, 30분 안 병합, 최고 q와 그 칸의 원인
  - `choose_threshold(val_q_by_space, candidates, target_per_day, days)`: 목표 이하인 최저 임계값과 후보 표
  - `combine(*alarm_sets)`: 합집합 후 다시 병합(원인은 최고 q 쪽)
  - `freeze_thresholds()`, `require_thresholds()`: 해시 검사
- [X] T007 `src/anomaly/evaluate.py`
  - `match(alarms, faults, slack, f3_h)`: 경보 → 고장, 지연, 헛경보
  - `scorecard()`: 탐지기 × 종류(n, 잡음, 탐지율, 지연 중앙·최대, 원인 정확도)
  - `false_per_day()`: 전체·실제 이상 후보 제외
  - `criteria()`: 결합 ≥ 80%, 헛경보 ≤ 1/일
  - `curve()`: 임계값 후보별 탐지율·헛경보·지연
- [X] T008 [P] `tests/anomaly/test_alarms.py` (unit)
  - 병합(30분 안 두 구간 → 하나)
  - 분위수 변환
  - **임계값 누수**: 시험 점수를 바꿔도 `choose_threshold` 결과가 같다(검증 점수만 입력)
  - `require_thresholds`: 파일 없음·변조 시 거부
- [X] T009 [P] `tests/anomaly/test_evaluate.py` (unit)
  - 손으로 만든 고장 3건과 경보 4개에서 탐지율·지연·헛경보·원인 정확도를 손 계산값과 맞춘다
  - F3 인정 구간 24시간
  - 사건 후보 겹침 표시
- [X] T010 `src/anomaly/__main__.py`: 하위 명령 뼈대(contracts/cli.md), `check`, `info`

---

## Phase 3: User Story 1 — 평가판 (P1)

- [X] T011 [US1] `src/anomaly/inject.py`: F1~F5 주입(research R2)
  - 후보 시각 뽑기: 관측 칸, 간격, 사용 시간대, 시험 기간 안
  - 크기 무작위(씨앗)
  - F2는 장치 전 항목, F3은 경사 + 24시간 유지, F5는 아침 기준 상승분 ×1.8
  - 정답 표·평가판 저장, 해시 기록
- [X] T012 [P] [US1] `tests/anomaly/test_inject.py` (unit)
  - 같은 씨앗 → 같은 정답 표
  - `fault_id == 0`인 행은 원본과 같다
  - F2 값이 정상 범위 안이다
  - F4·F5는 사용 시간대·수업일에 시작한다
  - 같은 공간 간격 ≥ 12시간
  - 종류마다 건수가 정확하다
- [X] T013 [US1] `anomaly inject`를 실데이터로 실행하고, 종류 × 공간 건수와 해시를 기록한다

---

## Phase 4: User Story 4 — 규칙 탐지 (P1)

- [X] T014 [US4] `src/anomaly/rules.py`: 급변(앞뒤 반대 방향 k·s), 값 고정(참조 장치 변화 / 단독 사용 시간대). 점수 0/1과 원인 장치
- [X] T015 [P] [US4] `tests/anomaly/test_rules.py` (unit)
  - 합성 튐 1칸 → 급변 경보, 지연 5분
  - 단조 상승(수업 시작)은 급변이 아니다
  - 장치 고정 2시간 + 이웃 변함 → 값 고정
  - 밤의 습도 양자화 반복(이웃도 안 변함) → 경보 없음
- [X] T016 [US4] 검증·평가판 규칙 점수(`anomaly score rule_spike rule_stuck`)

---

## Phase 5: User Story 2 — 오토인코더 (P1, GPU — 과제 3 실험 뒤)

- [X] T017 [US2] `src/anomaly/autoencoder.py`
  - 창 데이터: 정규화 값·관측 표시·시간 특징, 학습 제외 구간(B0·값 고정 후보)
  - 1D 합성곱 AE
  - 마스크 MSE, 조기 종료(검증)
  - 점수: 창 끝 3칸 열 오차, 열별 검증 중앙값으로 나눔, 합. 상위 3 원인
- [X] T018 [P] [US2] `tests/anomaly/test_autoencoder.py` (unit): 형상, 마스크 칸 무시, 큰 튐 주입 시 그 열이 1순위 원인
- [X] T019 [US2] `anomaly train`을 실데이터로 실행한다(학습 시간·메모리 기록). 이어서 `anomaly score ae`

---

## Phase 6: User Story 3 — 임계값 (P1)

- [X] T020 [US3] `anomaly threshold`: 탐지기별 검증 기준 선택·고정. 후보 표(검증 하루당 경보)를 남긴다
- [X] T021 [US3] 검증 기간 경보 중 과제 2 사건 후보와 겹치는 수를 기록한다(실제 이상 가능성)

---

## Phase 7: User Story 5 — 성적표·운영 제안 (P1)

- [X] T022 [US5] `anomaly evaluate`: 모델 / 규칙 / 결합 경보, 매칭, 성적표, 판정표, 임계값 곡선을 낸다(`results.json`)
- [X] T023 [US5] 미탐 원인 분석
  - 종류별로 놓친 고장의 사례를 본다(크기, 시간대, 공간, 원인 열)
  - 모델이 못 잡는 종류와 규칙이 못 잡는 종류를 정리한다(확인 기준 3)
- [X] T024 [US5] 운영 제안 표: 공간별 임계값, 알림 대상(센서 고장 → 센서 관리 담당, 설비 → 시설 담당, 공기질 → 공간 사용자), 예상 하루 경보 수
- [X] T025 [US5] `src/anomaly/bundle.py`: 과제 8용 묶음

---

## Phase 8: User Story 6 — 심화 (P3)

- [X] T026 [US6] `src/anomaly/forecast_err.py`: 평가판 공간 대표값 → 과제 3 묶음으로 15분 예측 → 정규화 잔차 점수. 같은 임계값·지표 (과제 3 `forecast test` 뒤)
- [X] T027 [US6] `src/anomaly/drift.py`: 새벽 기준 + 이웃 비교. F3 성적·헛경보를 기본 탐지기와 비교
- [X] T028 [P] [US6] `tests/anomaly/test_drift.py` (unit): 합성 영점 +300ppm 경사 → 이웃 비교 경보, 다음 새벽 기준 경보. 정상 계절 변화에서는 경보 없음

---

## Phase 9: Polish & 산출물

- [X] T029 `src/anomaly/report.py` + `anomaly report`
  - 결과 표(`tables/`)와 그래프: 임계값–탐지율·헛경보 곡선, 종류별 지연 분포, 대표 고장 사례
- [X] T030 산출물 본문을 쓴다
  - `docs/deliverables/task4/01-scorecard.md`(성적표·판정·미탐 이유)
  - `02-threshold-tradeoff.md`(곡선·임계값 근거)
  - `03-operations.md`(운영 제안)
  - 측정 조건과 해석을 단다
- [X] T031 심화 문서 `docs/advanced/task4/01-forecast-error-detector.md`, `02-drift-detector.md`
- [X] T032 점검 문서 `docs/tasks/task4-anomaly-detection.md`
  - 해결할 것 5, 확인 기준 3, 산출물 3, 생각해 볼 점 3, 심화 2를 대응시킨다
  - **확인 명령**은 실행해 맞춘 뒤 적는다
- [X] T033 [P] `docs/deliverables/README.md`, `docs/README.md`, `.github/README.md`, `CLAUDE.md`를 갱신한다
- [X] T034 전체 시험을 실행하고 두 저장소를 커밋·푸시한다

---

## Dependencies & Execution Order

- Setup → Foundational(T004~T010) → US1(T011~T013) → US4(T014~T016)
- US2(T017~T018 코드는 언제든, **T019 학습은 과제 3 실험 종료 뒤**) → US3(T020~T021) → US5(T022~T025)
- US6: T027·T028은 언제든, T026은 과제 3 `forecast test`(모델 묶음) 뒤
- Polish 마지막

## Implementation Strategy

1. **MVP**: 평가판 + 규칙 + 임계값 + 평가. GPU 없이 확인 기준 판정 틀을 먼저 완성한다(규칙만의 성적).
2. 오토인코더(과제 3 뒤)를 더해 모델 / 규칙 / 결합 성적표를 완성한다.
3. 심화 2개 → 문서.
