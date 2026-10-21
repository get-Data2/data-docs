# 수집기 운영 가이드 (과제 1)

MQTT로 들어오는 센서·냉난방 메시지를 MySQL `cf_u36_d3`에 쌓는 상시 서비스 `wonil-collector`의 운영 방법이다.
설계 근거는 data-docs의 [plan](../specs/001-data-collection/plan.md)과 [research](../specs/001-data-collection/research.md)에 있다.

## 구성

```text
MQTT 브로커 ──구독(QoS1)──► 해석(config/collector.yaml) ──► 로컬 스풀 var/spool.db ──2초마다 일괄──► MySQL
                               │ 중복(fCnt) 버림                    │ DB 장애 시 여기 쌓임(최대 40만 행)
                               └─ 매분 ingest_stats, 사건 collector_event
```

## 코드 구성 (코드 저장소 `get-Data2/data`)

```text
src/collector/
├── __main__.py   명령줄: run · check · check-config · init-db · report · replay · backfill
├── config.py     설정(config/collector.yaml) 읽기·검증, 공간 매핑(장치 지정 → 별칭 → unmapped), 항목 별칭, 기대 주기
├── pathexpr.py   설정의 경로 표현식 해석기(a.b, a[*].b, min(), topic[n], values.*, split())
├── timeutil.py   시각 해석(ISO·epoch ms/s) — 저장 UTC, 표시 KST
├── parser.py     MQTT 메시지 → 측정값 행. 기준 시각 결정(nsTime 우선, 허용 범위 밖이면 수신 시각)
├── dedup.py      (장치, fCnt) 2시간 창 — ChirpStack 이중 발행 걸러내기
├── spool.py      SQLite 스풀(var/spool.db) — DB에 쓰기 전에 보드에 먼저 남김(무손실)
├── store.py      MySQL 연결·DDL(schema.sql)·일괄 기록(멱등)
├── service.py    상시 수집기: MQTT 구독(재접속·무수신 감시·client_id 충돌 감지) + 기록 스레드(백오프·거부 행 격리)
├── report.py     현황 보고: 커버율·빠진 구간·원인 단서·관측 주기
├── capacity.py   저장소 크기·1년 추정
├── replay.py     저장된 메시지 파일을 해석만 해 보기(형식 변경 대응 확인)
├── influx.py     InfluxDB 읽기 전용 Flux 클라이언트
├── backfill.py   InfluxDB 과거 데이터 → MySQL 적재(구간별 재개·멱등)
└── check.py      연결 점검(서비스·MQTT·MySQL·InfluxDB)
config/collector.yaml   원천 규칙·공간 매핑·항목 별칭·장치별 주기 — 형식 변경·센서 추가는 여기만 고친다
deploy/                 wonil-collector.service, install.sh, run_forever.sh(cron 대체), wonil-backfill.{service,timer}, install-backfill.sh
tests/                  unit(오프라인) · broker(보드 안 amqtt) · mysql(test_ 테이블) · live(실 브로커)
```

흐름은 이렇다. 메시지가 `service._on_message`로 들어오면 `parser.parse`가 측정값으로 풀고, `dedup`으로 중복을 거른 뒤
`spool.put_many`로 보드에 먼저 남긴다. 그다음 기록 스레드가 `store.write_batch`로 MySQL에 옮기고, 성공한 만큼 스풀에서 지운다(`spool.ack`).

## 명령 실행 위치

`.venv/bin/python -m collector …`는 **프로젝트 폴더(`~/Desktop/project`)에서** 입력한다. `.venv`, `config/`, `.env`가 그 폴더 기준 상대 경로이기 때문이다.
어디서든 쓰려면 `collector` 명령을 쓴다. `tools/collector`가 프로젝트 폴더로 이동해 `.venv` 파이썬으로 실행하며, `~/.local/bin/collector`에 링크돼 있다.

```bash
cd ~/Desktop/project && .venv/bin/python -m collector check     # 프로젝트 폴더에서
collector check                                                  # 어디서든(같은 동작)
collector report --hours 1
```

Claude Code 대화창에서는 앞에 `!`를 붙여 실행한다(예: `! collector check`).

## 연결 확인

**한 번에 점검**:

```bash
.venv/bin/python -m collector check            # 서비스·MQTT(10초 수신)·MySQL·InfluxDB
.venv/bin/python -m collector check --seconds 30 --skip-influx
```

```text
| 대상 | 결과 | 내용 |
| 수집 서비스 | ✅ | active, 재시작 1회, 메모리 24MB |
| MQTT 브로커 | ✅ | 접속 0.1초, 15초 동안 무시 토픽 3건, chirpstack 1건 |
| MySQL | ✅ | MySQL 8.0.41, 최근 5분 저장 352건, 마지막 저장 7초 전 |
| InfluxDB(과거 데이터) | ❌ | 접속 실패: URLError (사설망 — 유선 랜 경로 확인) |
```

- MQTT 점검은 **일회용 client_id**(`team2-check-xxxxxx`)로 접속한다. 운영 수집기 client_id를 쓰면 서로 밀어낸다.
- InfluxDB는 과거 데이터용이라 실패해도 종료 코드는 0이다. 수집 서비스·MQTT·MySQL 중 하나라도 실패하면 3이다.
- 비밀 값(주소·계정·토큰)은 출력하지 않는다.

**하나씩 볼 때**:

