# Contract: MQTT 결과 토픽 (`team2/inference/v1`)

- **브로커**: 설정 `mqtt.target`에 따라 팀 브로커 또는 보드 로컬 브로커(포트 1884). 발행 권한 확인 결과는 점검 문서에 적는다.
- **QoS**: 1
- **보존(retained)**: 아래 표의 "보존" 열. 새 구독자는 접속 즉시 공간·종류별 최신 1개를 받는다.
- **인코딩**: UTF-8 JSON. 시각은 ISO 8601이며 UTC는 `Z`, KST는 `+09:00`이다.
- **주기**: 매분 `:20`초 전후

| 토픽 | 보존 | 내용 |
|---|---|---|
| `team2/inference/v1/{space}/forecast` | ✅ | 60분 예측(CO₂·온도, 5분 간격)과 1,000ppm 첫 초과 예상 |
| `team2/inference/v1/{space}/anomaly` | ✅ | 이상 여부·점수·원인 상위 3 |
| `team2/inference/v1/{space}/rules` | ✅ | 데이터 끊김·값 고정·범위 밖 장치 목록 |
| `team2/inference/v1/{space}/state` | ✅ | 사용 여부(확률)·환기 선제 경보(확률) |
| `team2/inference/v1/{space}/alerts/active` | ✅ | 현재 활성 경보 목록 |
| `team2/inference/v1/{space}/alerts/events` | ❌ | 경보 이벤트(raised · renotified · cleared) |
| `team2/inference/v1/_service/health` | ✅ | 서비스 상태(마지막 주기, 판단별 가용, 실행 방식, 메모리) |

`{space}`는 `lab` · `office` · `meeting`이다(설정으로 추가).

## 공통 필드

| 필드 | 형 | 단위 | 설명 |
|---|---|---|---|
| schema | str | — | `team2.inference.v1` |
| space_id / space_name | str | — | `meeting` / `회의실` |
| t0 | str | UTC | 예측·판단 기준 시각(마지막 닫힌 5분 칸 끝) |
| computed_at | str | UTC | 계산 끝 |
| freshness_min | float | 분 | 마지막 관측 칸 나이 |
| stale | bool | — | 신선도 15분 초과 |
| availability | str | — | `ok` · `unavailable:<이유>` · `error:<이유>` |
| model | object | — | `{version, sha256, runtime}` |

## 예시

`team2/inference/v1/meeting/forecast`

```json
{
  "schema": "team2.inference.v1", "space_id": "meeting", "space_name": "회의실",
  "t0": "2026-10-22T05:10:00Z", "t0_kst": "2026-10-22T14:10:00+09:00", "computed_at": "2026-10-22T05:11:21Z",
  "freshness_min": 6.0, "stale": false, "availability": "ok",
  "unit": {"co2": "ppm", "temperature": "°C"}, "step_min": 5,
  "horizons_min": [5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60],
  "co2": [812, 845, 878, 905, 931, 958, 981, 1004, 1022, 1038, 1050, 1061],
  "temperature": [23.1, 23.2, 23.2, 23.3, 23.3, 23.4, 23.4, 23.5, 23.5, 23.6, 23.6, 23.6],
  "interval": null,
  "co2_over": {"threshold_ppm": 1000, "first_over_min": 40, "current_ppm": 790},
  "summary_ko": "회의실 CO₂ 40분 뒤 1,000ppm 초과 예상(현재 790ppm)",
  "model": {"version": "forecast-<해시12>", "sha256": "<16자>", "runtime": "torch-cuda"}
}
```

`team2/inference/v1/lab/anomaly`

```json
{
  "schema": "team2.inference.v1", "space_id": "lab", "t0": "2026-10-22T05:10:00Z", "freshness_min": 5.0,
  "availability": "ok", "is_anomaly": true, "score_q": 0.99991, "threshold_q": 0.99962, "since": "2026-10-22T04:55:00Z",
  "causes": [{"device_id": "24e124126d152862", "item": "temperature", "contribution": 0.71},
             {"device_id": "24e124128c067999", "item": "temperature", "contribution": 0.12}],
  "summary_ko": "실습실 온도 센서 …152862 이상(점수 상위 0.01%)"
}
```

`team2/inference/v1/lab/alerts/events`

```json
{
  "schema": "team2.inference.v1", "kind": "data_gap", "event": "raised", "space_id": "lab",
  "device_id": "24e124785c389818", "started_at": "2026-10-22T05:02:00Z", "status": "active",
  "evidence": {"last_seen": "2026-10-22T04:51:12Z", "limit_min": 10},
  "action_ko": "센서 전원·게이트웨이 확인", "notify_role": "센서 관리 담당"
}
```

## 경보 이벤트 규칙

- 처음 판정되면 `raised`를 보낸다.
- 계속되면 30분마다 `renotified`를 보낸다.
- 연속 3주기 정상이면 `cleared`를 보낸다.
- 같은 경보를 매분 보내지 않는다.

## 버전 정책

- 필드 추가는 같은 `v1`에서 한다. 구독자는 모르는 필드를 무시한다.
- 필드 삭제·의미 변경은 `v2` 토픽으로 옮기고, `v1`을 30일 병행한다.
