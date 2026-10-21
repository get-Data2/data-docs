# Implementation Plan: 데이터 이해와 기준선 (과제 2)

**Branch**: `002-data-baseline` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-data-baseline/spec.md`

## Summary

과제 1 저장소의 실데이터(2025-08 ~)를 **보드에 Parquet 사본**(저장만 달별 파일로 나눔)으로 한 번 떠 두고, 그 사본만으로 반복 분석한다.
학습 기간은 약 7개월(`v1-provisional`)~8개월(`v2-final`)이다. 단계는 다음과 같다.

1. **정제**: 두 축을 분리해 판정한다.
   - 축 A 센서 오류: 기종별 오류 코드·제원 범위
   - 축 B 실제 사건: 공간·시간 패턴
   - 결측은 과제 1과 같은 "자기 주기 2배" 정의로 판정하고, 15분 이하만 보간한다.
2. **공간 시계열**: 5분 칸, 센서 중앙값으로 만든다.
3. **분할**: 버전으로 고정한다.
   - `v1-provisional`: 지금 데이터로 바로 가능
   - `v2-final`: InfluxDB 미적재분을 적재한 뒤 2학기 시험
4. **진단서**: 결측·이상값·평일 패턴·일과표 대비 CO₂ 반응 지연·어긋나는 날·자기상관을 담는다.
5. **기준선 3종**: 5·15·30·60분 MAE를 낸다.

평가 시점과 정답 파일은 과제 3~6이 그대로 쓴다. 근거는 [research](research.md)에 있다.

## Technical Context

**Language/Version**: Python 3.12.3 (`.venv`)

**Primary Dependencies**: pandas 3.0, pyarrow 26, matplotlib 3.11(Agg, Noto Sans CJK KR), holidays 0.107(KR), PyYAML, PyMySQL(과제 1 `collector.store` 재사용). 모두 보드에 설치·임포트 확인함

**Storage**: 입력은 MySQL `cf_u36_d3.measurement`(읽기만), 작업 파일은 `var/cache`, `var/series`, `var/eval`(Parquet·JSON, git 제외), 결과 문서는 `docs/reports/task2/`

**Testing**: pytest. 새 마커 없이 `unit`(합성 데이터)을 쓰고, 실 DB 추출 시험은 `mysql`

**Target Platform**: Jetson aarch64, 오프라인 일괄 실행(서비스 아님)

**Project Type**: 분석 라이브러리 + CLI(`src/analysis/`)

**Performance Goals**: 14개월분 전체(`analysis all`) 10분 이내(SC-001), 추출 뒤 재실행은 몇 분 안

**Constraints**: 메모리 1GB 이하(보드 RAM 7.3GB, 수집기·향후 모델과 공존), 시험 기간 정보 누수 0, 원본 행 삭제 0

**Scale/Scope**: 원본 약 630만 행(세 항목) → 5분 공간 시계열 약 3공간 × 3항목 × 12만 칸

## Constitution Check

*GATE: Phase 0 전 통과, Phase 1 후 재확인.*

| 원칙 | 확인 | 결과 |
|---|---|---|
| I. 문서·비밀 | 문서 한국어, DB 접속은 `.env`(python-dotenv), 출력에 비밀 없음 | ✅ |
| II. 엣지 제약 | `.venv`+uv, 메모리 1GB 상한 목표, 월 단위 처리 | ✅ |
| III. 설정 주도 | 정제·분할·일과표·임계값을 `config/analysis.yaml`로, 기대 주기는 과제 1 설정 재사용 | ✅ |
| IV. 테스트 | 확인 기준마다 자동 시험(분할 누수, 같은 시점, 검산, 축 분리) | ✅ |
| V. 공정한 평가 | 정규화·데이터 기반 임계값은 학습 기간만, 평가 시점 파일 고정·해시, 표마다 측정 조건 | ✅ |
| VI. 복구 | 일괄 작업 — 단계별 산출물이 있어 중간부터 재실행 가능, 닫힌 달 재추출 안 함 | ✅ |
| VII. 해석 우선 | 진단서·기준선 표마다 "왜·무엇을" 해석 절, 산출물 문서에 확인 명령 | ✅ |

Phase 1 뒤 재확인: 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/002-data-baseline/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md
├── contracts/ (config.md, cli.md)
├── checklists/requirements.md
└── tasks.md   (/speckit-tasks)
```

### Source Code (repository root)

```text
src/analysis/
├── __main__.py      CLI: extract · clean · events · split · diagnose · baseline · all · info
├── config.py        config/analysis.yaml 읽기·검증(+ 과제 1 장치 주기 읽기)
├── extract.py       MySQL → 월별 Parquet 사본, manifest
├── quality.py       축 A(오류 코드·쌍·범위·포화)
├── gaps.py          결측 판정·짧은 결측 보간·인위 결측 시험
├── series.py        센서 5분 칸 → 공간 중앙값 시계열
├── events.py        축 B(화재·누수·밀집·단독 극단값)·값 고정 후보
├── split.py         분할 검증·학습 기간 정규화 통계
├── schedule.py      일과표·공휴일·반응 지연·어긋나는 날
├── diagnose.py      진단 표·그래프·진단서 Markdown
└── baseline.py      평가 시점·기준선 3종·MAE 표
config/analysis.yaml
tools/analysis       어디서든 실행용
tests/analysis/      test_quality · test_axes · test_gaps · test_series · test_split · test_baseline · test_schedule
docs/reports/task2/  진단서·기준선 표·그래프(PNG)
docs/tasks/task2-data-baseline.md   산출물 문서(확인 명령 포함)
```

**Structure Decision**: 과제 1 `src/collector/`와 나란히 `src/analysis/` 패키지를 둔다. DB 접속은 `collector.store.connect`를 재사용한다.
`pyproject.toml`에 `analysis` extra(pandas·pyarrow·matplotlib·holidays)를 추가한다. 수집 서비스는 이 의존성을 쓰지 않아 영향이 없다.

## Complexity Tracking

위반 없음.
