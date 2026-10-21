# Quickstart: 상태 추정 (과제 5)

명령은 Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 입력한다. `state …`는 어디서든 동작한다.

## 준비

```bash
~/.local/bin/uv pip install --python .venv/bin/python -e '.[test,analysis,forecast,state]'
analysis extract && analysis all     # 실시간 수집 구간(2026-10-20~)까지 사본 갱신(관찰 평가용)
state check
```

## 직접 관찰 기록(사람)

```bash
state observe lab occupied --people 18     # 실습실에 사람이 있음(지금 시각)
state observe lab empty                    # 모두 나감
state observe meeting occupied --people 4 --note "회의 시작"
state observe meeting stop                 # 관찰 끝
state observe --status                     # 진행률(목표: 공간별 수업일 2일, 총 20시간, 경계 4회)
```

## 검증 시나리오

| # | 시나리오 | 명령 | 기대 결과 |
|---|---|---|---|
| 1 | 단위 시험 | `.venv/bin/python -m pytest tests/state` | 모두 통과. 라벨 경계, 관찰 검증, 특징 인과성, 누수, 선택 고정, 지표 손 계산 포함 |
| 2 | 사용 여부 일정 기반 입력 금지 | `config/state.yaml`의 occupancy에 `time`을 넣고 `state check` | 오류로 거부 |
| 3 | 정답 | `state labels` | 환기 양성 비율(약 1~2%), 이미 초과 비율, 관찰과 일정 정답 불일치 표 |
| 4 | 학습·선택 | `state train all && state select` | 후보 표, 고른 분류기·임계값(환기 검증 재현율 ≥ 0.85) |
| 5 | 선택 전 평가 거부 | (`selection.json` 없이) `state evaluate` | 오류 |
| 6 | 평가 | `state evaluate` | 판정표: 관찰 기준 정확도·F1·비어있음→사용중 오류율, 환기 재현율·macro-F1 vs 규칙 |
| 7 | 오류 분석 | `state analyze` | 전환 뒤 경과별 오류, 상위 혼동 상황 3개, 헛경보 |
| 8 | 보고서 | `state report` | `docs/deliverables/task5/`, `docs/advanced/task5/` |

## 확인 기준과 판정 위치

| 확인 기준 | 위치 |
|---|---|
| 사용 여부: 관찰 기준 정확도·F1, 비어있음→사용중 오류율, 규칙보다 좋음 | `state evaluate` 판정표, `docs/deliverables/task5/01-scorecards.md` |
| 환기: 재현율 ≥ 85%, 규칙보다 macro-F1 높음 | 같은 표 |
| 혼동 행렬과 가장 헷갈리는 경우 | `01-scorecards.md`, `03-error-cases.md` |
