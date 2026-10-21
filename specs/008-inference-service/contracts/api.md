# Contract: 조회 API (HTTP JSON, 읽기 전용)

- **주소**: `http://<보드 주소>:<port>/v1/...`. 포트 기본값은 8080이고 사용자 확인 뒤 확정한다. 내부망·Tailscale 접근을 가정한다.
- **방식**: GET만, 인증 없음(읽기 전용, 이번 범위)
- **응답**: `application/json; charset=utf-8`. 필드는 MQTT 결과와 같다(contracts/mqtt.md).
- **스키마**: `GET /v1/openapi.json`(OpenAPI 3)

| 엔드포인트 | 설명 |
|---|---|
| `GET /v1/spaces` | 공간 목록과 공간별 판단 가용 상태 |
| `GET /v1/spaces/{space}` | 최신 결과 묶음(예측·이상·규칙·상태·활성 경보) |
| `GET /v1/spaces/{space}/forecast?minutes=30` | t0 기준 지정 분 뒤의 CO₂·온도(5분 단위로 반올림, 5~60) |
| `GET /v1/spaces/{space}/anomaly` | 이상 판단 |
| `GET /v1/spaces/{space}/state` | 사용 여부·환기 선제 경보 |
| `GET /v1/alerts?status=active` | 활성 경보(전체 공간) |
| `GET /v1/health` | 서비스 상태: 마지막 주기, 처리 시간, 판단별 가용, 실행 방식, 메모리, 자기 성능 감시 |
| `GET /v1/openapi.json` | 스키마 |

## 예시

```bash
curl -s http://<보드 주소>:8080/v1/spaces/meeting/forecast?minutes=30
```

```json
{
  "schema": "team2.inference.v1", "space_id": "meeting", "t0": "2026-10-22T05:10:00Z", "minutes": 30,
  "target_time": "2026-10-22T05:40:00Z", "freshness_min": 6.0, "stale": false,
  "co2": {"value": 958, "unit": "ppm"}, "temperature": {"value": 23.4, "unit": "°C"},
  "co2_over": {"threshold_ppm": 1000, "first_over_min": 40},
  "summary_ko": "회의실 30분 뒤 CO₂ 958ppm, 온도 23.4°C 예상",
  "model": {"version": "forecast-<해시12>", "runtime": "torch-cuda"}
}
```

## 오류

| 코드 | 언제 | 본문 |
|---|---|---|
| 400 | `minutes`가 5~60 밖이거나 숫자가 아님 | `{"error": "minutes는 5~60"}` |
| 404 | 없는 공간·경로 | `{"error": "없는 공간: lab2"}` |
| 503 | 아직 첫 주기 전, 또는 그 판단이 `unavailable` | `{"error": "unavailable", "reason": "data_gap"}` |

응답 시간 목표는 p95 200ms 이하다(메모리 저장소 조회).
