---
description: "과제 8 상시 추론 서비스 작업 목록"
---

# Tasks: 상시 추론 서비스 (과제 8)

**Input**: `specs/008-inference-service/`의 spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: 포함한다(헌법 IV). 사용하는 마커는 다음과 같다.
- `unit`: 전처리 일치, 경보 상태 기계, 신선도, API
- `broker`: 보존 발행·새 구독자, amqtt
- `mysql`: 증분 읽기·기록, 시험 표
- `live`: 장애 시험 절차

**Organization**: US1~US5 = P1, US6 = P2, US7(심화) = P3.
- 실행 순서: 공통 전처리 정리(US3) → 서비스 뼈대(US1) → 장애(US4) → 제공(US2) → 경보 정책(US6) → 문서(US5) → 12시간 가동 → 심화
- 과제 3·4·5 묶음이 없으면 해당 판단은 `unavailable`로 두고 뼈대·장애 시험을 먼저 끝낸다.

## Format: `[ID] [P?] [Story] Description`

- 경로는 저장소 루트 기준. 소스 `src/service/`·`src/preprocess/`, 시험 `tests/service/`, 설정 `config/service.yaml`

---

## Phase 1: Setup

- [X] T001 [P] `config/service.yaml`을 contracts/config.md 그대로 작성한다(공개 저장소 — 내부 주소 없이)
- [X] T002 [P] `tools/infer` 래퍼(GPU 그룹 sg 재실행, `tools/forecast` 방식)를 만들고 `~/.local/bin/infer`에 링크한다. 명령 이름은 `infer`다(시스템 `/usr/sbin/service`와 겹치지 않게 2026-10-21 정함)
- [X] T003 [P] `tests/service/__init__.py`, `conftest.py`
  - 합성 1분 원본(공간 3, 회의실 10분 주기)
  - 가짜 모델 묶음(입력 규약 v1, 작은 가중치)
  - amqtt 고정물은 `tests/integration/conftest.py`를 재사용한다
- [X] T004 pytest 마커 확인(`unit`·`broker`·`mysql`·`live`는 이미 있음). `pyproject.toml` 패키지 탐색에 `service`·`preprocess`가 포함되는지 확인한다

## Phase 2: User Story 3 — 학습과 같은 전처리 (P1, 기반)

**⚠️ 서비스 작업 전 필수**

- [~] T005 [US3] (부분: `src/preprocess/`가 기존 함수를 감싸 서비스가 사용. 학습 코드를 이 모듈로 옮기는 정리는 T006 해시 확인과 함께 남음) `src/preprocess/`: 과제 2·3의 전처리를 행 단위 순수 함수로 꺼낸다. 대상은 다음과 같다.
  - 축 A 판정
  - 센서 5분 칸 평균(닫힌 칸만)
  - 결측 판정(자기 주기 × 2)
  - 공간 중앙값(제외 장치)
  - 인과적 채우기·관측 표시·경과 시간
  - 시간·일과표 특징

  `analysis`·`forecast`가 이 함수를 부르도록 바꾼다.
- [ ] T006 [US3] 회귀 확인: 정리 뒤 `analysis all`의 평가 시점 sha256(`2f878c11…`)과 과제 3 배열이 그대로인지, `pytest tests/analysis tests/forecast`가 모두 통과하는지 본다
- [X] T007 [P] [US3] `tests/service/test_preprocess_parity.py` (unit)
  - 같은 원본 구간을 학습 경로(사본 → 배열)와 서비스 경로(1분 원본 창 → 입력)로 만들어 모델 입력 텐서가 같다(1e-4)
  - 부분 칸을 쓰지 않는다
  - 65535 오류 코드가 칸 평균에 들어가지 않는다
- [ ] T008 [US3] (mysql) 실데이터 일치 시험: 2026-10-21 시각 몇 개를 골라 MySQL 원본 서비스 경로와 과제 2 사본 학습 경로를 비교한다 — `tests/service/test_parity_real.py`

## Phase 3: User Story 1 — 매분 계산 (P1)

