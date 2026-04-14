---
name: gcp-optimize
description: >
  GCP(Google Cloud Platform) 리소스 비용을 점검하고 최적화합니다.
  Cloud Run, Artifact Registry, BigQuery, Cloud Build, Cloud Storage, Logging 등
  주요 서비스의 낭비 요소를 찾아내고 자동 정리 정책을 설정합니다.
  월 1회 또는 비용 이상 감지 시 실행하세요.
allowed-tools: Bash, Read, Write, Edit
---

# GCP 운영 최적화 스킬

> 클라우드 비용의 80%는 20%의 리소스 낭비에서 발생합니다.
> 이 스킬은 GCP 프로젝트의 비용 누수를 체계적으로 점검하고 최적화합니다.

---

## 사전 설정

스킬 실행 전 프로젝트 정보를 확인합니다.

```bash
# 1. gcloud 경로 확인
GCLOUD=$(which gcloud 2>/dev/null || echo "$HOME/google-cloud-sdk/bin/gcloud")

# 2. 현재 프로젝트 확인
PROJECT=$($GCLOUD config get-value project 2>/dev/null)
echo "Project: $PROJECT"

# 3. 활성 리전 확인
$GCLOUD run services list --format="table(SERVICE,REGION)" --limit=5 2>/dev/null

# 4. Python 호환성 확인 (gcloud artifacts 명령에 Python 3.10+ 필요)
python3 --version
# Python 3.9 이하일 경우, 시스템에 설치된 3.10+ 경로를 찾아 설정:
# CLOUDSDK_PYTHON=$(which python3.12 || which python3.11 || which python3.10) 을 gcloud 앞에 붙여서 실행
```

---

## 체크 순서 (비용 영향도 순)

| 순서 | 항목 | 흔한 낭비 규모 | 난이도 |
|------|------|---------------|--------|
| 1 | Container Registry | $10~200/월 | 쉬움 |
| 2 | Cloud Run 설정 | $50~1000/월 | 중간 |
| 3 | Cloud Build | $5~50/월 | 쉬움 |
| 4 | BigQuery | $1~500/월 | 중간 |
| 5 | Cloud Logging | $5~50/월 | 쉬움 |
| 6 | Cloud Scheduler | $1~5/월 | 쉬움 |
| 7 | Cloud Storage | $5~200/월 | 쉬움 |
| 8 | 네트워크 / 기타 | 가변 | 어려움 |

> **참고**: 이 스킬은 서버리스/데이터 분석 중심 GCP 프로젝트에 최적화되어 있습니다.
> GCE, GKE 등 IaaS/컨테이너 오케스트레이션은 별도 점검이 필요합니다.

---

## 1. Container Registry / Artifact Registry 점검

**가장 흔한 비용 누수 원인.** Cloud Build로 빌드할 때마다 ~0.5~2GB 이미지가 쌓입니다.
월 30회 빌드하면 연간 200~700GB가 누적되어 **$20~70/월** 스토리지 비용이 발생합니다.

### 1-1. 현황 파악

```bash
# Artifact Registry 전체 저장소 목록 + 크기
$GCLOUD artifacts repositories list --project=$PROJECT \
  --format="table(REPOSITORY,LOCATION,SIZE)" 2>/dev/null

# gcr.io 이미지 수 확인
$GCLOUD container images list --repository=gcr.io/$PROJECT 2>/dev/null | while read repo; do
  count=$($GCLOUD container images list-tags "$repo" --format="get(digest)" --limit=999 2>/dev/null | wc -l)
  echo "$repo: ${count}개"
done
```

### 1-2. 진단 기준

| 상태 | 기준 | 조치 |
|------|------|------|
| 정상 | tagged 1~3개, untagged 0~5개 | 없음 |
| 주의 | untagged 10~50개 | 수동 정리 |
| 위험 | untagged 50개 이상 또는 총 50GB+ | 즉시 정리 + 자동정책 |

### 1-3. 정리 실행

