---
description: "과제 1 데이터 수집 체계 작업 목록"
---

# Tasks: 데이터 수집 체계 (과제 1)

**Input**: `specs/001-data-collection/`의 plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다. 헌법 IV에 따라 확인 기준은 pytest 자동 테스트로 검증해야 한다. 마커는 `unit`(오프라인),
`broker`(amqtt 시험 브로커), `mysql`(실 DB, `test_` 접두사), `live`(실 브로커)다.

**Organization**: 사용자 스토리(spec.md US1~US5)별로 묶는다. US1~US3은 P1, US4는 P2, US5는 P3(심화 B)다.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 병렬 가능(다른 파일, 미완료 작업에 의존 없음)
- 경로는 저장소 루트 기준(plan.md 구조: `src/collector/`, `config/`, `deploy/`, `tests/`, `docs/`)

---

## Phase 1: Setup (공통 기반)

- [X] T001 `pyproject.toml` 작성: 패키지 `collector`(src 레이아웃), 의존성 `paho-mqtt>=2.1`, `PyMySQL`, `PyYAML`, `python-dotenv`, extra `test` = `pytest`, `amqtt==0.12.*`; `[tool.pytest.ini_options]`에 마커 `unit`/`broker`/`mysql`/`live` 등록, 기본 `-m "unit or broker"`
- [X] T002 `~/.local/bin/uv pip install --python .venv/bin/python -e '.[test]'`로 설치 확인, `src/collector/__init__.py`(버전 `0.1.0`) 생성
- [X] T003 [P] `tests/conftest.py`: 고정물 `fixture_messages`(tests/fixtures/messages.jsonl 로드), `now_utc` 고정 시계, `mysql_conn`(`.env`를 python-dotenv로 로드, `test_` 접두사, 테스트 후 테이블 DROP — 접속 불가 시 skip)

---

## Phase 2: Foundational (모든 스토리의 전제)

**⚠️ 이 단계가 끝나야 스토리 작업을 시작한다**

- [X] T004 [P] `src/collector/pathexpr.py`: 경로 표현식 해석기 — `a.b.c`, `a[*].b`, `min(...)`/`first(...)`, `topic[n]`(음수 허용), `values.*`와 `$key`, `split(expr, "-", n)`; `compile(expr)` 시 구문 오류는 `PathExprError(위치, 이유)`; 값이 없으면 `None` (contracts/config.md "경로 표현식")
- [X] T005 [P] `tests/unit/test_pathexpr.py`: 위 문법 각각과 실측 메시지(ChirpStack `min(rxInfo[*].nsTime)`, xflow `split(values.*.$key,"-",0)`=`대표실`) 검증, 잘못된 식은 위치 포함 오류
- [X] T006 `src/collector/config.py`: YAML 로드 + 검증 → 불변 `Config` 데이터클래스(`spaces`, `devices`, `items`, `sources`, `ignore_topics`, `timing`, `mqtt`, `storage`, `spool`, `report`) 및 기본값(contracts/config.md 예시 값). 검증 규칙: `spaces[].id`는 `^[a-z][a-z0-9_]{1,31}$`·유일, 별칭 공간 간 중복 금지, `devices.*.space`는 존재하는 id, `sources[].name` 유일, `topic` 유효 MQTT 패턴, `device_id`·`measurements` 필수, 모든 경로식 사전 컴파일. 오류는 `ConfigError`로 `파일: 경로: 이유` 형식 목록. `unmapped` 공간(kind `unmapped`)을 자동 추가
- [X] T007 `src/collector/config.py`에 공간 결정 함수 `resolve_space(device_id, raw_location) -> (space_id, mapped_by)` — ① `devices.<id>.space`→`device` ② 이름·별칭 일치→`alias` ③ `unmapped`/`none` (data-model.md "space")
- [X] T008 [P] `config/collector.yaml`: contracts/config.md 예시 그대로(공간 8개, items, sources chirpstack/xflow, ignore_topics `gateway/+/state/#`)
- [X] T009 [P] `tests/unit/test_config.py`: 정상 설정 로드, 각 검증 규칙 위반 시 오류 메시지 위치, `resolve_space` 3가지 경로, `config/collector.yaml` 자체가 통과
- [X] T010 `src/collector/store.py`: `contracts/schema.sql`을 패키지 자원으로 읽어 `{p}` 치환 후 실행하는 `init_db(conn, prefix)`(멱등), `sync_spaces(conn, config)`(space 테이블 upsert), PyMySQL 연결 생성(`.env`의 `MYSQL_*`, `autocommit=False`, 세션 `time_zone='+00:00'`, connect/read/write timeout 10초). `contracts/schema.sql`을 `src/collector/schema.sql`로 복사해 사용
- [X] T011 `src/collector/__main__.py`: argparse CLI 골격 — 공통 `--config`(기본 `config/collector.yaml`), `--env`(기본 `.env`); 하위 명령 `check-config`(OK/오류, 종료 코드 2), `init-db`(`--prefix`), 이후 단계에서 `run`/`report`/`replay` 추가. 외부 접속 실패는 종료 코드 3 (contracts/cli.md)

