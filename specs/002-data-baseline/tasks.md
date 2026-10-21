---
description: "과제 2 데이터 이해와 기준선 작업 목록"
---

# Tasks: 데이터 이해와 기준선 (과제 2)

**Input**: `specs/002-data-baseline/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다. 헌법 IV에 따라 확인 기준(학습 기간만으로 정규화, 같은 시점 비교, 검산, 축 분리)을 pytest로 검증한다.
- 합성 데이터 시험: 마커 `unit`
- 실 DB 추출 시험: 마커 `mysql`

**Organization**: spec의 US1(정제 시계열)·US2(진단서)·US3(기준선)·US4(분할·누수)는 모두 P1이다.
의존 관계 때문에 **US1 → US4 → US3 → US2** 순서로 진행한다(기준선과 진단은 분할이 필요하다).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 병렬 가능(다른 파일, 미완료 작업 의존 없음)
- 경로는 저장소 루트 기준. 소스 `src/analysis/`, 시험 `tests/analysis/`, 설정 `config/analysis.yaml`, 결과 `docs/reports/task2/`

---

## Phase 1: Setup

- [X] T001 `pyproject.toml`에 extra `analysis = ["pandas>=3.0", "pyarrow>=26", "matplotlib>=3.11", "holidays>=0.107"]` 추가하고, 패키지 탐색에 `analysis`가 포함되는지 확인한 뒤 `~/.local/bin/uv pip install --python .venv/bin/python -e '.[test,analysis]'`로 설치
- [X] T002 [P] `tools/analysis` 실행 래퍼 작성(`tools/collector`와 같은 방식: 프로젝트 폴더로 이동해 `.venv/bin/python -m analysis "$@"`). 실행 권한을 주고 `~/.local/bin/analysis`에 링크
- [X] T003 [P] `config/analysis.yaml`을 contracts/config.md 예시 그대로 작성(점심 11:30~13:00 반영, `split.active: v1-provisional`)
- [X] T004 [P] `tests/analysis/__init__.py`, `tests/analysis/conftest.py` 작성 — 고정물 `acfg`(분석 설정), `make_raw(rows)`(원본 사본 형식 DataFrame 생성: ts UTC, space_id, device_id, item, value, source), `periods`(과제 1 설정에서 읽은 장치별 기대 주기)

---

## Phase 2: Foundational

**⚠️ 스토리 작업 전 필수**

- [X] T005 `src/analysis/config.py`: YAML 로드·검증(분할 순서·겹침·경계 1일 간격, 공간 targets 존재 경고). 과제 1 `config/collector.yaml`에서 장치별 `expected_period_s`와 `periodic`을 읽는다. 설정 해시(sha256, 정렬된 JSON)를 계산한다. 오류는 `파일: 경로: 이유` 형식
- [X] T006 `src/analysis/extract.py`: `collector.store.connect`로 접속해 달마다 한 번씩 읽는다.
  - 쿼리: `SELECT ts, space_id, device_id, item, value, source FROM measurement WHERE item IN (...) AND ts >= 월초 AND ts < 다음 달 초`
  - 저장: `var/cache/raw/YYYY-MM.parquet`(ts는 timestamp[ms, UTC], 문자열 열은 category)
  - 기록: `var/cache/manifest.json`에 `{rows, sha256, extracted_at, closed}`. 지난 달은 `closed=true`이면 건너뛰고, `--refresh`면 다시 읽는다
  - 결과: 월별 행 수 표를 돌려준다
- [X] T007 `src/analysis/__main__.py`: argparse CLI 골격. 명령은 `extract`·`clean`·`events`·`split`·`diagnose`·`baseline`·`all`·`info`, 공통 `--config`·`--split`. 종료 코드는 0 / 2 설정 / 3 DB / 4 데이터 부족(contracts/cli.md)
- [X] T008 [P] `tests/analysis/test_config.py`: 정상 설정 로드, 분할 겹침·순서·경계 간격 위반 시 오류, 과제 1 주기 읽기(`24e124126d152590` → 120, 문 센서 → periodic False)

**Checkpoint**: `analysis extract` 결과 `var/cache/raw/`에 2025-08 ~ 2026-10 사본, `analysis info`에 월별 행 수가 나온다.

---

## Phase 3: User Story 1 — 믿을 수 있는 분석용 시계열 (P1) 🎯 MVP

**Goal**: 축 A 품질 표시, 결측 판정·짧은 결측 보간, 5분 공간 중앙값 시계열, 축 B 사건·의심·값 고정 목록.

**Independent Test**: 합성 원본으로 품질 표시·보간·공간 값이 손 계산과 같고, `ok + saturated + error = 원본 행 수`가 성립한다.

### Tests

- [X] T009 [P] [US1] `tests/analysis/test_quality.py`
  - A1: CO₂ 65535·0 → error
  - A2: 같은 장치·시각의 습도 127.5와 온도 −0.1 → 둘 다 error. 한쪽만 있으면 error가 아니다
  - A3: 범위 밖 → error
  - A4: CO₂ 5000 → saturated
  - 원본 행 수가 보존된다(SC-006)
- [X] T010 [P] [US1] `tests/analysis/test_gaps.py`
  - 60초 장치 간격 150초 → 결측, 110초 → 결측 아님
  - 600초 장치 간격 900초 → 결측 아님
  - 짧은 결측 10분 → 선형 보간 + filled 표시
  - 20분 결측 → 보간하지 않음
  - 인위 결측 시험 함수가 길이별 MAE 표를 돌려준다
- [X] T011 [P] [US1] `tests/analysis/test_series.py`
  - 센서 3대의 5분 칸 평균 → 중앙값과 `n_sensors`
  - `exclude_from_space` 센서는 빠진다
  - error 값은 빠지고 saturated는 남는다
- [X] T012 [P] [US1] `tests/analysis/test_axes.py` — 합성 시나리오 4개(SC-007)
  - 화재: 실습실 센서 2대 온도가 30분에 25 → 45℃, 습도 하락, 뒤에 끊김 → 축 A error 0, B1 1건
  - 오류 코드 쌍 → A error, 사건 0
  - 단독 튐(센서 1대 1칸 +800ppm) → B0
  - 밀집: CO₂ 센서 2대가 3000ppm으로 15분 → B3
  - SC-008: `axis_a` 설정을 바꿔도 `events.detect`의 입력 계약이 같다(축 B는 품질 표시만 소비)

### Implementation

- [X] T013 [US1] `src/analysis/quality.py`: `apply_axis_a(raw, cfg) -> DataFrame[ts, device_id, item, quality, rule]`. 원본은 바꾸지 않는다
  - A1 `error_codes`
  - A2 `error_pairs`: device_id·ts로 짝을 맞춘다
  - A3 `ranges` + `models` 덮어쓰기(기종은 `device` 테이블의 model을 extract에서 함께 받거나 별도 조회)
  - A4 `saturation`
- [X] T014 [US1] `src/analysis/gaps.py`
  - `find_gaps(device_ts, period_s, factor=2)`
  - `interpolate_short(sensor_bins, max_min=15)`: 같은 장치 내 선형 보간, filled 표시
  - `interpolation_test(series, lengths=[5,10,15,30,60], rng_seed=0)`: 결측 없는 학습 구간에 인위 결측을 넣고 길이별 MAE와 칸 안 잡음(표준편차 중앙값)을 낸다
- [X] T015 [US1] `src/analysis/series.py`
  - 처리: 센서별 5분 칸 평균(error 제외) → 짧은 결측 보간 → 공간별 중앙값
  - 결과 열: `bin, space_id, item, value, n_sensors, filled`
  - 저장: `var/series/space_5min.parquet`
  - `--interval` 후보(1·5·15분)로도 만들 수 있게 매개변수화한다
- [X] T016 [US1] `src/analysis/events.py`
  - `detect(series_by_sensor, quality, cfg, train_stats) -> events DataFrame[space_id, kind, start, end, devices, evidence, rule_version]`
    - B1 화재(센서 2대 이상, 30분 상승 ≥5℃, 습도 하락, 이후 끊김·포화 가산)
    - B2 누수(습도 ≥95% 30분)
    - B3 밀집(max(2500, 학습 p99.9), 센서 2대 이상 또는 1대 3칸)
    - B0(공간 중앙값 대비 학습 MAD×10)
  - `stuck_candidates(...)`: 2시간 이상 같은 값이고 같은 공간 다른 센서는 변한 경우
  - 저장: `var/cache/events.parquet`, `docs/reports/task2/events.md`
- [X] T017 [US1] `src/analysis/__main__.py`에 `clean`·`events` 연결
  - `clean`은 규칙별 건수 표와 검산 줄(`ok N + saturated N + error N = 원본 N`)을 출력한다
  - 학습 기간 통계(p99.9, MAD)는 분할 버전의 학습 기간에서만 계산한다(T019에서 받는다. 없으면 분할 먼저 실행)

**Checkpoint**: `analysis clean && analysis events` 실행 결과 검산 일치, 실데이터 오류 코드 건수가 과제 1 조사(65535 1,067건, 쌍 402건)와 일치.

---

## Phase 4: User Story 4 — 시간 순 분할과 누수 방지 (P1)

**Goal**: 분할 버전 고정, 학습 기간만으로 정규화 통계, 자기상관 근거.

**Independent Test**: 시험 기간 값을 바꿔도 정규화 통계와 학습 기반 임계값이 같다.

- [X] T018 [P] [US4] `tests/analysis/test_split.py`
  - 분할 경계 계산은 KST 날짜 → UTC 구간
  - 시험 기간 값을 ×10 해도 `norm_stats`·`train_quantile`·MAD가 같다(SC-002)
  - 학습 기간에 시험 기간 칸이 하나도 섞이지 않는다
- [X] T019 [US4] `src/analysis/split.py`
  - `periods(cfg, version) -> {train, val, test: (start_utc, end_utc)}`
  - `fit_train_stats(series, train)`: 공간×항목 평균·표준편차·p99.9·MAD, 칸 수, 학습 구간 데이터 해시. 저장은 `var/eval/norm_<split>.json`
  - `autocorr(series, lags=[1, 12, 288])`
- [X] T020 [US4] `__main__.py`에 `split` 연결. 분할 기간별 칸 수·데이터 비율 표를 출력하고, 학습·검증·시험 중 어느 하나라도 대상 공간 데이터가 없으면 종료 코드 4와 이유

---

## Phase 5: User Story 3 — 기준선 성능표 (P1)

**Goal**: 평가 시점 파일(과제 3~6 공유)과 기준선 3종 MAE 표.

**Independent Test**: 같은 평가 시점 파일로 MAE를 다시 계산하면 같은 표가 나오고, 세 기준선의 시점 수가 같다.

- [X] T021 [P] [US3] `tests/analysis/test_baseline.py`
  - 합성 사인파 + 일주기 시계열에서 손 계산과 일치
    - 현재값 유지 = y(t0)
    - 1시간 평균 = mean(y[t0−55분..t0])
    - 어제 = y(t0+h−24h)
  - 평가 시점 조건: 정답이 보간 칸이면 제외, 1시간 칸 50% 미만이면 제외, 어제 값 없으면 제외. 세 방법 시점 집합이 같다(SC-004)
  - `usage_hours`: 평일 08~19시, 공휴일 제외. 2026-10-09 한글날이 제외되는지 확인
- [X] T022 [US3] `src/analysis/baseline.py`
  - `build_eval_points(series, test, cfg) -> DataFrame[space_id, item, t0, horizon_min, y_true, usage_hours]`
  - 메타데이터: `var/eval/eval_points_<split>.parquet` + `.json`(split, 행 수, sha256, 설정 해시, 생성 시각)
  - `predict_baselines(series, points)`
  - `mae_table(preds)`: 공간 × 항목 × 지평 × 방법, 전체·사용 시간대, 평가 시점 수
- [X] T023 [US3] `baseline` 명령 연결 → `var/eval/baseline_<split>.parquet`, `docs/reports/task2/baseline_<split>.md`
  - 문서 머리에 측정 조건: 데이터 기간, 실데이터, 분할, 분석 간격, 설정 해시, 평가 시점 sha256
  - 지평별 1위 방법 요약 줄
- [X] T024 [US3] `info` 명령: 사본·시계열·분할·평가 시점 파일의 존재·행 수·sha256, 기준선 결과가 쓴 평가 시점 sha256과 일치하는지 표시

---

## Phase 6: User Story 2 — 데이터 진단서 (P1)

**Goal**: 1~2쪽 진단서 — 정제 규칙·분할·패턴 그래프·일정–CO₂ 반응·결측·간격 비교·보간 시험.

**Independent Test**: `analysis diagnose`로 모든 표·그래프를 다시 만들 수 있고 숫자가 정제 결과와 같다.

- [X] T025 [P] [US2] `tests/analysis/test_schedule.py`
  - 일과표 행 → 날짜별 사건 시각(09:00·11:30·13:00·18:00 KST, 공휴일 제외)
  - 합성 CO₂가 09:07에 +40ppm → 반응 지연 7분
  - 60분 내 변화 없으면 무반응
  - 토요일 사용 수준 상승 → 어긋나는 날
- [X] T026 [US2] `src/analysis/schedule.py`: `events_for(date_range, cfg)`, `reaction_lags(series, events, pre_min=15, window_min=60, delta_ppm=30)`, `mismatch_days(...)`. 실제 자료 CSV(`schedule.file`)도 같은 함수로 읽는다
- [X] T027 [US2] `src/analysis/diagnose.py` 표
  - 공간·항목별 결측률(긴 결측 칸 비율)·제거 이상값(축 A error) 수·보간 칸 비율·사건/의심 건수
  - 장치별 기록 주기(간격 중앙값·p99)와 긴 결측 목록
  - 간격 후보 1·5·15분 비교(칸당 값 수, 결측률, 칸 안 잡음)
  - 보간 시험 표
  - 자기상관(1칸·1시간·1일)
  - 반응 지연 분포 요약·어긋나는 날 목록
- [X] T028 [US2] `src/analysis/diagnose.py` 그래프(matplotlib Agg, 폰트 `Noto Sans CJK KR`, `docs/reports/task2/*.png`)
  - 공간별 평일 시간대 평균 CO₂·온도(학습 기간, 일과표 사건 세로선)
  - 주말 비교
  - 반응 지연 히스토그램
  - 자기상관 막대
- [X] T029 [US2] `diagnose` 명령 → `docs/reports/task2/diagnosis_<split>.md`
  - 표·그래프 링크·측정 조건 머리
  - 해석 자리 표시(사람이 쓰는 절)는 비워 두지 말고 자동으로 사실 요약을 넣는다. 예: "평일 09시대 실습실 CO₂ 상승 시작 중앙 N분"

---

## Phase 7: Polish & 산출물

- [X] T030 `all` 명령: extract → clean → split → events → diagnose → baseline 순서. 단계별 걸린 시간을 출력하고 전체 10분 이내(SC-001)를 확인. 메모리 최대치 기록(`resource.getrusage`)
- [X] T031 실데이터로 `analysis all --split v1-provisional` 실행. 결과 숫자를 확인하고 이상한 점은 원인을 찾아 기록
- [X] T032 `docs/tasks/task2-data-baseline.md` 산출물 문서 채우기
  - 해결할 것 5항목·확인 기준 3·산출물 2·생각해 볼 점 2에 결과·측정 조건·해석
  - **확인 명령 표**: 실행해 결과를 맞춘 뒤 기재
- [X] T033 [P] 진단서 해석 절 작성
  - 정제 규칙의 이유
  - 가장 강한 기준선과 이유
  - '어제 같은 시각'이 약한 이유와 강할 수 있는 공간
  - 고장 구간을 학습·평가에 넣을지
- [X] T034 [P] `.github/README.md`·`docs/README.md` 진행 표, `CLAUDE.md` 진행 상황 갱신, 두 저장소 커밋·푸시
- [ ] T035 (선택, 이월) InfluxDB 미적재분(2026-07-23 ~ 10-20)을 적재한 뒤 `analysis extract --refresh && analysis all --split v2-final` 실행 → 2학기 시험 기준선을 추가 비교로 싣는다(과제 1 T055 완료가 선행 조건)
  - 2026-10-21 결정: 유선 랜 단절로 적재 불가 → `v1-provisional`을 과제 3~6 공식 평가 기준으로 채택하고 과제 2를 완료 처리

---

## Dependencies & Execution Order

```text
Phase 1 → Phase 2 → US1(정제) → US4(분할) → US3(기준선) → US2(진단) → Phase 7
                                      └─ 학습 기간 통계(T019)를 US1의 events(T016·T017)가 사용
T035는 과제 1 T055(InfluxDB 적재) 뒤
```

- 테스트(T009~T012, T018, T021, T025)는 각 스토리 구현 전에 작성한다(서로 다른 파일이라 병렬 가능).
- T016 축 B의 학습 기반 임계값은 T019가 필요하다. 그래서 events 실행은 split 뒤에 한다(`all` 순서에 반영).

## Parallel Examples

- Phase 1: T002, T003, T004
- US1: T009, T010, T011, T012(시험 4개 동시 작성)
- Phase 7: T033, T034

## Implementation Strategy

1. **MVP = Phase 1~3 + US4**: 믿을 수 있는 5분 공간 시계열과 분할. 이후 모든 과제의 입력이 된다.
2. US3 기준선: 과제 3의 비교 대상과 평가 시점 파일을 확정한다.
3. US2 진단서: 그래프와 해석을 쓴다.
4. 망이 복구되면 T035로 v2-final 확정 결과를 만든다.
