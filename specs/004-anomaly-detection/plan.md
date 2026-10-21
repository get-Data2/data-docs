# Implementation Plan: 이상 탐지 (과제 4)

**Branch**: `004-anomaly-detection` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/004-anomaly-detection/spec.md`

## Summary

정답이 있는 고장 자료가 없으므로, 실데이터 **시험 기간 사본에 고장 5종(종류마다 12건, 총 60건)을 주입**해 평가판을 만든다. 탐지기는 네 가지다.

| 탐지기 | 방식 | 근거 |
|---|---|---|
| 모델 | 공간별 1D 합성곱 오토인코더. 학습 기간 정상만 학습, 복원 오차 = 이상 점수, 열 기여 = 원인 센서 | research R3 |
| 규칙 | 급변(한 칸 튐) · 값 고정(이웃 센서는 변함) | research R4 |
| 심화 1 | 과제 3 예측 오차 | research R7 |
| 심화 2 | 영점 틀어짐 전용(새벽 CO₂ 기준 + 이웃 센서 비교) | research R8 |

- **임계값**: 검증 기간(주입 없음) 하루당 경보 0.5회 이하가 되는 가장 낮은 값으로 정하고 `thresholds.json`에 고정한다. 그 뒤에만 평가판을 채점한다.
- **채점**: 같은 매칭 규칙으로 종류별 탐지율·지연·하루당 헛경보·원인 정확도를 내고, 모델 / 규칙 / 결합 성적표와 임계값 곡선을 만든다.
- **산출물**: `docs/deliverables/task4/`, `docs/advanced/task4/`

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**
- PyTorch 2.14.1+cu130: 오토인코더. 과제 3과 같은 결정성 설정
- pandas, numpy, pyarrow, matplotlib, holidays
- 재사용
  - `analysis`: 설정·일과표·사용 시간대
  - `forecast`: 결정성 설정·장치 선택(`forecast.env`), 모델 묶음 읽기(심화 1)

**Storage**
- 입력(읽기만)
  - `var/series/sensor_5min.parquet`, `space_5min.parquet`
  - `var/cache/events.parquet`, `var/cache/stuck.parquet`, `var/cache/devices.parquet`
  - `config/analysis.yaml`(분할·일과표)
  - `var/forecast/bundle/`(심화 1)
- 작업 파일: `var/anomaly/`(git 제외)

**Testing**: pytest `unit`. 시험할 것은 다음과 같다.
- 주입 재현성, 주입 밖 값이 원본과 같음
- 규칙 탐지(합성)
- 매칭·지표 계산
- **임계값 누수**: 평가판을 바꿔도 임계값이 같음
- 오토인코더 형상·기여도

**Target Platform**: Jetson Orin Nano Super. 오프라인 일괄 실행

**Project Type**: 탐지·평가 라이브러리 + CLI(`src/anomaly/`)

**Performance Goals**
- 주입·규칙·평가 전체 5분 이내
- 오토인코더 학습: 공간당 5분 이내(GPU)

**Constraints**
- 학습 메모리 2GB 이하, GPU는 과제 3 실험 종료 뒤 사용
- 임계값 선택에 시험 기간 미사용
- 원본 파일 변경 없음

**Scale/Scope**
- 시험 57일 × 288칸 × 공간 3
- 열: 실습실 21, 사무실 5, 회의실 3
- 학습 창 약 18만 개(공간 합)

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 한국어 문서. DB를 읽지 않음(과제 2 사본) | ✅ |
| II. 엣지 제약 | 작은 오토인코더(파라미터 수만), 배치 상한, GPU는 과제 3 뒤 | ✅ |
| III. 설정 주도 | 주입 규칙·탐지기 파라미터·임계값 목표·매칭 여유·알림 대상을 `config/anomaly.yaml`에 둔다 | ✅ |
| IV. 테스트 | 확인 기준 계산(탐지율·헛경보)과 임계값 누수를 자동 시험으로 검증한다 | ✅ |
| V. 공정한 평가 | 학습은 학습 기간, 임계값은 검증 기간(주입 없음), 채점은 `thresholds.json` 고정 뒤. 결과표마다 측정 조건과 "실데이터 + 주입 고장" 명시 | ✅ |
| VI. 복구 | 단계 산출 파일이 있어 중간부터 재실행 가능 | ✅ |
| VII. 해석 우선 | 성적표·임계값 곡선·운영 제안마다 해석. 미탐 종류의 이유(확인 기준 3) | ✅ |

Phase 1 뒤 재확인: 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/004-anomaly-detection/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (config.md, cli.md)
├── checklists/requirements.md
└── tasks.md   (/speckit-tasks)
```

### Source Code (repository root)

```text
src/anomaly/
├── __main__.py     CLI: check · inject · train · score · threshold · evaluate · report · info
├── config.py       config/anomaly.yaml 읽기·검증, 해시
├── data.py         센서 넓은 표(공간별 열), 정규화 통계(학습 기간), 관측 표시·앞으로 채우기, 공간 대표값 재계산
├── inject.py       고장 5종 주입·정답 표
├── autoencoder.py  1D 합성곱 오토인코더, 창 데이터, 학습, 점수·열 기여
├── rules.py        급변·값 고정 규칙
├── drift.py        영점 틀어짐 전용(새벽 기준·이웃 비교)
├── forecast_err.py 과제 3 예측 오차 점수
├── alarms.py       점수 → 분위수 → 경보(병합), 임계값 선택(검증), 결합
├── evaluate.py     매칭·지표·성적표·임계값 곡선·원인 정확도·사건 후보 겹침
├── report.py       결과 표·그래프(자동), 운영 제안 표
└── bundle.py       과제 8용 묶음
config/anomaly.yaml
tools/anomaly       어디서든 실행(+ GPU 그룹 sg 재실행, tools/forecast와 같은 방식)
tests/anomaly/      test_inject · test_rules · test_alarms(누수 포함) · test_evaluate · test_autoencoder · test_drift
docs/deliverables/task4/   01-scorecard.md · 02-threshold-tradeoff.md · 03-operations.md · figures/ · tables/
docs/advanced/task4/       01-forecast-error-detector.md · 02-drift-detector.md
docs/tasks/task4-anomaly-detection.md
```

**Structure Decision**
- 과제별 패키지 관례대로 `src/anomaly/`를 둔다. 결정성·장치 설정은 `forecast.env`를 재사용한다(같은 보드 규칙).
- `pyproject.toml`은 `forecast` extra(torch)를 공유하므로 새 extra가 필요 없다.

## Complexity Tracking

위반 없음.