- [X] T009 [US1] `src/service/config.py`: 로드·검증(contracts/config.md)
- [X] T010 [US1] `src/service/window.py`: 시작 창 읽기, 증분 읽기(`ts > last − late_min`), 늦은 행 보정, DB 시간 초과
- [X] T011 [US1] `src/service/bundles.py`
  - 과제 3·4·5 묶음 적재와 입력 규약 버전 검사
  - 실행 방식 선택(과제 7 결과 → TensorRT, 없으면 torch GPU/CPU)
  - 실패 시 해당 판단 `unavailable`
- [X] T012 [US1] `src/service/infer.py`
  - 예측 12칸 × 2와 1,000ppm 첫 초과 시각
  - 이상 점수(분위수 표)·원인 상위 3
  - 사용 여부·환기 경보
- [X] T013 [US1] `src/service/rules.py`: 장치 끊김(과제 1 `report` 판정 재사용), 값 고정(과제 4 규칙 파라미터), 범위 밖(1분 원본·축 A 범위)
- [X] T014 [US1] `src/service/store.py`(최신 결과, 잠금), `src/service/loop.py`(매분 `:20`, 건너뜀, 감시 스레드)
- [X] T015 [US1] `src/service/__main__.py`: `once`, `run` 뼈대
- [X] T016 [P] [US1] `tests/service/test_cycle.py` (unit)
  - 합성 창에서 한 주기 결과 형식(contracts/mqtt.md 필드)
  - 묶음 없음 → `unavailable`
  - 주기 겹침 → `skipped`
- [X] T017 [US1] 실데이터로 `infer once` 실행: 공간별 결과, 처리 시간, 메모리 기록

## Phase 4: User Story 4 — 장애 (P1)

- [X] T018 [US4] 신선도(stale 15분·stop 60분), 공간·판단 단위 예외 격리, 주기 상태(ok·partial·failed·skipped)
- [X] T019 [US4] `src/service/records.py`: `svc_cycle`·`svc_alert`·`svc_forecast_log`·`svc_event` DDL(팀 DB, 접두사 `svc_`), 일괄 기록, SQLite 스풀(과제 1 `collector.spool` 재사용)
- [X] T020 [P] [US4] `tests/service/test_failures.py` (unit)
  - DB 시간 초과 주기 → `partial`, 다음 주기 `ok`
  - 모델 예외 격리
  - 감시 스레드: 진행 없음 → 종료 신호
- [X] T021 [US4] `deploy/wonil-inference.service`(Restart=always, 의존: 네트워크, 환경 변수 `CUBLAS_WORKSPACE_CONFIG` 등), `deploy/install-inference.sh`
- [ ] T022 [US4] `infer fault-test collector --minutes 10`: 단계 기록(`svc_event`), 끊김 경보 발생·해제 시각 측정

## Phase 5: User Story 2 — 제공 (P1)

- [X] T023 [US2] **사용자 확인**(2026-10-21 결정, research "결정 기록")
  - (a) MQTT: 보드 로컬 브로커(B안), `0.0.0.0:11883`, 구독 익명·발행은 서비스 계정만. 공용 서버 발행 시험은 하지 않는다
  - (b) API 포트 18080, 이 보드에서 상시 운영
- [X] T023a [US2] `deploy/wonil-broker.service`(amqtt 상시, Restart=always)와 설치 스크립트, 브로커 설정(포트 11883, 익명 구독 허용, 발행 계정·토픽 ACL `team2/inference/v1/#`). 발행 비밀번호는 `.env`에 키만 추가하고 출력하지 않는다
- [X] T024 [US2] `src/service/publish.py`: 보존 발행(QoS 1), 재접속 시 최신 재발행, 없어진 공간의 보존 메시지 지우기. 보드 amqtt 브로커(`wonil-broker`, T023a)에 발행
- [X] T025 [P] [US2] `tests/service/test_publish.py` (broker): 새 구독자 1초 안 최신값, 브로커 재시작 뒤 재발행
- [X] T026 [US2] `src/service/api.py`: 라우터(contracts/api.md), 400·404·503, OpenAPI JSON
- [X] T027 [P] [US2] `tests/service/test_api.py` (unit): 엔드포인트별 응답 형식, `minutes` 검증, p95 200ms(로컬)

## Phase 6: User Story 6 — 경보 정책 (P2)

