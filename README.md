# ☁️ GCP Optimize — 클라우드 비용 다이어트 스킬

> **한 줄 요약**: GCP 청구서가 새는 곳을 자동으로 찾아주고, 다시는 안 새도록 막아주는 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 스킬입니다.

`/gcp-optimize` 한 번이면 Cloud Run·BigQuery·Artifact Registry 같은 주요 서비스를 훑어
**"어디서 돈이 새는지 → 얼마인지 → 어떻게 막는지"** 를 리포트로 뽑아줍니다.

---

## 😱 이 스킬이 해결하는 문제

클라우드 비용은 보통 **조용히** 터집니다. 누가 악의로 쓴 게 아니라, 그냥 아무도 안 본 사이에 쌓입니다.

- 🗑️ **빌드 찌꺼기 누적** — Cloud Build로 배포할 때마다 1~2GB 이미지가 쌓입니다. 정리 정책이 없으면 1년이면 수백 GB → 매달 스토리지 요금.
- 🔥 **Cloud Run 설정 한 줄** — `cpu-throttling=false` 하나로 유휴 상태에서도 CPU가 24시간 과금됩니다. 월 +$500~$1,000 차이.
- 💣 **BigQuery 풀스캔 한 방** — 파티션 필터를 깜빡한 쿼리 하나가 대형 테이블 전체를 스캔합니다. 그게 **매시간 도는 배치**라면? 일일 스캔이 수백 GB로 부풀고, 일회성 재계산 한 번에 수십만 원이 찍힙니다.

> 💡 **핵심 통찰**: 클라우드 비용의 **80%는 20%의 리소스 낭비**에서 나옵니다.
> 그리고 그 낭비는 "사람이 조심하면 된다"로는 절대 안 막힙니다 — **사람은 반드시 빠뜨립니다.**
> 이 스킬은 낭비를 찾아주는 동시에, 빠뜨려도 비용이 안 터지는 **자동 안전망**을 깔아줍니다.

---

## 🚀 빠른 시작

### 1. 설치

스킬 파일(`skill.md`) 하나만 복사하면 끝입니다.

```bash
# 글로벌 설치 — 모든 프로젝트에서 /gcp-optimize 사용 가능
git clone https://github.com/melocream/GCP-optimize-skill.git
mkdir -p ~/.claude/skills/gcp-optimize
cp GCP-optimize-skill/skill.md ~/.claude/skills/gcp-optimize/skill.md
```

```bash
# 프로젝트별 설치 — 이 레포에서만 사용
mkdir -p <your-project>/.claude/skills/gcp-optimize
cp GCP-optimize-skill/skill.md <your-project>/.claude/skills/gcp-optimize/skill.md
```

### 2. 실행

Claude Code에서 한 줄이면 됩니다.

```
> /gcp-optimize
```

스킬이 알아서 `gcloud` / `bq` 로 현황을 조회하고, 항목별 진단 + 비용 추정 + 조치 목록을 리포트로 출력합니다.

### 3. 언제 실행하나요?

| 타이밍 | 이유 |
|--------|------|
| 📅 **월 1회 정기 점검** | 빌드 찌꺼기·로그·미사용 리비전은 천천히 쌓입니다. 한 달에 한 번이면 충분히 잡힙니다. |
| 🚨 **청구서가 평소보다 튀었을 때** | "이번 달 왜 이렇게 많지?" 싶을 때 가장 먼저 돌려보세요. 보통 BigQuery 풀스캔이나 Cloud Run 설정이 범인입니다. |
| 🆕 **새 서비스/배치 배포 직후** | 새로 만든 Cloud Run·스케줄러·BigQuery 쿼리가 비용 함정을 안고 있는지 확인. |

> ✅ **이 한 줄이면 됩니다**: `/gcp-optimize` — 나머지는 스킬이 알아서 합니다.

### 📋 사전 준비물

| 도구 | 용도 | 필수? |
|------|------|-------|
| `gcloud` CLI | 대부분의 점검 | ✅ 필수 |
| `bq` CLI | BigQuery 점검 | BigQuery 쓸 때 |
| `gsutil` | Cloud Storage 점검 | GCS 쓸 때 |
| GCP 권한 (Viewer+) | 현황 조회 | ✅ 필수 |
| AR Administrator 권한 | 이미지 삭제 | 정리 실행 시 |
| Python 3.10+ | `gcloud artifacts` 명령 | Artifact Registry 정리 시 |

---

## 🔍 무엇을 점검하나

스킬은 **비용 영향이 큰 순서**로 8개 영역을 훑습니다. 각 영역은 "언제 신경 쓰나 → 무엇을 체크 → 어떻게 고치나" 흐름으로 안내합니다.

