# Implementation Plan: 상시 추론 서비스 (과제 8)

**Branch**: `008-inference-service` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/008-inference-service/spec.md`

## Summary

프로세스 하나로 매분 `:20`초에 주기를 돌린다. 주기 하나의 흐름은 다음과 같다.

1. MySQL에서 최근 5시간 창을 증분으로 읽는다.
2. **학습과 같은 함수**로 전처리한다(축 A → 센서 5분 칸 → 공간 중앙값 → 인과적 배열). 닫힌 칸만 쓴다.
3. 과제 3·4·5 모델 묶음으로 공간마다 다음을 계산한다.
   - 60분 예측과 1,000ppm 초과 예상 시각
   - 이상 점수·원인
   - 규칙 경보(끊김·값 고정·범위 밖)
   - 사용 여부·환기 선제 경보
4. 경보는 상태 기계(발생·재알림 30분·해제 3주기)로 정리한다.
5. 결과를 MQTT 보존 메시지(`team2/inference/v1/{space}/{kind}`)로 발행하고, 같은 프로세스의 조회 API(HTTP JSON, 표준 라이브러리)로 제공한다.

운영과 기록
- 운영은 systemd 사용자 서비스 `wonil-inference`(Restart=always, Linger)로 한다.
- 주기·경보·예측 기록은 팀 DB 새 표(DB가 끊기면 SQLite 스풀)에 남기고, `infer report`가 12시간 요약을 만든다.
- 묶음이 없거나 깨진 판단은 `unavailable`로 끄고 나머지는 계속 돈다.

근거는 [research](research.md)에 있다.

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**
- 기존: paho-mqtt, PyMySQL, PyYAML, python-dotenv, pandas, numpy, holidays, PyTorch cu130
- 선택: 과제 7이 TensorRT를 고르면 `tensorrt-cu13`
- HTTP: 표준 라이브러리 `http.server`(새 의존성 없음)
- 재사용
  - `collector.store.connect`, `collector.spool`, `collector.report`의 끊김 판정
  - `analysis`의 축 A·칸·중앙값·일과표
  - `forecast`의 배열·특징·모델
  - `anomaly`의 오토인코더·규칙
  - 과제 5 `state` 분류기

**Storage**
- 입력: MySQL `measurement`(읽기만)
- 기록: MySQL 새 표 `svc_cycle`·`svc_alert`·`svc_forecast_log`, 로컬 스풀 `var/service/spool.db`, 상태 파일 `var/service/state.json`(경보 상태 기계 재시작 복원)

**Testing**: pytest
- `unit`: 전처리 일치(합성), 경보 상태 기계, 신선도, 부분 칸 제외, 결과 형식 스키마
- `broker`: 보드 안 amqtt로 보존 발행·새 구독자 수신·브로커 재시작 복구
- `mysql`: 실 DB 시험 표로 증분 읽기·기록
- `live`: 실서비스 장애 시험(수집기 10분 정지) 절차

**Target Platform**: Jetson Orin Nano Super, systemd 사용자 서비스

**Project Type**: 상시 서비스(데몬) + CLI + HTTP API

**Performance Goals**
- 주기 p95 20초 이하
- API p95 200ms 이하
- 새 구독자 1초 안 최신값
- 부팅 뒤 3분 안 첫 결과

**Constraints**
- 프로세스 메모리 1GB 이하
- 학습·서비스 전처리 동일
- 팀 브로커 발행은 사용자 승인 뒤 권한 확인
- sudo 없음

**Scale/Scope**: 공간 3(설정으로 확장), 매분 1주기, 하루 1,440주기, 결과 토픽 공간당 6개

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 한국어 문서. DB·브로커 접속은 `.env`(로드만). 결과·로그·API 응답에 비밀 없음. 공개 저장소에 내부 주소를 쓰지 않음(설정 예시는 자리표시자) | ✅ |
| II. 엣지 제약 | 프로세스 하나, 모델 한 번 적재, 1GB 목표, 새 의존성 없음(HTTP 표준 라이브러리), sudo 없이 사용자 서비스 | ✅ |
| III. 설정 주도 | 공간 목록·주기 시각·창 길이·신선도 한도·경보 재알림·발행 대상(팀/로컬)·API 포트·묶음 경로를 `config/service.yaml`에 둔다 | ✅ |
| IV. 테스트 | 확인 기준마다 시험: 전처리 일치(unit), 보존 발행·새 구독자(broker), 끊김 경보·복귀(unit + live 절차), API 응답(unit), 12시간 기록(측정 기록) | ✅ |
| V. 공정한 평가 | 자기 성능 감시는 과제 3 시험 MAE를 기준으로 실시간 채점. 요약표에 측정 조건(기간·묶음 버전·실행 방식·전력 모드) | ✅ |
| **VI. 사람 손 없는 복구** | 아래 표 | ✅ |
| VII. 해석 우선 | 12시간 요약에 처리 시간·경보·장애 시험의 원인과 조치. 결과에 권장 행동·근거 필드 | ✅ |

**VI 세부 — 장애마다 복구 경로**

| 장애 | 복구 |
|---|---|
| 수집기 정지·데이터 끊김 | `data_gap` 경보 + `stale` 표시, 계속 동작, 새 데이터가 오면 3주기 뒤 `cleared` |
| DB 끊김·느림 | 읽기 시간 초과 → 그 주기 `partial`, 다음 주기 재시도. 기록은 SQLite 스풀이 흡수 |
| 브로커 끊김 | paho 자동 재접속(최대 30초 간격, 과제 1과 같음), 재접속 시 최신 결과 보존 재발행 |
| 주기 중 예외 | 공간·판단 단위로 격리, 주기 기록 `failed`/`partial`, 다음 주기 정상 |
| 프로세스 종료·멈춤 | systemd `Restart=always`, 주기 정지 감시(감시 스레드가 3주기 연속 진행 없으면 종료 → 재시작) |
| 재부팅 | 사용자 서비스 + `Linger=yes`로 로그인 없이 시작, 경보 상태는 `state.json`으로 복원 |
| 묶음 깨짐 | 그 판단만 `unavailable`, 서비스는 계속 |

Phase 1 뒤 재확인: 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/008-inference-service/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (cli.md, config.md, mqtt.md, api.md)
├── checklists/requirements.md
└── tasks.md
```

