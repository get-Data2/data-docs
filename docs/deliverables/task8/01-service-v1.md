# 과제 8 산출물 ① 예측 서비스 v1 — 운영 상태·구성·운영 방법

> **측정 조건**: 실데이터(과제 1 수집기 → MySQL), Jetson Orin Nano Super 25W(클럭 고정 안 함), 2026-10-21 20:17 KST 기동 기준.
> 같은 보드에서 수집기, 다른 계정(t2) 작업이 함께 돈다.

## 운영 상태(2026-10-21 20:48 KST)

| 항목 | 값 |
|---|---|
| 서비스 | `wonil-inference`(systemd 사용자 서비스, 부팅 시 자동 시작·`Linger=yes`), 브로커 `wonil-broker` |
| 주기 | 매분 :20초. 공간 3곳(실습실·사무실·회의실) |
| 판단 | 예측 `ok` · 이상 `ok` · 상태 `ok` · 규칙 `ok` |
| 모델 | 예측 `forecast-52aff4f0c4bf`(과제 3 GRU 2시간) · 이상 `anomaly-240088924310`(과제 4 공간별 오토인코더) · 상태 `state-1cd4d961a332`(과제 5 HGB) |
| 실행 방식 | 예측 **ONNX Runtime CPU**(`onnxruntime-cpu`), 이상·상태 PyTorch CPU. 과제 7 결과 `var/edgeopt/runtime/runtime.json`을 읽어 고른다. ONNX가 지금 가중치에서 만든 것이 아니면 PyTorch로 내려간다. 12시간 기록(10-21 20:20 ~ 10-22 08:20)은 `torch-cuda`로 했고, 10-22 08:54에 바꿨다 |
| 처리 시간 | 주기 전체 p50 0.8초, p95 2.4초(같은 보드에서 다른 실험이 돌 때 포함). 목표 20초 |
| 메모리 | 프로세스 RSS **689MB**(ONNX Runtime CPU). PyTorch GPU일 때는 1,409MB에서 12시간 평탄. 상한 1.5GB |
| 제공 | MQTT 보드 브로커 `:11883`(보존 메시지 16토픽), 조회 API `:18080` |

12시간 가동 요약은 [산출물 ②](02-12h-run-summary.md), 결과 형식은 [산출물 ③](03-result-format.md)에 있다.

## 구성

```
수집기(과제 1) ──MySQL measurement(1분 원본)──▶ wonil-inference
                                                   │ 1. 창 읽기: 최근 5시간, 이후 증분(늦은 행 10분 보정)
                                                   │ 2. 공통 전처리(src/preprocess): 축 A → 닫힌 5분 칸 → 공간 중앙값 → 앞으로 채우기
                                                   │ 3. 판단: 예측(과제 3) · 이상(과제 4) · 상태(과제 5) · 규칙(끊김·값 고정·범위 밖)
                                                   │ 4. 경보 상태 기계(생김·재알림 30분·해제 3주기)
                                                   ├──▶ 보드 브로커 :11883  team2/inference/v1/…(보존)
                                                   ├──▶ 조회 API :18080   /v1/spaces/{space}/…
                                                   └──▶ 기록: var/service/*.jsonl + MySQL svc_cycle·svc_alert·svc_forecast_log·svc_event
```

| 부분 | 코드 | 역할 |
|---|---|---|
| 창 읽기 | `src/service/window.py` | MySQL 증분 읽기. **자동 커밋**(아래 "발견한 문제") |
| 전처리 | `src/preprocess/`, `src/service/prep.py` | 학습과 같은 함수. 진행 중인 칸은 쓰지 않음 |
| 묶음 | `src/service/bundles.py` | 입력 규약 버전 검사, 실패하면 그 판단만 `unavailable` |
| 판단 | `src/service/infer.py`, `rules.py` | 판단마다 예외 격리 |
| 경보 | `src/service/alerts.py` | 매분 같은 경보 반복 금지, 재시작 뒤 중복 `raised` 없음 |
| 제공 | `publish.py`, `api.py`, `broker.py` | 보존 발행, 재접속 시 최신값 재발행 / GET 전용 API |
| 감시 | `loop.py`, `selftest.py` | 감시 스레드(3주기 진행 없으면 종료 → systemd 재시작), 60분 뒤 자기 채점 |

## 장애에 버티는 방법

