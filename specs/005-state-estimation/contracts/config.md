# Contract: `config/state.yaml`

```yaml
version: 1
analysis_config: config/analysis.yaml
split: v1-provisional
inputs:
  series: var/series/space_5min.parquet
  norm: var/eval/norm_v1-provisional.json
  booking_estimated: var/forecast/booking_estimated.parquet
  booking_file: null                 # 실제 예약 CSV(space_id,start,end, KST). 있으면 추정 예약 대신 사용
  forecast_bundle: var/forecast/bundle
  observations: data/observations.csv   # 사람 기록, git 제외
spaces: [lab, office, meeting]

labels:
  occupancy:
    meeting_source: booking_estimated   # booking_estimated | booking_file
  vent:
    threshold_ppm: 1000
    horizon_min: 30
    min_future_obs: 0.5              # (t, t+30분] 관측 비율 하한, 미만이면 판단 불가

observe:
  timezone: Asia/Seoul
  target:                            # 목표량(제안, research R2)
    class_days_per_space: 2
    total_hours: 20
    boundary_visits_per_space: 4     # 09:00·11:30·13:00·18:00 ±30분
    meeting: {meetings: 2, empty_hours: 2}

features:
  window_min: 30
  min_input_obs: 0.5
  groups:
    occupancy: [level, delta, shape, obs]                    # time·schedule 금지(설정 검증)
    vent: [level, delta, shape, obs, time, schedule]
    leak: [level, delta, shape, obs, time, schedule]          # 누수 비교 실험 전용(사용 여부 + 일정)

models:
  candidates:
    hgb: {max_depth: [3, 6, null], learning_rate: [0.05, 0.1, 0.2], class_weight: balanced}
    logreg: {C: [0.1, 1.0, 10.0], class_weight: balanced}
    mlp: {hidden: [32, 64], layers: 2, epochs: 30, class_weight: balanced}
  seeds: [0, 1, 2]

selection:
  occupancy: {metric: macro_f1, threshold: max_macro_f1}
  vent: {metric: macro_f1, threshold: {min_recall: 0.85, then: max_precision}}
  tie_rule: simpler_within_1sd      # logreg < hgb < mlp

rules:
  occupancy: {window_min: 15, rise_ppm: 30}
  vent: {co2_ppm: 850}

advanced:
  unseen_space: true
  noshow: {after_start_min: 15}
  unbooked: {min_minutes: 15}
```

## 검증 규칙

- `features.groups.occupancy`에 `time` 또는 `schedule`이 있으면 거부한다("사용 여부 입력에 일정 정보 금지", 누수 실험은 `leak`만).
- `labels.vent.threshold_ppm ≤ rules.vent.co2_ppm`이면 경고한다(규칙이 이미 초과 상태를 보게 됨).
- `selection.vent.threshold.min_recall`은 0과 1 사이여야 한다.
- `booking_file`이 있으면 열 `space_id`·`start`·`end`가 있어야 한다.
- `split`은 분석 설정에 있어야 한다.

설정 해시는 `inputs`를 뺀 정렬 JSON sha256 앞 12자리다.
