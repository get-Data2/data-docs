# Implementation Plan: 엣지 최적화 (과제 7)

**Branch**: `007-edge-optimization` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/007-edge-optimization/spec.md`

## Summary

과제 3(예측)·4(오토인코더)·5(분류기) 모델 묶음을 **ONNX → TensorRT FP16 엔진**으로 바꾼다. 실행 방식 4종의 지연·자원·결과 차이를 같은 조건에서 잰다.

| 실행 방식 | 비고 |
|---|---|
| PyTorch CPU | 기준(결과 차이의 기준값) |
| PyTorch GPU | |
| ONNX Runtime | CPU·CUDA EP |
| TensorRT | FP16 |

- **측정**
  - 배치 1·3(현재 공간 수)·4(확장 가정), 워밍업 50 · 반복 1,000
  - 평균·p95 지연, `tegrastats` 전력·GPU 사용률·온도, 프로세스 메모리
- **결과 차이**: 과제 3 시험 표본 전체에서 원 단위로 잰다(CO₂ ppm 최대 ≤ 1, 분류 바뀐 비율 ≤ 0.1%).
- **설계 원칙**: 원 단위 복원은 엔진 밖(FP32)에서 한다. 정규화 공간에서 끝내야 FP16으로 1ppm 기준을 지킬 수 있다(research R3·R7).
- **전제 확인**: 시스템에 TensorRT가 없어 pip 휠(`tensorrt-cu13` 11.4)을 쓴다. Orin에서 동작하는지 **첫 작업으로 확인**하고, 실패 시 대안 순서(research R2)를 따른다.
- **심화**
  - INT8: 학습 기간 표본으로 보정
  - 동시 적재 총 메모리와 로컬 LLM 크기 가정별 조합
- **결론**: 과제 8 서비스가 쓸 실행 방식과 실행 묶음(`var/edgeopt/runtime/`)을 정한다.

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**
- PyTorch 2.14.1+cu130: 설치 완료. CPU 대비 수치 일치를 확인했다
- 추가 설치(pip, `.venv`): `onnx`, `onnxruntime-gpu` 1.31.0, `tensorrt-cu13` 11.4.0.106. aarch64 휠이 있는 것은 dry-run으로 확인했다
- 재사용
  - `forecast`(모델 정의·Windows·평가 시점 표본·결정성 설정·장치 선택)
  - `anomaly`(오토인코더 정의·입력)
  - `state`(과제 5 분류기)

**Storage**
- 입력(읽기만): `var/forecast/bundle/`, `var/forecast/runs/`(B1 세 모델), `var/anomaly/bundle/`, `var/state/bundle/`, 과제 2 평가 시점 파일
- 작업: `var/edgeopt/`(git 제외)

**Testing**: pytest `unit`(합성 작은 모델). 시험할 것은 다음과 같다.
- ONNX 내보내기와 PyTorch 결과 일치
- 지연 통계 계산(p95)
- tegrastats 줄 해석
- 원 단위 차이 계산
- 판정표

마커 `trt`: 실제 TensorRT가 필요한 시험. 기본에서 제외하고, 동작 확인 뒤 실행한다.

**Target Platform**: Jetson Orin Nano Super, 25W, 클럭 고정 안 함. 오프라인 측정이다

**Project Type**: 변환·측정 라이브러리 + CLI(`src/edgeopt/`)

**Performance Goals**: 전체 측정(모델 약 7개 × 방식 4 × 배치 3) 1시간 이내

**Constraints**
- 측정 중 우리 학습 프로세스 없음
- 다른 계정 GPU 사용 기록
- 측정 방식마다 새 프로세스

**Scale/Scope**
- 모델: 예측 1 + 비교 3 + 오토인코더 3 + 분류기 1
- 결과 차이 표본 약 4만(과제 3 시험)

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 한국어 문서. 비밀 없음(DB·브로커 미사용) | ✅ |
| II. 엣지 제약 | 이 과제의 목적 자체. sudo 없이 pip 휠·`tegrastats`만 사용, 메모리·전력 기록, 동시 적재 측정 | ✅ |
| III. 설정 주도 | 대상 모델·실행 방식·배치·워밍업·반복·작업 공간·정밀도·보정 표본 수를 `config/edgeopt.yaml`에 둔다 | ✅ |
| IV. 테스트 | 확인 기준 1(원 단위 차이 판정)과 지연 통계 계산을 자동 시험으로 검증한다. TensorRT 의존 시험은 마커로 분리 | ✅ |
| V. 공정한 평가 | 같은 입력·같은 조건으로 비교한다. INT8 보정은 학습 기간 표본만 쓴다. 측정 조건(25W·클럭 고정 안 함·반복·워밍업·온도·경합)을 기록한다 | ✅ |
| VI. 복구 | 일괄 측정. 모델·방식 단위로 결과 파일을 남겨 이어서 실행하고, 한 조합 실패가 전체를 멈추지 않는다 | ✅ |
| VII. 해석 우선 | 보고서에 결론(과제 8 실행 방식)과 Q1~Q3 답을 측정 숫자로 쓴다 | ✅ |

Phase 1 뒤 재확인: 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/007-edge-optimization/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (config.md, cli.md)
├── checklists/requirements.md
└── tasks.md
```

### Source Code (repository root)

```text
src/edgeopt/
├── __main__.py     CLI: probe · export · build · bench · diff · colocate · report · runtime · info
├── config.py       config/edgeopt.yaml 읽기·검증, 대상 모델 목록 해석(묶음 → 모델 정의·입력 규약)
├── models.py       모델 묶음 읽기(과제 3·4·5) → torch 모듈 + 예시 입력 + 원 단위 복원 함수
├── export.py       ONNX 내보내기·검증(ORT CPU vs PyTorch CPU)
├── trt.py          TensorRT 엔진 생성(FP16·INT8 보정)·실행 래퍼, 생성 환경 기록
├── runners.py      실행 방식 공통 인터페이스(torch_cpu · torch_gpu · ort_cpu · ort_cuda · ort_trt · trt_fp16 · trt_int8)
├── bench.py        지연 측정(워밍업·반복·동기화·통계), 새 프로세스 실행, 측정 전 경합 검사
├── tegra.py        tegrastats 기록·해석·구간 맞추기, 전력·에너지 계산
├── diff.py         시험 표본 전체 예측 → 원 단위 차이·바뀐 비율·판정
├── colocate.py     동시 적재 측정, LLM 크기 가정 표
└── report.py       표·그래프(자동) → docs/deliverables/task7/tables, figures
config/edgeopt.yaml
tools/edgeopt       어디서든 실행(+ GPU 그룹 sg 재실행, tools/forecast와 같은 방식, TensorRT lib 경로)
tests/edgeopt/      test_export · test_bench_stats · test_tegra · test_diff (unit), test_trt (마커 trt)
docs/deliverables/task7/   01-optimization-report.md · tables/ · figures/
docs/advanced/task7/       01-int8.md · 02-colocation-llm.md
docs/tasks/task7-edge-optimization.md
```

**Structure Decision**
- 과제별 패키지 관례대로 `src/edgeopt/`를 둔다. 모델 정의는 각 과제 패키지에서 가져온다. 다시 정의하면 학습·서비스와 어긋날 위험이 있기 때문이다.
- `pyproject.toml`에 extra `edgeopt = ["onnx", "onnxruntime-gpu", "tensorrt-cu13"]`를 추가한다. 수집기·분석에는 영향이 없다.

## Complexity Tracking

위반 없음.
