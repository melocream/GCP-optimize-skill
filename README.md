# GCP Optimize — Claude Code Skill

GCP 프로젝트의 비용 누수를 체계적으로 점검하고 최적화하는 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 스킬입니다.

A Claude Code skill that systematically audits and optimizes GCP project costs.

---

## 점검 항목 (Audit Scope)

| # | 서비스 | 흔한 낭비 규모 |
|---|--------|---------------|
| 1 | Container Registry / Artifact Registry | $10~200/mo |
| 2 | Cloud Run (CPU, memory, min-instances) | $50~1,000/mo |
| 3 | Cloud Build | $5~50/mo |
| 4 | BigQuery (query scan + partitioning) | $1~500/mo |
| 5 | Cloud Logging | $5~50/mo |
| 6 | Cloud Scheduler | $1~5/mo |
| 7 | Cloud Storage (lifecycle, storage class) | $5~200/mo |
| 8 | Network / Others | varies |

> **Note**: This skill is optimized for serverless / data-analytics GCP projects.
> For GCE, GKE, or other IaaS workloads, additional tooling is recommended.

## 주요 기능 (Features)

- **Container Registry**: 미사용 이미지 탐지 및 삭제, 자동 정리 정책(lifecycle) 설정
- **Cloud Run**: CPU throttling, min-instances, 메모리 과다 할당 진단 및 비용 추정
- **Cloud Build**: 무료 범위 초과 여부 확인, Docker 빌드 최적화 가이드
- **BigQuery**: 일별 쿼리량/비용 분석, 파티셔닝 규칙 가이드, 비싼 쿼리 Top 5 탐지
- **Cloud Storage**: 버킷별 사용량 확인, Lifecycle 정책 설정, 스토리지 클래스 최적화
- **Cloud Logging**: 불필요한 로그 제외 필터 설정
- **비용 리포트**: 서비스별 예상 비용 및 절감 가능액을 포맷된 리포트로 출력

## 설치 방법 (Installation)

### 방법 1: 글로벌 설치 (모든 프로젝트에서 사용)

```bash
git clone https://github.com/melocream/GCP-optimize-skill.git
mkdir -p ~/.claude/skills/gcp-optimize
cp GCP-optimize-skill/skill.md ~/.claude/skills/gcp-optimize/skill.md
```

### 방법 2: 프로젝트별 설치

```bash
git clone https://github.com/melocream/GCP-optimize-skill.git
mkdir -p <your-project>/.claude/skills/gcp-optimize
cp GCP-optimize-skill/skill.md <your-project>/.claude/skills/gcp-optimize/skill.md
```

설치 후 Claude Code에서 `/gcp-optimize`로 실행합니다.

## 사전 요구사항 (Prerequisites)

| 도구 | 용도 | 필수 여부 |
|------|------|----------|
| `gcloud` CLI | 대부분의 점검 | 필수 |
| `bq` CLI | BigQuery 점검 | BigQuery 사용 시 |
| `gsutil` | Cloud Storage 점검 | GCS 사용 시 |
| GCP 프로젝트 권한 | Viewer 이상 | 필수 |
| AR Administrator 권한 | Container Registry 정리 | 이미지 삭제 시 |
| Python 3.10+ | `gcloud artifacts` 명령 | Artifact Registry 사용 시 |

## 사용 예시 (Usage)

```
> /gcp-optimize

===================================
  GCP 월간 비용 점검 리포트
  프로젝트: my-project
  점검일: 2026-03-25
===================================

  [1] Container Registry
      이미지: 3개 (4.2 GB)
      상태: 정리 필요
      자동정리: 미설정
      예상 비용: $0.42/월

  [2] Cloud Run
      서비스: 2개
      설정: 2vCPU / 4GiB / min=1 / throttle=on
      상태: 최적
      예상 비용: $64/월

  [3] Cloud Build
      빌드: 45회/월 | 총 180분
      상태: 무료 범위
      예상 비용: $0/월

  [4] BigQuery
      쿼리: 120 GB/월 | 스토리지: 8 GB
      상태: 무료 범위
      예상 비용: $0/월

  [5] Cloud Logging
      로그 제외 필터: 미설정
      예상 비용: $3/월

  [6] Cloud Scheduler
      활성 Job: 5개 | 실패: 1개
      예상 비용: $0.20/월

  [7] Cloud Storage
      버킷: 3개 | 총 용량: 25 GB
      Lifecycle 정책: 미설정
      예상 비용: $0.50/월

  -----------------------------------
  예상 총 비용: $68.12/월
  절감 가능액: $12.50/월
  -----------------------------------

  조치 필요:
    1. Container Registry untagged 이미지 정리 + 자동정리 정책 설정
    2. Cloud Logging 제외 필터 설정 (health check, DEBUG)
    3. Cloud Scheduler 실패 Job 1개 점검
    4. Cloud Storage Lifecycle 정책 설정

  다음 점검: 2026-04-25
===================================
```

## 라이선스 (License)

MIT
