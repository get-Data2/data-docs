# 과제 1. 데이터 수집 체계 — 점검 문서

> 상태: 🟢 **완료(운영 중)**. 24시간 최종 현황표 통과(2026-10-21, 핵심 항목 최저 98.6%). 남은 것은 InfluxDB 미적재분(보드 유선 링크 복구 대기)뿐이다.
> 측정 조건: Jetson(L4T R39, aarch64), 실데이터, 수집 2026-10-20 16:33 KST~, 보드 전력 모드 기본값(측정에 영향 없음).
> 계획: [spec](../../specs/001-data-collection/spec.md) · [plan](../../specs/001-data-collection/plan.md) · [tasks](../../specs/001-data-collection/tasks.md) ·
> 코드: [get-Data2/data](https://github.com/get-Data2/data) `src/collector/`
>
> 이 문서는 README 항목(해결할 것·확인 기준·생각해 볼 점·심화)별 대응과 확인 명령을 모은 **점검 문서**다. 산출물 본문은 [산출물 폴더](../deliverables/README.md)에 있다.
> 산출물: [① 수집 현황 보고](../deliverables/task1/01-coverage-report-interim.md) · [② 형식 변경 대응 기록](../deliverables/task1/02-format-change.md)

## 요약

MQTT 두 원천(ChirpStack LoRaWAN 센서, xflow 냉난방)을 구독해 **공간 × 장치 × 항목 × 기준 시각**으로 MySQL에 쌓는다.
상시 서비스로 운영 중이다.
- 메시지 형식과 공간은 **설정 파일만** 고쳐 대응한다.
- 브로커 재시작, DB 장애, 재부팅 뒤에도 **사람 손 없이** 다시 수집한다.
- 명령 하나로 공간·항목별 커버율과 빠진 구간의 원인 단서를 본다.
- 과거 데이터는 InfluxDB에서 978만 행을 같은 형식으로 적재했다.

가장 의외였던 발견은 세 가지다.
1. 게이트웨이 한 대의 **시계가 4분 55초 빠르다**. 원천 시각을 그대로 썼다면 LoRa 행의 **40%** 시각이 틀렸을 것이다.
2. ChirpStack이 **같은 업링크를 두 번 발행**한다(메시지의 3.4%).
3. 센서의 보고 주기가 **기종이 아니라 장치마다 다르다**(1·2·10분).

---

## 해결할 것

| # | README 요구 | 해결 방법 | 근거·위치 |
|---|---|---|---|
| 1 | MQTT 센서 메시지를 **공간·항목·시각별로 저장**하는 수집기를 상시 가동 | `wonil-collector`(systemd 사용자 서비스)가 구독 → 해석 → 보드 내 스풀 → MySQL `measurement` 일괄 기록. 기준 시각은 네트워크 서버 시각(`nsTime`). 원천 시각과 수신 시각도 함께 저장 | [data-model](../../specs/001-data-collection/data-model.md), [research R1·R2](../../specs/001-data-collection/research.md) |
| 2 | 형식 변경·새 센서에 **설정만 바꿔** 대응 | `config/collector.yaml`의 원천 규칙(경로 표현식), 공간 매핑(장치 지정 → 별칭 → unmapped), 항목 별칭, 장치별 주기. 반영은 `reload`(재시작 없음) | [형식 변경 대응 기록](../deliverables/task1/02-format-change.md), [설정 형식](../../specs/001-data-collection/contracts/config.md) |
| 3 | 브로커 재시작·네트워크 끊김·재부팅 뒤 **사람 손 없이 재수집** | 재접속(최대 30초 간격), 구독 확인 뒤 접속 판정, 5분 무수신 시 강제 재접속, client_id 충돌 감지, DB 장애는 스풀이 흡수(최대 40만 행), 거부 행 격리, `Restart=always`, `Linger=yes` | [운영 가이드](../ops/collector-ops.md) |
| 4 | 공간·항목별 **건수, 마지막 수신, 커버율**을 한눈에 | `collector report`: ① 확인 필요 장치 ② 공간·항목별 ③ 장치별(관측 주기 포함) ④ 빠진 구간과 원인 단서. 연결 점검은 `collector check` | [중간 현황 보고](../deliverables/task1/01-coverage-report-interim.md) |

### 저장 정확성 검증 (해결할 것 1의 근거)

- 방법: `tools/verify_ingest.py`. 수집기 코드를 쓰지 않고 원본 MQTT 메시지에서 기대 행을 직접 계산해 MySQL과 양방향으로 대조한다.
- 결과
  - 3분 표본: 메시지 52건 → 기대 행 **212/212 일치**. 그 시간대 DB 행 중 원본에 없는 행 0개.
  - 90초 표본: **94/94 일치**.
- 공간 8개 매핑, 미지정 0건이다. 기준 시각은 `nsTime`과 밀리초까지 일치했고, 이중 발행은 1행만 저장됐다.
- 장비 현황(MQTT 실시간 22대 정상, 과거 데이터에만 있는 3대 중 확실한 수신 중단 1대·판단 보류 2대, 배터리 1% 교체 시급 1대)과 공간별 장비 목록은 [장비 현황](../reports/2026-10-21-devices.md)에 있다.

## 확인 기준

| 확인 기준 | 결과 | 판정 | 측정 조건 |
|---|---|---|---|
| 브로커를 껐다 켜도 **1분 안에** 수집 재개 | 시험 브로커를 10초 껐다 켠 뒤 **6.0초** 만에 첫 저장(자동 시험). 운영 중 실제 MQTT 끊김(keepalive timeout)도 **15초** 만에 재접속 | ✅ | 시험: 보드 내 amqtt, `pytest -m broker`. 운영: 2026-10-20 17:35 실데이터 |
| 보드를 재부팅해도 **자동으로** 다시 시작 | 부팅 17:28:37 → 수집기 시작 +15초 → 첫 저장 **+28초**. 사용자 로그인(17:29)보다 먼저. 데이터 공백 46초 | ✅ | 2026-10-20 실제 재부팅, `Linger=yes` |
| 24시간 커버율 **95% 이상**, 빠진 구간은 원인 설명 | **24시간 최종: 핵심 항목(co2·온도·습도) 공간별 최저 98.6%, 95% 미만 0개**. 빠진 구간은 모두 원인 단서 있음. 집중회의실 실내기 온도(0.1%)는 끊김이 아니라 켜졌을 때만 보내는 조건부 항목이라 판정 제외(`periodic_items`, 24시간 중 1번 켜짐) | ✅ | 2026-10-20 16:40 ~ 10-21 16:40 KST(24시간), 실데이터, 커버율 = 기대 주기 칸 기준 |

## 산출물

| README 산출물 | 위치 | 상태 |
|---|---|---|
| 수집 현황 보고(공간·항목별 커버율 표, 결측 구간과 추정 원인) | [deliverables/task1/01-coverage-report.md](../deliverables/task1/01-coverage-report.md)(24시간 최종), 해석이 있는 [중간본](../deliverables/task1/01-coverage-report-interim.md) | ✅ 24시간 최저 98.6% |
| 메시지 형식 변경 상황 가정과 대응 기록 | [deliverables/task1/02-format-change.md](../deliverables/task1/02-format-change.md) | ✅ 가정 2개(필드 이름 변경, 새 원천), 설정만 수정·코드 0줄 |

산출물이 아닌 부속 기록: [운영 가이드](../ops/collector-ops.md), [장비 현황](../reports/2026-10-21-devices.md), [InfluxDB 적재 보고](../reports/2026-10-21-backfill.md), 심화 결과 [보관 정책](../advanced/task1/01-storage-policy.md)

## 생각해 볼 점

**Q1. 시각은 센서가 찍은 것과 수신한 것 중 무엇을 저장해야 하나? 네트워크가 30초 밀렸다가 한꺼번에 도착하면?**

- **둘 다 저장하고, 기준 시각은 "네트워크 서버가 받은 시각(`nsTime`)"으로 정했다.**
- 실측 근거
  - ChirpStack의 원천 시각(`time`)은 게이트웨이 시계를 그대로 쓴다. 게이트웨이 `…dccc`의 시계는 4분 55초 빠르다.
  - 운영 16.7시간 동안 LoRa 34,351행 중 **13,850행(40%)**의 원천 시각이 수신보다 2분 넘게 미래였다.
  - `nsTime` 기준으로는 수신과의 차이가 평균 0.25초, 최대 0.9초다.
- 수신 시각만 쓰면 30초 밀렸다가 한꺼번에 도착할 때 측정 간격이 뭉개진다. `nsTime`은 네트워크 서버가 무선 수신 순간에 찍으므로 순서와 간격이 보존된다.
- 수신 시각(`received_at`)은 지연 분석용으로 따로 남긴다. 원천 시각(`source_ts`)은 시계 오차 진단용으로 남긴다.
- 기준 시각이 수신보다 2분 넘게 미래이거나 1시간 넘게 과거이면 수신 시각으로 바꾸고 `ts_corrected=1`로 표시한다. 지금까지 0건이었다.
- 냉난방(xflow)의 `ts`는 수신보다 평균 29초, 최대 70초 이르다. 장비가 상태를 갱신한 시각으로 보이며 허용 범위 안이다.

**Q2. 수집과 학습·추론이 같은 저장소를 동시에 쓰면 어떤 문제가 생기나?**

- 문제
  - 큰 범위를 읽는 분석 쿼리가 디스크·버퍼 풀을 차지해 수집 쓰기가 느려지거나 시간 초과가 날 수 있다.
  - 읽는 도중 새 행이 들어와 같은 쿼리를 두 번 돌리면 결과가 달라질 수 있다(재현성 문제).
  - 분석이 테이블을 잠그는 작업(대량 수정, 인덱스 추가)을 하면 수집이 멈춘다.
- 대응
  - 수집기는 **보드 내 스풀**을 거쳐 2초마다 일괄로 쓴다. DB가 느리거나 끊겨도 받은 값을 잃지 않는다. 실제로 62초 DB 장애 때 손실 0이었다.
  - 측정값 테이블에는 범위 조회용 인덱스(`공간·항목·시각`, `시각`)를 두어 분석 쿼리가 필요한 범위만 읽게 했다.
  - 과제 2부터 분석은 기간을 정해 **한 번 읽어 보드의 파일 사본으로** 반복한다(재현성 + 수집 보호, 과제 2 spec 가정).
  - 이상적으로는 읽기 전용 복제본을 쓰는 것이다. 지금 권한(팀 DB 하나)으로는 할 수 없어 위 방식으로 대신한다.

## 심화(B)

| 심화 항목 | 결과 | 문서 |
|---|---|---|
| 1년 뒤 저장소 크기 추정과 다운샘플링·보관 정책 | 지금 속도(하루 약 10만 행, 행당 241바이트)로 **1년 8.2GB**. 원본 90일 + 이후 5분 집계면 **4.0GB**, 여기에 냉난방 상태값을 "바뀔 때만 + 15분 생존 기록"으로 저장하면 **약 2.9GB**. 저장량의 절반은 냉난방 반복값이고(야간 100% 반복), 환경 센서 원본은 값 고정 고장 탐지(과제 4)의 근거라 90일은 지킨다 | [storage-policy.md](../advanced/task1/01-storage-policy.md) |
| 같은 메시지가 두 번 오는 경우(재전송) 찾아 처리 | 실측: ChirpStack이 게이트웨이 조합만 다른 같은 업링크(같은 fCnt)를 두 번 발행. `deduplicationId`와 `nsTime`(1ms 차이)이 달라 DB 유일키로는 못 막는다. (장치, fCnt)를 2시간 동안 기억해 한 번만 저장한다. 16.7시간 동안 **600건(메시지의 3.4%)** 걸러냄(`ingest_stats.duplicate`). 재처리·재접속 중복은 유일키(장치·항목·기준 시각)로 막는다 | [advanced/task1/02-duplicate-messages.md](../advanced/task1/02-duplicate-messages.md) |

## 확인 명령

모든 명령은 Jetson 터미널에서 입력한다. `collector`는 어느 폴더에서든 동작하고, `.venv/bin/python …`과 `deploy/…`는
`cd ~/Desktop/project` 뒤에 입력한다. Claude Code 대화창에서는 앞에 `!`를 붙인다.

| 확인할 것 | 명령 | 정상 결과 |
|---|---|---|
| 연결 전체(서비스·MQTT·MySQL·InfluxDB) | `collector check` | 수집 서비스·MQTT·MySQL ✅ (InfluxDB는 유선망 있을 때만 ✅) |
| 수집기가 상시 가동 중 | `systemctl --user status wonil-collector` | `active (running)` |
| 수집기 로그 | `journalctl --user -u wonil-collector -n 30 --no-pager` | 마지막 줄 근처에 `MQTT 접속, 구독 4개` |
| **해결 1** 공간·항목·시각별 저장이 정확한가 | `.venv/bin/python tools/verify_ingest.py 120` | `기대 행 N개: {'일치': N}`, `원본에 없는 행 0개`, 종료 코드 0 |
| 실제 저장된 값 보기 | 아래 SQL ① | 공간·항목별 최근 값 |
| **해결 2** 설정 검증 | `collector check-config` | `OK: 공간 9개, 원천 2개, 구독 토픽 4개` |
| 설정만으로 형식 변경 대응 | `.venv/bin/python -m pytest tests/unit/test_format_change.py` | `3 passed` |
| 저장 없이 해석해 보기 | `collector replay --file tests/fixtures/messages.jsonl --limit 10` | 공간·항목·시각 표, `실패 0` |
| 설정 반영(재시작 없이) | `systemctl --user reload wonil-collector` 후 `journalctl --user -u wonil-collector -n 2` | `설정 다시 읽음` |
| **해결 3** 브로커 재시작 후 1분 안 재수집 | `.venv/bin/python -m pytest -m broker tests/integration/test_broker_restart.py -s` | `브로커 재시작 → 첫 저장 N초`(N ≤ 60), `1 passed` |
| DB 장애 무손실·거부 행 격리 | `.venv/bin/python -m pytest -m broker tests/integration/test_spool_lossless.py` | `4 passed` |
| 재부팅 자동 시작 설정 | `loginctl show-user $USER -p Linger` / `systemctl --user is-enabled wonil-collector` | `Linger=yes` / `enabled` |
| 재부팅 뒤 첫 저장 시각 | 아래 SQL ② | 부팅 시각(`uptime -s`) + 1분 안 |
| 실제 끊김·복구 기록 | 아래 SQL ③ | `mqtt_disconnected` 뒤 `mqtt_connected`, `db_down` 뒤 `db_up` |
| **해결 4** 현황 보고(커버율·빠진 구간·원인) | `collector report --hours 24` | 표 ①~④, 핵심 항목 커버율 95% 이상 |
| 보고서를 파일로 | `collector report --hours 24 --md docs/deliverables/task1/01-coverage-report.md` | Markdown 파일 생성 |
| **심화** 저장소 1년 추정 | `collector report --capacity` | 1년 크기 표(원본·집계 정책) |
| **심화** 재전송(이중 발행) 거르기 | `.venv/bin/python -m pytest tests/unit/test_dedup.py` 그리고 아래 SQL ④ | `3 passed`, 중복 버림 건수 |
| 과거 데이터 적재 진행 | `systemctl --user list-timers wonil-backfill.timer` / `journalctl --user -u wonil-backfill -n 5` | 다음 실행 시각 / 구간별 적재 행 수 |
| 전체 자동 시험 | `.venv/bin/python -m pytest -m "unit or broker or mysql"` | 모두 passed |

```sql
-- ① 공간·항목별 최근 값(KST)
SELECT space_id, item, value, unit, CONVERT_TZ(ts,'+00:00','+09:00') AS ts_kst
FROM measurement WHERE ts > UTC_TIMESTAMP() - INTERVAL 10 MINUTE AND item IN ('co2','temperature','humidity')
ORDER BY space_id, item, ts DESC;

-- ② 재부팅 뒤 첫 저장(부팅 시각을 KST로 넣는다)
SELECT CONVERT_TZ(MIN(received_at),'+00:00','+09:00') FROM measurement
WHERE received_at > CONVERT_TZ('2026-10-20 17:28:37','+09:00','+00:00') AND source<>'influx';

-- ③ 수집기 사건(최근 20건)
SELECT CONVERT_TZ(ts,'+00:00','+09:00') AS ts_kst, kind, detail FROM collector_event ORDER BY id DESC LIMIT 20;

-- ④ 분 단위 수신 통계 합계(재전송으로 버린 메시지 = duplicate)
SELECT SUM(messages), SUM(`stored`), SUM(duplicate), SUM(failed) FROM ingest_stats;
```

SQL은 MySQL 클라이언트나 Crowfoot(워크스페이스 "데이터 수집" → 연결 MySQL #3)에서 실행한다.

## 추가로 한 것(README 범위 밖)

- **InfluxDB 과거 데이터 적재**: 2025-08-08 ~ 2026-07-22, **978만 행**을 같은 공간 매핑으로 적재했다. 하루치 대조 결과는 37,401 = 37,401이다.
  유선망이 끊겨 나머지 90일은 대기 중이고, 타이머가 10분마다 재시도한다. 장치를 특정하지 못한 75만 행은 복원할 계획이다. [적재 보고](../reports/2026-10-21-backfill.md)
- **Crowfoot ERD**: "데이터 수집" 워크스페이스, 문서 684에 요구사항 6개를 넣었다(모두 반영 상태).
- **연결 점검 명령** `collector check`: 서비스·MQTT·MySQL·InfluxDB를 한 번에 확인한다.

## 한계와 다음 행동

| 한계 | 영향 | 다음 행동 |
|---|---|---|
| 실 브로커는 우리가 끌 수 없어 "브로커 재시작"은 보드 내 시험 브로커로 검증 | 실 브로커 특유의 동작(세션 유지 등)은 미검증 | 운영 중 실제 끊김 기록으로 보완(15초 재접속 1건 확보). 계속 사건을 모은다 |
| InfluxDB가 유선망(현재 `NO-CARRIER`)에서만 닿음 | 2026-07-23~10-20 과거 데이터 미적재 | 랜 케이블 확인. 연결되면 자동 적재 |
| 냉난방 데이터는 2026-10-20부터만 있음 | 과제 3~5에서 냉난방 상태를 입력으로 쓰기 어려움 | 수집 기간이 쌓이면 재평가 |
| 팀 DB 용량 한도를 모름 | 1년 8.2GB가 한도를 넘을 수 있음 | 한도 확인 후 보관 정책 B·C 적용 여부 결정 |
