# 06. 데이터 저장 — PostgreSQL + pgvector (제로베이스)

> 목표: 수집·지표화한 데이터를 **시계열 DB(PostgreSQL)** 에 적재하고, 의미 검색용 **벡터 DB(pgvector)** 를 같은 Postgres에 얹는다. (MVP는 별도 벡터DB 없이 pgvector 하나로 충분)
> 선행: [01_개발환경_세팅.md](01_개발환경_세팅.md)

## 1. Docker로 Postgres + pgvector 띄우기 (가장 쉬움)
1. Docker Desktop 설치: https://www.docker.com/products/docker-desktop
2. pgvector 내장 이미지로 컨테이너 실행:
```bash
docker run -d --name trend-db \
  -e POSTGRES_USER=trend \
  -e POSTGRES_PASSWORD=trend \
  -e POSTGRES_DB=trend \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```
3. `.env` 확인: `DATABASE_URL=postgresql://trend:trend@localhost:5432/trend`
> 클라우드 대안: AWS RDS/Supabase/Neon 등 관리형 Postgres(대부분 pgvector 지원). 운영 단계에서 전환.

## 2. 파이썬 클라이언트 설치
```bash
pip install "psycopg[binary]"        # psycopg3
pip freeze > requirements.txt
```

## 3. 스키마 설계 (시계열 지표 중심)
`src/storage/schema.sql`:
```sql
-- 확장: pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- 감시 키워드(아이템)
CREATE TABLE IF NOT EXISTS keywords (
    id          SERIAL PRIMARY KEY,
    term        TEXT UNIQUE NOT NULL,
    category    TEXT,                       -- 예: 빵/초콜릿/음료
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- 일자별 지표(원문 아님 — 집계만): 트렌드 스코어링의 입력
CREATE TABLE IF NOT EXISTS daily_metrics (
    keyword_id     INT REFERENCES keywords(id),
    metric_date    DATE NOT NULL,
    source         TEXT NOT NULL,           -- naver_search/datalab/google/youtube...
    mention_count  INT  DEFAULT 0,          -- 신규 언급/글 수
    search_ratio   REAL,                    -- 검색 관심도(0~100, 데이터랩/구글)
    sentiment_pos  REAL,                    -- 긍정 비율(0~1)
    extra          JSONB,                   -- 소스별 부가지표
    PRIMARY KEY (keyword_id, metric_date, source)
);

-- (선택) 글 단위 메타 + 임베딩(의미검색)
CREATE TABLE IF NOT EXISTS documents (
    id          BIGSERIAL PRIMARY KEY,
    keyword_id  INT REFERENCES keywords(id),
    source      TEXT,
    title       TEXT,
    url         TEXT,
    posted_at   TIMESTAMPTZ,
    embedding   vector(768)                 -- 임베딩 차원은 모델에 맞게(아래 §6)
);
CREATE INDEX IF NOT EXISTS idx_metrics_date ON daily_metrics(metric_date);
```
적용:
```bash
docker exec -i trend-db psql -U trend -d trend < src/storage/schema.sql
```

## 4. 파이썬에서 적재/조회
`src/storage/db.py`:
```python
import psycopg
import config

def conn():
    return psycopg.connect(config.DATABASE_URL)

def upsert_keyword(term: str, category: str | None = None) -> int:
    with conn() as c, c.cursor() as cur:
        cur.execute(
            "INSERT INTO keywords(term, category) VALUES (%s, %s) "
            "ON CONFLICT (term) DO UPDATE SET category = COALESCE(EXCLUDED.category, keywords.category) "
            "RETURNING id",
            (term, category),
        )
        return cur.fetchone()[0]

def save_metric(keyword_id, metric_date, source, mention_count=0, search_ratio=None, sentiment_pos=None, extra=None):
    import json
    with conn() as c, c.cursor() as cur:
        cur.execute(
            "INSERT INTO daily_metrics(keyword_id, metric_date, source, mention_count, search_ratio, sentiment_pos, extra) "
            "VALUES (%s,%s,%s,%s,%s,%s,%s) "
            "ON CONFLICT (keyword_id, metric_date, source) DO UPDATE SET "
            "mention_count=EXCLUDED.mention_count, search_ratio=EXCLUDED.search_ratio, "
            "sentiment_pos=EXCLUDED.sentiment_pos, extra=EXCLUDED.extra",
            (keyword_id, metric_date, source, mention_count, search_ratio, sentiment_pos,
             json.dumps(extra) if extra else None),
        )

if __name__ == "__main__":
    kid = upsert_keyword("두바이초콜릿", "초콜릿")
    save_metric(kid, "2026-06-15", "naver_search", mention_count=1240, sentiment_pos=0.78)
    print("저장 OK, keyword_id:", kid)
```

## 5. 시계열 조회 예 (스코어링 입력)
```sql
-- 최근 30일 언급량 추이
SELECT metric_date, mention_count
FROM daily_metrics m JOIN keywords k ON k.id = m.keyword_id
WHERE k.term = '두바이초콜릿' AND m.source = 'naver_search'
ORDER BY metric_date DESC LIMIT 30;
```

## 6. 임베딩(의미검색)은 어떻게 채우나
- **Anthropic은 임베딩 API를 제공하지 않는다.** 두 가지 선택:
  - **오픈소스(무료·로컬)**: `sentence-transformers`의 다국어 모델(예: `paraphrase-multilingual-MiniLM-L12-v2`, 384차원) → 비용 0, MVP 적합. (스키마 `vector(384)`로 맞출 것)
  - **Voyage AI**(Anthropic 권장 파트너) 등 임베딩 API: 품질↑, 사용량 과금.
```bash
pip install sentence-transformers
```
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2")  # 384차원
vec = model.encode("두바이초콜릿 후기").tolist()   # documents.embedding 에 저장
```
> 임베딩은 "비슷한 글 묶기/유사 트렌드 검색"에 쓰며, MVP에선 후순위. 트렌드 스코어링은 임베딩 없이도 가능(09).

## 7. 트러블슈팅
- **연결 거부**: 컨테이너 실행 여부 `docker ps`, 포트 5432 충돌 확인.
- **extension "vector" 없음**: pgvector 이미지(`pgvector/pgvector`)인지 확인 후 `CREATE EXTENSION vector;`.
- **차원 불일치**: 임베딩 모델 차원과 `vector(N)`이 일치해야 함.

## 다음 단계
→ [07_LLM_Claude_분류감성인사이트.md](07_LLM_Claude_분류감성인사이트.md)
