# 04. 유튜브 Data API (제로베이스)

> 목표: 디저트 키워드 영상의 **조회수·좋아요·댓글수·업로드 추이**를 수집한다. 바이럴 선행지표.
> 선행: [01_개발환경_세팅.md](01_개발환경_세팅.md) · Google 계정

## 1. GCP 프로젝트 + API 키 발급
1. https://console.cloud.google.com 접속 → Google 계정 로그인.
2. 상단 프로젝트 선택 → **새 프로젝트** → 이름 `ddalgiro-trend` → 만들기.
3. 좌측 메뉴 **API 및 서비스 → 라이브러리** → "YouTube Data API v3" 검색 → **사용 설정(Enable)**.
4. **API 및 서비스 → 사용자 인증 정보 → 사용자 인증 정보 만들기 → API 키**.
5. 발급된 키를 `.env`에 입력:
   ```
   YOUTUBE_API_KEY=발급받은_키
   ```
6. (권장) API 키 클릭 → **키 제한**: "API 제한"에서 YouTube Data API v3만 허용.

## 2. 쿼터(무료 한도)와 비용 설계 — 중요
- 무료 쿼터: **하루 10,000 units**(프로젝트당).
- 작업별 비용: `search.list` = **100 units**, `videos.list` = **1 unit**, `commentThreads.list` = 1 unit.
- 즉 검색은 비싸다 → **하루 검색 횟수 = 10,000 / 100 = 최대 100회**.
- 설계: 키워드 검색은 아껴 쓰고(대표 키워드 위주), 영상 통계 조회(videos.list)는 저렴하므로 폭넓게.

## 3. 설치
```bash
pip install google-api-python-client
pip freeze > requirements.txt
```

## 4. 키워드 영상 검색 + 통계 수집
`src/collectors/youtube.py`:
```python
from googleapiclient.discovery import build
import config

yt = build("youtube", "v3", developerKey=config.YOUTUBE_API_KEY)

def search_videos(query: str, published_after: str, max_results: int = 25):
    """published_after: RFC3339, 예 '2026-06-01T00:00:00Z'. (search.list = 100 units)"""
    resp = yt.search().list(
        q=query, part="snippet", type="video",
        order="date", publishedAfter=published_after, maxResults=max_results,
    ).execute()
    return [
        {"video_id": it["id"]["videoId"],
         "title": it["snippet"]["title"],
         "channel": it["snippet"]["channelTitle"],
         "published": it["snippet"]["publishedAt"]}
        for it in resp.get("items", [])
    ]

def video_stats(video_ids: list[str]):
    """여러 영상 통계를 한 번에 (videos.list = 1 unit, id 최대 50개)"""
    resp = yt.videos().list(part="statistics", id=",".join(video_ids[:50])).execute()
    out = {}
    for it in resp.get("items", []):
        s = it["statistics"]
        out[it["id"]] = {
            "views": int(s.get("viewCount", 0)),
            "likes": int(s.get("likeCount", 0)),
            "comments": int(s.get("commentCount", 0)),
        }
    return out

if __name__ == "__main__":
    vids = search_videos("두바이초콜릿", "2026-06-01T00:00:00Z", max_results=10)
    stats = video_stats([v["video_id"] for v in vids])
    for v in vids:
        st = stats.get(v["video_id"], {})
        print(v["title"][:30], "조회수:", st.get("views"))
```
실행:
```bash
python -m src.collectors.youtube
```

## 5. 트렌드 신호로 쓰기
- 키워드별 **신규 영상 수 + 합산 조회수/좋아요/댓글** → "영상 기반 관심도"로 지표화.
- 짧은 기간(예: 최근 7일) 업로드 급증 + 조회수 급증 = 바이럴 초기 신호 → 스코어링(09) 가중.

## 6. 트러블슈팅
- **403 quotaExceeded**: 일일 쿼터 초과 → search 횟수 줄이고 videos.list 위주로. 다음 날 리셋(태평양 시간 자정).
- **403 accessNotConfigured**: API가 프로젝트에 사용 설정 안 됨 → §1-3 다시.
- **keyInvalid**: 키 오타/제한 설정 충돌.
- 댓글 텍스트 수집 시 개인정보·약관 유의(`docs/03` §5).

## 다음 단계
→ [05_웹크롤링_기초와_준법.md](05_웹크롤링_기초와_준법.md) 또는 [06 저장](06_데이터저장_postgres_pgvector.md).
