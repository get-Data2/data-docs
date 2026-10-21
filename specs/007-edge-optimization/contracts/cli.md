# Contract: `edgeopt` CLI

실행은 `edgeopt <명령>`(→ `tools/edgeopt`, 어디서든) 또는 `.venv/bin/python -m edgeopt <명령>`이다.
`tools/edgeopt`가 하는 일은 다음과 같다.
- GPU 그룹 처리(`tools/forecast`와 같음)
- pip TensorRT 공유 라이브러리 경로(`LD_LIBRARY_PATH`) 설정
- `CUBLAS_WORKSPACE_CONFIG` 설정

공통 옵션: `--config config/edgeopt.yaml`

| 명령 | 하는 일 | 출력 | 실패(종료 코드 ≠ 0) |
|---|---|---|---|
| `probe` | 합성 작은 모델로 ONNX 내보내기, ORT CPU·CUDA EP, TensorRT 엔진 생성·실행, ORT TensorRT EP 동작 확인 | 표(방식별 동작·결과 일치) → `var/edgeopt/probe.json` | 없음(실패도 기록) |
| `export [모델…]` | 묶음 → ONNX + 검증 | `onnx/`, `convert.json` | 묶음 없음 |
| `build [모델…] [--int8]` | ONNX → TensorRT FP16(·INT8) 엔진 | `engines/`, `convert.json` | `probe`에서 TensorRT 미동작 |
| `bench [--runner R…] [--model M…] [--batch B…]` | 조합마다 새 프로세스에서 지연·자원 측정 | `bench/`, `tegra/` | 우리 학습 프로세스가 실행 중이면 거부 |
| `diff [--runner R…] [--model M…]` | 시험 표본 전체 원 단위 차이, 판정 | `diff/`, `results.json` 판정표 | 기준 출력 없음 |
| `colocate [--runner R]` | 서비스 구성 동시 적재 메모리·지연, LLM 조합 표 | `colocate.json` | — |
| `report` | 표·그래프 생성 | `docs/deliverables/task7/tables`, `figures`, `docs/advanced/task7/tables` | 측정 없음 |
| `runtime --runner R` | 과제 8용 실행 묶음 고정 | `runtime/` | 그 방식의 판정 실패 |
| `info` | 변환·측정 현황, 버전 | 표 | — |
| `all` | probe → export → build → bench → diff → colocate → report | 위 전부 | 단계 실패 시 다음 단계로(기록) |