```bash
# 방법 A: gcr.io 이미지 정리 (latest만 유지)
# Python 3.10+ 필요 시: CLOUDSDK_PYTHON=$(which python3.12 || which python3.11) 앞에 붙이기

# Step 1: untagged 이미지 목록 확인
$GCLOUD container images list-tags gcr.io/$PROJECT/<IMAGE_NAME> \
  --filter="NOT tags:latest" --format="get(digest)" --limit=999 2>/dev/null | wc -l

# Step 2: 삭제 실행
$GCLOUD container images list-tags gcr.io/$PROJECT/<IMAGE_NAME> \
  --filter="NOT tags:latest" --format="get(digest)" --limit=999 2>/dev/null \
  | while read digest; do
    $GCLOUD container images delete \
      "gcr.io/$PROJECT/<IMAGE_NAME>@sha256:$digest" \
      --force-delete-tags --quiet 2>/dev/null
  done

# 방법 B: Artifact Registry API 사용 (gcr.io가 AR로 마이그레이션된 경우)
CLOUDSDK_PYTHON=$(which python3.12 || which python3.11 || which python3.10) \
  $GCLOUD artifacts docker images list \
    us-docker.pkg.dev/$PROJECT/gcr.io/<IMAGE_NAME> \
    --include-tags --filter="NOT tags:*" \
    --format="csv[no-heading](version)" 2>/dev/null \
  | while read version; do
    CLOUDSDK_PYTHON=$(which python3.12 || which python3.11 || which python3.10) \
      $GCLOUD artifacts docker images delete \
        "us-docker.pkg.dev/$PROJECT/gcr.io/<IMAGE_NAME>@$version" \
        --quiet --delete-tags 2>/dev/null
  done

# 방법 C: 사용하지 않는 레포지토리 통째로 삭제
$GCLOUD artifacts repositories delete <REPO_NAME> \
  --project=$PROJECT --location=<LOCATION> --quiet
```

### 1-4. 자동 정리 정책 설정 (1회만 설정하면 영구 적용)

```bash
cat > /tmp/ar-cleanup-policy.json << 'EOF'
[
  {
    "name": "delete-old-untagged",
    "action": {"type": "Delete"},
    "condition": {
      "tagState": "untagged",
      "olderThan": "604800s"
    }
  }
]
EOF

# gcr.io 레포지토리에 적용
$GCLOUD artifacts repositories set-cleanup-policies gcr.io \
  --project=$PROJECT --location=us \
  --policy=/tmp/ar-cleanup-policy.json --no-dry-run

# 다른 레포지토리가 있다면 각각 적용
# $GCLOUD artifacts repositories set-cleanup-policies <REPO> \
#   --project=$PROJECT --location=<LOCATION> \
#   --policy=/tmp/ar-cleanup-policy.json --no-dry-run
```

| 정책 | 대상 | 보존 기간 | 효과 |
|------|------|----------|------|
| delete-old-untagged | 태그 없는 이미지 | 7일 | 빌드 잔여물 자동 삭제 |
| (optional) keep-recent | 모든 이미지 | 90일 | 오래된 태그 이미지도 정리 |

---

## 2. Cloud Run 설정 점검

Cloud Run은 설정 하나로 월 $50↔$1000 차이가 납니다.

### 2-1. 현재 설정 확인

```bash
# 모든 Cloud Run 서비스 목록
$GCLOUD run services list --project=$PROJECT \
  --format="table(SERVICE,REGION,LAST_DEPLOYED_AT)" 2>/dev/null

# 서비스별 상세 설정
$GCLOUD run services describe <SERVICE_NAME> \
  --project=$PROJECT --region=<REGION> \
  --format="yaml(spec.template.spec.containers[0].resources,spec.template.metadata.annotations)" \
  2>/dev/null
```

### 2-2. 핵심 설정 체크포인트

| 설정 | 권장값 | 잘못 설정 시 비용 |
|------|--------|------------------|
| `cpu-throttling` | `true` | false면 유휴 시에도 CPU 과금 (월 +$500~$1000) |
| `min-instances` | `0~1` | 2+면 메모리 비용 x2, x3... |
| `max-instances` | `5~10` | 100이면 트래픽 스파이크에 폭탄 |
| CPU | 작업에 맞게 | 과다 할당 시 낭비 |
| Memory | 작업에 맞게 | 과다 할당 시 낭비 |
| `startup-cpu-boost` | `true` | 콜드스타트 시간 단축, 약간의 추가 비용 |

### 2-3. 비용 추정 공식

```
# 유휴 비용 (min-instances > 0, cpu-throttling=true 기준)
idle_cost = min_instances × memory_gib × 2,592,000초 × $0.00000250/GiB-초

# 예: min-instances=1, 4GiB → $25.92/월 (메모리만)

# 활성 비용
active_cost = avg_concurrent × cpu × active_seconds × $0.00002400/vCPU-초
            + avg_concurrent × memory_gib × active_seconds × $0.00000250/GiB-초

# cpu-throttling=false면 CPU도 24/7 과금:
always_on_cpu = min_instances × cpu × 2,592,000초 × $0.00002400 = $62.21/vCPU/월
```

