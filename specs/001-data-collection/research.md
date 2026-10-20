# Research: 데이터 수집 체계 (과제 1)

측정 조건: 2026-10-20 실 브로커 3분 관찰(53건), Jetson L4T R39 aarch64, Python 3.12.3.

## R1. 기준 시각을 무엇으로 할 것인가

- **Decision**: LoRaWAN은 `rxInfo[].nsTime` 중 가장 이른 값을 쓰고, 없으면 수신 시각을 쓴다. 냉난방은 `ts`(epoch ms)를
  쓰고, 없으면 수신 시각을 쓴다. 기준 시각이 수신 시각보다 2분 넘게 미래이거나 1시간 넘게 과거이면 수신 시각으로
  바꾸고 `ts_corrected=1`로 표시한다. 원천 `time`/`ts`, 수신 시각, 기준 시각은 모두 저장한다.
- **Rationale**: 실측에서 게이트웨이 `…dccc`의 `gwTime`이 약 4분 55초 빨랐다. ChirpStack `time`은 이 값을 그대로 써서
  미래 시각이 된다. 같은 메시지라도 다른 게이트웨이(`…9304`)를 거치면 정상이다. `nsTime`은 네트워크 서버 한 곳의
  시계라 일관된다. 수신 시각만 쓰면 네트워크가 밀렸다가 한꺼번에 도착할 때 간격이 무너진다(README "생각해 볼 점").
- **Alternatives**: `time` 그대로 쓰기(5분 오차, 기각). 수신 시각만 쓰기(지연 몰림에 취약, 기각). `gwTime` 보정
  (게이트웨이별 오프셋 추정이 필요해 복잡하고, `nsTime`이 있으면 불필요하므로 기각).

## R2. 수신 → 저장 경로와 무손실

- **Decision**: MQTT 콜백은 해석한 측정값을 **로컬 SQLite 스풀(WAL)**에 즉시 넣는다. 별도 기록 스레드가 2초마다
  최대 1,000건씩 MySQL에 `INSERT … ON DUPLICATE KEY UPDATE`(멱등)로 옮기고, 성공한 건만 스풀에서 지운다.
  실패하면 지수 백오프(최대 30초)로 다시 시도한다. 스풀 한도는 기본 24시간 분량(행 수 상한)이며, 넘으면 오래된
  것부터 지우고 `spool_drop` 사건을 남긴다.
- **Rationale**: 저장소 장애, 프로세스 비정상 종료, 재부팅 어느 경우에도 받은 측정값을 잃지 않는다(SC-008).
  기록을 일괄로 하면 학습·분석 읽기와 경쟁하는 짧은 쓰기 구간이 줄어든다(헌법 운영 제약).
- **Alternatives**: 메모리 큐만 쓰기(재시작 시 손실, 기각). MySQL에 직접 쓰고 실패 시에만 스풀(경로가 둘이 되어 시험하기
  어려우므로 기각).

## R3. 브로커 재접속 1분 보장

- **Decision**: paho-mqtt 2.x의 `loop_start()` 자동 재접속과 `reconnect_delay_set(min=1, max=30)`, `keepalive=30`을 쓴다.
  첫 접속 실패는 `connect_async`로 같은 재시도 경로에 태운다. 감시자가 **접속은 됐는데 5분 동안 메시지가 없으면** 강제로
  다시 접속한다. 고정 client_id와 `clean_session=False`, QoS 1로 구독해 짧은 끊김 동안 브로커에 쌓인 메시지도 받는다
  (브로커가 지원하는 범위에서).
- **Rationale**: 브로커 재시작 시 TCP가 끊기므로 즉시 감지하고, 최대 30초 간격으로 재시도하므로 복귀 후 30초 안에
  재접속된다. 조용히 죽은 연결도 keepalive 1.5배인 45초 안에 감지한다.
- **Alternatives**: 직접 만든 재접속 루프(paho 기능과 중복, 기각).

## R4. 브로커 재시작 자동 시험

- **Decision**: 보드 안에서 순수 파이썬 브로커 **amqtt 0.12**(설치·동작 확인)를 시험 고정물(fixture)로 띄운다.
  수집기가 붙은 상태에서 브로커를 종료하고 10초 뒤 같은 포트로 재시작한다. 이어 측정 메시지를 발행하고, 재시작부터 첫
  저장까지의 시간이 60초 이하인지 확인한다. 실 브로커는 우리가 끌 수 없으므로 실 환경 재접속은 `collector_event`
  기록으로 사후 확인한다.
- **Alternatives**: mosquitto 바이너리(sudo 필요, 기각). 실 브로커 앞의 TCP 프록시 차단(네트워크 끊김 시험으로는 쓸모
  있으나 실 브로커 의존이라 보조로만 사용).

## R5. 재부팅 자동 시작 (sudo 없음)

- **Decision**: systemd **사용자** 서비스 `team2-collector.service`를 쓴다(`Restart=always`, `RestartSec=5`,
  `WantedBy=default.target`). 재부팅 자동 시작은 1순위로 `loginctl enable-linger $USER`를 쓴다. polkit이 본인 계정을
  허용하면 sudo 없이 된다. 이것이 거부되면 2순위로 `crontab @reboot`에서 감시 스크립트(`deploy/run_forever.sh`)를
  실행한다. 이 스크립트는 수집기를 `nohup`으로 재실행하며 잠금 파일로 중복 실행을 막는다. systemd 사용자 서비스와 cron 감시가
  동시에 돌지 않도록 같은 잠금 파일(`flock`)을 쓴다.
