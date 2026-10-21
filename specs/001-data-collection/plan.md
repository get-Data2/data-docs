# Implementation Plan: 데이터 수집 체계 (과제 1)

**Branch**: `001-data-collection` | **Date**: 2026-10-20 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-data-collection/spec.md`

## Summary

MQTT 두 원천(ChirpStack LoRaWAN, xflow 냉난방)을 구독한다. **설정 파일의 원천 규칙과 공간 매핑**으로 메시지를
`공간 × 장치 × 항목 × 기준 시각` 측정값으로 풀어 MySQL에 쌓는다. 수신 → 로컬 SQLite 스풀 → 일괄 기록 구조로 저장소
장애와 재시작에도 손실이 없게 한다. paho 자동 재접속(최대 30초 간격)으로 브로커 복귀 후 1분 안에 재수집한다.
systemd 사용자 서비스와 linger(안 되면 cron `@reboot` 감시 스크립트)로 재부팅 뒤 자동 시작한다. `report` 명령이
공간·항목별 건수, 마지막 수신, 커버율, 빠진 구간과 원인 단서를 표와 Markdown으로 낸다.
기준 시각은 게이트웨이 시계 오차(실측 +4분 55초)를 피해 네트워크 서버 시각을 쓴다([research R1](research.md)).

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`, `uv`로 설치)

**Primary Dependencies**: paho-mqtt 2.1, PyMySQL, PyYAML, python-dotenv. 시험용으로 pytest, amqtt 0.12(순수 파이썬 시험 브로커, 설치·동작 확인함)

**Storage**: MySQL 8.0 `cf_u36_d3`(측정값·장치·공간·사건·통계), 로컬 SQLite `var/spool.db`(쓰기 앞 스풀)

**Testing**: pytest. 마커는 `unit`(오프라인), `broker`(amqtt 시험 브로커), `mysql`(실 DB, `test_` 접두사 테이블), `live`(실 브로커)

**Target Platform**: Jetson L4T R39 aarch64, systemd 사용자 서비스, sudo·Docker 없음

**Project Type**: 상시 실행 데몬 + CLI(단일 파이썬 패키지)

**Performance Goals**: 현재 약 20건/분 메시지(측정값 약 150건/분)를 지연 5초 이내로 기록한다. 10배 증가에도 여유가 있어야 한다. 24시간 보고는 10초 이내다.

**Constraints**: 메모리 200MB 이하, 브로커 복귀 후 1분 안 재수집, 저장소 10분 장애 시 손실 0, 코드 수정 없는 형식 변경 대응

**Scale/Scope**: 장치 약 20대, 공간 8개(실측) + 합성 4개, 하루 약 20만 행

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 보호 | 문서 한국어. `.env`는 `python-dotenv`로만 로드. 로그·사건 `detail`에 접속 정보를 넣지 않는다. README 수정 없음 | ✅ |
| II. 엣지 자원 제약 | `.venv` + uv, 사용자 서비스, 메모리 상한(`MemoryMax=256M`)을 서비스 파일에 명시 | ✅ |
| III. 설정 주도 | 원천 규칙·공간 매핑·항목 대응을 YAML로 둔다. 시작 시 검증하고, 형식 변경 대응을 `replay`로 확인한다 | ✅ |
| IV. 테스트로 검증 | 확인 기준 3개를 각각 자동 테스트나 재현 절차로 연결(아래 표) | ✅ |
| V. 공정한 평가 | 보고 머리에 측정 조건(기간·실/합성·생성 시각)을 적는다 | ✅ |
| VI. 사람 손 없는 복구 | 자동 재접속, 스풀, `Restart=always`, linger/cron, 사건 기록 | ✅ |
| VII. 해석 우선 | 빠진 구간마다 원인 단서를 붙이고, 보고서(`docs/`)에 원인과 조치를 쓴다 | ✅ |

확인 기준 ↔ 검증 연결:

| README 확인 기준 | 검증 |
|---|---|
| 브로커를 껐다 켜도 1분 안에 재수집 | `tests/integration/test_broker_restart.py`(amqtt 종료→재시작→첫 저장까지 ≤60초) |
| 보드 재부팅 후 자동 재시작 | `deploy/` 설치 후 실제 재부팅 → `collector_event`의 `start` 시각과 부팅 시각 비교. 절차·결과는 `docs/` 기록 |
| 24시간 커버율 95% 이상, 결측 원인 설명 | `report --hours 24` 실행 결과를 `docs/reports/`에 보관하고 결측 원인 해석을 덧붙인다. 계산 정확성은 `tests/unit/test_coverage.py` |

Phase 1 후 재확인: 설계에 위반 없음. Complexity Tracking 비움.

## Project Structure

소스·설정·배포·시험·문서·계획을 디렉토리로 나눈다. **계획(specs)과 결과 문서(docs)는 git으로 올려 함께 본다.**

### Documentation (this feature)

```text
specs/001-data-collection/          # 계획: 무엇을·어떻게 (Spec Kit 산출물)
├── spec.md                         # 요구사항·수용 기준
├── plan.md                         # 이 파일
├── research.md                     # 결정과 근거(R1~R10)
├── data-model.md                   # 테이블·스풀·공간 매핑
├── quickstart.md                   # 설치·검증 절차
├── contracts/                      # 외부 약속: 설정 형식, CLI, DDL
│   ├── config.md
│   ├── cli.md
│   └── schema.sql
├── checklists/requirements.md
└── tasks.md                        # /speckit-tasks 산출물
```

### Source Code (repository root)

```text
src/collector/                      # 소스 코드 (패키지)
├── __main__.py                     # CLI 진입점: run / check-config / init-db / report / replay
├── config.py                       # YAML 읽기·검증, 공간 매핑 해석
├── pathexpr.py                     # 경로 표현식 해석기
├── parser.py                       # 메시지 → 측정값(원천 규칙 적용, 기준 시각 결정)
├── spool.py                        # SQLite 스풀
├── store.py                        # MySQL 기록·DDL
├── service.py                      # MQTT 구독, 기록 스레드, 감시자, 신호 처리
└── report.py                       # 커버율·빠진 구간·원인 단서·용량 추정

config/collector.yaml               # 원천 규칙 + 공간 매핑 (형식 변경은 여기만)
deploy/                             # 상시 운영
├── wonil-collector.service         # systemd 사용자 서비스
├── run_forever.sh                  # linger 불가 시 cron @reboot 감시 스크립트
└── install.sh                      # 서비스 등록·linger 시도·cron 대체
tests/
├── fixtures/messages.jsonl         # 실측 메시지 표본(2026-10-20)
├── unit/                           # 경로식·파서·설정 검증·커버율 계산
└── integration/                    # 시험 브로커 재시작, 스풀 무손실, DB 기록
docs/                               # 결과 문서 (보고서에 들어갈 것)
├── collector-ops.md                # 운영 가이드: 설치·상태 확인·장애 대응
├── format-change-case.md           # 산출물: 형식 변경 대응 기록
└── reports/                        # 산출물: 수집 현황 보고(날짜별)
var/                                # 실행 산출물(스풀·로그), git 제외
```

**Structure Decision**: 단일 패키지 `src/` 레이아웃을 쓴다. 이후 과제(진단·예측·서비스)는 `src/` 아래에 형제 패키지로
추가한다(예: `src/forecast/`). `specs/`는 "계획", `docs/`는 "결과와 운영"으로 역할을 나눈다. 실행 산출물 `var/`는 커밋하지
않는다.

## Complexity Tracking

위반 없음.
