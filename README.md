# GCP Optimize — Claude Code Skill

GCP 프로젝트의 비용 누수를 체계적으로 점검하고 최적화하는 Claude Code 스킬입니다.

## 점검 항목

| # | 항목 | 흔한 낭비 규모 |
|---|------|---------------|
| 1 | Container Registry / Artifact Registry | $10~200/월 |
| 2 | Cloud Run 설정 (CPU, 메모리, min-instances) | $50~1000/월 |
| 3 | Cloud Build | $5~50/월 |
| 4 | BigQuery (쿼리 스캔량 + 파티셔닝) | $1~500/월 |
| 5 | Cloud Logging | $5~50/월 |
| 6 | Cloud Scheduler | $1~5/월 |
| 7 | 네트워크 / 기타 | 가변 |

## 주요 기능

- **Container Registry**: 미사용 이미지 탐지 및 삭제, 자동 정리 정책(lifecycle) 설정
- **Cloud Run**: CPU throttling, min-instances, 메모리 과다 할당 진단
- **BigQuery**: 일별 쿼리량/비용 분석, 파티셔닝 규칙 가이드, 불필요한 테이블 탐지
- **Cloud Logging**: 불필요한 로그 제외 필터 설정
- **비용 리포트**: 서비스별 예상 비용 및 절감 가능액 산출

## 설치 방법

### 방법 1: 글로벌 설치 (모든 프로젝트에서 사용)

```bash
cp -r gcp-optimize ~/.claude/skills/
```

설치 후 Claude Code에서 `/gcp-optimize` 로 실행합니다.

### 방법 2: 프로젝트별 설치

```bash
cp -r gcp-optimize <your-project>/.claude/skills/
```

## 사전 요구사항

- `gcloud` CLI 설치 및 인증
- `bq` CLI (BigQuery 점검 시)
- GCP 프로젝트 접근 권한 (Viewer 이상)
- Container Registry 정리 시 `Artifact Registry Administrator` 권한

## 사용 예시

```
> /gcp-optimize

===================================
  GCP 월간 비용 점검 리포트
  프로젝트: my-project
  점검일: 2026-03-25
===================================

  [1] Container Registry
      이미지: 1개 (2.1 GB)
      상태: OK
      자동정리: 설정됨
      예상 비용: $0.21/월

  [2] Cloud Run
      서비스: 1개
      설정: 2vCPU / 4GiB / min=1 / throttle=on
      상태: 최적
      예상 비용: $64/월

  ...

  -----------------------------------
  예상 총 비용: $85/월
  절감 가능액: $0/월
  -----------------------------------
===================================
```

## 라이선스

MIT
