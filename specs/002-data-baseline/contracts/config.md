# Contract: 분석 설정 `config/analysis.yaml`

정제·분할·일과표를 바꿀 때는 이 파일만 고친다. 장치별 기대 주기는 과제 1 `config/collector.yaml devices.*.expected_period_s`를 읽는다
(두 과제가 같은 결측 정의를 쓴다). 설정이 바뀌면 설정 해시가 바뀌고, 결과 파일 메타데이터에 남는다.

```yaml
version: 1

extract:
  items: [co2, temperature, humidity, battery]
  start: "2025-08-08"              # KST 날짜
  cache_dir: var/cache

axis_a:                            # 센서 오류 — 값 하나 + 기종으로 판정(원본은 지우지 않음)
  error_codes:                     # A1
    - {item: co2, values: [65535, 0]}
  error_pairs:                     # A2 — 같은 장치·같은 시각에 함께 나오면 둘 다 error
    - {temperature: -0.1, humidity: 127.5}
  ranges:                          # A3 — 기본, models로 기종별 덮어쓰기
    co2: [0, 10000]                # (0, 10000] — 0은 A1에서 처리
    temperature: [-40, 85]
    humidity: [0, 100]
  saturation:                      # A4 — 측정 범위 끝값에 붙음. 오류가 아님, 축 B 재료
    co2: {upper: 10000}            # 2026-10-21 실측: AM107이 정확히 10000 4건. AM103은 7,029까지 측정 → 5000은 상한이 아님
  pair_tolerance_s: 1              # A2 쌍을 같은 장치 1초 안에서 찾음(과거분은 1ms 차이 행이 있음)
  models: {}                       # 예: {AM103: {temperature: [0, 50]}}

axis_b:                            # 실제 사건 — 공간·시간 패턴
  fire:      {min_sensors: 2, temp_rise_c: 5, window_min: 30, humidity_drop: true}
  leak:      {humidity_min: 95, min_duration_min: 30}
  crowding:  {co2_floor: 2500, train_quantile: 0.999, min_sensors: 2, single_sensor_min_bins: 3}
  outlier:   {mad_k: 10, min_scale: {co2: 30, temperature: 0.3, humidity: 2}}   # B0 의심 — 센서별 편향·MAD, 분해능 미만 MAD는 바닥값
  stuck:     {min_hours: 2, require_others_change: true,
              others_min_change: {co2: 50, temperature: 1.0, humidity: 3}}   # 밤 안정기 양자화 반복을 거르기 위함

space:
  aggregate: median                # 센서 칸 평균의 중앙값
  exclude_from_space: [24e124136d151523]     # 실습실 '복도' 센서 — 진단에는 사용
  targets: [lab, office, meeting]            # 기준선 대상 공간

gaps:
  missing_factor: 2                # 결측 = 간격 > 기대 주기 × 2
  short_max_min: 15                # 짧은 결측(보간) 상한
  interpolation_test_min: [5, 10, 15, 30, 60]

interval:
  minutes: 5
  candidates: [1, 5, 15]

split:
  active: v1-provisional
  versions:
    v1-provisional: {train: ["2025-09-04", "2026-03-31"], val: ["2026-04-02", "2026-05-09"], test: ["2026-05-27", "2026-07-22"]}
    v2-final:       {train: ["2025-09-04", "2026-05-09"], val: ["2026-05-27", "2026-08-31"], test: ["2026-09-02", "2026-10-19"]}

baseline:
  items: [co2, temperature]
  horizons_min: [5, 15, 30, 60]
  mean_window_min: 60
  mean_min_coverage: 0.5
  usage_hours: {weekdays: [mon, tue, wed, thu, fri], start: "08:00", end: "19:00", exclude_holidays: true}

schedule:                          # 고정 일과표(2026-10-21 결정). 실제 자료는 file로 교체
  file: null                       # 예: data/schedule.csv
  exclude_holidays: true
  rows:
    - {space: lab,    days: [mon, tue, wed, thu, fri], start: "09:00", end: "11:30", kind: class}
    - {space: lab,    days: [mon, tue, wed, thu, fri], start: "13:00", end: "18:00", kind: class}
    - {space: office, days: [mon, tue, wed, thu, fri], start: "09:00", end: "11:30", kind: work}
    - {space: office, days: [mon, tue, wed, thu, fri], start: "13:00", end: "18:00", kind: work}
  reaction: {pre_min: 15, window_min: 60, delta_ppm: 30}

report:
  out_dir: docs/reports/task2
  font: "Noto Sans CJK KR"
```

## 검증 규칙

- 분할: 학습 < 검증 < 시험 순서이고 겹치지 않아야 한다. 각 경계 사이 간격은 1일 이상이다.
- `space.targets`와 `exclude_from_space`는 과제 1 공간·장치에 있어야 한다(경고).
- `axis_a`·`axis_b`는 서로를 참조하지 않는다(SC-008). 한쪽을 바꿔도 다른 쪽 결과 파일은 다시 만들 필요가 없다.
