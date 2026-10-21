# 산출물

과제 안내서(README)의 각 과제 **"산출물"** 항목에 해당하는 문서만 모은 폴더다. 최종 결과 보고서는 이 폴더의 문서를 묶어 만든다.

- 해결할 것·확인 기준·생각해 볼 점·심화(B)의 항목별 대응과 확인 명령은 [과제별 점검 문서](../tasks/)에 있다.
- 운영 가이드, 측정 기록, 심화 결과는 [docs 첫 화면](../README.md)의 "그 밖의 문서"에 있다.

| 과제 | README 산출물 | 문서 | 상태 |
|---|---|---|---|
| 1. 데이터 수집 체계 | 수집 현황 보고(공간·항목별 커버율 표, 결측 구간과 추정 원인) | [task1/01-coverage-report.md](task1/01-coverage-report.md)(24시간 최종) · [중간본(해석 포함)](task1/01-coverage-report-interim.md) | ✅ 24시간 최저 98.6% |
| | 메시지 형식 변경 상황 가정과 대응 기록 | [task1/02-format-change.md](task1/02-format-change.md) | ✅ |
| 2. 데이터 이해와 기준선 | 데이터 진단서(정제 규칙, 분할 기준, 패턴 그래프, 일정–CO₂ 반응 분석) | [task2/01-data-diagnosis.md](task2/01-data-diagnosis.md) | ✅ 그래프 3개(`task2/figures/`) |
| | 기준선 성능표 | [task2/baseline_v1-provisional.md](task2/baseline_v1-provisional.md) | ✅ `analysis baseline`이 생성 |
| 3. 60분 예측 | 모델 비교표 | [task3/01-model-comparison.md](task3/01-model-comparison.md) | ✅ 실습실 CO₂ 60분 53% 개선 |
| | 공간별 오차 분석과 원인 | [task3/02-space-error-analysis.md](task3/02-space-error-analysis.md) | ✅ |
| | 일정 정보 포함·제외 비교와 결론 | [task3/03-schedule-value.md](task3/03-schedule-value.md) | ✅ 3.3ppm(8%) 감소 |
| 4. 이상 탐지 | 탐지 성적표 / 임계값–탐지율·헛경보 표·그래프 / 운영 제안 | `task4/` | ⚪ |
| 5. 상태 추정 | 두 분류기 성적표·혼동 행렬 / 일정 기반 정답과 직접 관찰의 차이 / 많이 틀리는 상황 사례 | `task5/` | ⚪ |
| 6. 사전학습 모델 비교 | 비교표와 채택 의견(반 쪽) | `task6/` | ⚪ |
| 7. 엣지 최적화 | 최적화 보고서(실행 방식 × 모델별 지연·오차·메모리·전력, 결론) | `task7/` | ⚪ |
| 8. 상시 추론 서비스 | 예측 서비스 v1 | [task8/01-service-v1.md](task8/01-service-v1.md) | ✅ 상시 가동 중 |
| | 12시간 가동 로그 요약 | [task8/02-12h-run-summary.md](task8/02-12h-run-summary.md) | 🟡 장애 시험 완료, 12시간 표 10-22 08:25 |
| | 결과 형식 문서 | [task8/03-result-format.md](task8/03-result-format.md) | ✅ |

파일 이름 규칙: `taskN/NN-이름.md`. 번호는 README 산출물 목록의 순서다. 프로그램이 만드는 표는 생성 이름을 그대로 쓴다(예: `baseline_<분할>.md`).
