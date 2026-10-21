# Implementation Plan: 상태 추정 (과제 5)

**Branch**: `005-state-estimation` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/005-state-estimation/spec.md`

## Summary

두 분류기를 만든다.

| 문제 | 정답 | 입력 | 기준선 |
|---|---|---|---|
| 사용 여부 | 일정 기반(학습·검증·시험) + **직접 관찰**(최종 평가) | 센서 특징만(일정·시각 금지) | 최근 15분 CO₂ +30ppm |
| 환기 선제 경보 | 지금 < 1,000, 30분 안 ≥ 1,000. 이미 초과한 칸은 제외 | 센서 + 시간 + 일정 | 현재 CO₂ ≥ 850, 과제 3 예측 판정 |

- **특징**: 학습·서비스 공통 전처리(`src/preprocess/`) 위에서 인과적으로 만든다.
- **분류기**: HistGradientBoosting·로지스틱 회귀·작은 MLP를 검증 macro-F1로 고른다.
- **임계값**: 환기 경보는 검증 재현율 ≥ 85% 중 정밀도 최대, 사용 여부는 macro-F1 최대.
- **절차**: 선택을 `selection.json`에 고정한 뒤에만 시험·관찰 평가를 한다.
- **관찰**: `state observe` 명령으로 사람이 기록하고, 일정 기반 정답의 오류를 먼저 잰다.
- **산출물**: `docs/deliverables/task5/`(성적표·혼동 행렬, 정답 차이 분석, 오류 사례), `docs/advanced/task5/`(처음 보는 공간, 노쇼·예약 없는 사용)

근거는 [research](research.md)에 있다.

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**
- scikit-learn 1.9.1: aarch64 설치 가능 확인, `state` extra로 추가
- PyTorch 2.14.1+cu130: MLP 후보
- pandas, numpy, pyarrow, matplotlib, holidays
- 재사용
  - `analysis`: 설정·일과표·분할·사용 시간대
  - `src/preprocess/`: 공통 전처리(과제 8에서 꺼냄. 그 전에는 `analysis`·`forecast.data` 함수를 직접 부르고, 공통 모듈이 생기면 그쪽으로 바꿈)
  - `forecast.env`: 결정성·장치
  - `forecast`: 과제 3 묶음 읽기(Q4 비교)

**Storage**
- 입력(읽기만)
  - `var/series/space_5min.parquet`, `var/eval/norm_v1-provisional.json`
  - `config/analysis.yaml`, `var/forecast/booking_estimated.parquet`, `var/forecast/bundle/`
  - `data/observations.csv`: 사람 기록, git 제외
- 작업: `var/state/`(git 제외)

**Testing**: pytest `unit`(합성)
- 정답 라벨(이미 초과 제외 경계)
- 관찰 CSV 검증, 관찰 명령의 구간 열고 닫기
- 특징 인과성, 사용 여부 입력에 일정·시각 금지
- 임계값 선택, 선택 고정 전 평가 거부, **누수 시험**(시험 기간 변조 → 선택 동일)
- 지표 계산(손 계산 대조)

**Target Platform**: Jetson Orin Nano Super. 오프라인 일괄 + 관찰 기록 명령

**Project Type**: 분류·평가 라이브러리 + CLI(`src/state/`)

**Performance Goals**: 학습·평가 전체 30분 이내(SC-007), 관찰 명령 즉시(1초 안)

**Constraints**
- 최대 메모리 2GB 이하(목표 1GB)
- 시험 기간 누수 0
- 관찰 기록은 공개 저장소 밖(`data/`)

**Scale/Scope**
- 학습 칸 약 13만(3공간), 검증 약 3만, 시험 약 4만
- 특징 약 40개
- 관찰 목표 20시간(약 240칸)

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 한국어 문서. 관찰 기록은 공개 저장소 밖(`data/` git 제외), 관찰자는 머리글자 | ✅ |
| II. 엣지 제약 | 트리·선형은 CPU, 특징 표 약 20MB, 메모리 목표 1GB, 학습은 과제 3 뒤 | ✅ |
| III. 설정 주도 | 정답 기준(1,000ppm·30분), 특징 묶음, 후보·격자, 임계 목표(재현율 0.85), 단순 규칙, 노쇼 시간 기준, 예약 파일 경로를 `config/state.yaml`에 둔다 | ✅ |
| IV. 테스트 | 확인 기준 계산(재현율·macro-F1·오류율)과 누수·선택 고정을 자동 시험으로 검증한다 | ✅ |
| V. 공정한 평가 | 학습은 학습 기간, 선택·임계값은 검증 기간, `selection.json` 고정 뒤 시험·관찰 평가. 정답 출처(일정·관찰)를 결과표마다 표시. 회의실 일정 기반 평가는 순환이라 참고로만 | ✅ |
| VI. 복구 | 단계 산출 파일로 중간부터 재실행. 관찰 명령은 파일에 덧붙이기만 함(덮어쓰기 없음) | ✅ |
| VII. 해석 우선 | 혼동 행렬·상위 혼동 상황·전환 직후 오류·헛경보마다 원인과 운영 의미를 적는다 | ✅ |

Phase 1 뒤 재확인: 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/005-state-estimation/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (config.md, cli.md)
├── checklists/requirements.md
└── tasks.md   (/speckit-tasks)
```