### 1️⃣ Container / Artifact Registry — 가장 흔한 범인

- **언제 신경 쓰나**: Cloud Build로 자주 배포하는 프로젝트라면 거의 100% 해당.
- **무엇을 체크**: 태그 없는(untagged) 이미지가 몇 개나 쌓였는지. 50개 이상 또는 총 50GB+ 면 위험.
- **어떻게 고치나**: 오래된 untagged 이미지 삭제 + **자동 정리 정책(cleanup policy)** 한 번 설정 → 이후 영구 자동 정리.

> 💰 흔한 낭비: **$10~200/월** · 난이도: 쉬움

### 2️⃣ Cloud Run — 설정 하나로 월 $1,000 차이

- **언제 신경 쓰나**: 서버리스 API·배치를 Cloud Run에 올렸다면.
- **무엇을 체크**: `cpu-throttling`(반드시 `true`), `min-instances`(보통 0~1), `max-instances`(폭탄 방지), CPU/메모리 과다 할당.
- **어떻게 고치나**: 설정 교정 + 트래픽 0% 오래된 리비전 삭제. 배치 작업이면 `min-instances=0`, API면 `min=1 + throttle=on`.

> 💰 흔한 낭비: **$50~1,000/월** · 난이도: 중간

### 3️⃣ Cloud Build — 보통은 무료 범위

- **언제 신경 쓰나**: CI/CD로 자주 빌드하는 경우.
- **무엇을 체크**: 하루 120분 무료 범위를 넘는지.
- **어떻게 고치나**: Docker 레이어 캐시·`.dockerignore`·멀티스테이지 빌드로 빌드 시간 단축.

> 💰 흔한 낭비: **$5~50/월** · 난이도: 쉬움

### 4️⃣ BigQuery — 풀스캔 한 방이면 청구 폭탄 ⚠️

- **언제 신경 쓰나**: on-demand BigQuery로 쿼리·배치를 돌린다면 (스캔한 바이트만큼 과금, $6.25/TB).
- **무엇을 체크**: 일별 스캔량, 가장 비싼 쿼리 Top 5, 파티션 필터 누락 여부, `SELECT *` 남용.
- **어떻게 고치나**: 파티셔닝 + 파티션 필터 강제 + **아래 7개 가드레일 패턴**(다층 안전망)으로 풀스캔 자체를 차단.

