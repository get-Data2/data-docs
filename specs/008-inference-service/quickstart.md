# Quickstart: 상시 추론 서비스 (과제 8)

명령은 Jetson 터미널에서 `cd ~/Desktop/project` 뒤에 입력한다. `infer …`는 어디서든 동작한다.

## 준비

```bash
forecast info; anomaly info                 # 과제 3·4 묶음 확인(과제 5 묶음은 var/state/bundle)
service once                                # 주기 한 번(발행 없음) — 공간별 결과 JSON
deploy/install-inference.sh                 # 사용자 서비스 wonil-inference 등록·시작
systemctl --user status wonil-inference
```

발행 대상은 보드 로컬 브로커(`wonil-broker`, 포트 11883)다(2026-10-21 결정). 서비스보다 먼저 떠 있어야 한다. 공용 MQTT 서버에는 발행하지 않는다.

## 검증 시나리오

| # | 시나리오 | 명령 | 기대 결과 |
|---|---|---|---|
| 1 | 단위 시험 | `.venv/bin/python -m pytest tests/service -m unit` | 전처리 일치, 경보 상태 기계, 신선도, 부분 칸 제외, API 응답 형식 |
| 2 | 보존 발행 | `.venv/bin/python -m pytest tests/service -m broker` | 새 구독자가 1초 안에 공간·종류별 최신값 수신, 브로커 재시작 뒤 재발행 |
| 3 | 조회 API | `curl -s localhost:8080/v1/spaces/meeting/forecast?minutes=30` | 30분 뒤 CO₂·온도, 단위, t0, 신선도 |
| 4 | **수집기 10분 정지** | `infer fault-test collector --minutes 10` | 기준 시간 안에 `data_gap` raised, 재시작 뒤 `cleared`와 정상 결과 |
| 5 | 주기 실패 회복 | (시험용 설정으로 DB 시간 초과 유도) | 그 주기 `partial`/`failed`, 다음 주기 `ok` |
| 6 | 재부팅 | 보드 재부팅 | 로그인 없이 3분 안 첫 결과(`svc_cycle`) |
| 7 | **12시간 가동** | 12시간 뒤 `infer report --from … --to … --md docs/deliverables/task8/02-12h-run-summary.md` | 예정 720주기 중 빠짐 0, 처리 시간 p95 ≤ 20초, 경보 내역 |
| 8 | 결과 형식 문서 확인 | 문서의 구독·조회 예제를 그대로 실행 | 문서 예시와 같은 형식 |

## 확인 기준과 판정 위치

| 확인 기준 | 위치 |
|---|---|
| 12시간 무중단 | 시나리오 7, `docs/deliverables/task8/02-12h-run-summary.md` |
| 수집기 10분 정지 → 끊김 경보 → 복귀 | 시나리오 4, 같은 요약의 장애 시험 절 |
| 조회 API로 공간별 예측·이상·상태 | 시나리오 3, `docs/deliverables/task8/03-result-format.md` 예제 |
| 결과 형식 문서 | `docs/deliverables/task8/03-result-format.md` |