### Source Code (repository root)

```text
src/preprocess/        학습·서비스 공통 전처리(과제 2·3에서 꺼낸 행 단위 순수 함수) — analysis·forecast가 이것을 부르도록 정리
src/service/
├── __main__.py        CLI: run · once · status · report · selftest · reload
├── config.py          config/service.yaml 읽기·검증
├── window.py          MySQL 증분 읽기, 메모리 창(최근 5시간), 늦은 행 보정
├── features.py        공통 전처리 호출 → 공간별 모델 입력(닫힌 칸만)
├── bundles.py         과제 3·4·5 묶음 적재·규약 검사, 실행 방식 선택(TensorRT/PyTorch GPU/CPU)
├── infer.py           예측·1,000ppm 초과 시각, 이상 점수·원인, 사용 여부·환기 경보
├── rules.py           끊김(과제 1 기준)·값 고정·범위 밖(1분 원본)
├── alerts.py          경보 상태 기계(발생·재알림·해제), state.json 복원
├── publish.py         MQTT 보존 발행(팀/로컬), 재접속 재발행
├── api.py             HTTP JSON 라우터(ThreadingHTTPServer), OpenAPI
├── store.py           최신 결과 메모리 저장소(잠금)
├── records.py         svc_* 표 기록 + SQLite 스풀
├── loop.py            주기 실행·건너뜀·감시 스레드
├── report.py          12시간 요약 Markdown
├── selftest.py        자기 성능 감시(심화 2)
└── control_sim.py     제어 규칙 시뮬레이션(심화 1)
config/service.yaml
deploy/wonil-inference.service · deploy/install-inference.sh
tools/infer          어디서든 실행(+ GPU 그룹 sg 재실행)
tests/service/         test_preprocess_parity · test_alerts · test_freshness · test_api · test_publish(broker) · test_records(mysql)
docs/deliverables/task8/   01-service-v1.md · 02-12h-run-summary.md · 03-result-format.md(다른 팀용)
docs/advanced/task8/       01-control-rules.md · 02-self-monitoring.md
docs/tasks/task8-inference-service.md
```

**Structure Decision**
- 서비스는 새 패키지 `src/service/`에 둔다.
- 전처리는 `src/preprocess/`로 꺼내 학습(analysis·forecast)과 서비스가 **같은 함수**를 부르게 한다. 학습 결과(평가 시점 해시 등)는 바뀌지 않아야 하며, 이를 시험으로 확인한다.
- 배포 방식은 과제 1 수집기와 같다(사용자 서비스, 설치 스크립트, `tools/` 래퍼).

## Complexity Tracking

| 추가 구조 | 필요한 이유 | 더 단순한 대안을 버린 이유 |
|---|---|---|
| `src/preprocess/` 공통 모듈(기존 코드 정리) | 학습·서비스 전처리 동일 보장(해결할 것 3) | 서비스용 복사본은 시간이 지나며 어긋남 |