**Checkpoint**: `collector check-config` → OK, `init-db` → 테이블 5개 생성

---

## Phase 3: User Story 1 — 센서 메시지를 공간·항목·시각별로 쌓는다 (P1) 🎯 MVP

**Goal**: 실 브로커의 모든 측정값이 공간·항목·기준 시각과 함께 MySQL에 저장된다.

**Independent Test**: 10분 가동 후 `measurement`에 실측 장치 전부의 항목이 들어 있고, `ts`가 수신 시각과 ±2분 안이며
`unmapped`가 없다.

### Tests

- [X] T012 [P] [US1] `tests/unit/test_parser.py`: 고정물로 검증 — ChirpStack 메시지가 `object`의 숫자 항목별 측정값으로 풀림, `ts`=가장 이른 `nsTime`(시계 빠른 게이트웨이 `…dccc` 메시지도 수신 시각 ±5초), `source_ts`=`time`, `frame_cnt`=`fCnt`; xflow `power:false`→`0.0`, `op_mode:"unknown"`→무시 건수; `gateway/…/state/conn`→무시; 깨진 JSON→failed; 기준 시각이 미래 2분 초과/과거 1시간 초과면 `received_at`으로 대체·`ts_corrected=1`; 공간 매핑 `lab`/`office`/`ceo_room`/`plant`
- [X] T013 [P] [US1] `tests/integration/test_store.py` (`mysql`): 같은 `(device_id, item, ts)` 두 번 기록 시 1행(멱등), `device.last_seen`은 더 큰 값만 반영, 값 `0`(깨진 값)도 저장

### Implementation

- [X] T014 [US1] `src/collector/parser.py`: `parse(topic, payload_bytes, received_at, config) -> ParseResult(measurements, device_infos, ignored, failed_reason)`. `Measurement` 필드는 data-model.md `measurement`와 동일(`ts`, `space_id`, `device_id`, `item`, `value`(유한수만, NaN·inf 거부), `unit`, `source`, `received_at`, `source_ts`, `ts_corrected`, `frame_cnt`). 시각 문자열(ISO, 나노초 포함)·epoch ms/s 자동 인식, 모두 UTC. 항목 이름은 `items.*.aliases`로 표준화(별칭에 없으면 원래 이름 그대로), 단위는 `items.*.unit`. bool→1/0, 문자열·사전·리스트 값은 무시 건수. `devices.<id>.enabled: false`면 무시
- [X] T015 [US1] `src/collector/spool.py`: SQLite(WAL) `outbox(seq INTEGER PRIMARY KEY, kind TEXT, payload TEXT)`; `put_many(kind, rows)`, `peek(limit) -> [(seq, kind, payload)]`, `ack(max_seq)`, `count()`, 한도 `spool.max_rows`(기본 400,000) 초과 시 가장 오래된 `measurement`부터 삭제하고 삭제 건수·시각 범위를 반환(호출자가 `spool_drop` 사건 기록). 스레드 안전(연결은 스레드별)
- [X] T016 [US1] `src/collector/store.py`에 `write_batch(conn, prefix, items)`: kind별로 `measurement`는 `INSERT … ON DUPLICATE KEY UPDATE id=id`, `device`는 upsert(`last_seen=GREATEST(...)`, `space_id`/`mapped_by`/`raw_location` 갱신), `event`/`stats` insert(`stats`는 PK 충돌 시 합산). 한 트랜잭션
- [X] T017 [US1] `src/collector/service.py`: `Collector` — paho `Client(CallbackAPIVersion.VERSION2, client_id=mqtt.client_id, clean_session=False)`, `.env`의 `MQTT_HOST/PORT/USER/PASS`, `on_connect`에서 모든 `sources[].topic`/`also_topics` + `ignore_topics`를 QoS `mqtt.qos`로 구독, `on_message`→`parse`→`spool.put_many`; 기록 스레드가 `storage.flush_interval_s`(2초)마다 `peek(batch_rows)`→`write_batch`→`ack`; 매분 `ingest_stats` 1행(메시지 0건이어도) 스풀에 추가; `start`/`stop` 사건; `max_lag_s` 집계
- [X] T018 [US1] `src/collector/__main__.py`에 `run`(Collector 실행, SIGTERM/SIGINT 시 스풀 비우기 최대 10초 후 종료 — FR-012)과 `init-db`가 `sync_spaces`까지 수행하도록 연결
- [X] T019 [US1] 실 브로커로 10분 가동 검증(`live`): `tests/integration/test_live_smoke.py` — `run`을 하위 프로세스로 3분 실행 후 `test_` 테이블에 `co2`·`temperature`가 있고 `unmapped` 0건, `ts`와 `received_at` 차이 120초 이하

