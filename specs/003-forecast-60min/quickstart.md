# Quickstart: 60분 예측 (과제 3)

명령은 Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 입력한다. `forecast …`는 어디서든 동작한다.

## 준비

```bash
# torch(cu130, aarch64)는 별도 index에서 설치
~/.local/bin/uv pip install --python .venv/bin/python torch --index-url https://download.pytorch.org/whl/cu130
~/.local/bin/uv pip install --python .venv/bin/python -e '.[test,analysis,forecast]'
analysis info          # 과제 2 입력 확인: 평가 시점 sha256 2f878c11d09ab68d… 일치
forecast check         # GPU(Orin)·결정성·입력 해시·설정 확인
```

정상 결과: `장치 cuda (Orin)`, `평가 시점 해시 일치`, `설정 ok`. GPU가 `False`이면 `id`로 현재 그룹을 확인한다.
`tools/forecast`가 `sg`로 다시 실행하는지도 본다.

## 검증 시나리오

| # | 시나리오 | 명령 | 기대 결과 |
|---|---|---|---|
| 1 | 단위 시험(합성 데이터) | `.venv/bin/python -m pytest tests/forecast -m unit` | 모두 통과. 인과적 입력, 마스크 손실, 형상, 반복 예측, 재개 시험 포함 |
| 2 | **누수 시험**(확인 기준 3) | `.venv/bin/python -m pytest tests/forecast/test_leakage.py` | 시험 기간 값을 바꿔도 가중치·조기 종료 에폭·선택 결과가 같다 |
| 3 | 배열·표본 | `forecast build` | 공간별 학습·검증·시험 표본 수, 예측 불가 수, 추정 예약 건수 |
| 4 | 기본 실험 1회 | `forecast train D0 --seed 0` | 몇 분 안에 끝나고 검증 MAE 표, `run.json` 생성. 최대 메모리 2GB 이하 |
| 5 | 전체 실험 | `forecast run all` | 단계별로 진행. 끊긴 뒤 다시 실행하면 끝난 실행은 건너뜀 |
| 6 | 선택 고정 | `forecast select` | `selection.json`, 단계별 고른 값과 이유 |
| 7 | 시험 평가 | `forecast test` | 평가 시점 행 수가 기준선과 같음, 확인 기준 판정표 |
| 8 | 선택 전 시험 거부 | (`selection.json` 없이) `forecast test` | 오류로 거부 |
| 9 | 보고서 | `forecast report` | `docs/deliverables/task3/` 3개, `docs/advanced/task3/` 3개, 그래프 |

## 확인 기준과 판정 위치

| 확인 기준 | 판정 위치 |
|---|---|
| CO₂ 60분 MAE ≤ 현재값 유지 × 0.7 | `forecast test` 판정표, `docs/deliverables/task3/01-model-comparison.md` |
| 온도 60분 MAE < 현재값 유지 | 같은 표 |
| 시험 기간을 학습·선택에 쓰지 않음 | 시나리오 2 + 8, `selection.json`의 고정 시각 < 시험 평가 시각 |
