# 형식 변경 대응 기록 (과제 1 산출물)

- 측정 조건: 2026-10-20 실 브로커에서 3분간 받은 메시지 표본(`tests/fixtures/messages.jsonl`)을 변형한 시험 메시지,
  수집기 v0.1.0, 확인 도구는 `python -m collector replay`(해석만 하고 저장하지 않음)
- 자동 검증: `tests/unit/test_format_change.py` — 같은 코드에 설정 조각만 바꿔 넣어 전후를 비교한다.
- 결론: 두 상황 모두 **`config/collector.yaml`만 고쳐 대응했다. `src/` 코드 수정은 0줄이다.**

## 상황 1. 센서 펌웨어가 필드 이름을 바꿈 (`co2` → `co2_ppm`)

**가정**: CO₂ 센서(EM500-CO2, AM103 등) 펌웨어를 올렸더니 `object.co2`가 `object.co2_ppm`으로 바뀌어 온다.

**영향(대응 전)**: 수집은 멈추지 않는다. 하지만 값이 `co2_ppm`이라는 **다른 항목**으로, 단위 없이 쌓인다.
이후 과제의 예측·탐지가 읽는 `co2` 계열은 그 시점부터 끊긴다. 현황 보고에서는 공간별 `co2` 커버율이 떨어지고
`co2_ppm` 항목이 새로 보이는 것으로 드러난다.

```text
| lab | 24e124126d152862 | co2_ppm | 879.0 |  | 15:42:06 |  |
```

**바꾼 설정**: 항목 별칭에 새 이름 하나를 더했다.

```diff
 items:
-  co2:                {unit: ppm,  aliases: [co2]}
+  co2:                {unit: ppm,  aliases: [co2, co2_ppm]}
```

**대응 후**: 같은 메시지가 기존 표준 항목 `co2`(단위 ppm)로 저장된다. 옛 펌웨어와 새 펌웨어가 섞여 있어도 둘 다 `co2`다.

```text
| lab | 24e124126d152862 | co2 | 879.0 | ppm | 15:42:06 |  |
```

## 상황 2. 다른 제품의 새 센서가 전혀 다른 형식으로 들어옴

**가정**: 회의실과 라운지에 다른 회사 게이트웨이를 달았다. 이 게이트웨이는 토픽 `lab/v2/{장치}/telemetry`에 아래 형식으로 보낸다.
라운지는 지금까지 없던 공간이다.

```json
{"dev": "nx-01", "room": "회의실", "at": 1792478500, "data": {"co2": 812, "temp_c": 23.1, "rssi": -70}}
```

**영향(대응 전)**: 구독 규칙이 없어 아예 받지 않는다.

```text
메시지 3 · 측정값 0 · 무시 메시지 3
```

**바꾼 설정**: 원천 규칙 하나, 공간 하나, 항목 별칭 하나를 추가했다.

```diff
 spaces:
+  - {id: lounge, name: 라운지, kind: lounge, aliases: [라운지]}
 items:
-  temperature: {unit: degC, aliases: [temperature, current_temperature]}
+  temperature: {unit: degC, aliases: [temperature, current_temperature, temp_c]}
 sources:
+  - name: labv2
+    topic: lab/v2/+/telemetry
+    device_id: dev
+    space: room
+    measurements: data
+    time: [at]            # epoch 초 자동 인식
```

**대응 후**: 기존 공간(회의실 → `meeting`)과 새 공간(라운지 → `lounge`)으로 매핑되어 저장된다.
`temp_c`는 표준 항목 `temperature`가 되고, `rssi`처럼 대응표에 없는 필드는 원래 이름으로 들어온다.

```text
| lounge  | nx-02 | co2         | 640.0 | ppm  | 15:42:40 |  |
| lounge  | nx-02 | temperature | 22.4  | degC | 15:42:40 |  |
| meeting | nx-01 | co2         | 830.0 | ppm  | 15:43:40 |  |
메시지 3 · 측정값 9 · 무시 메시지 0
```

## 운영 절차와 안전장치

1. 바뀐 메시지 몇 개를 JSONL로 저장한다(`{"t": 수신 epoch, "topic": ..., "payload": "..."}`).
2. 설정을 고친 뒤 `python -m collector check-config`로 검증한다. 틀리면 `파일: 위치: 이유`로 거부된다.
3. `python -m collector replay --file <저장한 파일>`로 저장 없이 결과를 확인한다.
4. `systemctl --user reload team2-collector`(SIGHUP)로 반영한다. 재시작은 필요 없다.
   - 검증에 실패하면 기존 설정을 유지하고 `config_rejected` 사건을 남긴다.
   - 성공하면 구독 토픽의 차이만 반영하고 `config_reload` 사건을 남긴다.

## 해석: 이 방식이 못 하는 것

- **값의 의미가 바뀌는 변경**(예: 단위가 ppm에서 %로 바뀜)은 이름 대응만으로는 잡히지 않는다. 숫자가 그대로 쌓이므로
  과제 2의 범위 검사(물리적으로 불가능한 값)와 과제 4의 이상 탐지가 그 역할을 맡는다.
- **값이 사전 안에 있지 않은 형식**(예: CSV 문자열 페이로드)은 경로 표현식으로 풀 수 없다. 이런 원천이 생기면
  해석기 종류를 하나 더하는 코드 변경이 필요하다. 현재 두 원천(ChirpStack, xflow)과 일반적인 JSON 형식은 모두 설정으로 표현된다.
- 대응표에 없는 새 필드는 원래 이름으로 쌓이므로 **조용히 새어 들어갈 수 있다**. 현황 보고 ② 공간·항목 표에 처음 보는
  항목이 나타나는지 주기적으로 확인해야 한다.