> 💰 흔한 낭비: **$1~500/월** · 난이도: 중간 · 👉 [BigQuery 비용 가드레일 7패턴](#-bigquery-비용-가드레일-7패턴) 꼭 읽으세요

### 5️⃣ Cloud Logging — 안 보는 로그도 돈

- **언제 신경 쓰나**: DEBUG 로그·health check 로그가 쏟아지는 서비스.
- **무엇을 체크**: 월 50GB 무료 범위 초과 여부.
- **어떻게 고치나**: 로그 제외 필터로 DEBUG·health check·verbose stdout 컷 → 볼륨 30~60% 절감.

> 💰 흔한 낭비: **$5~50/월** · 난이도: 쉬움

### 6️⃣ Cloud Scheduler — 작지만 새는 곳

- **언제 신경 쓰나**: 크론 잡이 여러 개일 때.
- **무엇을 체크**: 실패(FAILED) 반복 잡, 과다 빈도 잡, 더는 안 쓰는 잡.
- **어떻게 고치나**: 실패 잡 수정/일시정지, 빈도 조정, 불필요 잡 삭제. (3개까지 무료)

> 💰 흔한 낭비: **$1~5/월** · 난이도: 쉬움

### 7️⃣ Cloud Storage — 방치되기 쉬운 버킷

- **언제 신경 쓰나**: 백업·로그·임시 파일을 GCS에 쌓는 경우.
- **무엇을 체크**: 버킷별 용량, Lifecycle 정책 유무, 불필요한 버전 관리.
- **어떻게 고치나**: Lifecycle 정책(90일→Nearline, 365일→삭제) + 접근 빈도 낮은 데이터는 Coldline/Archive로.

> 💰 흔한 낭비: **$5~200/월** · 난이도: 쉬움

### 8️⃣ 네트워크 / 기타

- **언제 신경 쓰나**: 리전 간 트래픽이 많거나 egress가 큰 경우.
- **무엇을 체크**: Cloud Run ↔ BigQuery 같은 리전 배치 여부, Secret Manager 버전 수.
- **어떻게 고치나**: 통신하는 서비스를 같은 리전에 배치 → 내부 트래픽 무료.

> 💰 흔한 낭비: 가변 · 난이도: 어려움

---

## 🛡️ BigQuery 비용 가드레일 7패턴

> 📖 **전체 상세는 [`skill.md`](./skill.md)의 "4-5. BigQuery 비용 가드레일" 절**에 있습니다.
> 여기서는 유튜브에서 화면 보며 설명하기 좋게 핵심만 요약합니다.

### 왜 7개씩이나 필요한가 (사고 스토리)

on-demand BigQuery는 **스캔한 바이트만큼** 과금됩니다. 파티션 프루닝이 빠진 쿼리 하나가
대형 테이블을 풀스캔하면 단건이 수 GB~수백 GB이고, 그게 **배치/스케줄러로 매시간·매일 반복**되면
비용이 선형이 아니라 **누적·폭증**합니다. 실제 사례에서 미프루닝 쿼리가 쌓이며 일일 스캔이
~485GB까지 부풀었고, 전체 재계산 일회성 풀스캔이 며칠 만에 수십만 원대 청구를 냈습니다.

> 🔑 **핵심 교훈**: 개별 쿼리의 파티션 필터는 **1차 방어일 뿐**, 사람·에이전트는 반드시 빠뜨립니다.
> 그래서 아래 7개 패턴은 "조심하자"가 아니라 **빠뜨려도 비용이 안 터지게 만드는 다층 안전망**입니다.

### 7패턴 한눈에

| # | 패턴 | 한 줄 설명 |
|---|------|-----------|
| 1 | 🧰 **공용 클라이언트** | raw `bigquery.Client()` 금지. `maximum_bytes_billed` 백스톱이 박힌 래퍼만 사용 → 상한 초과 쿼리는 **과금 0으로 즉시 실패**. |
| 2 | 📅 **파티션 필터** | 파티션 테이블엔 날짜/시간 컬럼 **범위 비교**(`>=`/`<`/`BETWEEN`)를 WHERE에. `ORDER BY`는 필터가 아닙니다. |
| 3 | 🎯 **SELECT \* 금지** | BigQuery는 컬럼 지향 → 읽은 컬럼만 과금. 대형 테이블엔 필요한 컬럼만 명시. |
| 4 | 🚦 **정적 린터 + CI 게이트** | PR diff의 **추가(+) 라인만** 검사 → 신규 안티패턴은 머지 전 차단, 레거시는 안 건드림. |
| 5 | 📊 **일일 쿼터 + 알럿** | project/user 단위 일일 바이트 cap + 예산 알림(50/90/100%). 위가 다 뚫려도 막는 최후 방어선. |
| 6 | 🔁 **멱등 적재** | 재실행 안전한 `MERGE` 또는 파티션 단위 `DELETE+INSERT`. 중복 누적 → 다운스트림 풀스캔 비용 차단. |
| 7 | 🏷️ **화이트리스트** | 백필·마이그레이션·ML 같은 **의도적 풀스캔**은 명시적 예외 경로로 격리. 단, 좁게 유지. |

### 🧱 다층 방어 요약표 — "한 층은 반드시 뚫린다"

| 방어층 | 막는 것 | 책임 주체 |
|--------|---------|-----------|
| 파티션 필터·컬럼 선택 (패턴 2·3) | 풀스캔 자체 | 쿼리 작성자 |
| PR 린터 (패턴 4) | 신규 안티패턴 머지 | CI 게이트 |
| `maximum_bytes_billed` 백스톱 (패턴 1) | 필터 누락이 과금되는 것 | 공용 클라이언트 |
| 일일 쿼터 + 알럿 (패턴 5) | 위 전부 뚫린 폭주 | 조직 정책 |
| 멱등 적재 (패턴 6) | 재실행 중복·누적 비용 | 배치 설계 |
| 화이트리스트 (패턴 7) | 정당한 풀스캔이 막히는 것 | 명시적 예외 |

> 🛡️ **설계 철학**: 어느 한 층이 실패해도 다음 층이 비용 폭증을 막습니다. 단일 방어를 믿지 않습니다.

---

## 📝 예시 / Before-After

### BigQuery 쿼리: 파티션 필터 한 줄의 차이

```sql
-- ❌ BEFORE: created_at이 ORDER BY에만 등장 → 풀스캔 (수 GB)
SELECT id, status FROM `ds.events` ORDER BY created_at DESC

-- ✅ AFTER: WHERE에 범위 비교 → 하루치만 스캔 (수 MB)
SELECT id, status FROM `ds.events`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
```

### BigQuery 클라이언트: 상한 백스톱 박기

```python
# ❌ BEFORE: 상한 없는 raw 클라이언트 → 풀스캔이 과금된 뒤에야 알게 됨
from google.cloud import bigquery
client = bigquery.Client(project=PROJECT)

# ✅ AFTER: maximum_bytes_billed 백스톱 → 상한 초과 시 과금 0으로 즉시 실패
client = safe_bq_client(project=PROJECT)            # 기본 20GB 상한
light  = safe_bq_client(project=PROJECT, max_gb=2)  # 경량 배치는 타이트하게
```

### 실행 전 스캔량 추정 (과금 0)

```bash
# 큰 쿼리를 돌리기 전, dry-run으로 스캔량을 먼저 확인 — 예상보다 크면 쿼리부터 고친다
bq query --use_legacy_sql=false --dry_run \
  'SELECT id FROM `ds.events` WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)'
```

### Artifact Registry: 자동 정리 정책 (한 번 설정 = 영구 적용)

```bash
cat > /tmp/ar-cleanup-policy.json << 'EOF'
[
  {
    "name": "delete-old-untagged",
    "action": {"type": "Delete"},
    "condition": {"tagState": "untagged", "olderThan": "604800s"}
  }
]
EOF

gcloud artifacts repositories set-cleanup-policies gcr.io \
  --project=$PROJECT --location=us \
  --policy=/tmp/ar-cleanup-policy.json --no-dry-run
```

### 리포트 출력 예시

```
===================================
  GCP 월간 비용 점검 리포트
  프로젝트: my-project
  점검일: 2026-06-18
===================================

  [1] Container Registry
      이미지: 3개 (4.2 GB)
      상태: 정리 필요 → 자동정리: 미설정
      예상 비용: $0.42/월

  [2] Cloud Run
      서비스: 2개 | 2vCPU / 4GiB / min=1 / throttle=on
      상태: 최적
      예상 비용: $64/월

  [4] BigQuery
      쿼리: 120 GB/월 | 스토리지: 8 GB
      상태: 무료 범위
      예상 비용: $0/월

  -----------------------------------
  예상 총 비용: $68.12/월
  절감 가능액: $12.50/월
  -----------------------------------

  조치 필요:
    1. Container Registry untagged 이미지 정리 + 자동정리 정책
    2. Cloud Logging 제외 필터 설정 (health check, DEBUG)
    3. Cloud Scheduler 실패 Job 1개 점검

  다음 점검: 2026-07-18
===================================
```

---

## ❓ FAQ

**Q. Container Registry 정리하면 프로덕션에 영향 있나요?**
A. 없습니다. `latest` 태그 이미지만 유지하면 됩니다. Cloud Run은 현재 배포된 리비전 이미지를 자체 캐시합니다. untagged 이미지는 이전 빌드 잔여물이라 안전하게 삭제 가능합니다.

**Q. `cpu-throttling=true`로 바꾸면 성능이 떨어지나요?**
A. 요청 처리 중에는 동일한 CPU를 받습니다. 유휴 시에만 CPU를 회수합니다. 대부분의 웹/API/배치 서비스는 차이가 없습니다. 백그라운드 연산이 항상 돌아야 하는 경우만 `false`가 필요합니다.

**Q. `min-instances=0`으로 바꾸면 얼마나 절약되나요?**
A. 4GiB 메모리 기준 약 $26/월 절약. 다만 콜드스타트 시 10~30초 지연이 생깁니다. 배치 작업이면 OK, 실시간 API 서버면 사용자 경험을 따져보세요.

**Q. 일일 쿼터 cap을 걸면 위험하지 않나요?**
A. cap을 너무 낮게 잡으면 정상 쿼리(서빙 포함)까지 차단됩니다. **평소 사용량의 3~5배**로 넉넉하게 잡아 catastrophic 폭주만 거르게 하세요.

**Q. 스트리밍으로 막 넣은 데이터를 UPDATE/DELETE 하려니 에러가 나요.**
A. `insertAll`(스트리밍 삽입)로 적재한 행은 수십 분간 **스트리밍 버퍼**에 머물러 DML 대상이 안 됩니다. 멱등 재적재가 필요한 배치는 스트리밍 대신 **load job** 또는 `INSERT ... SELECT`를 쓰세요.

**Q. Lifecycle 정책은 설정 후 추가 작업이 필요한가요?**
A. 아닙니다. 한 번 설정하면 GCP가 자동으로 조건에 맞는 객체를 전환/삭제합니다.

**Q. GKE나 GCE도 점검되나요?**
A. 이 스킬은 **서버리스/데이터 분석 중심**(Cloud Run·BigQuery 등)에 최적화되어 있습니다. IaaS/컨테이너 오케스트레이션(GCE·GKE)은 별도 도구가 필요합니다.

---

## 📂 더 보기

- 📘 **[`skill.md`](./skill.md)** — 모든 점검 명령어, 비용 단가 참조표, BigQuery 가드레일 7패턴 전체 구현
- 🔗 [GCP Pricing](https://cloud.google.com/pricing) — 최신 단가 확인
- 🔗 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — 스킬 실행 환경

## 라이선스

MIT
