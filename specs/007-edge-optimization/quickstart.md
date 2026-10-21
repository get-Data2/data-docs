# Quickstart: 엣지 최적화 (과제 7)

명령은 Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 입력한다. `edgeopt …`는 어디서든 동작한다.

## 준비

```bash
~/.local/bin/uv pip install --python .venv/bin/python onnx onnxruntime-gpu==1.31.0 tensorrt-cu13==11.4.0.106
~/.local/bin/uv pip install --python .venv/bin/python -e '.[test,analysis,forecast,edgeopt]'
nvpmodel -q                  # 25W 확인(측정 조건)
pgrep -af "forecast run|anomaly train"   # 우리 학습이 돌고 있으면 기다린다
edgeopt probe                # pip TensorRT·ORT GPU가 이 보드에서 동작하는지
```

정상 결과: `ort_cuda 동작·일치`, `trt_fp16 동작·일치`. TensorRT가 실패하면 research R2의 대안 순서를 따른다.

## 검증 시나리오

| # | 시나리오 | 명령 | 기대 결과 |
|---|---|---|---|
| 1 | 단위 시험 | `.venv/bin/python -m pytest tests/edgeopt -m unit` | 통과(내보내기 일치, 지연 통계, tegrastats 해석, 원 단위 차이 판정) |
| 2 | TensorRT 시험 | `.venv/bin/python -m pytest tests/edgeopt -m trt` | 합성 모델 엔진 결과 일치(probe 성공 뒤) |
| 3 | 변환 | `edgeopt export && edgeopt build` | 모델마다 ONNX·FP16 엔진, `convert.json`에 실패 사유 포함 |
| 4 | 측정 | `edgeopt bench` | 방식 × 모델 × 배치 측정 파일, 측정 조건·전력·메모리 포함 |
| 5 | 학습 중 측정 거부 | (과제 3 학습 중) `edgeopt bench` | 거부 메시지 |
| 6 | 결과 차이 | `edgeopt diff` | 판정표: CO₂ 최대 차이 ≤ 1ppm, 분류 바뀐 비율 ≤ 0.1% |
| 7 | 동시 적재 | `edgeopt colocate` | 총 메모리, LLM 크기 가정별 조합 표 |
| 8 | 보고서 | `edgeopt report` | `docs/deliverables/task7/`, `docs/advanced/task7/` 표·그래프 |
| 9 | 과제 8 묶음 | `edgeopt runtime --runner trt_fp16` | `var/edgeopt/runtime/runtime.json` |

## 확인 기준과 판정 위치

| 확인 기준 | 위치 |
|---|---|
| CO₂ 예측 차이 ≤ 1ppm | `edgeopt diff` 판정표, `docs/deliverables/task7/01-optimization-report.md` |
| 분류 바뀐 비율 ≤ 0.1% | 같은 표 |
| 측정 조건 기록 | 측정 파일 `conditions`, 보고서 측정 조건 블록 |
