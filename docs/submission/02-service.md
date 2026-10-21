# 상시 서비스 — 팀 Jetson에서 가동 중인 수집기와 추론 서비스

> 제출 시각 기준 상태(2026-10-22): 세 서비스 모두 `active`. 추론 서비스는 2026-10-21 20:16부터 재시작 없이 가동 중이다(12시간 기록 08:20 종료).
> **조회 API 주소의 실제 IP는 공개 저장소에 싣지 않는다**(내부망·Tailscale 주소). 제출할 때 아래 `<팀 Jetson 주소>` 자리에 넣는다.
> 실제 주소는 보드에서 `hostname -I`(교육장 망)와 `tailscale ip -4`(외부)로 확인하거나, 보드의 `var/submission/addresses.md`(git 제외)를 본다.

## 조회 API

| 항목 | 값 |
|---|---|
| **주소** | `http://<팀 Jetson 주소>:18080` |
| 시작점 | `GET /v1/health`(서비스 상태) · `GET /v1/spaces`(공간 목록) · `GET /v1/openapi.json`(명세) |
| 대표 질의 | `GET /v1/spaces/meeting/forecast?minutes=30` — 회의실 30분 뒤 CO₂·온도 |
| 인증 | 없음(읽기 전용 GET) |
| 결과 발행(MQTT) | 같은 주소 포트 **11883**, 토픽 `team2/inference/v1/#`(보존 메시지, 새 구독자도 바로 최신값) |
| 사용법 | [결과 형식 문서](03-result-format.md) |

## 가동 중인 서비스

| 서비스 | systemd 사용자 서비스 | 역할 | 외부 포트 | 재부팅 |
|---|---|---|---|---|
| 수집기 | `wonil-collector` | 팀 MQTT(센서·냉난방) 구독 → MySQL `measurement` 저장. 형식·공간은 설정 파일로 | 없음(구독만) | 자동 시작(`enabled` + `Linger=yes`, 2026-10-20 실제 재부팅으로 확인) |
| 추론 서비스 | `wonil-inference` | 매분 공간 3곳의 60분 예측·이상·상태·규칙 경보 계산, 조회 API 제공, 결과 발행 | 18080 | 자동 시작(같은 설정) |
| 결과 브로커 | `wonil-broker` | 결과 발행용 보드 로컬 MQTT. 익명은 구독만, 발행은 서비스 계정만 | 11883 | 자동 시작 |

## 지금 상태를 확인하는 방법

| 무엇을 | 어디서 | 명령 | 정상 |
|---|---|---|---|
| 서비스가 도는가 | 보드 | `systemctl --user is-active wonil-collector wonil-inference wonil-broker` | `active` 3줄 |
| 판단이 정상인가 | 보드 | `infer status` | `상태 ok`, forecast·anomaly·state `ok` |
| 외부에서 응답하는가 | 아무 곳(망 안·Tailscale) | `curl -s http://<팀 Jetson 주소>:18080/v1/health` | `"status": "ok"` |
| 데이터가 쌓이는가 | 보드 | `collector report --hours 1` | 공간·항목별 커버율 |
| 가동 기록 | 보드 | `infer report --from "2026-10-21 20:20"` | 주기·처리 시간·경보·장애 시험 표 |

## 운영 기록 요약

| 항목 | 값 |
|---|---|
| 수집 | 2026-10-20 16:33 ~ 상시. 24시간 커버율 최저 98.6% |
| 추론 | 2026-10-21 20:16 ~ 상시. 01:30 기준 312/312 주기 정상, 주기 p50 0.8초 |
| 장애 시험 | 수집기 10분 정지 → 끊김 경보(10분 뒤) → 해제(재가동 2.5~3.5분 뒤), 주기 실패 0 |
| 다음 변경(예정) | 12시간 기록 뒤 추론 실행 방식을 ONNX Runtime CPU로 교체(과제 7 결정, 결과 동일·메모리 −0.6GB) |