**Checkpoint**: MVP — 수집기가 실데이터를 쌓는다.

---

## Phase 4: User Story 2 — 끊겨도 사람 손 없이 다시 수집한다 (P1)

**Goal**: 브로커 재시작·DB 장애·프로세스 종료·재부팅 후 자동 복구한다.

**Independent Test**: 시험 브로커 재시작 후 60초 안에 새 측정값 저장, DB 10분 차단 후 손실 0.

### Tests

- [X] T020 [P] [US2] `tests/integration/conftest.py`: amqtt 브로커 고정물(`127.0.0.1`, 빈 포트, 익명 허용, 별도 스레드 이벤트 루프, `stop()`/`start()`로 같은 포트 재시작) + 시험용 발행기(고정물 메시지를 토픽 그대로 재발행, `nsTime`/`ts`를 현재 시각으로 갱신)
- [X] T021 [P] [US2] `tests/integration/test_broker_restart.py` (`broker`): 수집기(저장소는 가짜 기록기 또는 `test_` DB) 가동 → 브로커 정지 10초 → 재시작 → 1초마다 발행 → **재시작부터 첫 저장까지 ≤ 60초** 단언, `mqtt_disconnected`·`mqtt_connected` 사건 존재 (SC-001)
- [X] T022 [P] [US2] `tests/integration/test_spool_lossless.py` (`broker`): 기록기를 실패하도록 주입(`DB down`) 상태에서 600건 발행 → 스풀 600건 → 기록기 복구 → 스풀 0, 저장 600건(손실 0), `db_down`/`db_up` 사건 (SC-008); 한도 초과 시 오래된 것부터 삭제·`spool_drop` 사건
- [X] T023 [P] [US2] `tests/integration/test_restart_resume.py` (`broker`): 스풀에 미기록 건이 남은 상태로 프로세스 강제 종료(SIGKILL) → 재시작 시 남은 건 기록

### Implementation

- [X] T024 [US2] `src/collector/service.py`: `reconnect_delay_set(1, mqtt.reconnect_max_s)`, `keepalive=mqtt.keepalive_s`, `connect_async`+`loop_start`로 첫 접속 실패도 같은 재시도 경로; `on_disconnect`→`mqtt_disconnected`(사유 코드만, 주소·계정 금지), `on_connect`→`mqtt_connected`; 감시 스레드: 접속 상태인데 `mqtt.stale_reconnect_s`(300초) 동안 메시지 0건이면 `reconnect()`와 `stale_reconnect` 사건
- [X] T025 [US2] `src/collector/service.py` 기록 스레드: `write_batch` 실패 시 연결 폐기, 지수 백오프(1→2→…→30초), 최초 실패 시 `db_down`, 성공 복귀 시 `db_up`(끊긴 시간·밀린 건수 포함); 스풀 한도 초과 삭제 시 `spool_drop` 사건
- [X] T026 [P] [US2] `deploy/wonil-collector.service`: `[Service] Type=simple`, `WorkingDirectory=%h/Desktop/project`, `ExecStart=/usr/bin/flock -n %h/Desktop/project/var/collector.lock %h/Desktop/project/.venv/bin/python -m collector run`, `ExecReload=/bin/kill -HUP $MAINPID`, `Restart=always`, `RestartSec=5`, `MemoryMax=256M`, `[Install] WantedBy=default.target`
- [X] T027 [P] [US2] `deploy/run_forever.sh`: cron 대체 감시 — `flock -n var/collector.lock`로 이미 돌면 종료, 아니면 무한 루프(수집기 실행, 종료 시 5초 후 재실행), 로그 `var/collector.log`(10MB 회전)
- [X] T028 [US2] `deploy/install.sh`: `mkdir -p var ~/.config/systemd/user` → 서비스 링크·`daemon-reload`·`enable --now`; `loginctl enable-linger "$USER"` 시도 후 `loginctl show-user -p Linger` 확인; 실패 시 `crontab`에 `@reboot …/deploy/run_forever.sh`를 중복 없이 추가하고 결과 출력 (research R5)
- [X] T029 [US2] 실제 설치 실행 및 linger/cron 중 무엇이 적용됐는지 확인, 재부팅 시험 절차·결과를 `docs/collector-ops.md`에 기록(부팅 시각 `uptime -s` vs 첫 `start` 사건 vs 첫 저장 시각) (SC-002)

