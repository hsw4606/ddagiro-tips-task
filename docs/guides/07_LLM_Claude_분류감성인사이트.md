# 07. LLM(Claude)으로 분류·감성·인사이트 (제로베이스)

> 목표: Claude API로 (1) 디저트 글 **분류**, (2) **감성 분석**(대량, 저가 모델), (3) **트렌드 인사이트 생성**(소량, 고급 모델)을 구현한다. 비용은 **티어링 + 배치 + 캐싱**으로 통제.
> 선행: [01_개발환경_세팅.md](01_개발환경_세팅.md), [06 저장](06_데이터저장_postgres_pgvector.md)
> 원칙: **수치 계산은 코드, 해석·생성만 LLM**(환각 통제). 단가·산정은 `docs/05_비용가이드.md`.

## 1. 계정 + API 키
1. https://console.anthropic.com (= platform.claude.com) 가입.
2. **결제수단 등록**(사용량 과금) + 초기 크레딧 충전.
3. **API Keys → Create Key** → 발급값을 `.env`에:
   ```
   ANTHROPIC_API_KEY=sk-ant-...
   ```
4. (권장) **Usage limits**에서 월 사용 한도(budget cap) 설정 → 폭주 방지.

## 2. 설치
```bash
pip install anthropic
pip freeze > requirements.txt
```

## 3. 모델 선택 (티어링)
| 작업 | 모델 ID | 입력/출력 $1M | 이유 |
|------|---------|---------------|------|
| 분류·감성(대량) | `claude-haiku-4-5` | $1 / $5 | 건수 많고 단순 → 최저가 |
| 중간 난도 | `claude-sonnet-4-6` | $3 / $15 | 균형 |
| 인사이트(소량·고품질) | `claude-opus-4-8` | $5 / $25 | 전략 브리핑 |

## 4. 분류 + 감성 — 구조화 출력(Structured Output)
"디저트 관련 여부 + 광고성 여부 + 감성"을 한 번에. Pydantic 스키마로 결과를 강제해 파싱 오류를 없앤다.
`src/analysis/classify.py`:
```python
import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()   # ANTHROPIC_API_KEY 자동 사용

class Label(BaseModel):
    is_dessert: bool          # 디저트 관련 글인가
    is_ad: bool               # 광고/체험단성인가
    sentiment: str            # "positive" | "neutral" | "negative"
    item: str | None          # 핵심 디저트 아이템명(없으면 null)

def classify(text: str) -> Label:
    resp = client.messages.parse(
        model="claude-haiku-4-5",
        max_tokens=256,
        system="너는 한국 디저트 트렌드 분석가다. 주어진 글을 정확히 분류하라.",
        messages=[{"role": "user", "content": text}],
        output_format=Label,
    )
    return resp.parsed_output

if __name__ == "__main__":
    r = classify("두바이초콜릿 카페 다녀왔는데 겉바속촉 너무 맛있어요!")
    print(r)
```
> `messages.parse` + `output_format`(Pydantic)은 스키마 검증을 자동 처리한다. (구버전 SDK면 `output_config={"format": {...}}` 사용)

## 5. 대량 처리 비용 절감 — Batch API (50% 할인)
실시간이 필요 없는 야간 분류·감성은 **배치**로. 최대 10만 건/배치, 보통 1시간 내 완료, **모든 토큰 50% 할인**.
`src/analysis/classify_batch.py`:
```python
import anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

def submit(texts: list[str]):
    reqs = [
        Request(
            custom_id=f"doc-{i}",
            params=MessageCreateParamsNonStreaming(
                model="claude-haiku-4-5",
                max_tokens=256,
                system="너는 한국 디저트 트렌드 분석가다. is_dessert/is_ad/sentiment/item을 JSON으로만 답하라.",
                messages=[{"role": "user", "content": t}],
            ),
        )
        for i, t in enumerate(texts)
    ]
    batch = client.messages.batches.create(requests=reqs)
    return batch.id

def collect(batch_id: str):
    for res in client.messages.batches.results(batch_id):
        if res.result.type == "succeeded":
            msg = res.result.message
            text = next((b.text for b in msg.content if b.type == "text"), "")
            yield res.custom_id, text
```
> 제출 후 `client.messages.batches.retrieve(batch_id).processing_status == "ended"` 가 될 때까지 폴링 → `collect`.

## 6. 프롬프트 캐싱 — 공통 지침 재사용 (입력비 절감)
분류 지침처럼 매 요청 공통인 큰 프롬프트는 캐싱하면 입력 토큰이 캐시읽기(~0.1×)로 청구된다.
```python
resp = client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=256,
    system=[{
        "type": "text",
        "text": LONG_CLASSIFY_GUIDE,                 # 길고 고정된 지침
        "cache_control": {"type": "ephemeral"},      # 캐시 지정
    }],
    messages=[{"role": "user", "content": text}],
)
```

## 7. 인사이트 생성 — 고급 모델 (소량)
정량 지표(코드로 계산한 증가율·확산단계 등)와 근거를 **입력**으로 주고, "왜 뜨는가 / 누가 만들면 좋은가 / 어떻게 준비하나"를 자연어로 생성.
`src/analysis/insight.py`:
```python
import anthropic
client = anthropic.Anthropic()

def brief(item: str, metrics: dict, sources: list[str]) -> str:
    prompt = f"""아이템: {item}
정량 지표(코드 계산값): {metrics}
근거 출처: {sources}

위 '지표'를 근거로 입점 업체용 1페이지 브리핑을 작성하라.
- 왜 지금 뜨는가 (지표 근거로만)
- 어떤 업장이 만들면 좋은가
- 준비 포인트(난이도/원가/진입 타이밍)
- 반드시 출처를 함께 표기. 지표에 없는 수치는 지어내지 말 것."""
    resp = client.messages.create(
        model="claude-opus-4-8",       # 또는 claude-sonnet-4-6
        max_tokens=1500,
        messages=[{"role": "user", "content": prompt}],
    )
    return next(b.text for b in resp.content if b.type == "text")
```
> 핵심: **숫자는 코드가 계산해 prompt에 주입**하고, LLM은 설명/추천 문장만 쓴다 → 환각·계산오류 방지(`docs/04` §4, `docs/02` §5.4).

## 8. 비용 가드 (실무)
- 대량 작업은 **반드시 Haiku + Batch**. Opus를 대량에 쓰지 말 것.
- 월 토큰 상한/알림 설정(콘솔). 일일 처리 건수 cap.
- 캐싱으로 공통 프롬프트 입력비 절감.
- 추정·재계산: `docs/05_비용가이드.md` §3 워크시트.

## 9. (대안) OpenAI/Gemini로 교체 시
같은 작업을 OpenAI(GPT-5.4 Mini 등)·Gemini(Flash-Lite 등)로도 구현 가능(단가 `docs/05` §2). SDK만 다르고 구조(분류=구조화출력, 대량=배치, 인사이트=고급모델)는 동일. 선택은 **한국어/도메인 품질 PoC**로 결정.

## 10. 트러블슈팅
- **401**: `ANTHROPIC_API_KEY` 미설정/오타.
- **429 rate_limit**: SDK가 자동 백오프 재시도. 대량은 Batch로 전환.
- **결과가 스키마와 다름**: `messages.parse`+Pydantic 사용, 또는 시스템 프롬프트에 출력형식 명시.

## 다음 단계
→ [08_스케줄러_자동화.md](08_스케줄러_자동화.md) · [09 스코어링/백테스팅](09_트렌드스코어링_백테스팅.md)
