# 저장소 크기 추정과 보관 정책 (과제 1 심화 B)

- 측정 조건: 2026-10-21 09시 KST, MySQL `cf_u36_d3`, 실데이터
  - 과거 적재분: InfluxDB, 2025-08-08 ~ 2026-07-22, 약 985만 행
  - 실시간 수집: 2026-10-20 16:33부터 16.6시간
- 도구: `collector report --capacity`, 그리고 아래 SQL 측정

## 1. 지금 크기

| 테이블 | 행 | 데이터+인덱스 |
|---|---|---|
| measurement | 9,848,989 | 2,263 MB |
| 나머지 4개 | 1천여 행 | 0.1 MB |

- 행당 **241바이트**(인덱스 3개 포함 실측)다. 인덱스는 유일키(장치·항목·시각), 공간·항목·시각, 시각이다.

## 2. 1년 뒤 추정

| 시나리오 | 하루 | 1년 행 수 | 1년 크기 |
|---|---|---|---|
| A. 지금처럼 원본 전부 보관 | 99,670행 | 3,638만 | **8.2 GB** |
| B. 원본 90일 + 이후 5분 집계(평균·최소·최대·건수) | — | 1,567만 | **4.0 GB** |
| C. B + 냉난방 상태값은 바뀔 때만 저장(아래) | 약 5.3만 행 | 약 1,030만 | **약 2.9 GB**(추정) |

- C의 산식은 다음과 같다.
  - 하루 행 수: LoRa 3.4만 + xflow(40계열 × 15분 생존 기록 96회 + 변경분) 약 0.4만
  - 1년: 원본 90일분 477만 행 + 이후 275일은 집계 하루 약 2만 행(1분 계열 1/5, 10분 계열 그대로) × 1.3배 크기
  - 결과: 약 2.9GB

**처음 추정이 틀렸던 이유**: 과거 적재분의 하루 평균(2.2만 행)으로 계산했더니 1년 1.8GB가 나왔다. 그런데 이 평균에는
장기 정지 구간과 아직 적재하지 못한 89일이 섞여 있고, 냉난방(xflow) 데이터는 아예 없다. 그래서 현재 속도의 1/4로 잡혔다.
**속도는 최근 실시간 수집으로 재야 한다.** 도구도 그렇게 고쳤다.

## 3. 어디서 늘어나는가

| 원천 | 활성 계열 | 하루 행 | 비율 | 직전 값과 같은 행 |
|---|---|---|---|---|
| LoRaWAN 센서(chirpstack) | 61 | 3.4만 | 49% | 69% |
| 냉난방(xflow) | 40 | 3.5만 | **51%** | **100%**(야간, 전원 꺼짐) |

- xflow는 장치마다 상태 항목 19개를 1분마다 다시 보낸다. 전원, 모드, 오류 코드, 온라인 여부, 누적 전력량 등이다.
  대부분은 하루에 몇 번 바뀌지 않는다. 저장량의 절반이 "바뀌지 않았다"는 정보다.

## 4. 제안하는 보관 정책

| 대상 | 정책 | 이유 |
|---|---|---|
| LoRaWAN 환경 센서(CO₂·온습도 등) | **원본 90일**, 이후 5분 집계(평균·최소·최대·건수) | 예측(과제 3)은 최근 원본 해상도가 필요하다. 과거는 패턴 분석용 5분이면 충분하다. 같은 값이 반복되는 기록은 **값 고정 고장(과제 4) 탐지의 근거**이므로 원본 기간 안에서는 지우지 않는다 |
| 냉난방 상태값(xflow) | **바뀔 때만 저장 + 15분마다 생존 기록 1행** | 100%가 반복값이다. 생존 기록을 남겨야 "안 바뀜"과 "데이터 없음"을 구분할 수 있다 |
| 수집기 사건·분 통계 | 전부 보관 | 하루 1,500행 미만이고 결측 원인 설명의 근거다 |
| 과거 적재분(InfluxDB) | 같은 정책 적용 | 2026-07 이전은 바로 5분 집계 대상이다 |

### 실행 방법(아직 실행하지 않음)

5분 집계 테이블은 다음과 같이 만든다.

```sql
CREATE TABLE IF NOT EXISTS measurement_5m (
  bucket    DATETIME    NOT NULL,          -- 5분 칸 시작(UTC)
  space_id  VARCHAR(32) NOT NULL,
  device_id VARCHAR(64) NOT NULL,
  item      VARCHAR(64) NOT NULL,
  v_avg DOUBLE NOT NULL, v_min DOUBLE NOT NULL, v_max DOUBLE NOT NULL, n INT NOT NULL,
  PRIMARY KEY (device_id, item, bucket),
  KEY ix5_space_item_bucket (space_id, item, bucket)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

90일 지난 원본은 하루씩 옮기고 지운다(멱등).

```sql
INSERT INTO measurement_5m (bucket, space_id, device_id, item, v_avg, v_min, v_max, n)
SELECT FROM_UNIXTIME(FLOOR(UNIX_TIMESTAMP(ts)/300)*300), space_id, device_id, item, AVG(value), MIN(value), MAX(value), COUNT(*)
FROM measurement WHERE ts >= @day AND ts < @day + INTERVAL 1 DAY AND ts < UTC_TIMESTAMP() - INTERVAL 90 DAY
GROUP BY 1, 2, 3, 4
ON DUPLICATE KEY UPDATE v_avg=VALUES(v_avg), v_min=VALUES(v_min), v_max=VALUES(v_max), n=VALUES(n);
DELETE FROM measurement WHERE ts >= @day AND ts < @day + INTERVAL 1 DAY AND ts < UTC_TIMESTAMP() - INTERVAL 90 DAY;
```

- 집계와 삭제는 팀 합의 뒤에 하루 단위 작업으로 붙인다. 삭제는 되돌릴 수 없으므로 집계 건수를 대조한 뒤에만 한다.
- xflow "바뀔 때만 저장"은 수집기 해석 단계에 원천별 옵션으로 넣을 수 있다. 다만 과제 1 현황 보고의 커버율 정의(칸마다 값이 있는가)와
  충돌한다. 그래서 생존 기록 간격(15분)을 기대 주기로 삼도록 보고서도 함께 바꿔야 한다.

## 5. 해석

- 지금 정책 그대로면 1년에 약 8GB다. 팀 DB 용량 한도를 모르므로 **한도를 먼저 확인**해야 한다.
- 집계(B)만으로 8.2GB → 4.0GB, 냉난방 반복값 제거까지 하면(C) 약 2.9GB다. 하루 유입량은 냉난방 반복값 제거로 절반이 된다.
  환경 센서 원본은 과제 3·4의 재료이므로 90일은 지킨다.
- 측정이 16.6시간(야간 포함)뿐이라, 냉난방이 켜지는 주간에는 xflow 반복률이 100%보다 낮다. 1주일 수집 뒤 다시 잰다.
