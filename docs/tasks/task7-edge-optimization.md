# 과제 7. 엣지 최적화 — 점검 문서

> 상태: 🟢 **측정·판정 완료(2026-10-21)**.
> - TensorRT FP16은 GRU·LSTM에서 CO₂ 1ppm 기준을 넘는다(최대 2.2~3.0ppm).
> - TensorRT FP32와 ONNX Runtime CPU는 모든 모델이 통과한다.
> - 서비스 실행 방식은 **ONNX Runtime CPU**로 정했다. 2026-10-22 08:54 과제 8 서비스에 적용했다(서비스 메모리 1,409 → 689MB, 결과 차이 최대 0.0005ppm).
>
> 측정 조건: Jetson Orin Nano Super, **25W, 클럭 고정 안 함**(sudo 없음), 워밍업 50·반복 1,000(3구간), 조합마다 새 프로세스, tegrastats 1초 간격.
> 소프트웨어: PyTorch 2.14.1+cu130, ONNX Runtime 1.31.0, TensorRT 10.16.2(JetPack 시스템 판).
>
> 산출물: [최적화 보고서](../deliverables/task7/01-optimization-report.md) · 심화: [① INT8](../advanced/task7/01-int8.md) · [② 동시 적재·LLM](../advanced/task7/02-colocation-llm.md)
> 계획: [spec](../../specs/007-edge-optimization/spec.md) · [plan](../../specs/007-edge-optimization/plan.md) · [research](../../specs/007-edge-optimization/research.md) · [tasks](../../specs/007-edge-optimization/tasks.md)

## 요약

- 예측(GRU·LSTM·TCN)과 오토인코더 3개를 모두 ONNX와 TensorRT(FP16·FP32·INT8)로 변환했다. 상태 분류기(트리)는 CPU 그대로 둔다.
- **이 크기의 모델에서는 GPU가 이득이 아니다.**
  - GPU 경로는 복사·실행·동기화의 고정 비용이 약 1ms다.
  - ONNX Runtime CPU는 예측 0.21ms, 오토인코더 0.07ms로 가장 빠르고, 결과도 PyTorch와 같다.
- GRU·LSTM을 FP16으로 바꾸면 순환 계산에서 반올림이 쌓여 최대 2~3ppm 달라진다. 평균은 0.03ppm이다.
- INT8은 빌드는 되지만 FP16과 같은 엔진이 됐다. 속도·정확도 이득이 없다.

---

## 해결할 것

