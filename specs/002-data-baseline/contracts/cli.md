# Contract: 명령줄 `analysis`

- 어디서든: `analysis <명령>`(→ `tools/analysis`, `~/.local/bin/analysis`)
- 프로젝트 폴더에서: `.venv/bin/python -m analysis <명령>`
- 공통 옵션: `--config config/analysis.yaml`, `--split <버전>`(기본 `split.active`)
- 종료 코드: 0 성공, 2 설정 오류, 3 DB 접속 실패, 4 데이터 부족(분할 기간에 데이터 없음)

| 명령 | 하는 일 | 출력 |
|---|---|---|
| `extract [--refresh]` | MySQL → 월별 Parquet 사본(닫힌 달은 건너뜀) | `var/cache/raw/*.parquet`, `manifest.json` |
| `clean` | 축 A 품질 표시, 결측 판정·짧은 결측 보간, 5분 공간 시계열 | `var/cache/quality/`, `var/series/space_5min.parquet`, 정제 건수 표 |
| `events` | 축 B 사건 후보·의심 목록, 값 고정 후보 | `var/cache/events.parquet`, `docs/reports/task2/events.md` |
| `split` | 분할 검증, 학습 기간 정규화 통계 | `var/eval/norm_<split>.json` |
| `diagnose` | 결측률·제거 수·평일 패턴·주말 비교·반응 지연·어긋나는 날·자기상관·간격 비교·보간 시험 | `docs/reports/task2/diagnosis_<split>.md` + PNG |
| `baseline` | 평가 시점 생성, 기준선 3종 예측, MAE 표 | `var/eval/eval_points_<split>.parquet`(+json), `var/eval/baseline_<split>.parquet`, `docs/reports/task2/baseline_<split>.md` |
| `all` | 위 전부를 순서대로 | 위 전부, 걸린 시간 |
| `info` | 사본·분할·평가 시점 파일 상태와 해시 | 표 |

## 출력 표 예 (`baseline`)

```markdown
| 공간 | 항목 | 지평 | 현재값 유지 | 최근 1시간 평균 | 어제 같은 시각 | 평가 시점 수 |
|---|---|---|---|---|---|---|
| 실습실(lab) | co2 | 60분 | 48.2 | 61.0 | 95.3 | 15,880 |
```
(숫자는 형식 예시다.)
