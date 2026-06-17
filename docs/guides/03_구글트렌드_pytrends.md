# 03. 구글 트렌드 (pytrends) (제로베이스)

> 목표: 구글 검색 트렌드의 키워드 관심도 추이를 수집한다. **비공식 라이브러리**이므로 차단(429) 대비가 핵심.
> 선행: [01_개발환경_세팅.md](01_개발환경_세팅.md)

## 1. 성격·주의 (먼저 읽기)
- 구글은 트렌드용 **공식 API가 없다**. `pytrends`는 내부 엔드포인트를 호출하는 **비공식** 라이브러리 → 정책 변경·차단 위험이 있다.
- 반환값은 네이버 데이터랩과 같은 **상대 지수(0~100)**.
- **MVP에서는 네이버 데이터랩을 1차로**, 구글 트렌드는 보조(해외·교차검증)로 쓰는 것을 권장.

## 2. 설치
```bash
pip install pytrends
pip freeze > requirements.txt
```

## 3. 기본 사용법
`src/collectors/google_trends.py`:
```python
import time
from pytrends.request import TrendReq

# hl=언어, tz=시간대(분 단위, 한국=540)
pytrends = TrendReq(hl="ko-KR", tz=540)

def interest(keywords: list[str], timeframe: str = "today 3-m", geo: str = "KR"):
    """keywords: 최대 5개. timeframe 예: 'today 3-m', 'today 12-m', 'now 7-d'"""
    pytrends.build_payload(keywords[:5], timeframe=timeframe, geo=geo)
    df = pytrends.interest_over_time()   # pandas DataFrame
    if "isPartial" in df.columns:
        df = df.drop(columns=["isPartial"])
    return df

if __name__ == "__main__":
    df = interest(["두바이초콜릿", "약과", "소금빵"])
    print(df.tail())          # 최근 추이
```
실행:
```bash
python -m src.collectors.google_trends
```
> `pandas`가 없으면 `pip install pandas`. (pytrends가 보통 함께 설치)

## 4. 차단(429) 대비 — 실무 팁
- **키워드는 한 번에 최대 5개**까지. 더 많으면 여러 번 나눠 호출하되 사이에 `time.sleep(랜덤 1~5초)`.
- 호출 빈도를 낮춘다(하루 1회 배치면 충분 — 트렌드는 일 단위로 변함).
- 429가 잦으면: 재시도(backoff), 호출량 축소, 또는 구글 트렌드 의존을 줄이고 네이버 데이터랩 비중↑.
```python
import random, time
def safe_interest(keywords, **kw):
    for attempt in range(3):
        try:
            return interest(keywords, **kw)
        except Exception as e:
            wait = (2 ** attempt) + random.random()
            print(f"재시도 {attempt+1} ({wait:.1f}s): {e}")
            time.sleep(wait)
    raise RuntimeError("google trends 호출 실패")
```

## 5. 해외 선행지표 활용 (콜드스타트 전략)
`geo`를 바꿔 일본·미국 등 해외 관심도를 함께 수집 → 국내보다 먼저 뜬 아이템을 조기 포착(`docs/03` §2).
```python
interest(["dubai chocolate"], geo="JP")   # 일본
interest(["dubai chocolate"], geo="US")   # 미국
```

## 6. 트러블슈팅
- **429 / TooManyRequestsError**: §4 적용. 그래도 막히면 시간을 두고 재시도.
- **빈 DataFrame**: 검색량이 너무 적은 키워드 → 더 일반적 표현으로 시도.
- **버전 이슈**: pandas 호환 문제 시 `pip install --upgrade pytrends pandas`.

## 다음 단계
→ [04_유튜브_데이터API.md](04_유튜브_데이터API.md) 또는 [06 저장](06_데이터저장_postgres_pgvector.md).