- [X] T028 [US6] `src/service/alerts.py`
  - 상태 기계(raised·renotified·cleared)
  - 재알림 30분, 해제 3주기
  - `state.json` 복원
  - 권장 행동·알림 대상(과제 4 운영 제안)
- [X] T029 [P] [US6] `tests/service/test_alerts.py` (unit)
  - 매분 같은 판정에 이벤트 1번
  - 30분 재알림
  - 깜빡임 억제
  - 재시작 뒤 중복 raised 없음

## Phase 7: User Story 5 — 결과 형식 문서와 운영 (P1)

- [~] T030 [US5] (초안: `docs/deliverables/task8/03-result-format.md` — 예측 값 예시는 과제 3 묶음 뒤 실측으로 교체) `docs/deliverables/task8/03-result-format.md`(다른 팀용): 토픽·필드·단위·값 범위·예시·API·오류·버전 정책·경보 규칙·구독/조회 예제. **예제를 실제로 실행해 맞춘 뒤** 쓴다
- [ ] T031 [US5] 서비스 설치·12시간 가동 시작(`deploy/install-inference.sh`). 가동 중 장애 시험(수집기 10분 정지)을 수행한다
- [X] T032 [US5] `src/service/report.py` + `infer report`: 12시간 요약(주기 수·처리 시간 분포·경보 내역·장애 시험·메모리, 측정 조건)
- [ ] T033 [US5] `docs/deliverables/task8/01-service-v1.md`(운영 상태·구성·운영 방법), `02-12h-run-summary.md`(요약과 해석)
- [ ] T034 [US5] 재부팅 시험(사용자 일정): 로그인 없이 첫 결과까지 시간. GPU 그룹이 서비스에 반영됐는지(실행 방식) 확인

## Phase 8: User Story 7 — 심화 (P3)

- [ ] T035 [US7] `src/service/control_sim.py` + `infer simulate-control --days 7`
  - 감쇠 시간 상수 추정(과제 2 퇴실 뒤 감쇠)
  - 규칙 적용 재생
  - 1,000ppm 초과 시간 비교와 민감도(k=2·3)
- [X] T036 [US7] `src/service/selftest.py`: 예측 기록 채점, 지평별 24시간 이동 MAE, `model_drift` 경보, health 반영 — `tests/service/test_selftest.py`
- [ ] T037 [US7] `docs/advanced/task8/01-control-rules.md`, `02-self-monitoring.md`(가정·결과·해석)

## Phase 9: Polish

- [ ] T038 점검 문서 `docs/tasks/task8-inference-service.md`
  - 해결할 것 5, 확인 기준 4, 산출물 3, 생각해 볼 점 3(부분 칸·재발송·LLM 도구), 심화 2를 대응시킨다
  - **확인 명령**은 실행해 맞춘 뒤 적는다
- [ ] T039 [P] `docs/deliverables/README.md`, `docs/README.md`, `.github/README.md`(빠른 사용·API 주소 자리표시자), `CLAUDE.md`를 갱신한다
- [ ] T040 전체 시험(`unit or broker or mysql`). 비밀·내부 주소를 점검한 뒤 두 저장소를 커밋·푸시한다

---

## Dependencies & Execution Order

- Setup → US3(T005~T008, **학습 회귀 확인 필수**) → US1(T009~T017) → US4(T018~T022) → US2(T023~T027, T023 사용자 확인 선행) → US6(T028~T029) → US5(T030~T034) → US7 → Polish
- 외부 의존
  - 과제 3 묶음(예측), 과제 4 묶음(이상), 과제 5 묶음(상태), 과제 7 실행 방식
  - 없으면 `unavailable`로 진행하고, 생기면 `infer reload`
- 12시간 가동(T031)은 세 묶음이 붙은 뒤 시작하는 것이 바람직하다. 그 전 가동은 뼈대·장애 검증용으로 따로 기록한다.

## Implementation Strategy

1. **MVP**: 공통 전처리 + 예측 판단 하나 + 규칙 경보 + API + 로컬 발행 + 사용자 서비스. 이것만으로 장애 시험(수집기 10분 정지)과 12시간 가동 틀을 검증한다.
2. 이상·상태 판단, 경보 정책, 결과 형식 문서를 더한다.
3. 12시간 정식 가동 → 요약 → 심화.