**Checkpoint**: 상시 운영 가능.

---

## Phase 5: User Story 3 — 지금 잘 쌓이고 있는지 한눈에 본다 (P1)

**Goal**: 명령 하나로 공간·항목별 건수, 마지막 수신, 커버율, 빠진 구간과 원인 단서를 표·Markdown으로 낸다.

**Independent Test**: 결측을 넣은 합성 측정값으로 커버율·빠진 구간이 손 계산과 일치한다.

### Tests

- [X] T030 [P] [US3] `tests/unit/test_coverage.py`: 순수 함수 검증 — 60분 중 57칸 → 95.0%, 장치 2대 합산(어느 장치든 있으면 채움), 기대 주기 600초 장치의 기대 칸 수, 연속 빈 칸 `report.gap_min_minutes`(2분) 이상만 빠진 구간, 전 장치 동시 결측 판정, 사건과 겹침 판정, KST 표시
- [X] T031 [P] [US3] `tests/integration/test_report_db.py` (`mysql`): `test_` 테이블에 합성 측정값·사건 삽입 → `report` 결과 표 4개 값 확인, `--md` 파일 머리에 기간·실/합성·생성 시각

### Implementation

- [X] T032 [US3] `src/collector/report.py` 계산부(순수 함수): `coverage(minutes_by_series, start, end, period_s)`, `gaps(...)`, `cause_hints(gap, all_series_gaps, events, device_info)` — 단서 규칙: 전 장치 동시 결측 + `mqtt_disconnected`/`stop`/ingest_stats 없음 → "수집기·브로커·네트워크"; 해당 장치만 → "센서·배터리·전파"(배터리 항목 최근값 포함); 수집기 사건 없는 전체 결측 → "원천(게이트웨이·브로커) 측"
- [X] T033 [US3] `src/collector/report.py` 조회부: 기간 내 `SELECT DISTINCT space_id, item, device_id, FLOOR(UNIX_TIMESTAMP(ts)/60)`, `device`, `collector_event`, `ingest_stats` 조회(인덱스 `ix_ts` 범위), 끊김 장치(`last_seen`이 `report.stale_after_s` 600초 초과), 미매핑 장치·별칭에 없는 `raw_location` 목록
- [X] T034 [US3] `src/collector/report.py` 출력부: 표 ① 미매핑·끊김 장치 ② 공간·항목(장치 수, 건수, 마지막 수신 KST, 커버율) ③ 장치별 ④ 빠진 구간(장치, 공간, 시작, 끝, 길이, 원인 단서); 화면은 고정폭 표, `--md`는 Markdown + 머리에 측정 조건(기간 KST, `실데이터`, 생성 시각) (contracts/cli.md 예시 형식)
- [X] T035 [US3] `src/collector/__main__.py`에 `report` 연결: `--hours`(기본 24) 또는 `--from/--to`(KST), `--space`, `--by device|space`, `--md PATH`; 24시간 10초 이내(SC-005) 실측

**Checkpoint**: 과제 1 확인 기준 3개를 모두 검증할 수단이 갖춰진다.

---

## Phase 6: User Story 4 — 형식이 바뀌거나 센서가 늘어도 설정만 고친다 (P2)

