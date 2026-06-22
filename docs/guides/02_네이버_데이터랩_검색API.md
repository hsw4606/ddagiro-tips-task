# 02. 네이버 데이터랩 + 검색 API (제로베이스)

> 목표: 네이버 개발자 앱을 등록하고, **검색량 추이(데이터랩)** 와 **블로그·뉴스·카페 글(검색 API)** 을 무료로 수집한다. MVP의 1차 데이터 소스.
> 선행: [01_개발환경_세팅.md](01_개발환경_세팅.md)

## 1. 네이버 개발자 앱 등록 (키 발급)

1. <https://developers.naver.com> 접속 → 네이버 계정 로그인.
2. 상단 **Application → 애플리케이션 등록**.
3. 입력:
   - **애플리케이션 이름**: 예) `ddalgiro-trend`
   - **사용 API**: **검색** + **데이터랩(검색어트렌드)** 둘 다 선택.
   - **환경 추가**: `WEB 설정` 선택 후 서비스 URL에 `http://localhost` 입력(개발용).
4. 등록 완료 → **Client ID / Client Secret** 발급됨.
5. 발급값을 `.env`에 입력:

   ```ini
   NAVER_CLIENT_ID=발급받은_ID
   NAVER_CLIENT_SECRET=발급받은_SECRET
   ```

## 2. 호출 한도 (무료)

- 검색 API: 앱당 **하루 25,000회**(2026년 기준, 콘솔에서 확인).
- 데이터랩: 하루 1,000회 수준. → MVP(키워드 200개)에 충분.

> 정확한 한도·정책은 콘솔 및 개발자 문서에서 확인. 한도 초과 시 429 응답.

## 3. 검색 API — 블로그·뉴스·카페 글 수집

- 엔드포인트(JSON): `https://openapi.naver.com/v1/search/blog.json` (뉴스는 `news.json`, 카페는 `cafearticle.json`)
- 인증: 헤더 `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- 주요 파라미터: `query`(검색어), `display`(최대 100), `start`(최대 1000), `sort`(`date`=최신순/`sim`=정확도순)

`src/collectors/naver_search.py`:

```python
import requests
import config

BASE = "https://openapi.naver.com/v1/search"
HEADERS = {
    "X-Naver-Client-Id": config.NAVER_CLIENT_ID,
    "X-Naver-Client-Secret": config.NAVER_CLIENT_SECRET,
}

def search(kind: str, query: str, display: int = 100, sort: str = "date") -> list[dict]:
    """kind: 'blog' | 'news' | 'cafearticle'"""
    url = f"{BASE}/{kind}.json"
    params = {"query": query, "display": display, "sort": sort}
    r = requests.get(url, headers=HEADERS, params=params, timeout=10)
    r.raise_for_status()           # 4xx/5xx면 예외
    items = r.json()["items"]
    # 원문 전체는 보관하지 않고, 분석에 필요한 메타만 추림 (docs/04 §0 원칙)
    return [
        {
            "title": _strip_tags(it["title"]),
            "link": it["link"],
            "desc": _strip_tags(it.get("description", "")),
            "postdate": it.get("postdate") or it.get("pubDate"),
        }
        for it in items
    ]

def _strip_tags(s: str) -> str:
    return s.replace("<b>", "").replace("</b>", "").replace("&quot;", '"').replace("&amp;", "&")

if __name__ == "__main__":
    rows = search("blog", "두바이초콜릿")
    print(f"{len(rows)}건 수집")
    for r in rows[:3]:
        print("-", r["title"], r["postdate"])
```

실행:

```bash
pip install requests   # (01에서 이미 설치했으면 생략)
python -m src.collectors.naver_search
```

## 4. 데이터랩 — 검색량 추이 수집 (트렌드 핵심 신호)

- 엔드포인트: `POST https://openapi.naver.com/v1/datalab/search`
- 헤더: 위 두 개 + `Content-Type: application/json`
- 본문(JSON): `startDate`, `endDate`(yyyy-mm-dd), `timeUnit`(`date`/`week`/`month`), `keywordGroups`(최대 5그룹)
- **반환값은 절대 검색수가 아니라 기간 내 최댓값=100 기준의 상대 비율**(0~100). → 증가율·모멘텀 계산엔 충분.

`src/collectors/naver_datalab.py`:

```python
import requests
import config

URL = "https://openapi.naver.com/v1/datalab/search"
HEADERS = {
    "X-Naver-Client-Id": config.NAVER_CLIENT_ID,
    "X-Naver-Client-Secret": config.NAVER_CLIENT_SECRET,
    "Content-Type": "application/json",
}

def search_trend(keywords: list[str], start: str, end: str, time_unit: str = "date"):
    """keywords 각각을 그룹으로(최대 5개). 반환: 그룹별 [{period, ratio}, ...]"""
    body = {
        "startDate": start,
        "endDate": end,
        "timeUnit": time_unit,
        "keywordGroups": [{"groupName": k, "keywords": [k]} for k in keywords[:5]],
    }
    r = requests.post(URL, headers=HEADERS, json=body, timeout=10)
    r.raise_for_status()
    return r.json()["results"]

if __name__ == "__main__":
    res = search_trend(["두바이초콜릿", "약과", "소금빵"], "2026-01-01", "2026-06-15", "week")
    for g in res:
        pts = g["data"]
        print(g["title"], "최근값:", pts[-1]["ratio"] if pts else "N/A")
```

## 5. 수집 → 지표화 연결 (개념)

- 검색 API 결과 → "키워드별 일자별 신규 글 수 + 텍스트(분류·감성 입력)".
- 데이터랩 결과 → "키워드별 검색 관심도 추이"(트렌드 스코어링의 핵심 입력).
- 두 신호를 [06 저장](06_데이터저장_postgres_pgvector.md)에 적재 → [09 스코어링](09_트렌드스코어링_백테스팅.md)에서 결합.

## 6. 주의·트러블슈팅

- **401 Unauthorized**: 헤더 키 이름/값 오타, 또는 앱에 해당 API 미등록. 콘솔에서 "사용 API"에 검색·데이터랩이 추가됐는지 확인.
- **429 Too Many Requests**: 일일 한도 초과 → 호출 간 `time.sleep`, 키워드 수/주기 조정.
- **검색 API는 최신순(sort=date)** 으로 받아 "신규 유입분"만 집계해야 중복이 줄어든다.
- **준법**: 본문 전재 금지, 메타·집계만 보관(`docs/04` §5). 과도한 호출 금지.

## 다음 단계

→ [03_구글트렌드_pytrends.md](03_구글트렌드_pytrends.md) 또는 바로 [06 저장](06_데이터저장_postgres_pgvector.md).
