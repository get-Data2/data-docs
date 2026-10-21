# Contract: `pretrained` CLI

실행은 `pretrained <명령>`(→ `tools/pretrained`, 어디서든) 또는 `.venv/bin/python -m pretrained <명령>`이다.
GPU 그룹 처리·환경 변수는 `tools/forecast`와 같다. `fetch` 이후 명령은 `HF_HUB_OFFLINE=1`로 실행한다.

공통 옵션: `--config config/pretrained.yaml`, `--device auto|cuda|cpu`

| 명령 | 하는 일 | 출력 | 실패(종료 코드 ≠ 0) |
|---|---|---|---|
| `check` | 입력 파일·해시, 장치, 패키지·가중치 존재, 설정 검증 | 표 | 입력 없음·해시 불일치·설정 오류 |
| `fetch` | 설정의 모델을 리비전 고정으로 받아 `var/pretrained/hf/`에 저장 | `models.json` | 네트워크 실패(이미 받은 모델은 건너뜀) |
| `predict val` | 검증 표본에서 크기 × 컨텍스트 조합 예측(심화 공변량 포함) | `preds/*__val.parquet` | 가중치 없음 |
| `select` | 검증 기준 크기·컨텍스트·앙상블 가중치 고정 | `selection.json`, 후보 표 | 검증 예측 없음, 과제 3 검증 예측 없음(앙상블만 생략하고 경고) |
| `test` | 선택된 조합(+크기별 비교, 공변량)으로 시험 예측·채점 | `preds/*__test.parquet`, 비교표 | `selection.json` 없음·변조 |
| `bench` | 모델 × 장치 × 배치 지연·메모리·전력 측정(과제 3 모델 포함) | `bench/*.json`, 표 | 다른 GPU 작업이 돌고 있으면 경고 후 계속(`--strict`면 거부) |
| `report` | 결과 표·그래프·예산표 | `docs/deliverables/task6/`, `docs/advanced/task6/` | 결과 없음 |
| `info` | 단계별 파일 상태·해시 | 표 | — |