### 2-4. 리비전 정리

```bash
# 사용하지 않는 리비전 확인
$GCLOUD run revisions list --service=<SERVICE_NAME> \
  --project=$PROJECT --region=<REGION> \
  --format="table(REVISION,ACTIVE,LAST_DEPLOYED_AT)" --limit=20 2>/dev/null

# 트래픽 0%인 오래된 리비전 삭제
$GCLOUD run revisions delete <REVISION_NAME> \
  --project=$PROJECT --region=<REGION> --quiet
```

### 2-5. 최적화 권장사항

| 상황 | 권장 |
|------|------|
| API 서버 (항상 응답 필요) | min=1, cpu-throttling=true |
| 배치 작업 (스케줄러로 트리거) | min=0 가능 (콜드스타트 허용 시) |
| 트래픽 적은 서비스 | min=0, concurrency 높게 |
| ML 모델 서빙 | min=1 (모델 로드 시간), memory 충분히 |

---

## 3. Cloud Build 점검

### 3-1. 현황 확인

```bash
# 최근 30일 빌드 횟수 + 시간
$GCLOUD builds list --project=$PROJECT --region=global \
  --filter="createTime>='$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d)'" \
  --format="table(createTime.date(),duration,status)" --limit=30 2>/dev/null
```

### 3-2. 무료 범위

- **120분/일** 무료 (e1-medium 기준)
- 초과 시 **$0.003/빌드분**
- 하루 2회 × 12분 빌드 = 24분 → 무료 범위 내

### 3-3. 최적화 팁

| 방법 | 효과 |
|------|------|
| Docker 레이어 캐시 활용 | 빌드 시간 50~80% 단축 |
| `.dockerignore` 최적화 | 빌드 컨텍스트 크기 감소 |
| Multi-stage 빌드 | 최종 이미지 크기 감소 → AR 스토리지 절약 |
| 불필요한 재빌드 방지 | CI 조건부 트리거 설정 |

---

## 4. BigQuery 점검

### 4-1. 사용량 확인

```bash
# 최근 30일 일별 쿼리량
bq query --use_legacy_sql=false --project_id=$PROJECT \
  "SELECT
    DATE(creation_time) as dt,
    COUNT(*) as queries,
    ROUND(SUM(total_bytes_billed)/1e9, 2) as gb_billed,
    ROUND(SUM(total_bytes_billed)/1e12 * 6.25, 4) as est_cost_usd
   FROM \`region-US\`.INFORMATION_SCHEMA.JOBS
   WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   AND job_type = 'QUERY'
   GROUP BY dt ORDER BY dt DESC LIMIT 10"

# 가장 비싼 쿼리 Top 5
bq query --use_legacy_sql=false --project_id=$PROJECT \
  "SELECT
    ROUND(total_bytes_billed/1e9, 2) as gb,
    ROUND(total_bytes_billed/1e12 * 6.25, 4) as cost_usd,
    SUBSTR(query, 1, 120) as query_preview
   FROM \`region-US\`.INFORMATION_SCHEMA.JOBS
   WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
   AND job_type = 'QUERY' AND total_bytes_billed > 0
   ORDER BY total_bytes_billed DESC LIMIT 5"

# 테이블별 스토리지
bq query --use_legacy_sql=false --project_id=$PROJECT \
  "SELECT
    table_name,
    ROUND(total_rows/1e6, 2) as rows_M,
    ROUND(total_logical_bytes/1e9, 2) as gb
   FROM \`<DATASET>\`.INFORMATION_SCHEMA.TABLE_STORAGE
   ORDER BY total_logical_bytes DESC"
```

### 4-2. 비용 기준

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| On-demand 쿼리 | $6.25/TB 스캔 | 첫 1TB/월 무료 |
| Active 스토리지 | $0.02/GB/월 | 첫 10GB 무료 |
| Long-term 스토리지 | $0.01/GB/월 | 90일 미수정 테이블 자동 전환 |

### 4-3. 파티셔닝 규칙 (가장 중요!)

BigQuery 파티셔닝은 쿼리 비용의 **90% 이상**을 절감할 수 있는 핵심 최적화입니다.
UPDATE/SELECT 모든 쿼리에 파티션 컬럼을 WHERE 절에 반드시 포함해야 합니다.