### Source Code (repository root)

```text
src/state/
├── __main__.py      CLI: check · labels · observe · features · train · select · evaluate · analyze · report · info · all
├── config.py        config/state.yaml 읽기·검증(사용 여부 특징에 time·schedule 금지, 누수 실험만 예외), 해시
├── labels.py        사용 여부 일정 기반 정답, 환기 정답(양성·음성·이미 초과·판단 불가)
├── observe.py       관찰 CSV 읽기·검증·덧붙이기, 열린 구간 관리, 현황, 일정 정답과 비교
├── features.py      공통 전처리 위 인과적 특징 묶음(level·delta·shape·obs·time·schedule)
├── models.py        후보 분류기(HGB·로지스틱·MLP) 공통 인터페이스, MLP는 PyTorch
├── train.py         후보 × 격자 × 씨앗 학습, 검증 예측·지표, 이어서 실행
├── select.py        검증 선택·임계값(재현율 ≥ 0.85 중 정밀도 최대 / macro-F1 최대)·selection.json 고정·검사
├── evaluate.py      시험·관찰 평가, 단순 규칙, 과제 3 예측 판정, 확인 기준 판정표
├── analyze.py       전환 뒤 경과별 오류, 인원별 재현율, 상위 혼동 상황, 헛경보, 그래프
├── advanced.py      처음 보는 공간, 노쇼·예약 없는 사용 규칙
├── report.py        결과 표·그래프(자동)
└── bundle.py        과제 8용 묶음(bundle.json)
config/state.yaml
tools/state          어디서든 실행(+ GPU 그룹 sg 재실행, tools/forecast와 같음)
tests/state/         test_labels · test_observe · test_features · test_select(누수 포함) · test_evaluate · test_advanced
data/observations.csv   사람 기록(git 제외)
docs/deliverables/task5/   01-scorecards.md · 02-label-vs-observation.md · 03-error-cases.md · figures/ · tables/
docs/advanced/task5/       01-unseen-space.md · 02-noshow-unbooked.md
docs/tasks/task5-state-estimation.md
```

**Structure Decision**
- 과제별 패키지 관례대로 `src/state/`를 둔다.
- 특징은 공통 전처리(`src/preprocess/`, 과제 8 T006에서 꺼냄) 위에서 만든다. 그 전에는 같은 함수가 있는 `forecast.data`·`analysis`를 부르고, 공통 모듈이 생기면 import만 바꾼다. 결과는 같아야 한다(시험).
- `pyproject.toml`에 `state = ["scikit-learn>=1.9"]` extra를 추가한다. torch는 `forecast` extra를 공유한다.
- `data/`를 `.gitignore`에 추가한다.

## Complexity Tracking

위반 없음.
