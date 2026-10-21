# Contract: `config/pretrained.yaml`

```yaml
version: 1
analysis_config: config/analysis.yaml
forecast_config: config/forecast.yaml          # 과제 3 평가 규칙·특징·결과 위치
split: v1-provisional
inputs:
  series: var/series/space_5min.parquet
  eval_points: var/eval/eval_points_v1-provisional.parquet
  eval_points_sha256: 2f878c11d09ab68d83eb9301ead583de41d6c9abb56eab43542b2cf38390e797
  baseline: var/eval/baseline_v1-provisional.parquet
  forecast_root: var/forecast
spaces: [lab, office, meeting]
item: co2
horizon_bins: 12
quantiles: [0.1, 0.5, 0.9]
min_context_obs: 0.1
missing: nan                                  # nan(모델에 결측 그대로) | ffill(앞으로 채움)

models:                                       # 리비전은 fetch 때 확인한 커밋 해시로 고정
  bolt_tiny:  {repo: amazon/chronos-bolt-tiny,  revision: null}
  bolt_mini:  {repo: amazon/chronos-bolt-mini,  revision: null}
  bolt_small: {repo: amazon/chronos-bolt-small, revision: null}
  chronos2:   {repo: amazon/chronos-2, revision: null, covariates: true}   # 심화 1(지원 확인 뒤)
dtype: {cuda: bfloat16, cpu: float32}         # 구현 때 fp32 대비 차이 기록
context_min: [60, 240, 1440, 10080]           # 1시간·4시간·1일·1주
covariates:                                   # 심화 1
  past: [temperature, humidity]
  known: [time, schedule]                     # forecast.features 재사용

ensemble:
  w_grid: {min: 0.0, max: 1.0, step: 0.1}
  modes: [global, per_cell]                   # 하나의 w / 공간 × 지평별 w — 검증으로 선택

bench:
  warmup: 10
  repeats: 100
  batches: [1, 3]
  devices: [cuda, cpu]
  tegrastats_interval_ms: 100
  idle_seconds: 30

budget:                                       # 메모리 예산(MB). 관찰값은 측정으로 덮어씀
  board_total: 7486
  limit_ratio: 0.85
  observed: {os_desktop: null, collector: null, other_accounts: 1000}
  llm_assumptions: {"2B-4bit": 1600, "4B-4bit": 3000, "8B-4bit": 5500}   # 사용자 확인 항목
```

## 검증 규칙

- `split`이 분석 설정에 없음
- `eval_points_sha256` 불일치
- `quantiles`에 0.1·0.5·0.9가 없음
- `context_min`이 5의 배수가 아님
- `missing`이 `nan`·`ffill`이 아님
- `models`가 비어 있음

설정 해시는 `inputs`를 뺀 정렬 JSON sha256 앞 12자리다.
