# Contract: `anomaly` CLI

실행은 `anomaly <명령>`(→ `tools/anomaly`, 어디서든) 또는 `.venv/bin/python -m anomaly <명령>`이다.
GPU 그룹 처리와 환경 변수는 `tools/forecast`와 같다.

공통 옵션: `--config config/anomaly.yaml`, `--device auto|cuda|cpu`

| 명령 | 하는 일 | 출력 | 실패(종료 코드 ≠ 0) |
|---|---|---|---|
| `check` | 입력 파일·해시, 장치, 설정 검증 | 표 | 입력 없음·설정 오류 |
| `inject` | 시험 기간 사본에 고장 주입 | `faults.parquet`, `board.parquet`, 종류 × 공간 건수 표 | 건수를 채우지 못함(후보 시각 부족) |
| `train` | 공간별 오토인코더 학습(학습 기간, 사건 후보 제외) | `models/`, 학습 기록 | — |
| `score [탐지기…]` | 검증 기간·평가판 점수 계산(기본: 전부, 심화 1은 모델 묶음이 있을 때만) | `scores/` | 모델 없음 |
| `threshold` | 검증 점수로 임계값 선택·고정 | `thresholds.json`, 후보 표 | 검증 점수 없음 |
| `evaluate` | 평가판 경보·매칭·지표·결합, 확인 기준 판정 | `alarms/`, `results.json`, 판정표 | `thresholds.json` 없음·해시 불일치 |
| `report` | 결과 표·그래프·운영 제안 | `docs/deliverables/task4/`, `docs/advanced/task4/` | 결과 없음 |
| `info` | 단계별 파일 상태와 해시 | 표 | — |
| `all` | inject → train → score → threshold → evaluate → report | 위 전부 | 단계 실패 시 멈춤 |