**Goal**: 필드 이름 변경·새 원천 추가를 코드 수정 없이 설정으로 대응한다.

**Independent Test**: 바뀐 메시지가 설정 변경 전에는 무시되고, 변경 후 기존 항목으로 저장된다.

### Tests

- [X] T036 [P] [US4] `tests/fixtures/format_change/`: `co2_renamed.jsonl`(ChirpStack `object.co2`→`co2_ppm`), `new_source.jsonl`(가상 원천 `lab/v2/{장치}/telemetry`, 구조 `{"dev":..., "room":..., "at": epoch_s, "data": {"co2":...}}`), 각각 대응 설정 조각 `*.yaml`
- [X] T037 [P] [US4] `tests/unit/test_format_change.py`: 설정 변경 전후 `parse` 결과 비교(전: `co2_ppm` 비표준 항목 / 새 원천 무시, 후: `co2`로 저장 / 새 원천 수집), `src/` 아래 파일은 수정하지 않음을 전제로 함; 새 장치가 별칭으로 자동 매핑, 모르는 표기는 `unmapped`

### Implementation

- [X] T038 [US4] `src/collector/service.py`: SIGHUP 시 설정 다시 읽기 — 검증 성공 시 구독 목록 차이만큼 subscribe/unsubscribe 후 교체·`config_reload` 사건·`sync_spaces`, 실패 시 기존 유지·`config_rejected` 사건(오류 요약)
- [X] T039 [US4] `src/collector/__main__.py`에 `replay --file PATH [--limit N] [--config ...]`: JSONL(`topic`, `payload`, 선택 `t`)을 해석만 해서 측정값 표·무시/실패 요약 출력(저장 안 함)
- [X] T040 [US4] `docs/format-change-case.md`: 산출물 — 가정 상황 2개(필드 이름 변경, 새 원천 추가), 바꾼 설정 diff, `replay` 전후 출력, 코드 수정 0줄 확인(`git diff --stat src/` 빈 결과)

---

## Phase 7: User Story 5 — 재전송을 찾아내고 1년 뒤 저장소 크기를 안다 (P3, 심화 B)

**Goal**: 재전송을 한 번만 저장하고, 저장소 1년 크기와 보관 정책을 제시한다.

**Independent Test**: 같은 `(장치, fCnt)` 메시지를 `nsTime`만 바꿔 두 번 넣으면 저장 1회, `duplicate` 1.

- [X] T041 [P] [US5] `tests/unit/test_dedup.py`: 같은 장치·`fCnt`·다른 `deduplicationId`/`nsTime` → 두 번째는 duplicate, 2시간 창 밖이면 다시 허용, `fCnt` 리셋(작아짐) 시 허용
- [X] T042 [US5] `src/collector/service.py`(또는 `src/collector/dedup.py`): 최근 2시간 `(device_id, frame_cnt)` 창, 중복 시 저장 안 하고 `ingest_stats.duplicate` 증가 (research R8)
- [X] T043 [US5] `src/collector/report.py`: 장치별 중복 건수 표 추가, `report --capacity` — `information_schema.TABLES` 데이터·인덱스 크기, 최근 일 평균 행 수 → 1년 선형 추정, 정책(원본 90일 + 이후 5분 평균·최소·최대 `measurement_5m`) 적용 시 크기 (research R9)
- [X] T044 [P] [US5] `docs/storage-policy.md`: 1년 추정 결과와 보관·다운샘플링 정책, `measurement_5m` 생성·이관 SQL(실행은 하지 않음)

---

## Phase 8: Polish & 산출물

- [X] T045 [P] `docs/collector-ops.md`: 운영 가이드 — 설치(`deploy/install.sh`), 상태 확인(`systemctl --user status`, `journalctl --user -u wonil-collector`, `report --hours 1`), 장애 대응표(브로커·DB·디스크·설정 오류), 설정 반영(`reload`)
- [ ] T046 24시간 연속 수집 후 `report --hours 24 --md docs/reports/<날짜>-coverage.md` 생성, 아래에 결측 구간별 원인 해석과 조치 작성 (SC-003, SC-004, 헌법 VII)
- [X] T047 [P] 수집기 메모리 측정(`systemctl --user show -p MemoryCurrent`, 1시간 간격 3회) → 200MB 이하 확인, `docs/collector-ops.md`에 측정 조건과 함께 기록 (SC-007)
- [X] T048 [P] `.github/README.md`·`docs/README.md` 진행 표 갱신, `CLAUDE.md` 진행 상황 체크, 두 저장소 커밋·푸시(`git push`, `tools/docs-git push`)
- [X] T049 quickstart.md 절차 전체를 처음부터 따라 해 보고 어긋난 곳 수정

