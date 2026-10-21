# Implementation Plan: 60분 예측 (과제 3)

**Branch**: `003-forecast-60min` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/003-forecast-60min/spec.md`

## Summary

과제 2의 공간 5분 시계열과 평가 기준(분할 `v1-provisional`, 평가 시점 파일, 학습 기간 정규화)을 그대로 받아,
실습실·사무실·회의실의 CO₂·온도를 **60분 앞까지 12칸** 예측하는 모델을 학습한다. 방식의 핵심은 다음과 같다.

1. **표본**: (공간, t0)마다 과거 구간 입력과 미래 12칸의 "미리 알 수 있는 정보"(시각·요일·공휴일·일과표·예약)를 받아 CO₂·온도 24개 값을 낸다.
   - 입력은 관측 칸만 쓰고, 빈 칸은 앞으로만 채운다(누수 방지).
   - 정답도 관측 칸만 쓴다.
2. **모델**: GRU·LSTM·TCN 인코더 + 미래 정보 결합 헤드. 기본 목표는 **변화량**이다. 출력이 0이면 곧 현재값 유지다.
3. **선택 절차**: 설계 축(과거 길이 → 입력 묶음 → 출력 방식 → 목표)을 하나씩 **검증 MAE**로 고정한다.
   - 심화 1(과거 1·2·4시간 × 모델 3종)도 검증으로 최종 모델을 고른다.
   - `selection.json`이 고정된 뒤에만 시험 평가가 동작한다.
4. **시험 평가**(한 번): 최종 모델, 일정 제외 쌍(일정 정보의 가치), 심화 2(미래 예약 — 회의실은 추정 예약, 상한), 심화 3(처음 보는 공간).
5. **보고**
   - 비교표: 기준선 3종 + 모델, 5·15·30·60분, 전체·사용 시간대
   - 확인 기준 판정표, 공간별·일과 단계별 오차 분석, 결론 문장
   - 저장 위치: 산출물은 `docs/deliverables/task3/`, 심화는 `docs/advanced/task3/`

학습은 GPU(Orin, PyTorch cu130, TF32 끔·결정적 설정)로 한다. 그룹 권한이 아직 반영되지 않은 세션에서는 래퍼가 `sg`로 다시 실행한다.
근거는 [research](research.md)에 있다.

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**
- PyTorch 2.14.1+cu130: 설치 완료. GPU 인식과 CPU 대비 수치 일치를 확인했다.
- 과제 2 의존성: pandas, pyarrow, numpy, matplotlib, holidays, PyYAML
- 과제 2 패키지 `analysis`의 설정·일과표·평가 시점 함수를 재사용한다.

**Storage**
- 입력(읽기만, 과제 2 산출)
  - `var/series/space_5min.parquet`
  - `var/eval/eval_points_v1-provisional.parquet`
  - `var/eval/baseline_v1-provisional.parquet`
  - `var/eval/norm_v1-provisional.json`
  - `var/cache/events.parquet`
- 작업: `var/forecast/`(실행 기록·가중치·예측·선택 파일·모델 묶음, git 제외)
- 결과 문서: `docs/deliverables/task3/`, `docs/advanced/task3/`

**Testing**: pytest. 합성 시계열 `unit` 시험(누수·인과성·마스크·형상·재개)과 실데이터 소규모 `forecast` 마커 시험(GPU 있으면 GPU)

**Target Platform**: Jetson Orin Nano Super(aarch64, CUDA 13.2, 25W 모드). 오프라인 일괄 학습이며 서비스가 아니다

**Project Type**: 학습·평가 라이브러리 + CLI(`src/forecast/`)

**Performance Goals**
- 실험 전체(약 70회) GPU 하루 안(SC-007). 추정 5~6시간
- 과거 2시간 GRU 한 실행 약 3분

**Constraints**
- 학습 프로세스 최대 메모리 2GB 이하(SC-008, 목표 1.5GB). 수집기와 공존
- 시험 기간 누수 0(SC-003)
- 평가 시점 행 단위 일치(SC-004)

**Scale/Scope**
- 학습 표본 약 13만(lab 6만, office 4.4만, meeting 3만 t0), 검증 약 3만, 시험 평가 행 약 11.7만(CO₂·온도 × 지평 4)
- 출력: 2항목 × 12칸

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 문서 한국어. DB를 읽지 않는다(과제 2 사본만 사용) | ✅ |
| II. 엣지 제약 | `.venv`+uv, 학습 메모리 2GB 상한, 표본을 인덱스로 잘라 메모리에 펼치지 않음, sudo 없이 `sg` 래퍼 | ✅ |
| III. 설정 주도 | 입력 묶음·과거 길이·출력 방식·목표·모델 크기·씨앗·실험 목록·추정 예약 규칙·예약 파일 경로를 `config/forecast.yaml`에 둔다 | ✅ |
| IV. 테스트 | 확인 기준 3(누수)을 자동 시험으로 검증한다(시험 값 변조 → 가중치 동일). 인과적 입력, 마스크 손실, 평가 행 일치, 재개 시험도 둔다 | ✅ |
| V. 공정한 평가 | 과제 2 평가 시점 파일·해시·정규화 그대로 사용. 선택은 검증만, `selection.json` 고정 뒤 시험 평가. 결과표마다 측정 조건(장치·씨앗 수·전력 모드) | ✅ |
| VI. 복구 | 실험 단위로 결과를 저장하고, 끊겨도 이어서 실행. 한 실험 실패가 전체를 멈추지 않음 | ✅ |
| VII. 해석 우선 | 비교표·공간별 분석·일정 가치·심화마다 해석 절과 결론 문장. 점검 문서에 확인 명령 | ✅ |

Phase 1 뒤 재확인: 위반 없음.
- 추정 예약은 시험 기간 실제 값에서 유도된다. 그러나 **상한 실험으로 명시**하고 기본 비교표·확인 기준 판정에는 쓰지 않는다. 그래서 V 위반이 아니다.

## Project Structure

### Documentation (this feature)

```text
specs/003-forecast-60min/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (config.md, cli.md)
├── checklists/requirements.md
└── tasks.md   (/speckit-tasks)
```

### Source Code (repository root)

```text
src/forecast/
├── __main__.py      CLI: check · build · train · run · select · test · report · info
├── config.py        config/forecast.yaml 읽기·검증, 실험 설정 해시
├── env.py           장치 선택, 결정성 설정, 버전 기록
├── data.py          공간 배열 만들기(관측만·앞으로 채우기·표시), 분할별 t0 목록, 평가 시점 파일 해시 확인
├── features.py      시간·공휴일·일과표·예약 특징(과거·미래 12칸), 추정 예약 생성
├── dataset.py       (공간, t0) → 입력 텐서·미래 정보·정답·마스크(인덱스로 자르기)
├── models.py        GRU·LSTM·TCN 인코더 + 헤드, 반복 예측 래퍼
├── train.py         학습 루프(마스크 L1, 조기 종료, 씨앗), 검증 예측 저장
├── experiments.py   실험 목록·단계 선택(D0~D4, B1)·selection.json·이어서 실행
├── evaluate.py      시험 예측(평가 시점 파일 행), 기준선과 결합, MAE·개선율·판정
├── analysis.py      공간별·일과 단계별·상위 오차·창문 개방 추정·사건 구간 분석, 그래프
└── report.py        산출물·심화 Markdown 생성
config/forecast.yaml
tools/forecast       어디서든 실행 + GPU 그룹 없으면 sg로 다시 실행
tests/forecast/      test_data(인과성·마스크) · test_features · test_models(형상·반복) · test_leakage · test_evaluate(행 일치) · test_experiments(재개·선택 고정)
docs/deliverables/task3/   01-model-comparison.md · 02-space-error-analysis.md · 03-schedule-value.md · figures/
docs/advanced/task3/       01-lookback-model.md · 02-future-info.md · 03-unseen-space.md
docs/tasks/task3-forecast-60min.md   점검 문서(확인 명령 포함)
```

**Structure Decision**
- `src/analysis/` 옆에 `src/forecast/` 패키지를 둔다. 일과표·공휴일·사용 시간대 판정은 `analysis.schedule`과 `analysis.baseline.usage_mask`를 재사용한다.
  두 과제의 정의가 같아야 하기 때문이다.
- `pyproject.toml`에 `forecast` extra(torch)를 추가한다. torch는 cu130 index에서 설치한다(quickstart).
- 수집기(`collector`)는 torch를 쓰지 않으므로 영향이 없다.

## Complexity Tracking

위반 없음.
