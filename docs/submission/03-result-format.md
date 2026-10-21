# 결과 형식 문서 — team2 상시 추론 서비스 v1

> 다른 팀(대시보드, 로컬 LLM 에이전트, 냉난방 제어)이 이 문서만 보고 결과를 받아 쓰도록 정리했다. 상세판·전체 응답 예시는 [과제 8 산출물 ③](../deliverables/task8/03-result-format.md)에 있다.
> 아래 예시는 실제 응답이다(2026-10-21 20:48~20:49 KST).

## 1. 무엇을, 언제

- 공간 `lab`(실습실)·`office`(사무실)·`meeting`(회의실)마다 **매분** 계산한다(매분 :20초 무렵).
- 판단은 **닫힌 5분 칸**까지의 데이터로 한다. `t0` = 마지막 닫힌 칸의 끝이다. 예: 13:03에 계산하면 `t0` = 13:00이다.
- 시각은 ISO 8601이다. `t0`는 UTC(`Z`)이고, `t0_kst`는 +09:00이다.

| 판단 | 필드 | 단위·범위 |
|---|---|---|
| 60분 예측 | `forecast.co2[12]`, `forecast.temperature[12]`, `forecast.horizons_min`(5~60, 5분 간격) | ppm, ℃. `co2[i]`는 `t0 + horizons_min[i]`분에 끝나는 5분 칸의 평균 |
| 바로 쓸 경보 | `co2_over.first_over_min` | 분. n = t0 뒤 n분에 처음 1,000ppm 이상, `0` = 이미 넘음, `null` = 60분 안에 안 넘음 |
| 이상 | `anomaly.is_anomaly`, `score_q`(0~1 분위수), `causes[]`(장치·항목·기여 0~1, 상위 3) | |
| 규칙 경보 | `rules.data_gap[]`·`stuck[]`·`out_of_range[]` | 장치 목록 |
| 상태 | `state.p_occupied`·`occupied`, `state.p_vent`·`ventilation_soon`(30분 안 1,000ppm) | 확률 0~1 |
| 신뢰 | `freshness_min`(데이터 나이, 분), `stale`(15분 초과), `availability.<판단>` | `ok` · `unavailable:<이유>` · `error:<예외>` |
| 사람·LLM용 | `summary_ko` | 한 줄 요약 |

`unavailable` 이유
- `no_bundle`: 모델 없음
- `data_gap`: 60분 넘게 데이터 없음 → 예측 중단
- `stale`: 15분 초과 → 이상 판단 보류
- `input_missing`·`no_result`: 입력 관측 부족. 회의실 상태는 시간의 약 3분의 1만 나온다(10분 주기 센서)

## 2. 조회 API — `http://<팀 Jetson 주소>:18080` (GET, 인증 없음)

| 엔드포인트 | 내용 |
|---|---|
| `/v1/spaces` | 공간 목록과 판단별 가용 상태 |
| `/v1/spaces/{space}` | 최신 결과 묶음 전체 |
| `/v1/spaces/{space}/forecast?minutes=30` | 지정 분(5~60) 뒤 CO₂·온도 |
| `/v1/spaces/{space}/anomaly` · `/state` | 이상 · 상태 |
| `/v1/alerts?status=active` | 활성 경보 |
| `/v1/health` · `/v1/openapi.json` | 서비스 상태(자기 성능 감시 포함) · 명세 |

오류 코드는 다음과 같다.

| 코드 | 언제 | 본문 |
|---|---|---|
| 400 | `minutes`가 5~60 밖 | `{"error": "minutes는 5~60"}` |
| 404 | 없는 공간 | `{"error": "없는 공간: x"}` |
| 503 | 첫 주기 전이거나 판단 불가 | `{"error": "unavailable", "reason": …}` |

```bash
curl -s "http://<팀 Jetson 주소>:18080/v1/spaces/meeting/forecast?minutes=30"
```
```json
{"schema": "team2.inference.v1", "space_id": "meeting", "t0": "2026-10-21T11:45:00Z", "minutes": 30,
 "target_time": "2026-10-21T12:15:00Z", "freshness_min": 19.3, "stale": true,
 "co2": {"value": 625.7, "unit": "ppm"}, "temperature": {"value": 20.06, "unit": "°C"},
 "co2_over": {"threshold_ppm": 1000.0, "first_over_min": null, "current_ppm": 641.0},
 "summary_ko": "회의실 30분 뒤 CO₂ 626ppm, 온도 20.1°C 예상",
 "model": {"version": "forecast-52aff4f0c4bf", "sha256": "696085dc53f9b69d", "runtime": "torch-cuda"}}
```

## 3. MQTT — `<팀 Jetson 주소>:11883`

- 구독은 익명으로 할 수 있다. QoS 1, UTF-8 JSON이다. **보존 메시지**라 새로 구독해도 최신값을 바로 받는다(실측 8ms 안 16개).
- 토픽은 `team2/inference/v1/{space}/forecast` · `/anomaly` · `/rules` · `/state` · `/alerts/active`(보존), `/alerts/events`(비보존), `team2/inference/v1/_service/health`(보존)다.

```bash
mosquitto_sub -h <팀 Jetson 주소> -p 11883 -t 'team2/inference/v1/#' -v
```

## 4. 경보 규칙

| `kind` | 뜻 | 알림 대상 |
|---|---|---|
| `co2_over_forecast` · `ventilation_soon` | 60분 안 1,000ppm 초과 예상 · 30분 안 초과(분류기) | 공간 사용자 |
| `data_gap` · `stuck` · `out_of_range` | 장치 끊김(마지막 수신 > max(10분, 주기 × 2)) · 값 고정 · 오류값 | 센서 관리 담당 |
| `anomaly` · `model_drift` · `service_degraded` | 이상 감지 · 예측 오차가 평소 2배 · 판단 일부 실패 | 시설 담당 |

재발송 규칙은 다음과 같다. 같은 경보를 매분 보내지 않는다.
- 처음 판정되면 `raised`를 보낸다.
- 계속되면 30분마다 `renotified`를 보낸다.
- 연속 3주기 정상이면 `cleared`를 보낸다.
- 서비스가 재시작돼도 활성 경보를 다시 `raised`로 보내지 않는다.

## 5. 버전 정책

- 필드 추가는 `v1` 안에서 한다. 받는 쪽은 모르는 필드를 무시해야 한다.
- 필드 삭제·의미 변경은 `team2/inference/v2/…`·`/v2/…`로 옮기고, `v1`을 30일 함께 둔다.
- 응답의 `model.version`·`sha256`으로 어떤 모델이 냈는지 추적한다.