---

## Phase 9: User Story 6 — InfluxDB 과거 데이터 적재 (P1, 2026-10-20 추가)

**Goal**: InfluxDB `iot-bucket`(2025-08-08~, 약 1,470만 행)의 센서 항목을 수집기 시작 전까지 MySQL에 같은 형식으로 적재한다.

**Independent Test**: 하루치 적재 뒤 같은 날 InfluxDB와 MySQL의 장치·항목별 건수가 일치하고, 다시 실행해도 행이 늘지 않는다.

- [X] T050 [US6] `config/collector.yaml`·`src/collector/config.py`에 `backfill` 섹션(bucket, measurements, chunk_hours, progress_path) 추가, 공간 `storage`(창고) 추가, 시험 장치 `dev001` 수집 제외
- [X] T051 [P] [US6] `src/collector/influx.py`: 읽기 전용 Flux 조회 클라이언트(.env INFLUX_URL/TOKEN, 버킷 이름으로 orgID 찾기, annotated CSV 해석)
- [X] T052 [US6] `src/collector/backfill.py`: (location, point)→장치 추론표, 행 변환(공간 매핑·별칭·제외 장치·숫자 검사), 1일 구간 적재, 진행 파일로 재개, 끝 시각=수집기 첫 수신
- [X] T053 [US6] `src/collector/store.py`: `device_seen` 종류 — 있는 장치는 first_seen/last_seen만 넓히고 공간은 그대로
- [X] T054 [P] [US6] `tests/unit/test_backfill.py`(행 변환·장치 추론), `tests/integration/test_backfill_db.py`(mysql: 멱등·device_seen)
- [ ] T056 [US6] 2025-08-13~09-03 `device_id` 없는 행 75만 건 복원: InfluxDB에서 (location, point) 조합 조사 → `backfill.infer_table` 보강(항목으로 기종 좁히기 등) → 진행 파일에서 해당 날짜 지우고 재적재 → `docs/reports/*-backfill.md` 갱신
- [ ] T055 [US6] 전체 적재 실행(systemd-run 사용자 유닛, 로그 var/backfill.log), 결과를 `docs/reports/<날짜>-backfill.md`에 기록(건수 대조, 건너뛴 행과 사유)

---

## Dependencies & Execution Order

```text
Phase 1 Setup ─► Phase 2 Foundational ─► US1 (MVP) ─┬─► US2 복구 ─┐
                                                    ├─► US3 보고 ─┼─► Phase 8 (24h 보고는 US1~3 가동 후)
                                                    └─► US4 설정 ─┘
                                                         US5(심화)는 US1·US3 뒤 언제든
```

- US2·US3·US4는 US1(parser·spool·store·service 기본)만 있으면 서로 독립이다. 단 같은 파일 `service.py`를 고치는 T024·T025·T038·T042는 순서대로 한다.
- T046(24시간 보고)은 수집기를 상시 설치(T028)한 뒤 24시간이 지나야 할 수 있다. **US1 완료 즉시 설치해 수집을 시작**하는 것이 일정상 가장 중요하다.

## Parallel Examples

- Phase 2: T004+T005(경로식), T008(설정 파일), T009(설정 시험)는 동시에 진행할 수 있다.
- US1: T012(파서 시험)와 T013(저장 시험)은 구현 전에 동시에 작성한다.
- US2: T020~T023 시험과 T026·T027 배포 파일은 동시에 진행한다.
- US3: T030(계산 시험)과 T032(계산부)를 먼저 하고, 조회·출력은 그다음이다.

## Implementation Strategy

1. **MVP = Phase 1~3(US1)**: 끝나는 즉시 임시로라도 서비스 설치(T026·T028 선행 가능) → 데이터 축적 시작.
2. US2 → 확인 기준 1·2 자동 테스트 통과, 재부팅 시험 기록.
3. US3 → 24시간 뒤 확인 기준 3 보고서.
4. US4 → 형식 변경 산출물.
5. US5(심화 B)는 시간 여유가 있을 때 한다.