#### 파티셔닝 적용 대상 기준
- 테이블 크기 **100MB 이상** 또는 **100만행 이상**
- 날짜/시간 컬럼이 있는 테이블

#### 쿼리 작성 필수 규칙

```sql
-- ✅ 올바른 예: 파티션 컬럼(created_at)이 WHERE에 포함
SELECT * FROM `project.dataset.orders`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND status = 'pending'

-- ❌ 잘못된 예: 파티션이 아닌 컬럼(updated_at)만 사용
SELECT * FROM `project.dataset.orders`
WHERE updated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)

-- ✅ UPDATE에도 파티션 필터 필수
UPDATE `project.dataset.orders`
SET status = 'completed'
WHERE order_id = 'xxx'
  AND created_at >= TIMESTAMP('2026-03-01')
  AND created_at < TIMESTAMP('2026-03-02')

-- ❌ UPDATE에 파티션 필터 없음 → 풀 테이블 스캔
UPDATE `project.dataset.orders`
SET status = 'completed'
WHERE order_id = 'xxx'
```

#### JOIN 시 양쪽 테이블 모두 파티션 필터 적용

```sql
-- ✅ JOIN 양쪽에 파티션 필터
FROM `ds.orders` o
JOIN `ds.order_items` i ON o.order_id = i.order_id
WHERE o.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND i.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
```

#### 기존 테이블에 파티셔닝 추가하는 방법

BigQuery는 기존 테이블에 파티셔닝을 추가할 수 없으므로 테이블을 새로 만들어야 합니다.

```sql
-- Step 1: 파티셔닝된 새 테이블 생성
CREATE TABLE `project.dataset.table_new`
PARTITION BY DATE(created_at)
CLUSTER BY user_id
AS SELECT * FROM `project.dataset.table_old`;

-- Step 2: 행 수 검증
SELECT COUNT(*) FROM `project.dataset.table_new`;
SELECT COUNT(*) FROM `project.dataset.table_old`;

-- Step 3: 이름 교체
ALTER TABLE `project.dataset.table_old` RENAME TO `table_backup`;
ALTER TABLE `project.dataset.table_new` RENAME TO `table`;

-- Step 4: 검증 후 백업 삭제
DROP TABLE `project.dataset.table_backup`;
```

#### 파티셔닝 적용 체크리스트

1. 새 테이블 생성 시: 반드시 `PARTITION BY` 포함
2. 새 쿼리 작성 시: 파티션 컬럼을 WHERE에 포함
3. 코드 리뷰 시: 파티션 없는 쿼리 발견하면 즉시 수정
4. 월간 점검: `INFORMATION_SCHEMA.JOBS`에서 대용량 쿼리 Top 5 확인

### 4-4. 일반 최적화 팁

- **SELECT \* 금지**: 필요한 컬럼만 선택
- **LIMIT 활용**: 탐색적 쿼리에 LIMIT 추가
- **불필요한 테이블 삭제**: 임시/실험/staging 테이블 정리
- **사용하지 않는 API 비활성화**: Gemini Cloud Assist 등 불필요한 API가 활성화되어 있으면 비활성화

---

## 5. Cloud Logging 점검

Cloud Logging은 **수집 볼륨에 따라 과금**됩니다. 불필요한 로그가 쌓이면 월 $10~100 낭비.

### 5-1. 현황 확인

```bash
# 로그 사용량 메트릭
$GCLOUD logging read "timestamp>=\"$(date -v-1d +%Y-%m-%dT00:00:00Z 2>/dev/null || date -d '1 day ago' +%Y-%m-%dT00:00:00Z)\"" \
  --project=$PROJECT --limit=1 --format=json 2>/dev/null | head -5
```

### 5-2. 비용 기준

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| 로그 수집 | $0.50/GB | 첫 50GB/월 무료 |
| 로그 보관 (30일 초과) | $0.01/GB/월 | 기본 30일 보관 무료 |

### 5-3. 최적화: 로그 제외 필터

```bash
# 불필요한 DEBUG/stdout verbose 로그 제외
# 주의: 이 명령은 기존 _Default 싱크를 수정합니다
$GCLOUD logging sinks update _Default \
  --project=$PROJECT \
  --log-filter='NOT (resource.type="cloud_run_revision" AND severity="DEBUG")
AND NOT (resource.type="cloud_run_revision" AND jsonPayload.message=~"^GET /health")'
```

