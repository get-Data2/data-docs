# 과제 8 산출물 ③ 결과 형식 문서 — team2 상시 추론 서비스 v1

> 다른 팀이 이 문서만 보고 결과를 받아 쓸 수 있게 정리했다. 대상은 대시보드, 로컬 LLM 에이전트, 냉난방 제어다.
> 상태: 상시 가동 중(2026-10-21 20:17 KST~). 예측·이상·상태·규칙 판단이 모두 `ok`이다. 아래 예시는 모두 실제 응답이다(`curl`로 받아 그대로 실음).

## 1. 무엇을 주나

공간마다 **매분** 아래를 계산한다. 공간은 `lab`(실습실), `office`(사무실), `meeting`(회의실)이다.

| 판단 | 내용 | 단위 |
|---|---|---|
| `forecast` | 60분 앞까지 5분 간격(12칸) CO₂·온도 예측, 1,000ppm 첫 초과 예상 시점 | ppm, °C, 분 |
| `anomaly` | 이상 여부, 이상 점수(분위수), 원인 센서 상위 3 | 0~1 |
| `rules` | 데이터 끊김·값 고정·범위 밖 장치 목록 | — |
| `state` | 사용 여부(확률), 환기 선제 경보(30분 안 1,000ppm 초과 확률) | 0~1 |
| `alerts` | 활성 경보 목록과 경보 이벤트(생김·재알림·해제) | — |

받는 방법은 두 가지다.
- **MQTT 구독**: 새로 접속해도 최신값을 바로 받는다(보존 메시지)
- **조회 API**(HTTP GET): 필요할 때 묻는다

## 2. 시각 규칙 — 꼭 읽을 것

- 모든 판단은 **닫힌 5분 칸**까지의 데이터로 한다. 13:03:20에 계산하면 마지막 입력은 [12:55, 13:00) 칸이다.
- `t0`는 **마지막 닫힌 칸의 끝**이다(위 예에서 13:00). 정보는 최대 5분 늦다. 대신 학습 때와 같은 칸 정의를 쓴다.
- `forecast.co2[i]`는 `t0 + horizons_min[i]`분에 **끝나는** 5분 칸의 평균이다. 예: `horizons_min[5] = 30`이면 [t0+25분, t0+30분) 칸이다.
- `co2_over.first_over_min`
  - 숫자 n: t0 기준 n분 뒤 칸에서 처음 1,000ppm 이상이 예상된다
  - `0`: 이미 넘었다
  - `null`: 60분 안에는 넘지 않는다
- 시각은 ISO 8601이다. UTC는 `Z`, 사람이 볼 값은 `t0_kst`(+09:00)에 있다.
- `freshness_min`은 그 공간 대표값의 마지막 관측 칸 끝에서 지금까지의 분이다.
  - 15분을 넘으면 `stale: true`이고, 이상 판단은 보류한다.
  - 60분을 넘으면 예측을 멈추고 `unavailable:data_gap`으로 낸다.

## 3. MQTT

| 항목 | 값 |
|---|---|
| 브로커 | 팀 Jetson 보드의 로컬 브로커. 주소는 팀 Jetson 주소(내부망·Tailscale), **포트 11883** |
| 인증 | 구독은 계정 없이(익명) 된다. 발행은 서비스만 할 수 있다 |
| 구독 가능 토픽 | `team2/inference/v1/#` 아래만 |
| QoS | 1 |
| 인코딩 | UTF-8 JSON |

| 토픽 | 보존 | 내용 |
|---|---|---|
| `team2/inference/v1/{space}/forecast` | ✅ | 예측·1,000ppm 첫 초과 |
| `team2/inference/v1/{space}/anomaly` | ✅ | 이상 판단 |
| `team2/inference/v1/{space}/rules` | ✅ | 끊김·값 고정·범위 밖 장치 |
| `team2/inference/v1/{space}/state` | ✅ | 사용 여부·환기 선제 경보 |
| `team2/inference/v1/{space}/alerts/active` | ✅ | 현재 활성 경보 |
| `team2/inference/v1/{space}/alerts/events` | ❌ | 경보 이벤트(생김 `raised`·재알림 `renotified`·해제 `cleared`) |
| `team2/inference/v1/_service/health` | ✅ | 서비스 상태 |

구독 예(mosquitto 클라이언트):

```bash
mosquitto_sub -h <팀 Jetson 주소> -p 11883 -t 'team2/inference/v1/#' -v
```

파이썬(paho-mqtt 2.x):

```python
import json, paho.mqtt.client as mqtt
c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
c.on_message = lambda cl, u, m: print(m.topic, json.loads(m.payload or b"{}"))
c.connect("<팀 Jetson 주소>", 11883)
c.subscribe("team2/inference/v1/+/forecast", qos=1)
c.loop_forever()
```

## 4. 조회 API

`http://<팀 Jetson 주소>:18080` — GET만, 인증 없음(읽기 전용). 응답은 `application/json; charset=utf-8`이다.

