# Quickstart: 사전학습 모델 비교 (과제 6)

명령은 Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 입력한다. `pretrained …`는 어디서든 동작한다.

## 준비

```bash
~/.local/bin/uv pip install --python .venv/bin/python -e '.[test,analysis,forecast,pretrained]'
.venv/bin/python -c "import torch; print(torch.__version__)"   # 2.14.1+cu130 그대로인지 확인
pretrained check
pretrained fetch          # 가중치를 받고 리비전 고정(인터넷 필요, 한 번만)
```

## 검증 시나리오

| # | 시나리오 | 명령 | 기대 결과 |
|---|---|---|---|
| 1 | 단위 시험 | `.venv/bin/python -m pytest tests/pretrained` | 모두 통과. 인과적 컨텍스트, 행 일치, 적중률, 선택 고정 전 시험 거부, 앙상블 검증 전용 |
| 2 | 검증 예측 | `pretrained predict val` | 크기 × 컨텍스트 조합 예측 파일 |
| 3 | 선택 고정 | `pretrained select` | 고른 크기·컨텍스트·앙상블 가중치와 이유 |
| 4 | 선택 전 시험 거부 | (`selection.json` 없이) `pretrained test` | 오류 |
| 5 | 시험 평가 | `pretrained test` | 세 방법 비교표, 평가 시점 수 = 기준선 표, 적중률 표 |
| 6 | 비용 측정 | `pretrained bench` | 모델 × 장치 × 배치 지연 평균·p95, RSS, GPU 할당, 전력, 측정 조건(클럭 고정 안 함) |
| 7 | 보고서 | `pretrained report` | `docs/deliverables/task6/` 비교표·판단표·예산표, `docs/advanced/task6/` |

## 확인 기준과 판정 위치

| 확인 기준 | 위치 |
|---|---|
| 세 방법을 같은 시점에서 비교 | `pretrained test` 출력의 행 수 일치 검사, `docs/deliverables/task6/01-comparison-adoption.md` |
| 정확도·적중률·추론 시간·메모리 네 근거 | 같은 문서의 판단표 |
