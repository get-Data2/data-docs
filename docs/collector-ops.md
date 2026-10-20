# 수집기 운영 가이드 (과제 1)

MQTT로 들어오는 센서·냉난방 메시지를 MySQL `cf_u36_d3`에 쌓는 상시 서비스 `team2-collector`의 운영 방법이다.
설계 근거는 data-docs의 [plan](../specs/001-data-collection/plan.md)과 [research](../specs/001-data-collection/research.md)에 있다.

## 구성

```text
MQTT 브로커 ──구독(QoS1)──► 해석(config/collector.yaml) ──► 로컬 스풀 var/spool.db ──2초마다 일괄──► MySQL
                               │ 중복(fCnt) 버림                    │ DB 장애 시 여기 쌓임(최대 40만 행)
                               └─ 매분 ingest_stats, 사건 collector_event
```

| 항목 | 값 |
|---|---|
| 서비스 | systemd **사용자** 서비스 `team2-collector.service` (`~/.config/systemd/user/`) |
| 부팅 시 자동 시작 | `loginctl enable-linger` → `Linger=yes` (2026-10-20 설치 시 sudo 없이 적용됨) |
| 중복 실행 방지 | `flock var/collector.lock` |
| 재시작 | 비정상 종료 시 5초 뒤 자동(`Restart=always`) |
| 메모리 상한 | `MemoryMax=256M` |
| 로그 | `journalctl --user -u team2-collector` |

## 설치·갱신

```bash
deploy/install.sh                    # 설정 검증 → 테이블 생성 → 서비스 등록·시작 → linger(안 되면 cron @reboot)
systemctl --user restart team2-collector   # 코드를 바꾼 뒤
systemctl --user reload team2-collector    # 설정(config/collector.yaml)만 바꾼 뒤 — 재시작 없이 반영
```

## 상태 확인

```bash
systemctl --user status team2-collector
journalctl --user -u team2-collector -n 50 --no-pager
.venv/bin/python -m collector report --hours 1          # 최근 1시간 현황
.venv/bin/python -m collector report --hours 24 --md docs/reports/$(date +%F)-coverage.md
```

현황 보고에는 표 네 개가 나온다.
- ① 확인 필요 장치: 공간 미지정, 10분 넘게 수신 없음
- ② 공간·항목별 건수·마지막 수신·커버율
- ③ 장치별 커버율
- ④ 빠진 구간과 원인 단서

## 장애 대응

| 증상(보고·로그) | 원인 단서 | 조치 |
|---|---|---|
| 전 장치 동시 결측 + `mqtt_disconnected` | 브로커 재시작·네트워크 | 자동 재접속(최대 30초 간격). 1분 넘게 지속되면 브로커 상태 확인 |
| 전 장치 동시 결측, 수집기 사건 없음 | 원천 측(게이트웨이·ChirpStack) | 게이트웨이 전원·네트워크 확인 |
| 분 통계(`ingest_stats`)까지 빔 | 수집기 미가동 | `systemctl --user status`, 보드 전원·재부팅 기록 확인 |
| 한 장치만 결측 | 센서 배터리·전파 | 보고의 배터리 값 확인(예: AM103 `24e124725d081175`는 2026-10-20 배터리 1%) |
| `db_down` 사건, 스풀 증가 | MySQL 접속 불가 | 자동 재시도(최대 30초 간격). 복구되면 `db_up`과 함께 밀린 행 기록 |
| `spool_drop` 사건 | DB 장애가 스풀 한도(약 24시간)를 넘김 | 버린 구간이 사건 detail에 남는다. 결측 원인으로 보고서에 적는다 |
| `bad_row` 사건 | DB가 거부한 행(형식 이상) | 해당 행만 버리고 나머지는 기록. 로그에서 장치·항목 확인 후 설정 점검 |
| `config_rejected` 사건 | 설정 검증 실패 | 기존 설정으로 계속 동작. `python -m collector check-config`로 오류 위치 확인 |
| `stale_reconnect` 사건 | 접속은 됐는데 5분 무수신 | 조용히 죽은 연결로 보고 자동 재접속 |
| `mqtt_flapping` 사건, 로그에 1초 간격 접속·끊김(`Unspecified error`) | 브로커에 **같은 client_id**를 쓰는 다른 클라이언트 | `config/collector.yaml`의 `mqtt.client_id`를 고유하게 바꾸고 재시작(2026-10-20 설치 직후 실제 발생, `team2-collector` → `team2-jetson-collector-a1`로 해결) |

## 확인 기준 검증 기록

| 확인 기준 | 방법 | 결과 | 측정 조건 |
|---|---|---|---|
| 브로커 재시작 후 1분 안 재수집 | `pytest -m broker tests/integration/test_broker_restart.py` (보드 안 amqtt 브로커를 10초 정지 후 재시작) | **6.0초** 만에 첫 저장 | 2026-10-20, Jetson, 시험 브로커, 재접속 상한 30초 |
| DB 장애 무손실 | `test_spool_lossless.py` (DB 실패 주입 중 메시지 600건) | 손실 0, `db_down`/`db_up` 기록 | 2026-10-20, 가짜 기록기 |
| 실 브로커 수집 | `pytest -m live` (3분) | 장치 18대·항목 30종, 미지정 공간 0, \|ts−수신\| 최대 69.7초 | 2026-10-20 16:29~16:32 KST, 실데이터 |
| 보드 재부팅 후 자동 시작 | 아래 절차 | 시험 예정 | |
| 24시간 커버율 95% | `report --hours 24` | 수집 24시간 뒤 | |

### 재부팅 시험 절차

1. 재부팅 전에 `uptime -s`와 `systemctl --user is-enabled team2-collector`(enabled)를 확인한다.
2. 재부팅한 뒤 **로그인하지 않고** 3분을 기다린다.
3. 다른 PC에서 SSH로 접속해 아래를 확인하고 결과를 위 표에 적는다.
   - `uptime -s`(부팅 시각)
   - `report --hours 1`의 사건 중 `start` 시각
   - 첫 저장 시각: `SELECT MIN(received_at) FROM measurement WHERE received_at > <부팅 시각>`

## 관찰된 원천 특성 (2026-10-20)

- 게이트웨이 `24e124fffef5dccc`의 시계가 약 **4분 55초 빠르다**. ChirpStack `time` 필드가 이 시각을 그대로 쓰므로
  기준 시각에는 `rxInfo[].nsTime`의 최소값을 쓴다. 원본 `time`은 `source_ts`에 남긴다.
- ChirpStack이 **같은 업링크(같은 fCnt)를 두 번 발행**하는 경우가 있다. 게이트웨이 1대로 받은 판과 2대로 받은 판이
  `deduplicationId`가 다른 채로 따로 온다. 수집기가 (장치, fCnt)를 2시간 동안 기억해 한 번만 저장한다(`ingest_stats.duplicate`).
- 냉난방(xflow) `ts`는 수신보다 최대 약 60초 이르다. 상태가 바뀐 시각으로 보이며 허용 범위 안이다.