| 장애 | 동작 | 확인 |
|---|---|---|
| 데이터 일부 끊김 | 장치마다 `data_gap` 경보(마지막 수신 > max(10분, 주기 × 2)). 15분 넘으면 `stale`·이상 판단 보류, 60분 넘으면 예측 중단 | 수집기 10분 정지 시험(산출물 ②) |
| 한 주기 실패 | DB 시간 초과 → `partial`, 다음 주기 정상. 판단 하나의 예외는 그 판단만 `error:` | `tests/service/test_cycle.py` |
| 프로세스 멈춤 | 감시 스레드가 종료 → `Restart=always`로 10초 뒤 재시작. 경보 상태는 `state.json`에서 복원 | 단위 시험 |
| 브로커 끊김 | 계산은 계속, 재접속하면 최신 보존 메시지 재발행 | `tests/service/test_publish.py` |
| 재부팅 | 사용자 로그인 없이 시작(`enabled` + `Linger=yes`) | 설정 확인 완료. 실제 재부팅 시험은 사용자 일정(보드를 다른 계정과 공유) |
| 메모리 | `MemoryMax=1536M`. 넘으면 cgroup이 서비스만 끝내고 재시작 | 12시간 동안 1,409MB 평탄(여유 127MB) → CPU 실행으로 689MB |

## 운영 방법

| 하고 싶은 것 | 명령(Jetson 터미널) | 정상 결과 |
|---|---|---|
| 상태 보기 | `infer status` | `상태 ok`, 판단 3개 `ok` |
| 지금 한 주기 결과 | `infer once` | 공간별 JSON |
| 새 모델 묶음 적용 | `infer reload` 또는 `systemctl --user reload wonil-inference` | 다음 주기에 `reload` 사건, 재시작 없음 |
| 가동 요약 | `infer report --from "2026-10-21 20:20"` | 주기·처리 시간·경보·사건 표 |
| 자기 채점 | `infer selftest` | 공간·항목·지평별 이동 MAE |
| 장애 시험 | `infer fault-test collector --minutes 10` | 수집기 정지 → 재시작, 사건 기록 |
| 로그 | `journalctl --user -u wonil-inference -f` | |
| 설치·갱신 | `deploy/install-inference.sh` | 두 서비스 `active` |

## 발견한 문제와 고친 것(2026-10-21)

1. **DB 스냅샷에 갇힘**
   - 서비스는 MySQL 연결 하나로 매분 읽는다. 과제 1 연결 설정은 자동 커밋이 꺼져 있다.
   - 그래서 첫 SELECT가 연 트랜잭션의 스냅샷(REPEATABLE READ)이 계속 쓰였다. 서비스는 시작 시각 이후 행을 보지 못했다.
   - 증상: 기동 10분 뒤 모든 장치에 `data_gap` 경보가 났고, 2시간 동안 재알림이 이어졌다. 수집기는 정상이었다.
   - 고침: 읽기 연결에 자동 커밋을 켰다(`window.py`). 재발 방지 시험 `tests/service/test_failures.py`를 추가했다.
   - 첫 장애 시험은 이 문제로 오염돼 버리고 다시 했다.
2. **다시 읽기 신호가 서비스를 재시작시킴**
   - GPU 그룹을 반영하려고 `sg`로 감싸 실행하므로, systemd의 주 프로세스는 파이썬이 아니라 `sg`다.
   - 이전 `ExecReload`는 HUP을 `sg`에 보냈다. 그러면 서비스가 다시 읽는 대신 재시작됐다.
   - 고침: 파이썬이 PID 파일(`var/service/infer.pid`)을 쓰고, `infer reload`가 그 프로세스에만 HUP을 보낸다.

## 한계

- **회의실 상태 판단은 시간의 약 25~36%만 나온다.** 과제 5 모델은 최근 30분 중 관측 칸이 50% 이상일 때만 판단하도록 학습됐다. 회의실은 10분 주기 센서 1대라 이 조건을 자주 못 채운다. 학습과 같은 조건을 지키려고 그대로 두었다. 과제 5에서 회의실용 창을 60분으로 늘려 다시 학습하는 것이 다음 할 일이다.
- 이상 판단은 과제 4 결과 그대로다. 시험 기간 헛경보가 하루 4.65회였으므로 이상 경보는 "확인 요청" 수준으로 쓴다.
- 조회 API는 인증 없는 읽기 전용이다. 보드 내부망·Tailscale 안에서만 열려 있다.
