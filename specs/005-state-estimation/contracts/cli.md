# Contract: `state` CLI

실행은 `state <명령>`(→ `tools/state`, 어디서든) 또는 `.venv/bin/python -m state <명령>`이다.
GPU 그룹 처리와 환경 변수는 `tools/forecast`와 같다.

공통 옵션: `--config config/state.yaml`, `--device auto|cuda|cpu`

| 명령 | 하는 일 | 출력 | 실패(종료 코드 ≠ 0) |
|---|---|---|---|
| `check` | 입력 파일·해시, 장치, 설정 검증(사용 여부 특징에 time·schedule 금지) | 표 | 입력 없음·설정 오류 |
| `observe <공간> occupied\|empty [--people N] [--note 텍스트] [--at "YYYY-MM-DD HH:MM"]` | 그 공간의 열린 구간을 닫고 새 구간을 연다(시각 기본 = 지금, KST) | 한 줄 확인 | 모르는 공간, 겹침 |
| `observe <공간> stop` | 열린 구간을 닫는다 | 한 줄 확인 | 열린 구간 없음 |
| `observe --status` | 공간별 관찰 시간, 상태 비율, 경계 포함 횟수, 목표 대비 진행률, 열린 구간 | 표 | — |
| `observe --check` | 관찰 CSV 검증, 문 열림과 어긋나는 구간 표시(research R3) | 표 | 형식 오류(줄 번호) |
| `labels` | 사용 여부·환기 정답 생성, 관찰과 일정 정답 비교 | `labels_*.parquet`, 비교 표 | — |
| `features` | 특징 표 생성 | `features.parquet`, `features.json` | 입력 없음 |
| `train [occupancy\|vent\|leak\|all]` | 후보 × 격자 × 씨앗 학습, 검증 지표(끝난 것은 건너뜀) | `runs/` | 실패는 기록하고 계속 |
| `select` | 검증 기준 후보·임계값 선택·고정 | `selection.json`, 후보 표 | 학습 미완료 |
| `evaluate` | 시험·관찰 평가, 단순 규칙·과제 3 판정 비교, 확인 기준 판정, 묶음 저장 | `eval/results.json`, `bundle/`, 판정표 | `selection.json` 없음·해시 불일치 |
| `analyze` | 오류 분석(전환 뒤 경과·인원·상위 혼동·헛경보), 심화 2개 | `eval/analysis.json` | 평가 결과 없음 |
| `report` | 결과 표·그래프 | `docs/deliverables/task5/`, `docs/advanced/task5/` | — |
| `info` | 단계별 파일·해시·관찰 진행률 | 표 | — |
| `all` | labels → features → train → select → evaluate → analyze → report | | 단계 실패 시 멈춤 |
