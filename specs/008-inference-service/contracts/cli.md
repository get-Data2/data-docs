# Contract: `infer` CLI

실행은 `infer <명령>`(→ `tools/infer`, 어디서든) 또는 `.venv/bin/python -m service <명령>`이다.
`tools/infer`는 GPU 그룹이 프로세스에 없고 계정에는 있으면 `sg video`·`sg render`로 다시 실행한다(`tools/forecast`와 같음).
상시 운영은 systemd 사용자 서비스 `wonil-inference`가 `infer run`을 실행한다.

공통 옵션: `--config config/service.yaml`, `--device auto|cuda|cpu`

| 명령 | 하는 일 | 출력 | 실패(종료 코드 ≠ 0) |
|---|---|---|---|
| `run` | 상시 실행: 매분 주기 + 조회 API + 발행. SIGTERM에 현재 주기를 마치고 종료 | 로그(journal) | 설정 오류(2), DB 첫 접속 실패는 재시도하며 계속 |
| `once [--at 시각]` | 주기 한 번 실행하고 결과 JSON 출력(발행 없음). `--at`은 과거 시각 재생 | 공간별 결과 | 입력 부족 |
| `status` | 실행 중인 서비스의 health(API 호출) 또는 마지막 주기 기록 요약 | 표 | 서비스 없음(3) |
| `report --from --to [--md 경로]` | 기간 요약: 주기 수·처리 시간 분포·경보 내역·장애 시험·메모리 | Markdown | 기록 없음 |
| `selftest` | 자기 성능 감시 현황(지평별 이동 MAE vs 과제 3 시험 MAE) | 표 | — |
| `reload` | 묶음 다시 적재(실행 중 서비스에 신호) | 결과 | 서비스 없음 |
| `simulate-control --days 7` | 심화 1 제어 규칙 시뮬레이션 | 표·Markdown | 데이터 부족 |
| `fault-test collector --minutes 10` | 장애 시험 보조: 수집기 정지 → 대기 → 재시작, 단계 기록(`svc_event`) | 진행 로그 | — |

설치: `deploy/install-inference.sh`는 사용자 서비스 등록·활성화와 `infer selftest`를 수행한다. `Linger=yes`는 과제 1에서 적용했다.
