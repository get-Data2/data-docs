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

## 폴더 구성

| 폴더 | 담는 것 |
|---|---|
| [`deliverables/`](deliverables/README.md) | **산출물만**: 과제 안내서의 "산출물" 항목에 해당하는 문서(보고서·표·그래프). 최종 결과 보고서의 재료 |
| [`tasks/`](tasks/) | **과제별 점검 문서**: 해결할 것·확인 기준·생각해 볼 점·심화(B)의 항목별 대응과 확인 명령 |
| [`advanced/`](advanced/) | 심화(B) 결과 문서(`advanced/taskN/NN-이름.md`, 번호는 README 심화 목록 순서). 과제 2는 README에 심화가 없다 |
| [`ops/`](ops/) | 운영 가이드(설치·상태 확인·장애 대응) |
| [`reports/`](reports/) | 측정·작업 기록(장비 현황, 적재 보고, 자동 생성 상세 표, 사건 목록) |

## 과제별 산출물과 점검 문서

| 과제 | 산출물 | 점검 문서 | 상태 |
|---|---|---|---|
| 1. 데이터 수집 체계 | [수집 현황 보고](deliverables/task1/01-coverage-report-interim.md) · [형식 변경 대응 기록](deliverables/task1/02-format-change.md) | [task1](tasks/task1-data-collection.md) | 🟢 완료(운영 중), 24h 최종표만 남음 |
| 2. 데이터 이해와 기준선 | [데이터 진단서](deliverables/task2/01-data-diagnosis.md) · [기준선 성능표](deliverables/task2/baseline_v1-provisional.md) | [task2](tasks/task2-data-baseline.md) | 🟢 완료 — 분할 `v1-provisional`이 과제 3~6 평가 기준(2학기 `v2-final`은 망 복구 시 추가) |
| 3. 60분 예측 | — | [task3](tasks/task3-forecast-60min.md) | ⚪ 점검표 |
| 4. 이상 탐지 | — | [task4](tasks/task4-anomaly-detection.md) | ⚪ 점검표 |
| 5. 상태 추정 | — | [task5](tasks/task5-state-estimation.md) | ⚪ 점검표 |
| 6. 사전학습 모델 비교 | — | [task6](tasks/task6-pretrained-models.md) | ⚪ 점검표 |
| 7. 엣지 최적화 | — | [task7](tasks/task7-edge-optimization.md) | ⚪ 점검표 |
| 8. 상시 추론 서비스 | — | [task8](tasks/task8-inference-service.md) | ⚪ 점검표 |

## 과제별 계획

| 과제 | 상태 | 요구사항 | 설계 | 작업 |
|---|---|---|---|---|
| 1. 데이터 수집 체계 | 🟢 상시 수집 중 | [spec](../specs/001-data-collection/spec.md) | [plan](../specs/001-data-collection/plan.md) · [research](../specs/001-data-collection/research.md) · [data-model](../specs/001-data-collection/data-model.md) · [contracts](../specs/001-data-collection/contracts/) | [tasks](../specs/001-data-collection/tasks.md) 53/56 |
| 2. 데이터 이해와 기준선 | 🟢 완료 | [spec](../specs/002-data-baseline/spec.md) | [plan](../specs/002-data-baseline/plan.md) · [research](../specs/002-data-baseline/research.md) · [data-model](../specs/002-data-baseline/data-model.md) · [contracts](../specs/002-data-baseline/contracts/) | [tasks](../specs/002-data-baseline/tasks.md) 34/35 |
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

## 그 밖의 문서

| 문서 | 내용 | 상태 |
|---|---|---|
| [ops/collector-ops.md](ops/collector-ops.md) | 수집기 운영 가이드·장애 대응·확인 기준 검증 기록 | ✅ |
| [advanced/task1/01-storage-policy.md](advanced/task1/01-storage-policy.md) | 과제 1 심화(B): 저장소 1년 추정(8.2GB)과 보관 정책 | ✅ |
| [advanced/task1/02-duplicate-messages.md](advanced/task1/02-duplicate-messages.md) | 과제 1 심화(B): 재전송(이중 발행 3.2%) 찾아 처리, 남은 중복 0건 | ✅ |
| [reports/2026-10-21-devices.md](reports/2026-10-21-devices.md) | 장비 현황(정상·수신 중단·판단 보류·배터리) | ✅ |
| [reports/2026-10-21-backfill.md](reports/2026-10-21-backfill.md) | InfluxDB 과거 데이터 적재 — 978만 행, 장치 미특정 75만 행 원인과 계획 | 🟡 유선망 복구 대기 |
| [reports/task2/diagnosis_v1-provisional.md](reports/task2/diagnosis_v1-provisional.md) | 과제 2 진단 상세 표(자동 생성, 진단서의 숫자 원본) | ✅ |
| [reports/task2/events.md](reports/task2/events.md) | 과제 2 사건 후보·값 고정 목록(과제 4 입력) | ✅ |

모든 결과표에는 측정 조건(데이터 기간, 실/합성, 시험 기간, 전력 모드)을 함께 적는다.