| 대상 | 명령 | 정상 |
|---|---|---|
| MQTT 로그인만 | `.venv/bin/python tools/mqtt_ping.py .env` | `MQTT: 접속 성공` |
| 수집기 MQTT 상태 | `journalctl --user -u wonil-collector -n 20 \| grep MQTT` | 마지막 줄이 `MQTT 접속, 구독 4개` |
| 수집기 최근 사건 | `report --hours 1`의 ④ 원인 단서, 또는 `SELECT kind, ts FROM collector_event ORDER BY id DESC LIMIT 10` | `mqtt_disconnected` 뒤에 `mqtt_connected` |
| MySQL 저장 | `SELECT MAX(received_at) FROM measurement` | 1분 이내 |
| InfluxDB 경로 | `ip route \| grep '^10\.'` | 유선 랜(`enP8p1s0`) 경로가 보여야 함. 없으면 사설망 InfluxDB에 닿지 않는다 |
| InfluxDB 응답 | `check` 결과의 InfluxDB 행 | `pass`와 최근 값 시각 |
| 과거 적재 진행 | `systemctl --user list-timers wonil-backfill.timer`, `journalctl --user -u wonil-backfill -n 5` | 완료되면 `var/backfill_done` 생성 |

## 서비스 설정

| 항목 | 값 |
|---|---|
| 서비스 | systemd **사용자** 서비스 `wonil-collector.service` (`~/.config/systemd/user/`) |
| 부팅 시 자동 시작 | `loginctl enable-linger` → `Linger=yes` (2026-10-20 설치 시 sudo 없이 적용됨) |
| 중복 실행 방지 | `flock var/collector.lock` |
| 재시작 | 비정상 종료 시 5초 뒤 자동(`Restart=always`) |
| 메모리 상한 | `MemoryMax=256M` |
| 로그 | `journalctl --user -u wonil-collector` |

## 설치·갱신

```bash
deploy/install.sh                    # 설정 검증 → 테이블 생성 → 서비스 등록·시작 → linger(안 되면 cron @reboot)
systemctl --user restart wonil-collector   # 코드를 바꾼 뒤
systemctl --user reload wonil-collector    # 설정(config/collector.yaml)만 바꾼 뒤 — 재시작 없이 반영
```

## 상태 확인

```bash
systemctl --user status wonil-collector
journalctl --user -u wonil-collector -n 50 --no-pager
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
| `reload` 뒤에 `stop`·`start` 사건(재시작됨) | (수정됨) 예전 서비스 파일은 HUP을 `flock`에 보내 프로세스가 죽었다 | 2026-10-21에 `ExecReload`를 자식 프로세스로 보내게 고쳤다. `deploy/install.sh`로 서비스 파일을 다시 설치 |
| `mqtt_flapping` 사건, 로그에 1초 간격 접속·끊김(`Unspecified error`) | 브로커에 **같은 client_id**를 쓰는 다른 클라이언트 | `config/collector.yaml`의 `mqtt.client_id`를 고유하게 바꾸고 재시작(2026-10-20 설치 직후 실제 발생, `team2-collector` → `team2-jetson-collector-a1`로 해결) |

## 확인 기준 검증 기록

| 확인 기준 | 방법 | 결과 | 측정 조건 |
|---|---|---|---|
| 브로커 재시작 후 1분 안 재수집 | `pytest -m broker tests/integration/test_broker_restart.py` (보드 안 amqtt 브로커를 10초 정지 후 재시작) | **6.0초** 만에 첫 저장 | 2026-10-20, Jetson, 시험 브로커, 재접속 상한 30초 |
| DB 장애 무손실 | `test_spool_lossless.py` (DB 실패 주입 중 메시지 600건) | 손실 0, `db_down`/`db_up` 기록 | 2026-10-20, 가짜 기록기 |
| 실 브로커 수집 | `pytest -m live` (3분) | 장치 18대·항목 30종, 미지정 공간 0, \|ts−수신\| 최대 69.7초 | 2026-10-20 16:29~16:32 KST, 실데이터 |
| 실제 장애 자동 복구 | 운영 중 사건 기록 | 2026-10-20 17:34(재부팅 6분 뒤) DB 기록 실패와 MQTT keepalive timeout이 함께 발생. 부팅 직후 네트워크(무선) 불안정으로 보인다. **DB 62초, MQTT 15초 만에 사람 손 없이 복구**, 스풀 덕분에 손실 0 | 실데이터, 운영 서비스 |
| 보드 재부팅 후 자동 시작 | 실제 재부팅 기록(`last -x`, journal, `collector_event`) | **통과**: 부팅 17:28:37 → 수집기 시작 17:28:52(15초) → MQTT 접속 17:28:59 → 첫 저장 17:29:04(**28초**). 첫 SSH 로그인 17:29보다 먼저 시작. 종료 때 스풀 잔여 7건을 기록하고 끝남. 데이터 공백 46초 | 2026-10-20 17:28 KST 사용자 재부팅, Linger=yes, 실데이터 |
| 24시간 커버율 95% | `report` | **중간 보고 16.3시간: 핵심 항목 공간별 최저 98.0%** ([보고](reports/2026-10-21-coverage-interim.md)). 24시간 최종본은 2026-10-21 16:33 이후 | 2026-10-20 16:40 ~ 10-21 09:00 KST, 실데이터 |
| 메모리 200MB 이하 | `systemctl --user show -p MemoryCurrent` | 20.0MB(10-20 16:34) → 21.8MB(16:41) → 25.5MB(10-21 09:04) → 24MB(10-21 오전 `check` 실행 시). 16시간 동안 늘지 않음 | 운영 서비스, `MemoryMax=256M` |

### 재부팅 시험 절차

1. 재부팅 전에 `uptime -s`와 `systemctl --user is-enabled wonil-collector`(enabled)를 확인한다.
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
