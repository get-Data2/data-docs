# Quickstart: 이상 탐지 (과제 4)

명령은 Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 입력한다. `anomaly …`는 어디서든 동작한다.

## 준비

```bash
analysis info        # 과제 2 입력 확인
anomaly check        # 입력·장치·설정
```

## 검증 시나리오

| # | 시나리오 | 명령 | 기대 결과 |
|---|---|---|---|
| 1 | 단위 시험 | `.venv/bin/python -m pytest tests/anomaly` | 모두 통과. 주입 재현, 주입 밖 값 동일, 규칙, 매칭·지표, 임계값 누수 포함 |
| 2 | 평가판 | `anomaly inject` | 고장 60건(종류마다 12), 종류 × 공간 표, 같은 씨앗이면 같은 해시 |
| 3 | 학습 | `anomaly train` | 공간별 학습 기록, 공간당 5분 이내, 메모리 2GB 이하 |
| 4 | 점수 | `anomaly score` | 검증·평가판 점수 파일 |
| 5 | 임계값 고정 | `anomaly threshold` | 탐지기별 고른 분위수와 검증 하루당 경보 ≤ 0.5 |
| 6 | 평가 | `anomaly evaluate` | 성적표, 확인 기준 판정(결합 ≥ 80%, 헛경보 ≤ 1/일) |
| 7 | 임계값 고정 전 평가 거부 | (`thresholds.json` 없이) `anomaly evaluate` | 오류 |
| 8 | 보고서 | `anomaly report` | `docs/deliverables/task4/`, `docs/advanced/task4/` 표·그래프 |

## 확인 기준과 판정 위치

| 확인 기준 | 위치 |
|---|---|
| 결합 탐지율 ≥ 80% | `anomaly evaluate` 판정표, `docs/deliverables/task4/01-scorecard.md` |
| 헛경보 ≤ 하루 1회(전체 공간 합) | 같은 표(실제 이상 후보 겹침 별도) |
| 모델·규칙이 못 잡는 종류와 이유 | 성적표의 종류별 열 + 해석 절 |
