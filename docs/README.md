# 시계열 데이터 추론 실습 — team2 (계획·문서)

팀 Jetson에서 상시 운영할 센서 데이터 추론 기능의 **계획(Spec Kit)**과 **결과 문서**를 모은다.

> 💻 **코드는 [get-Data2/data](https://github.com/get-Data2/data)**에 있다.

## 원칙

- [프로젝트 헌법](../.specify/memory/constitution.md): 7개 원칙
  - 한국어 문서와 비밀 보호
  - 엣지 자원 제약
  - 설정 주도
  - 테스트로 검증
  - 공정한 평가
  - 사람 손 없는 복구
  - 해석 우선

## 과제별 산출물 문서

README의 해결할 것·확인 기준·산출물·생각해 볼 점·심화(B)를 하나도 빠뜨리지 않고 대응시킨 문서다. 최종 결과 보고서는 이 8개를 묶어 만든다.

| 과제 | 문서 | 상태 |
|---|---|---|
| 1. 데이터 수집 체계 | [task1-data-collection.md](tasks/task1-data-collection.md) | 🟢 완료(운영 중), 24h 최종표만 남음 |
| 2. 데이터 이해와 기준선 | [task2-data-baseline.md](tasks/task2-data-baseline.md) | 🟡 spec 초안 |
| 3. 60분 예측 | [task3-forecast-60min.md](tasks/task3-forecast-60min.md) | ⚪ 점검표 |
| 4. 이상 탐지 | [task4-anomaly-detection.md](tasks/task4-anomaly-detection.md) | ⚪ 점검표 |
| 5. 상태 추정 | [task5-state-estimation.md](tasks/task5-state-estimation.md) | ⚪ 점검표 |
| 6. 사전학습 모델 비교 | [task6-pretrained-models.md](tasks/task6-pretrained-models.md) | ⚪ 점검표 |
| 7. 엣지 최적화 | [task7-edge-optimization.md](tasks/task7-edge-optimization.md) | ⚪ 점검표 |
| 8. 상시 추론 서비스 | [task8-inference-service.md](tasks/task8-inference-service.md) | ⚪ 점검표 |

## 과제별 계획

| 과제 | 상태 | 요구사항 | 설계 | 작업 |
|---|---|---|---|---|
| 1. 데이터 수집 체계 | 🟢 상시 수집 중 | [spec](../specs/001-data-collection/spec.md) | [plan](../specs/001-data-collection/plan.md) · [research](../specs/001-data-collection/research.md) · [data-model](../specs/001-data-collection/data-model.md) · [contracts](../specs/001-data-collection/contracts/) | [tasks](../specs/001-data-collection/tasks.md) 52/56 |
| 2. 데이터 이해와 기준선 | 🟡 spec 초안 | [spec](../specs/002-data-baseline/spec.md) | | |
| 3. 60분 예측 | ⚪ | | | |
| 4. 이상 탐지 | ⚪ | | | |
| 5. 상태 추정 | ⚪ | | | |
| 6. 사전학습 모델 비교 | ⚪ | | | |
| 7. 엣지 최적화 | ⚪ | | | |
| 8. 상시 추론 서비스 | ⚪ | | | |

### 과제 1 핵심 결정 (요약)

- **시각**: 게이트웨이 한 대의 시계가 약 5분 빠르다(2026-10-20 실측). 그래서 기준 시각은 네트워크 서버 시각(`nsTime`)으로 하고,
  원천 시각과 수신 시각도 함께 저장한다. → [research R1](../specs/001-data-collection/research.md)
- **공간 매핑**: 원천마다 다른 공간 표기를 설정의 공간 목록(영문 id, 별칭)으로 통일한다. 실측 공간은 8개다.
  → [research R10](../specs/001-data-collection/research.md)
- **무손실**: 수신한 값은 로컬 스풀을 거쳐 MySQL에 일괄 기록한다. DB가 끊겨도 받은 값을 잃지 않는다.
- **자동 복구**: 브로커 재접속은 최대 30초 간격으로 시도한다. 재부팅 뒤에는 systemd 사용자 서비스와 linger(또는 cron)로 자동 시작한다.
  시험 브로커를 재시작했을 때 **6.0초** 만에 다시 수집했다.
- **실측으로 찾은 문제**
  - ChirpStack이 같은 업링크를 두 번 발행한다. 그래서 (장치, fCnt)로 중복을 걸러 낸다.
  - 설치 직후 브로커에 같은 client_id를 쓰는 다른 클라이언트가 있어 1초마다 서로 밀어냈다. client_id를 바꾸고 감지 기능을 넣었다.

## 결과 문서

| 문서 | 상태 |
|---|---|
| [collector-ops.md](collector-ops.md): 수집기 운영 가이드·장애 대응·확인 기준 검증 기록 | ✅ |
| [format-change-case.md](format-change-case.md): 형식 변경 대응 기록(과제 1 산출물) | ✅ |
| [reports/2026-10-21-coverage-interim.md](reports/2026-10-21-coverage-interim.md): 수집 현황 보고(과제 1 산출물) — 중간 16.3시간, 핵심 항목 공간별 최저 98.0% | 🟡 24시간 최종본은 10-21 16:40 이후 |
| [reports/2026-10-21-backfill.md](reports/2026-10-21-backfill.md): InfluxDB 과거 데이터 적재 — 978만 행, 장치 미특정 75만 행 원인과 계획 | 🟡 유선망 복구 대기 |
| [storage-policy.md](storage-policy.md): 저장소 1년 추정(8.2GB)과 보관 정책(심화 B) | ✅ |

모든 결과표에는 측정 조건(데이터 기간, 실/합성, 시험 기간, 전력 모드)을 함께 적는다.
