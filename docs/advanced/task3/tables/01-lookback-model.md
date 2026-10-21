# 과제 3 심화 결과 표 ① 과거 길이 × 모델 (자동 생성)

> **측정 조건**
> - 실데이터(과제 2 공간 5분 시계열). 분할 `v1-provisional`: 학습 2025-09-04 ~ 2026-03-31, 검증 2026-04-02 ~ 2026-05-09, **시험 2026-05-27 ~ 2026-07-22**(KST)
> - 평가 시점 파일 sha256 `2f878c11d09ab68d…`(과제 2와 같은 행·같은 정답), 선택 고정 sha256 `900fb554ba710ae8…`(2026-10-21T08:23:22+00:00)
> - 장치 Orin(Jetson Orin Nano Super, 전력 모드 25W), torch 2.14.1+cu130, TF32 끔·결정적 설정, 씨앗 3개(평균 ± 표준편차)
> - 다시 만들기: `forecast report`(표·그래프), 해석은 사람이 쓴다

## 검증 기간(선택 기준)

| 후보 | 검증 정규화 MAE | 검증 실습실 CO₂ 60분(ppm) | 파라미터 | 학습 시간(초) | 에폭(최고) | 선택 |
|---|---|---|---|---|---|---|
| gru 1h [sensors+time+space+schedule] direct delta | 0.0486 ± 0.0002 | 30.83 | 68,440 | 92 | 20(11) |  |
| lstm 1h [sensors+time+space+schedule] direct delta | 0.0503 ± 0.0003 | 31.95 | 82,200 | 79 | 15(6) |  |
| tcn 1h [sensors+time+space+schedule] direct delta | 0.0497 ± 0.0001 | 30.88 | 69,208 | 157 | 19(10) |  |
| gru 2h [sensors+time+space+schedule] direct delta | 0.0480 ± 0.0003 | 31.11 | 68,440 | 83 | 14(5) | ◀ |
| lstm 2h [sensors+time+space+schedule] direct delta | 0.0504 ± 0.0005 | 32.56 | 82,200 | 91 | 14(5) |  |
| tcn 2h [sensors+time+space+schedule] direct delta | 0.0495 ± 0.0004 | 32.52 | 93,912 | 229 | 16(7) |  |
| gru 4h [sensors+time+space+schedule] direct delta | 0.0480 ± 0.0003 | 30.85 | 68,440 | 111 | 14(5) |  |
| lstm 4h [sensors+time+space+schedule] direct delta | 0.0504 ± 0.0004 | 31.44 | 82,200 | 142 | 14(5) |  |
| tcn 4h [sensors+time+space+schedule] direct delta | 0.0495 ± 0.0004 | 32.08 | 118,616 | 332 | 15(6) |  |

선택: **gru 2h [sensors+time+space+schedule] direct delta** — 최소(gru 4h [sensors+time+space+schedule] direct delta)와 1표준편차 안 → 더 단순한 쪽. 이 조합의 시험 결과는 결과 표 ①이다.