| 엔드포인트 | 설명 |
|---|---|
| `GET /v1/spaces` | 공간 목록과 판단별 가용 상태 |
| `GET /v1/spaces/{space}` | 최신 결과 묶음(예측·이상·규칙·상태·활성 경보) |
| `GET /v1/spaces/{space}/forecast?minutes=30` | t0 기준 지정 분 뒤 CO₂·온도(5~60, 5분 단위로 반올림) |
| `GET /v1/spaces/{space}/anomaly` | 이상 판단 |
| `GET /v1/spaces/{space}/state` | 사용 여부·환기 선제 경보 |
| `GET /v1/alerts?status=active` | 활성 경보(전체 공간) |
| `GET /v1/health` | 서비스 상태: 마지막 주기·처리 시간·판단별 가용·메모리·자기 성능 감시 |
| `GET /v1/openapi.json` | OpenAPI 3 스키마 |

| 코드 | 언제 | 본문 |
|---|---|---|
| 200 | 정상 | 결과 |
| 400 | `minutes`가 5~60 밖이거나 숫자가 아님 | `{"error": "minutes는 5~60"}` |
| 404 | 없는 공간·경로 | `{"error": "없는 공간: lab2"}` |
| 503 | 첫 주기 전이거나 그 판단이 불가 | `{"error": "unavailable", "reason": "no_bundle"}` |

```bash
curl -s "http://<팀 Jetson 주소>:18080/v1/spaces/meeting/forecast?minutes=30"
```

실제 응답(2026-10-21 20:49 KST):

```json
{"schema": "team2.inference.v1", "space_id": "meeting", "t0": "2026-10-21T11:45:00Z", "minutes": 30,
 "target_time": "2026-10-21T12:15:00Z", "freshness_min": 19.3, "stale": true,
 "co2": {"value": 625.7, "unit": "ppm"}, "temperature": {"value": 20.06, "unit": "°C"},
 "co2_over": {"threshold_ppm": 1000.0, "first_over_min": null, "current_ppm": 641.0},
 "summary_ko": "회의실 30분 뒤 CO₂ 626ppm, 온도 20.1°C 예상",
 "model": {"version": "forecast-52aff4f0c4bf", "sha256": "696085dc53f9b69d", "runtime": "torch-cuda"}}
```

이 응답은 수집기 정지 시험 직후라 `stale: true`(데이터 19분 전)다. 예측은 60분까지는 계속 내고, 받는 쪽이 `stale`을 보고 믿을지 정한다.

오류 응답(실제):

```bash
curl -s "http://<팀 Jetson 주소>:18080/v1/spaces/lab/forecast?minutes=61"   # {"error": "minutes는 5~60"}  (400)
curl -s "http://<팀 Jetson 주소>:18080/v1/spaces/x"                         # {"error": "없는 공간: x"}     (404)
```

## 5. 공통 필드

| 필드 | 형 | 설명 |
|---|---|---|
| `schema` | str | `team2.inference.v1` |
| `space_id` / `space_name` | str | `meeting` / `회의실` |
| `t0`, `t0_kst` | str | 기준 시각(2절) |
| `computed_at` | str | 계산이 끝난 시각(UTC) |
| `freshness_min` | float · null | 데이터 나이(분). 관측이 없으면 null |
| `stale` | bool | 15분 초과 |
| `availability` | str | `ok` · `unavailable:<이유>` · `error:<이유>` |
| `model` | object | `{version, sha256, runtime}`. runtime은 `tensorrt` · `torch-cuda` · `torch-cpu` |
| `summary_ko` | str | 한 줄 요약(LLM 에이전트·사람용) |

**가용성 이유**

| 값 | 뜻 |
|---|---|
| `unavailable:no_bundle` | 그 판단의 모델이 아직 없다 |
| `unavailable:data_gap` | 데이터가 60분 넘게 없다 |
| `unavailable:input_missing` | 입력 구간의 관측이 너무 적다 |
| `unavailable:stale` | 15분 넘게 오래돼 이상 판단을 보류했다 |
| `error:<예외>` | 그 판단 계산이 실패했다. 다른 판단은 계속 나온다 |

실제 결과 묶음 예(`GET /v1/spaces/lab`, 2026-10-21 20:48 KST, 예측 12칸은 앞 3칸만 표시):

