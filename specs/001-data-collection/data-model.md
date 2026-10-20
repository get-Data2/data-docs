# Data Model: 데이터 수집 체계 (과제 1)

저장소: MySQL 8.0 `cf_u36_d3`. 모든 시각은 UTC `DATETIME(3)`이며 세션 `time_zone='+00:00'`이다.
테이블 이름 접두사는 설정(`storage.table_prefix`, 기본 없음)으로 바꿀 수 있고, 시험은 `test_` 접두사를 쓴다.

## measurement — 측정값 (장치 × 항목 × 기준 시각)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | BIGINT UNSIGNED PK AUTO_INCREMENT | |
| ts | DATETIME(3) NOT NULL | 기준 시각(R1) |
| space_id | VARCHAR(32) NOT NULL | 공간 id(`space.space_id`). 수신 시점의 매핑을 그대로 기록. 매핑 실패 시 `unmapped` |
| device_id | VARCHAR(64) NOT NULL | devEui 또는 xflow 장치 id |
| item | VARCHAR(64) NOT NULL | 표준 항목 이름(설정의 대응표 적용 후, 예: `co2`) |
| value | DOUBLE NOT NULL | 숫자 값. 참거짓은 1/0 |
| unit | VARCHAR(16) NULL | 설정의 단위 |
| source | VARCHAR(32) NOT NULL | 원천 규칙 이름(`chirpstack`, `xflow` 등) |
| received_at | DATETIME(3) NOT NULL | 수집기 수신 시각 |
| source_ts | DATETIME(3) NULL | 원천이 찍은 시각(ChirpStack `time`, xflow `ts`) |
| ts_corrected | TINYINT(1) NOT NULL DEFAULT 0 | 허용 범위 밖이라 수신 시각으로 대체했으면 1 |
| frame_cnt | INT UNSIGNED NULL | LoRaWAN fCnt |

- 유일키 `uq_meas (device_id, item, ts)`: 재처리 멱등 보장(FR-007).
- 인덱스 `ix_space_item_ts (space_id, item, ts)`: 학습·분석 범위 조회용.
- 인덱스 `ix_ts (ts)`: 현황 보고용.
- 검증: `value`는 유한수(NaN·inf 거부). 깨진 값(0ppm)도 그대로 저장(FR-006).

## space — 공간 (설정 `spaces`를 시작 시 동기화)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| space_id | VARCHAR(32) PK | 영문 고정 id(`lab`, `office` …). 이후 과제의 MQTT 토픽·API 경로에 그대로 쓴다 |
| name | VARCHAR(64) NOT NULL | 한글 이름(`실습실`) |
| kind | VARCHAR(16) NOT NULL | `classroom` / `office` / `meeting` / `outdoor` / `plant` / `unmapped` |
| area_m2 | DOUBLE NULL | 면적(알면). 과제 5 "공간 크기" 분석용 |
| note | VARCHAR(256) NULL | |

- 공간 결정 순서(FR-020): ① 설정 `devices.<id>.space`(장치 지정) → ② 원천 규칙이 뽑은 공간 표기를 `spaces[].aliases`에서 찾기
  → ③ `unmapped`. 미매핑 장치는 현황 보고 맨 위에 따로 표시한다.
- 측정값은 **수신 시점의 space_id를 복사해 저장**한다. 나중에 장치를 다른 방으로 옮겨 매핑을 바꿔도 과거 데이터의 공간은
  바뀌지 않는다.
- 같은 이름이 원천마다 다르게 나오는 경우(LoRaWAN `사무실` 태그와 xflow `사무실` 실내기)도 별칭으로 한 공간에 모은다.

## device — 장치 목록 (수신 시 자동 등록·갱신)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| device_id | VARCHAR(64) PK | |
| source | VARCHAR(32) NOT NULL | |
| name | VARCHAR(128) NULL | deviceName 또는 xflow 장치명 |
| model | VARCHAR(64) NULL | deviceProfileName 또는 `idu`/`odu`/`controller` |
| space_id | VARCHAR(32) NOT NULL | 현재 공간 id |
| spot | VARCHAR(128) NULL | 공간 안 위치 설명(태그 `spot`) |
| raw_location | VARCHAR(128) NULL | 원천이 준 공간 표기 그대로(태그 `location`, 장치명 앞부분). 매핑 점검용 |
| mapped_by | VARCHAR(16) NOT NULL | `device`(장치 지정) / `alias`(표기 대응) / `none`(미매핑) |
| expected_period_s | INT NOT NULL DEFAULT 60 | 기대 보고 주기(FR-018, 설정에서 덮어씀) |
| first_seen | DATETIME(3) NOT NULL | |
| last_seen | DATETIME(3) NOT NULL | 마지막 수신 시각 |

갱신은 기록 스레드가 일괄로 `INSERT … ON DUPLICATE KEY UPDATE`한다. `last_seen`은 큰 값만 반영한다.

## collector_event — 수집기 사건 (FR-011)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | BIGINT UNSIGNED PK AUTO_INCREMENT | |
| ts | DATETIME(3) NOT NULL | |
| kind | VARCHAR(32) NOT NULL | `start`, `stop`, `mqtt_connected`, `mqtt_disconnected`, `db_down`, `db_up`, `spool_drop`, `config_reload`, `config_rejected`, `stale_reconnect` |
| detail | VARCHAR(512) NULL | 사유·건수 등(비밀 값 금지) |
| host | VARCHAR(64) NOT NULL | 보드 이름 |

인덱스 `ix_event_ts (ts)`(Crowfoot은 문서 안 인덱스 이름이 유일해야 함). 사건도 스풀을 거쳐 기록되므로, DB가 끊긴 동안의 사건도 나중에 남는다.

## ingest_stats — 분 단위 수신 통계 (수집기 생존 신호 겸용)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| minute | DATETIME NOT NULL | 수신 시각 기준 분 |
| host | VARCHAR(64) NOT NULL | |
| messages | INT NOT NULL | 받은 메시지 수 |
| stored | INT NOT NULL | 측정값으로 풀린 건수 |
| ignored | INT NOT NULL | 무시 토픽·비숫자 항목 수 |
| failed | INT NOT NULL | 해석 실패 메시지 수 |
| duplicate | INT NOT NULL | 재전송으로 버린 메시지 수 |
| max_lag_s | DOUBLE NULL | 수신 시각 − 기준 시각의 최대값 |

PK `(minute, host)`. 매분 한 줄을 쓴다(메시지가 0건이어도 쓴다). 이 줄이 없는 분은 **수집기가 돌지 않은 분**이다.

## 로컬 스풀 (SQLite, `var/spool.db`)

| 테이블 | 내용 |
|---|---|
| `outbox(seq INTEGER PK, kind TEXT, payload TEXT)` | `kind` ∈ {`measurement`, `device`, `event`, `stats`}, payload는 JSON |

- 기록 스레드는 `seq` 순서로 읽어 MySQL에 반영하고 성공한 `seq`까지 지운다.
- 한도: `spool.max_rows`(기본 400,000 ≈ 24시간 + 여유). 넘으면 오래된 `measurement`부터 지우고 `spool_drop` 사건을 남긴다.

## 상태 전이: 수집기 연결 상태

```text
STARTING → CONNECTING ⇄ CONNECTED
              ↑             │ (끊김 / 5분 무수신)
              └─────────────┘
DB: OK ⇄ DOWN (DOWN 동안 스풀만 증가, OK 복귀 시 소진)
```