| 제외 대상 | 효과 |
|----------|------|
| DEBUG 로그 | 전체 볼륨의 30~60% 절감 |
| Health check 로그 | 반복 로그 제거 |
| stdout verbose | 프레임워크 기본 출력 제거 |

---

## 6. Cloud Scheduler 점검

### 6-1. 현황 확인

```bash
$GCLOUD scheduler jobs list --project=$PROJECT --location=<REGION> \
  --format="table(ID,SCHEDULE,STATE,LAST_ATTEMPT_TIME,LAST_ATTEMPT_STATUS)" 2>/dev/null
```

### 6-2. 비용 기준

- **3개 무료**, 이후 **$0.10/job/월**
- 15개 job = ($15 - 3) × $0.10 = **$1.20/월** (미미함)

### 6-3. 체크포인트

| 확인 사항 | 조치 |
|----------|------|
| FAILED 반복 job | 원인 파악 후 수정 또는 PAUSE |
| 과다 빈도 job | 30분→1시간 등 조정 검토 |
| 더 이상 불필요한 job | PAUSE 또는 삭제 |

---

## 7. Cloud Storage 점검

GCS 버킷은 방치되기 쉬운 비용 항목입니다. 임시 파일, 오래된 백업, 불필요한 버전이 누적됩니다.

### 7-1. 현황 확인

```bash
# 모든 버킷 목록 + 위치
$GCLOUD storage buckets list --project=$PROJECT \
  --format="table(name,location,storageClass,timeCreated.date())" 2>/dev/null

# 버킷별 사용량 (gsutil 필요)
gsutil du -s gs://<BUCKET_NAME> 2>/dev/null
```

### 7-2. 비용 기준

| 항목 | Standard | Nearline | Coldline | Archive |
|------|----------|----------|----------|---------|
| 스토리지 | $0.020/GB/월 | $0.010/GB/월 | $0.004/GB/월 | $0.0012/GB/월 |
| 읽기 | $0.004/1만건 | $0.01/1만건 | $0.05/1만건 | $0.50/1만건 |
| 최소 보관 | 없음 | 30일 | 90일 | 365일 |

### 7-3. 최적화

```bash
# Lifecycle 정책: 90일 이후 Nearline으로 전환, 365일 이후 삭제
cat > /tmp/gcs-lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 90}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
EOF

gsutil lifecycle set /tmp/gcs-lifecycle.json gs://<BUCKET_NAME>

# 버전 관리 비활성화 (불필요 시)
gsutil versioning set off gs://<BUCKET_NAME>
```

| 조치 | 대상 | 효과 |
|------|------|------|
| Lifecycle 정책 | 로그, 백업, 임시 파일 버킷 | 자동 전환/삭제로 스토리지 절감 |
| 스토리지 클래스 변경 | 접근 빈도 낮은 데이터 | Nearline/Coldline으로 60~80% 절감 |
| 불필요 버킷 삭제 | 실험/staging 버킷 | 즉시 비용 제거 |

---

## 8. 네트워크 및 기타

### 8-1. Network Egress (데이터 전송)

| 구간 | 비용 |
|------|------|
| 같은 리전 내 | 무료 |
| 리전 간 | $0.01/GB |
| 인터넷으로 | $0.12/GB (첫 1TB), $0.085/GB (이후) |

**팁**: Cloud Run과 BigQuery를 같은 리전에 배치하면 내부 트래픽 무료

### 8-2. Secret Manager

- **6개 시크릿 버전 무료**, 이후 $0.06/시크릿 버전/월
- 10,000 액세스 무료, 이후 $0.03/10,000 액세스

### 8-3. Firebase / Identity Platform

| 항목 | 무료 범위 |
|------|----------|
| Authentication | MAU 50,000명 무료 |
| Hosting | 10GB 스토리지 + 360MB/일 전송 |
| Firestore | 1GB + 50,000 읽기/20,000 쓰기/일 |

---

## 비용 보고서 출력 형식

점검 완료 후 아래 형식으로 요약합니다:

```
===================================
  GCP 월간 비용 점검 리포트
  프로젝트: {PROJECT}
  점검일: {DATE}
===================================

  [1] Container Registry
      이미지: {n}개 ({size} GB)
      상태: {OK / 정리 필요 / 정리 완료}
      자동정리: {설정됨 / 미설정}
      예상 비용: ${cost}/월

  [2] Cloud Run
      서비스: {n}개
      설정: {cpu}vCPU / {mem}GiB / min={min} / throttle={on/off}
      상태: {최적 / 조정 필요}
      예상 비용: ${cost}/월

  [3] Cloud Build
      빌드: {n}회/월 | 총 {min}분
      상태: {무료 범위 / 초과}
      예상 비용: ${cost}/월

  [4] BigQuery
      쿼리: {gb} GB/월 | 스토리지: {storage_gb} GB
      상태: {무료 범위 / 과금 중}
      예상 비용: ${cost}/월

  [5] Cloud Logging
      로그 제외 필터: {설정됨 / 미설정}
      예상 비용: ${cost}/월

  [6] Cloud Scheduler
      활성 Job: {n}개 | 실패: {failed}개
      예상 비용: ${cost}/월

  [7] Cloud Storage
      버킷: {n}개 | 총 용량: {size} GB
      Lifecycle 정책: {설정됨 / 미설정}
      예상 비용: ${cost}/월

  -----------------------------------
  예상 총 비용: ${total}/월
  절감 가능액: ${savings}/월
  -----------------------------------

  조치 필요:
    1. {action item}
    2. {action item}

  다음 점검: {next_date}
===================================
```

---

## 비용 단가 참조표 (최신 가격은 [GCP Pricing](https://cloud.google.com/pricing) 참조)

### Cloud Run (Gen2)

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| vCPU | $0.00002400/vCPU-초 | 180,000 vCPU-초/월 |
| 메모리 | $0.00000250/GiB-초 | 360,000 GiB-초/월 |
| 요청 | $0.40/백만 요청 | 2백만 요청/월 |

### Artifact Registry

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| 스토리지 | $0.10/GB/월 | 0.5GB |
| 네트워크 (같은 리전) | 무료 | - |
| 네트워크 (인터넷) | $0.12/GB | 첫 1GB |

### BigQuery

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| On-demand 쿼리 | $6.25/TB | 1TB/월 |
| Active 스토리지 | $0.02/GB/월 | 10GB |
| Long-term 스토리지 | $0.01/GB/월 | - |
| Streaming Insert | $0.01/200MB | - |
| Storage API 읽기 | $1.10/TB | - |

### Cloud Storage

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| Standard 스토리지 | $0.020/GB/월 | 5GB |
| Nearline 스토리지 | $0.010/GB/월 | - |
| Coldline 스토리지 | $0.004/GB/월 | - |
| Archive 스토리지 | $0.0012/GB/월 | - |
| Class A 작업 (쓰기) | $0.005/1만건 | 5,000건 |
| Class B 작업 (읽기) | $0.0004/1만건 | 50,000건 |

### Cloud Build

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| 빌드 분 (e1-medium) | $0.003/분 | 120분/일 |
| 빌드 분 (e1-highcpu) | $0.006/분 | - |
| 빌드 분 (e2-highcpu-32) | $0.048/분 | - |

### Cloud Logging

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| 수집 | $0.50/GB | 50GB/월 |
| 보관 (30일 초과) | $0.01/GB/월 | - |

### Cloud Scheduler

| 항목 | 단가 | 무료 범위 |
|------|------|----------|
| Job | $0.10/job/월 | 3개 |

---

## 자주 묻는 질문 (FAQ)

**Q: Container Registry 정리 시 프로덕션에 영향이 있나요?**
A: `latest` 태그 이미지만 유지하면 됩니다. Cloud Run은 현재 배포된 리비전의 이미지를 자체 캐시합니다. untagged 이미지는 이전 빌드의 잔여물이므로 안전하게 삭제 가능합니다.

**Q: cpu-throttling을 true로 바꾸면 성능이 떨어지나요?**
A: 요청 처리 중에는 동일한 CPU를 받습니다. 유휴 시에만 CPU가 회수됩니다. 대부분의 웹/API/배치 서비스에서는 차이 없습니다. 백그라운드 연산이 필요한 경우만 false가 필요합니다.

**Q: min-instances=0으로 바꾸면 얼마나 절약되나요?**
A: 4GiB 메모리 기준 약 $26/월 절약. 다만 콜드스타트 시 10~30초 지연이 발생합니다. 배치 작업이라면 허용 가능하지만, API 서버라면 사용자 경험에 영향.

**Q: Lifecycle Policy 설정 후 추가 작업이 필요한가요?**
A: 아닙니다. 한 번 설정하면 GCP가 자동으로 조건에 맞는 이미지를 삭제합니다.