- **Rationale**: `Linger=no`에서는 로그인 전에 사용자 systemd가 뜨지 않는다. `@reboot`의 `systemctl --user`는 사용자
  버스가 없어 실패하므로 2순위는 systemd를 거치지 않는다.
- **Alternatives**: 시스템 서비스(sudo 필요, 기각).

## R6. 설정 형식과 메시지 해석

- **Decision**: YAML 설정(`config/collector.yaml`)에 원천 규칙 목록을 둔다. 각 규칙은 `topic`(MQTT 와일드카드),
  `measurements`(값 위치), `device_id`, `space`, `time`(우선순위 목록) 등을 **점 경로 표현식**으로 적는다.
  표현식 문법은 `a.b.c`, 배열 전체 `a[*].b`, 집계 `min(...)`, 토픽 조각 `topic[3]`, 사전 하나 펼치기 `values.*`(키를 장치명으로)
  정도로 작게 유지한다. `pyyaml`로 읽고, 직접 만든 검증기가 경로·필수 키·정규식을 검사한다.
- **Rationale**: 실측된 두 형식(ChirpStack, xflow)을 모두 표현하면서, 새 원천도 코드 없이 추가할 수 있는 최소 문법이다.
  JSONPath 라이브러리는 의존성을 늘리고 집계·토픽 조각을 표현하기 어렵다.
- **Alternatives**: 원천별 파이썬 플러그인(코드 수정 필요, 요구사항 2 위반으로 기각). jsonpath-ng(과함, 기각).

## R7. 커버율·빠진 구간 계산

- **Decision**: 보고 도구가 기간 안 측정값의 `(space_id, item, device, FLOOR(UNIX_TIMESTAMP(ts)/60))`를 DB에서
  `SELECT DISTINCT`로 가져오고, 파이썬에서 장치별·공간항목별 커버율과 연속 빈 칸(빠진 구간)을 계산한다. 기대 주기가
  1분보다 긴 장치는 기대 칸 수를 `기간/주기`로 잡는다. 빠진 구간 원인 단서는 세 가지로 붙인다. 같은 구간에
  전 장치가 비었는지, 그 구간과 겹치는 `collector_event`, 해당 장치의 직전 수신 간격이다.
- **Rationale**: 24시간이면 약 10~20만 행 수준이라 보드에서 수 초 안에 끝난다(SC-005). 인덱스 `(ts)`, `(space_id, item, ts)`로
  범위만 읽는다.

## R8. 재전송(심화 B)

- **Decision**: LoRaWAN `fCnt`를 `frame_cnt`로 저장한다. 수집기는 최근 2시간 `(device, fCnt)`를 메모리에 두고, 이미 본
  프레임이면 저장하지 않고 `duplicate` 통계만 올린다. 재시작 뒤에는 DB 유일키 `(device_id, item, ts)`가 같은 메시지의
  재처리를 막는다.
- **Rationale**: 재전송은 `deduplicationId`와 `nsTime`이 달라져 시각 유일키로는 못 막는다. `fCnt`는 장치 재가입 시
  0으로 돌아가므로 영구 유일키로 쓰지 않고 시간 창으로 제한한다.

## R9. 용량 추정(심화 B)

- **Decision**: `report --capacity`가 `information_schema.TABLES`의 데이터·인덱스 크기와 최근 일 평균 행 수로 1년 크기를
  선형 추정한다. 제안 정책은 다음과 같다. 원본 1분 값은 90일 보관한다. 그 뒤는 5분 평균·최소·최대만 남긴다
  (`measurement_5m`). 정책 적용 시 크기도 함께 낸다. 실제 삭제 작업은 이번 범위에서 문서와 SQL로만 제공한다.

## R10. 공간 매핑

- **Decision**: 공간은 설정의 `spaces` 목록(영문 id, 한글 이름, 종류, 별칭)으로 **명시적으로 정의**한다. 장치마다 공간을
  정하는 순서는 ① 장치 지정, ② 원천 표기(LoRaWAN 태그 `location`, xflow 장치명의 `-` 앞부분)를 별칭으로 찾기, ③ `unmapped`이다.
  측정값에는 수신 시점의 `space_id`를 저장한다.
- **Rationale**: 원천마다 공간 표기가 다르다(태그 `실습실` vs 장치명 `대표실-실내기`). 태그는 현장에서 누가 바꿔도
  모른다. 이후 과제(예측·상태 추정·서비스)는 공간 단위로 동작하므로 안정된 id가 필요하다. README의 합성 데이터 공간
  (강의실·회의실·라운지)도 같은 `spaces` 목록에 넣어 실데이터와 같은 체계로 다룬다.
- **Alternatives**: 태그 값을 그대로 공간으로 쓰기(표기 흔들림·원천 간 불일치, 기각).
- **실측 매핑 초안**(2026-10-20 관찰):

| space_id | 이름 | 종류 | 별칭(원천 표기) | 장치 |
|---|---|---|---|---|
| `lab` | 실습실 | classroom | 실습실 | EM500-CO2 ×2, AM107, EM300-TH ×2, EM320-TH(…9818), WS302 |
| `office` | 사무실 | office | 사무실 | AM103, EM320-TH(…9010), xflow 실내기 `사무실` |
| `office_out` | 사무실 밖 | outdoor | 사무실 밖 | EM300-TH(…1836) |
| `ceo_room` | 대표실 | office | 대표실 | xflow 실내기·실외기 |
| `dev_room` | 개발실 | office | 개발실 | xflow 실내기·실외기 |
| `meeting` | 회의실 | meeting | 회의실 | xflow 실내기 |
| `focus_meeting` | 집중회의실 | meeting | 집중회의실 | xflow 실내기 |
| `plant` | 공용 설비 | plant | controller | xflow 제어기 |
