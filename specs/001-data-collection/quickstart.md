# Quickstart: 데이터 수집 체계 검증 절차

설정 형식은 [contracts/config.md](contracts/config.md), 명령은 [contracts/cli.md](contracts/cli.md),
테이블은 [data-model.md](data-model.md)를 본다.

## 1. 준비

```bash
~/.local/bin/uv pip install --python .venv/bin/python -e '.[test]'
.venv/bin/python -m collector check-config        # → OK
.venv/bin/python -m collector init-db             # 테이블 생성 + spaces 동기화
.venv/bin/python -m collector replay --file tests/fixtures/messages.jsonl --limit 20
```

기대 결과: `replay`에 실측 메시지가 `space_id`/`item`/`ts`로 풀려 나온다. `unmapped`가 없어야 한다.
게이트웨이 시계가 빠른 메시지도 `ts`가 수신 시각과 몇 초 안으로 맞아야 한다.

## 2. 자동 테스트 (확인 기준)

```bash
.venv/bin/python -m pytest -m unit                 # 오프라인
.venv/bin/python -m pytest -m broker               # 시험 브로커: 재시작 후 ≤60초 재수집, 스풀 무손실, 거부 행 격리
.venv/bin/python -m pytest -m mysql                # 실 DB(test_ 접두사): 멱등 기록, 현황 보고, 과거 적재
.venv/bin/python -m pytest -m live                 # 실 브로커 3분(LIVE_SECONDS)
```

## 3. 상시 운영 설치

```bash
deploy/install.sh        # 사용자 서비스 등록·시작, linger 시도, 실패 시 cron @reboot 등록
.venv/bin/python -m collector check   # 서비스·MQTT·MySQL·InfluxDB 연결 점검
deploy/install-backfill.sh            # (선택) InfluxDB 과거 데이터 적재 타이머 — 사설망 경로가 있어야 함
systemctl --user status team2-collector
```

## 4. 재부팅 시험 (수동, 기록 필수)

1. `sudo` 없이 가능한 방법으로 재부팅한다(관리자 요청 또는 전원 재투입).
2. 부팅 후 로그인하지 않고 3분을 기다린다.
3. 다른 PC나 다시 로그인한 뒤 `report --hours 1`을 실행하고, `collector_event`의 `start` 시각과 부팅 시각(`uptime -s`)을
   비교해 `docs/collector-ops.md`에 기록한다.

## 5. 24시간 수집 현황 보고

```bash
.venv/bin/python -m collector report --hours 24 --md docs/reports/$(date +%F)-coverage.md
```

기대 결과: 공간·항목별 커버율이 95% 이상이어야 한다. 빠진 구간마다 원인 단서가 붙어야 한다.
해석(원인·조치)은 보고 파일 아래에 직접 덧붙인다.

## 6. 형식 변경 대응 시연

`tests/fixtures/format_change/`의 바뀐 메시지(예: `co2` → `co2_ppm`)로 `replay`를 실행하면 `ignored`로 나온다.
`config/collector.yaml`의 `items.co2.aliases`에 `co2_ppm`을 추가한 뒤 다시 `replay`하면 `co2`로 나온다.
전후 결과를 `docs/format-change-case.md`에 기록한다.