| # | README 요구 | 해결 방법 | 근거·위치 | 상태 |
|---|---|---|---|---|
| 1 | 예측·이상 탐지·분류 모델을 TensorRT FP16으로 | 예측 4·오토인코더 3 변환 성공(FP16 7~46초). 상태 분류기는 트리라 대상 아님. pip TensorRT 11.4는 sm87 빌더가 없어 JetPack 시스템 TensorRT 10.16을 연결 | [보고서 1절](../deliverables/task7/01-optimization-report.md#1-변환-해결할-것-1) | ✅ |
| 2 | 지연(평균·p95): PyTorch CPU/GPU, ONNX Runtime, TensorRT × 배치 1·4 | 7모델 × 6방식 × 배치 1·3·4 × CPU 스레드 6·1. ONNX Runtime CUDA는 이 GPU 커널이 없어 실패로 기록 | [보고서 3절](../deliverables/task7/01-optimization-report.md#3-지연--평균과-느린-쪽-5-해결할-것-2) | ✅ |
| 3 | 결과 차이를 원 단위로 | 과제 3 시험 표본 40,116개 전체로, PyTorch CPU 대비 CO₂·온도 최대·평균·p99 차이와 경보 바뀐 비율 | [보고서 2절](../deliverables/task7/01-optimization-report.md#2-결과-차이--원-단위-해결할-것-3-확인-기준-1) | ✅ |
| 4 | 전력·GPU·메모리 기록 | 측정마다 tegrastats(VDD_IN·GR3D·온도), RSS 최대, GPU 할당, TensorRT 장치 메모리. 1회 1ms 미만은 1초 전력계로 1회 에너지를 잴 수 없다는 한계를 명시 | [보고서 4절](../deliverables/task7/01-optimization-report.md#4-메모리전력-해결할-것-4) | ✅ |

## 확인 기준

| # | 기준 | 결과 | 판정 | 측정 조건 |
|---|---|---|---|---|
| 1 | CO₂ 차이 ≤ 1ppm, 분류 바뀐 비율 ≤ 0.1% | TensorRT FP16: GRU 2.43·LSTM 2.21ppm ❌, TCN 0.38 ✅, 오토인코더 ≤ 0.012% ✅. TensorRT FP32·ONNX Runtime CPU·PyTorch GPU: 모두 ≤ 0.008ppm, 0% ✅ | FP16 일부 ❌ / 채택 방식 ✅ | 시험 표본 40,116 |
| 2 | 측정 조건 기록 | 전력 모드 25W, 클럭 고정 안 함, 워밍업 50, 반복 1,000, 3구간, 스레드 수, 다른 GPU 프로세스 목록, 경합 표시(159개 중 2개)를 측정 파일마다 남김 | ✅ | `var/edgeopt/bench/*.json` |

## 산출물

| # | 산출물 | 위치 | 상태 |
|---|---|---|---|
| 1 | 최적화 보고서(방식 × 모델 지연·오차·메모리·전력 표, 결론) | [01-optimization-report.md](../deliverables/task7/01-optimization-report.md), 상세 [표](../deliverables/task7/tables/01-runners.md) | ✅ |

## 생각해 볼 점

**Q1. CPU로도 1ms면 충분히 빠른데 왜 굳이 변환하나?**

- 실측하면 **PyTorch CPU는 1ms가 아니다.** 예측 10.6ms, LSTM 24.5ms이고, 배치 4의 p95는 39ms다. 파이썬·연산자 호출 비용 때문이다.
- 변환의 이득은 "GPU로 옮기기"보다 **런타임을 가볍게 바꾸기**에서 나왔다. 같은 CPU에서 ONNX Runtime은 0.21ms로 50배 빠르다.
- 힌트의 세 가지로 보면 다음과 같다.
  - **함께 올라갈 다른 모델**: 로컬 LLM이 GPU와 메모리를 크게 쓴다. ONNX Runtime CPU는 GPU를 쓰지 않고, 추가 메모리가 38MB다(PyTorch GPU는 +614MB).
  - **전력**: 매분 1회라 차이가 작다. GPU를 깨우지 않는 쪽이 유리하다.
  - **공간 확장**: 배치 4도 0.5ms다. 공간이 수십 개로 늘어도 수 ms 안이다.
- 모델이 커지면(과제 6 사전학습 모델) 그때는 TensorRT가 맞다.

**Q2. GRU/LSTM과 TCN 중 무엇이 변환 후 더 빨라지나? 이유는?**

- **TCN이 훨씬 많이 빨라진다**(배치 1, PyTorch GPU → TensorRT FP16).
  - TCN: 3.36 → 0.63ms, **5.3배**
  - GRU: 1.32 → 1.14ms, **1.16배**
- TCN은 합성곱이라 24칸을 **한꺼번에** 계산한다. TensorRT가 층을 합치고(합성곱 + 활성화) 커널 수를 줄이면 그만큼 빨라진다. PyTorch에서는 층마다 커널을 따로 띄워 느렸다.
- GRU·LSTM은 칸마다 앞 칸의 결과가 필요해 **순서대로** 계산해야 한다. 합칠 여지가 적고, PyTorch도 이미 cuDNN 순환 커널 하나로 돌고 있었다.
- 정확도도 같은 이유로 갈린다. 순환은 FP16 반올림이 칸마다 쌓여 2~3ppm, TCN은 0.38ppm이다.

**Q3. 평균 지연보다 '느린 쪽 5%'가 상시 서비스에서 더 중요한 이유는?**

- 서비스는 매분 정해진 시간 안에 끝나야 한다. 평균이 빨라도 가끔 튀면 그 주기가 늦거나 겹친다.
- 같은 보드에서 다른 일이 돌수록 튐이 커진다. 실측 예는 다음과 같다.
  - ONNX Runtime CPU 스레드 6 LSTM은 평균 0.30ms인데 p99가 **3.75ms(12배)**였다. 스레드 1은 튀지 않았다.
  - PyTorch CPU 배치 4는 평균 13.5ms인데 p95가 **39ms**였다.
- 그래서 실행 방식을 고를 때 평균보다 p95·p99를 봤다. 서비스에는 **스레드 1**을 쓴다(공존하는 LLM과 CPU를 덜 다툼).
- 과제 8에서도 같은 현상이 보였다. 이 측정과 겹친 시간에 서비스 주기 p95가 0.8초 → 2.3초로 올랐다.

## 심화(B)

| # | 심화 항목 | 결과 | 문서 |
|---|---|---|---|
| 1 | INT8 변환, 정확도 손실 vs 속도 이득 | 학습 기간 표본 2,000개로 보정해 4모델 빌드 성공. 결과 차이·지연이 FP16과 같다(INT8 커널이 없는 순환층은 FP16으로 남음). TCN은 p95가 오히려 나빠짐. **이득 없음** | [01-int8.md](../advanced/task7/01-int8.md) |
| 2 | 4모델 동시 적재 메모리, LLM 조합 | ONNX Runtime CPU RSS 552MB(+38MB), PyTorch GPU 1,129MB(+614MB). LLM 4비트는 3B까지 안전, 7B는 여유 0.1GB(다른 계정 사용분을 빼면 부족) | [02-colocation-llm.md](../advanced/task7/02-colocation-llm.md) |

## 확인 명령

Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 실행한다. 2026-10-21 23:30에 실제로 실행해 맞췄다.

| 기능 | 명령 | 정상 결과 |
|---|---|---|
| 전력 모드 | `nvpmodel -q` | `NV Power Mode: 25W` |
| 시스템 TensorRT 연결 | `.venv/bin/python -c "import tensorrt as t; print(t.__version__)"` | `10.16.2.10` |
| 동작 확인 | `edgeopt probe` | `trt_fp16` ✅, `ort_cuda` ❌(커널 없음) |
| 변환 | `edgeopt export && edgeopt build` (`--fp32`, `--int8 --only-extra`) | 모델별 ✅와 빌드 시간 |
| 결과 차이 | `edgeopt diff` | 모델 × 방식 판정표 |
| 지연 측정 | `edgeopt bench`(약 1시간 30분) | 조합별 `ok 평균 …ms p95 …ms` |
| 동시 적재 | `edgeopt colocate --runner ort_cpu` | RSS 552MB, 한 주기 0.81ms, LLM 표 |
| 보고 표 | `edgeopt report` | `docs/deliverables/task7/tables/`, `docs/advanced/task7/tables/` |
| 서비스용 실행 묶음 | `edgeopt runtime --runner ort_cpu` | `var/edgeopt/runtime/runtime.json`(ONNX 4개, 해시) |
| 단위 시험 | `.venv/bin/python -m pytest tests/edgeopt` / `-m trt` | 21 passed / 1 passed |

## 남은 것

- 오토인코더 INT8: 학습 기간 보정 창을 따로 만들면 시도할 수 있다. 다만 위 결과로 보아 이득이 없을 것으로 본다.
