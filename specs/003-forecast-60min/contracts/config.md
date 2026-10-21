# Contract: `config/forecast.yaml`

```yaml
version: 1
analysis_config: config/analysis.yaml   # 분할·일과표·사용 시간대 공유(과제 2)
split: v1-provisional
inputs:
  series: var/series/space_5min.parquet
  eval_points: var/eval/eval_points_v1-provisional.parquet
  eval_points_sha256: 2f878c11d09ab68d83eb9301ead583de41d6c9abb56eab43542b2cf38390e797
  baseline: var/eval/baseline_v1-provisional.parquet
  norm: var/eval/norm_v1-provisional.json
  events: var/cache/events.parquet
spaces: [lab, office, meeting]
targets: [co2, temperature]
horizon_bins: 12                     # 5분 × 12 = 60분
min_input_obs: 0.1                   # 입력 구간 관측 비율 하한(미만이면 예측 불가)
age_cap_h: 6

booking:
  file: null                         # 실제 예약 CSV(space_id,start,end[,kind]). 있으면 추정보다 우선
  estimate:                          # 회의실 추정 예약(R10) — 상한 실험
    spaces: [meeting]
    rise_ppm: 300
    rise_window_min: 30
    end_drop_ppm_per_10min: 50
    round_min: 15
    min_len_min: 30

window_open:                         # 창문 개방 추정(분석만, R12)
  drop_ppm: 200
  within_min: 10

defaults:                            # D0 기본 설정
  model: gru
  size: {hidden: 64, layers: 2, channels: 64, kernel: 3, head: 128, dropout: 0.1}
  lookback_min: 120
  features: [sensors, time, space, schedule]
  output: direct
  target: delta
  train_spaces: [lab, office, meeting]
  seeds: [0, 1, 2]
  train: {lr: 0.001, weight_decay: 0.0001, batch: 512, max_epochs: 40, patience_lr: 5, patience_stop: 8}

stages:                              # 단계마다 기본 설정에서 바꿀 값 목록(앞 단계 선택이 누적 적용됨)
  D1: {lookback_min: [30, 120, 1440]}
  D2: {features: [[sensors], [sensors, time], [sensors, time, space], [sensors, time, space, schedule]]}
  D3: {output: [direct, recursive]}
  D4: {target: [delta, value]}
  B1: {lookback_min: [60, 120, 240], model: [gru, lstm, tcn]}   # 곱집합 9개
selection:
  metric: val_norm_mae               # 정규화 MAE 평균(공간 × 항목 × 지평)
  tie_rule: simpler_within_1sd       # 씨앗 1표준편차 안이면 단순한 쪽(짧은 과거, 적은 입력, direct, delta)
```

## 검증 규칙(시작 시 거부)

- `split`이 `analysis_config`에 없음
- `eval_points_sha256` 불일치
- `features`에 알 수 없는 이름이 있거나 `sensors`가 없음
- `lookback_min`이 5의 배수가 아님
- `booking.file`이 있는데 열(`space_id`, `start`, `end`)이 없음
- `seeds`가 비어 있음

설정 해시는 `seeds`·`inputs` 경로를 뺀 실험 설정에서 계산한다(data-model "실험 설정").
