# Quickstart: 과제 2 검증 절차

명령은 Jetson 터미널에서 입력한다.
- `analysis …`는 어디서든 동작한다.
- `.venv/bin/python …`은 `cd ~/Desktop/project` 뒤에 입력한다.
- Claude Code 대화창에서는 앞에 `!`를 붙인다.

설정은 [contracts/config.md](contracts/config.md), 명령은 [contracts/cli.md](contracts/cli.md), 파일은 [data-model.md](data-model.md)를 본다.

## 1. 준비

```bash
cd ~/Desktop/project
~/.local/bin/uv pip install --python .venv/bin/python -e '.[test,analysis]'
analysis info                       # 사본·분할·평가 시점 상태
```

## 2. 전체 실행

```bash
analysis all                        # 추출 → 정제 → 사건 → 분할 → 진단 → 기준선 (목표 10분 이내, SC-001)
```

기대 결과
- `docs/reports/task2/`에 진단서(`diagnosis_<split>.md`), 기준선 표(`baseline_<split>.md`), PNG 그래프가 생긴다.
- `var/eval/eval_points_<split>.parquet`과 그 SHA256이 생긴다.

## 3. 확인 기준별 검증

| 확인 기준 | 명령 | 정상 결과 |
|---|---|---|
| 정규화는 학습 기간만 | `.venv/bin/python -m pytest tests/analysis/test_split.py` | 시험 기간 값을 바꿔도 통계 불변(SC-002) |
| 기준선과 이후 모델이 같은 시점·조건 | `analysis info` | 평가 시점 SHA256이 기준선 결과 메타데이터와 같음. 세 기준선 시점 수 동일(SC-004) |
| 가장 강한 기준선과 이유 | `docs/reports/task2/baseline_<split>.md`의 해석 절 | 지평별 1위 방법과 근거 |
| 원본 행 보존·건수 검산 | `analysis clean` 출력의 검산 줄 | `ok + saturated + error = 원본`(SC-006) |
| 센서 오류와 실제 사건 분리 | `.venv/bin/python -m pytest tests/analysis/test_axes.py` | 화재 → B1, 오류 코드 → A error, 단독 튐 → B0, 밀집 → B3(SC-007, SC-008) |
| 짧은 결측 기준 근거 | 진단서 "보간 시험" 표 | 길이별 보간 MAE |

## 4. 분할 버전 바꾸기

InfluxDB 미적재 구간(2026-07-23 ~ 10-20)을 적재한 뒤 아래를 실행한다.

```bash
analysis extract --refresh
analysis all --split v2-final
```
