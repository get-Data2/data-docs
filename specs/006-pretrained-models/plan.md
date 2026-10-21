# Implementation Plan: 사전학습 모델 비교 (과제 6)

**Branch**: `006-pretrained-models` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/006-pretrained-models/spec.md`

## Summary

Chronos-Bolt(tiny·mini·small)로 시험 기간 CO₂를 **추가 학습 없이** 예측한다. 과제 2 평가 시점 파일의 CO₂ 행에서 기준선 3종·과제 3 직접 학습 모델과 비교한다.

1. **선택**: 컨텍스트 길이(1시간·4시간·1일·1주)와 모델 크기를 **검증 기간**으로 고르고 `selection.json`으로 고정한다. 그 뒤 시험을 한 번 평가한다.
2. **판단 근거 네 가지**를 표로 만든다.
   - 정확도: 공간 × 지평 MAE
   - 예측 구간: 10~90% 적중률·이탈 방향·폭
   - 비용: 보드 지연(평균·p95, 배치 1·3, GPU·CPU)
   - 메모리·전력: RSS·GPU 할당, `tegrastats` 전력
3. **채택 의견**: 위 근거와 로컬 LLM 공존 메모리 예산(가정 포함)으로 반 쪽 의견을 쓴다.
4. **심화**
   - 공변량 모델(Chronos-2): 온습도·시간·일과표를 넣은 비교
   - 섞은 예측: 직접 학습 + 사전학습, 가중치는 검증으로 결정

근거는 [research](research.md)에 있다.

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**
- `chronos-forecasting` 2.3.2: aarch64 설치 가능 확인. torch는 기존 2.14.1+cu130을 쓰며, 설치 때 바뀌지 않는지 확인한다
- pandas, numpy, pyarrow, matplotlib
- 재사용
  - `analysis`: 설정·사용 시간대
  - `forecast`: 평가 함수·특징·결정성·장치 선택·모델 묶음

**Storage**
- 입력(읽기만)
  - 과제 2: `var/series/space_5min.parquet`, `var/eval/eval_points_v1-provisional.parquet`, `var/eval/baseline_v1-provisional.parquet`
  - 과제 3: `var/forecast/selection.json`, `var/forecast/runs/*/test_pred.parquet`·`val_pred.parquet`, `var/forecast/stages.json`, `var/forecast/bundle/`
- 작업
  - `var/pretrained/`: 가중치 `hf/`, 예측 `preds/`, `selection.json`, `bench/`, `models.json`
  - git에서 제외한다

**Testing**: pytest `unit`(합성 시계열). 시험할 것은 다음과 같다.
- 컨텍스트 인과성
- 평가 행 일치
- 적중률 계산
- 선택 고정 전 시험 거부
- 앙상블 가중치가 검증만 사용

모델 다운로드가 필요한 시험은 마커 `pretrained`로 분리한다(기본 제외).

**Target Platform**: Jetson Orin Nano Super(25W 모드, 클럭 고정 안 함). 오프라인 일괄 실행

**Project Type**: 평가·측정 라이브러리 + CLI(`src/pretrained/`)

**Performance Goals**
- 시험 예측(약 3.1만 CO₂ t0 × 모델)이 GPU로 모델당 30분 이내
- 비용 측정은 모델·장치·배치 조합당 수 분

**Constraints**
- 메모리: 사전학습 모델 프로세스 최대 RSS를 기록한다. small도 2GB 이하일 것으로 예상하나 실측으로 확인한다
- 시험 기간 누수 0
- 다운로드 뒤 오프라인 재실행

**Scale/Scope**
- 모델 3 + 심화 1
- 컨텍스트 4종(검증)
- 공간 3, 지평 4

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 한국어 문서. HuggingFace 토큰이 필요하면 `.env`에만 두고 출력하지 않는다(공개 모델이라 불필요 예상) | ✅ |
| II. 엣지 제약 | `.venv`+uv, sg 래퍼, 메모리·전력 실측, 로컬 LLM 공존 예산표 | ✅ |
| III. 설정 주도 | 모델 이름·리비전·컨텍스트 후보·분위수·반복 수·LLM 가정을 `config/pretrained.yaml`에 둔다 | ✅ |
| IV. 테스트 | 같은 시점 비교(행 일치), 인과적 컨텍스트, 선택 고정 전 시험 거부, 적중률 계산을 자동 시험으로 | ✅ |
| V. 공정한 평가 | 과제 2 평가 시점·정답 그대로. 크기·컨텍스트·가중치는 검증 기간으로. 측정 조건(전력 모드, 클럭 고정 안 함, 반복·워밍업, 동시 작업) 기록 | ✅ |
| VI. 복구 | 단계 산출 파일이 있으면 건너뛰고 이어서 실행 | ✅ |
| VII. 해석 우선 | 비교표·적중률·비용마다 해석. 채택 의견은 근거 인용 | ✅ |

Phase 1 뒤 재확인: 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/006-pretrained-models/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (config.md, cli.md)
├── checklists/requirements.md
└── tasks.md
```

### Source Code (repository root)

```text
src/pretrained/
├── __main__.py     CLI: check · fetch · predict · select · test · bench · report · info
├── config.py       config/pretrained.yaml 읽기·검증, 해시
├── context.py      인과적 컨텍스트(관측만, NaN 유지 또는 앞으로 채우기), 검증 표본 t0, 시험 t0(평가 시점 파일)
├── models.py       모델 적재(리비전 고정·오프라인), 분위수 예측 래퍼(배치), 공변량 입력(Chronos-2)
├── predict.py      기간별 예측 → parquet(12칸 × 분위수 → 지평 5·15·30·60)
├── selection.py    검증 기준 크기·컨텍스트·앙상블 가중치 선택, selection.json 고정·검사
├── metrics.py      MAE 비교표(forecast.evaluate 재사용), 적중률·이탈·폭·보정 곡선
├── bench.py        지연·메모리·전력 측정 하네스(tegrastats 병행) — 과제 7 재사용
├── budget.py       메모리 예산표(관찰값 + 가정 LLM + 측정값)
└── report.py       결과 표·그래프
config/pretrained.yaml
tools/pretrained    어디서든 실행 + GPU 그룹 sg 재실행
tests/pretrained/   test_context · test_metrics · test_selection · test_bench(가짜 모델)
docs/deliverables/task6/   01-comparison-adoption.md(비교표와 채택 의견) · figures/ · tables/
docs/advanced/task6/       01-covariates.md · 02-ensemble.md
docs/tasks/task6-pretrained-models.md
```

**Structure Decision**
- 과제별 패키지 관례대로 `src/pretrained/`를 둔다.
- 평가 함수(`forecast.evaluate.mae_by_cell`·`baseline_by_cell`)와 특징(`forecast.features`)을 재사용해 정의를 과제 3과 맞춘다.
- `pyproject.toml`에 `pretrained = ["chronos-forecasting==2.3.2"]` extra를 추가한다.

## Complexity Tracking

위반 없음.
