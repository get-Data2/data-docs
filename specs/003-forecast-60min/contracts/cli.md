# Contract: `forecast` CLI

실행은 `forecast <명령>`(→ `tools/forecast`, 어디서든) 또는 프로젝트 폴더에서 `.venv/bin/python -m forecast <명령>`이다.
`tools/forecast`는 현재 프로세스에 `video` 그룹이 없고 계정에는 있으면 `sg video`·`sg render`로 자신을 다시 실행한다.

공통 옵션: `--config config/forecast.yaml`, `--device auto|cuda|cpu`(기본 auto)

| 명령 | 하는 일 | 출력 | 실패 조건(종료 코드 ≠ 0) |
|---|---|---|---|
| `check` | 장치(GPU 이름·CUDA·결정성), 과제 2 입력 파일 존재·해시, 설정 검증 | 표 한 장 | 입력 파일 없음, 평가 시점 해시 불일치, 설정 오류 |
| `build` | 공간 배열·특징·추정 예약을 만들어 캐시 | `var/forecast/arrays.npz`, `booking_estimated.parquet`, 표본 수 표 | 입력 없음 |
| `train <실험> [--seed N]` | 실험 하나 학습(씨앗 지정 없으면 설정의 씨앗 전부), 검증 예측 저장 | `runs/<실험>/<해시>-s<씨앗>/` | 학습 발산(NaN) → `failed` 기록 후 다음 씨앗 |
| `run <단계…>\|all [--retry-failed]` | 단계(D0~D4, B1, A2, A3) 실험을 순서대로, 끝난 실행은 건너뜀. 단계 끝마다 검증 기준 선택 반영 | 진행 로그, 단계 표 | 없음(실패는 기록하고 계속) |
| `select` | D1~D4·B1 결과로 최종 설정과 시험 대상 목록을 고정 | `var/forecast/selection.json` | 필요한 단계 미완료 |
| `test` | `selection.json`의 시험 대상만 시험 기간 예측·채점, 모델 묶음 저장 | `test_pred.parquet`, `var/forecast/bundle/` | 선택 파일 없음·해시 불일치 |
| `report` | 산출물·심화 Markdown·그래프 생성 | `docs/deliverables/task3/`, `docs/advanced/task3/` | 시험 결과 없음 |
| `info` | 실행 현황(단계별 완료/실패 수), 선택 상태, 해시 일치 | 표 | — |

단계 이름

| 단계 | 내용 |
|---|---|
| `D0` | 기본 설정 |
| `D1` | 과거 길이 30분 · 2시간 · 하루 |
| `D2` | 입력 묶음 누적(sensors → +time → +space → +schedule) |
| `D3` | 출력 방식(direct · recursive) |
| `D4` | 목표(delta · value) |
| `B1` | 심화 1: 과거 1·2·4시간 × GRU·LSTM·TCN |
| `A2` | 심화 2: 미래 예약 입력(`booking`, 회의실 추정 예약), 비교 대상은 같은 설정의 `booking` 없음 |
| `A3` | 심화 3: 처음 보는 공간(공간 하나씩 빼기) + 기준(`space` 끔, 세 공간 학습) |
| `S` | 일정 가치 쌍: 최종 설정 − `schedule`(US5, 선택에 쓰지 않음) |

`run all`의 순서는 D0 → D1 → D2 → D3 → D4 → B1 → S → A2 → A3이다. 그다음 `select` → `test` → `report`를 사람이 실행한다.
`run all --to-report`를 주면 이 셋까지 이어서 실행한다.
