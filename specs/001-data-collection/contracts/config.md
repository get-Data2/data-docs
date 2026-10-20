# Contract: 수집기 설정 파일 `config/collector.yaml`

메시지 형식 변경·센서 추가·공간 변경은 **이 파일만** 고쳐 대응한다(FR-019~022). 접속 정보는 넣지 않는다(`.env`).
반영: 수집기 재시작 또는 `systemctl --user reload team2-collector`(SIGHUP). 검증에 실패하면 시작을 거부하고,
가동 중 다시 읽기라면 기존 설정을 유지하며 `config_rejected` 사건을 남긴다.

## 최상위 키

| 키 | 필수 | 설명 |
|---|---|---|
| `version` | ✔ | 설정 형식 버전(현재 `1`) |
| `spaces` | ✔ | 공간 목록(공간 매핑의 기준) |
| `devices` | | 장치별 덮어쓰기(공간 지정, 기대 주기, 수집 제외) |
| `items` | | 표준 항목 이름 → 단위, 원천 필드 별칭 |
| `sources` | ✔ | 원천 규칙 목록 |
| `ignore_topics` | | 측정값이 아닌 토픽(건수만 집계) |
| `timing` | | 시각 허용 범위 |
| `mqtt` / `storage` / `spool` / `report` | | 동작 파라미터(기본값 있음) |

## 경로 표현식 (값 위치 지정)

| 문법 | 뜻 | 예 |
|---|---|---|
| `a.b.c` | 중첩 키 | `deviceInfo.tags.location` |
| `a[*].b` | 배열 모든 원소의 `b` | `rxInfo[*].nsTime` |
| `min(expr)` / `first(expr)` | 여러 값 중 최소 / 첫 값 | `min(rxInfo[*].nsTime)` |
| `topic[n]` | `/`로 나눈 토픽의 n번째 조각 | `topic[3]` → devEui |
| `values.*` | 사전의 유일한 키 하나로 내려감. 그 키는 `$key`로 참조 | xflow `values.{장치명}` |
| `split($key, "-", 0)` | 문자열 나누기 | `대표실-실내기` → `대표실` |

## 공간 결정 순서

1. `devices.<device_id>.space`가 있으면 그것을 쓴다(`mapped_by=device`).
2. 원천 규칙의 `space` 표현식 결과가 어느 공간의 `name` 또는 `aliases`와 같으면 그 공간을 쓴다(`mapped_by=alias`).
3. 둘 다 아니면 `unmapped`(`mapped_by=none`)로 저장하고, 현황 보고 맨 위에 표시한다.

## 예시 (실측 기준 초기 설정)

```yaml
version: 1

spaces:                      # 공간 매핑: id는 이후 과제의 토픽·API 경로에 쓰이므로 바꾸지 않는다
  - {id: lab,           name: 실습실,     kind: classroom, aliases: [실습실]}
  - {id: office,        name: 사무실,     kind: office,    aliases: [사무실]}
  - {id: office_out,    name: 사무실 밖,  kind: outdoor,   aliases: [사무실 밖]}
  - {id: ceo_room,      name: 대표실,     kind: office,    aliases: [대표실]}
  - {id: dev_room,      name: 개발실,     kind: office,    aliases: [개발실]}
  - {id: meeting,       name: 회의실,     kind: meeting,   aliases: [회의실]}
  - {id: focus_meeting, name: 집중회의실, kind: meeting,   aliases: [집중회의실]}
  - {id: plant,         name: 공용 설비,  kind: plant,     aliases: [controller]}

devices:                     # 선택: 태그가 틀리거나 비어 있을 때 장치 단위로 고정
  24e124785c389818: {space: lab}          # 태그에 spot이 없음 — location만으로도 되지만 명시
  # 24e124xxxxxxxxxx: {expected_period_s: 600}   # 10분 주기 센서
  # 24e124yyyyyyyyyy: {enabled: false}           # 수집 제외

items:                       # 표준 항목 이름: 단위 + 원천 필드 별칭
  co2:          {unit: ppm, aliases: [co2]}
  temperature:  {unit: degC, aliases: [temperature, current_temperature]}
  humidity:     {unit: "%", aliases: [humidity, current_humidity]}
  pressure:     {unit: hPa}
  tvoc:         {unit: ppb}
  battery:      {unit: "%"}
  power:        {unit: bool}
  target_temperature: {unit: degC}

sources:
  - name: chirpstack
    topic: application/+/device/+/event/up
    device_id: deviceInfo.devEui
    device_name: deviceInfo.deviceName
    model: deviceInfo.deviceProfileName
    space: deviceInfo.tags.location
    spot: deviceInfo.tags.spot
    measurements: object            # 이 사전의 숫자·참거짓 필드를 모두 측정값으로
    time: [min(rxInfo[*].nsTime)]   # 우선순위 목록. 모두 실패하면 수신 시각
    source_time: time
    frame_cnt: fCnt

  - name: xflow
    topic: xflow/+/+/+/status       # hvacr/idu, hvacr/odu
    also_topics: [xflow/controller/+/status]
    device_id: "topic[-2]"
    device_name: values.*.$key
    model: "topic[-3]"
    space: split(values.*.$key, "-", 0)
    measurements: values.*
    time: [ts]                      # epoch ms 자동 인식
    source_time: ts

ignore_topics:
  - gateway/+/state/#

timing:
  max_future_s: 120     # 기준 시각이 수신보다 이만큼 넘게 미래면 수신 시각으로 대체
  max_past_s: 3600

mqtt:   {keepalive_s: 30, reconnect_max_s: 30, stale_reconnect_s: 300, qos: 1, client_id: team2-collector}
storage: {table_prefix: "", batch_rows: 1000, flush_interval_s: 2}
spool:  {path: var/spool.db, max_rows: 400000}
report: {stale_after_s: 600, gap_min_minutes: 2}
```

## 검증 규칙

- `spaces[].id`는 `^[a-z][a-z0-9_]{1,31}$`이며 유일해야 한다. 별칭은 공간 사이에 겹치면 안 된다.
- `devices.*.space`는 `spaces`에 있는 id여야 한다.
- `sources[].name`은 유일해야 하고, `topic`은 올바른 MQTT 구독 패턴이어야 한다. `device_id`·`measurements`는 필수다.
- 경로 표현식은 시작 시 구문 검사를 한다. 실제 메시지에 경로가 없으면 그 메시지를 `failed`로 센다(수집기는 계속 돈다).
- 오류 메시지는 `config/collector.yaml: sources[1].time[0]: 알 수 없는 함수 'max2'` 형식으로 낸다.
