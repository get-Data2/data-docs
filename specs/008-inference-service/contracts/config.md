# Contract: `config/service.yaml`

```yaml
version: 1
analysis_config: config/analysis.yaml      # 공통 전처리 규칙(축 A·칸·중앙값·일과표)
collector_config: config/collector.yaml    # 장치 기대 주기·사건형 여부(끊김 판정)
spaces: [lab, office, meeting]

cycle:
  offset_s: 20                 # 매분 이 초에 시작
  window_h: 5                  # 메모리 창(가장 긴 입력 4시간 + 여유)
  late_min: 10                 # 늦게 도착한 행 보정
  db_timeout_s: 10
  watchdog_cycles: 3           # 연속 진행 없음 → 종료(→ systemd 재시작)

freshness:
  stale_min: 15                # 초과 시 stale 표시, 이상 판단 보류
  stop_min: 60                 # 초과 시 예측 중단(unavailable: data_gap)

bundles:
  forecast: var/forecast/bundle
  anomaly: var/anomaly/bundle
  state: var/state/bundle
  edge_selection: var/edgeopt/runtime/runtime.json   # 과제 7 결과(없으면 PyTorch)
  supported_input_spec: [1]

thresholds:
  co2_ppm: 1000

alerts:
  renotify_min: 30
  clear_after: 3
  roles:                       # 과제 4 운영 제안과 같은 역할 이름
    sensor_fault: 센서 관리 담당
    equipment_fault: 시설 담당
    air_quality: 공간 사용자

mqtt:
  target: local                # 2026-10-21 결정: 보드 로컬 브로커(B안). team은 대안으로만 남김
  topic_prefix: team2/inference/v1
  qos: 1
  local: {host: 127.0.0.1, port: 11883}     # 서비스는 같은 보드 브로커에 발행. 브로커는 0.0.0.0:11883에서 받아 다른 팀이 보드 주소로 구독
  # team: .env의 MQTT_* 사용(별도 client_id)
  client_id: team2-jetson-inference-a1

api:
  bind: 0.0.0.0
  port: 18080                  # 2026-10-21 결정: 흔한 8080을 피함(보드를 다른 계정과 공유). 시작 시 사용 중이면 알림
  max_minutes: 60

records:
  db_tables_prefix: svc_
  spool: var/service/spool.db
  keep_days: {forecast_log: 30, others: 90}

selfmonitor:                   # 심화 2
  window_h: 24
  drift_factor: 2.0            # 이동 MAE > 과제 3 시험 MAE × 2 → model_drift

control_sim:                   # 심화 1
  start_when: [ventilation_soon, co2_over_30min]
  stop_below_ppm: 800
  stop_hold_min: 30
  outdoor_ppm: 450
  speedup_k: [2, 3]            # 환기 시 감쇠 시간 상수 1/k
```

## 검증 규칙

- `spaces`가 분석 설정에 없음
- `freshness.stale_min ≥ stop_min`
- `mqtt.target`이 team·local이 아님
- `api.port`가 1024 미만(sudo 필요)
- `cycle.offset_s`가 0~50 밖

공개 저장소에 내부 주소를 쓰지 않는다. 팀 브로커 주소·계정은 `.env`에서만 읽는다.