```json
{"schema": "team2.inference.v1", "space_id": "lab", "space_name": "실습실",
 "t0": "2026-10-21T11:45:00Z", "t0_kst": "2026-10-21T20:45:00+09:00", "computed_at": "2026-10-21T11:48:20.757405Z",
 "freshness_min": 3.3, "stale": false,
 "availability": {"rules": "ok", "forecast": "ok", "anomaly": "ok", "state": "ok"},
 "models": {"forecast": {"version": "forecast-52aff4f0c4bf", "sha256": "696085dc53f9b69d", "runtime": "torch-cuda"},
            "anomaly": {"version": "anomaly-240088924310", "sha256": "6d04702384d48367", "runtime": "torch-cuda"},
            "state": {"version": "state-1cd4d961a332", "sha256": "d40b1dd327551ef2", "runtime": "torch-cuda"}},
 "forecast": {"unit": {"co2": "ppm", "temperature": "°C"}, "step_min": 5, "horizons_min": [5, 10, 15, "…", 60],
              "co2": [564.4, 563.7, 562.5, "…", 558.3], "temperature": [22.49, 22.48, 22.47, "…", 22.36], "interval": null},
 "co2_over": {"threshold_ppm": 1000.0, "first_over_min": null, "current_ppm": 564.5},
 "anomaly": {"is_anomaly": false, "score": 114.3035, "score_q": 0.91321, "threshold_q": 0.999,
             "causes": [{"device_id": "24e124128c067999", "item": "temperature", "contribution": 0.298},
                        {"device_id": "24e124126d152590", "item": "co2", "contribution": 0.262},
                        {"device_id": "24e124126d152862", "item": "temperature", "contribution": 0.146}]},
 "state": {"p_occupied": 0.1586, "occupied": false, "occupancy_threshold": 0.5908,
           "p_vent": 0.0001, "ventilation_soon": false, "vent_threshold": 0.6371},
 "rules": {"data_gap": [], "stuck": [], "out_of_range": []},
 "summary_ko": "실습실 CO₂ 60분 안 1,000ppm 초과 없음(현재 564ppm)"}
```

| 판단 | 필드 | 범위·뜻 |
|---|---|---|
| `anomaly` | `score_q` | 0~1. 과제 4 검증 기간 점수 분포에서의 분위수. `threshold_q`(0.999) 이상이면 `is_anomaly` |
| | `causes[].contribution` | 0~1, 합 1. 재구성 오차에서 그 장치·항목이 차지하는 몫 |
| `state` | `p_occupied`, `p_vent` | 0~1 확률. 각 `_threshold` 이상이면 `occupied`·`ventilation_soon` |
| | (없음) | 최근 30분 관측 칸이 50% 미만이면 `state`가 빠지고 `availability.state = unavailable:no_result`. 회의실(10분 주기 센서)은 시간의 약 3분의 1만 나온다 |

회의실 신선도는 평소 5~15분이다. 10분 주기 센서이기 때문이다(정상).

## 6. 경보

| 종류 `kind` | 뜻 | 알림 대상(역할) |
|---|---|---|
| `co2_over_forecast` | 60분 안 1,000ppm 초과 예상 | 공간 사용자 |
| `ventilation_soon` | 환기 선제 경보(과제 5 분류기) | 공간 사용자 |
| `anomaly` | 이상 감지(원인 센서 포함) | 시설 담당 |
| `data_gap` | 장치 데이터 끊김: 마지막 수신이 max(10분, 기대 주기 × 2)보다 오래됨 | 센서 관리 담당 |
| `stuck` | 값 고정: 1시간 이상 같은 값이고 이웃 센서는 변함 | 센서 관리 담당 |
| `out_of_range` | 오류 코드·측정 범위 밖 값 | 센서 관리 담당 |
| `model_drift` | 예측 오차가 평소의 2배 넘게 커짐(자기 성능 감시) | 시설 담당 |
| `service_degraded` | 서비스 판단 일부가 실패 | 시설 담당 |

**재발송 규칙**: 같은 경보를 매분 보내지 않는다.
- 처음 판정되면 `raised`를 보낸다.
- 계속되면 30분마다 `renotified`를 보낸다.
- 연속 3주기(3분) 정상이면 `cleared`를 보낸다.
- 서비스가 재시작돼도 활성 경보를 다시 `raised`로 보내지 않는다.

```json
{"schema": "team2.inference.v1", "kind": "data_gap", "event": "raised", "space_id": "lab",
 "device_id": "24e124785c389818", "started_at": "2026-10-22T05:02:00Z", "status": "active",
 "evidence": {"last_seen": "2026-10-22T04:51:12Z", "limit_min": 10, "age_min": 11.2},
 "action_ko": "센서 전원·게이트웨이 확인", "notify_role": "센서 관리 담당"}
```

## 7. 버전 정책

- 필드 추가는 같은 `v1`에서 한다. 받는 쪽은 모르는 필드를 무시해야 한다.
- 필드 삭제·의미 변경은 `team2/inference/v2/...`와 `/v2/...`로 옮기고, `v1`을 30일 함께 둔다.

## 8. 운영 정보

| 항목 | 값 |
|---|---|
| 서비스 | systemd 사용자 서비스 `wonil-inference`(재부팅 시 자동 시작), 브로커 `wonil-broker` |
| 주기 | 매분 20초 전후 |
| 상태 확인 | `GET /v1/health` 또는 토픽 `team2/inference/v1/_service/health` |
| 문의 | team2 |
