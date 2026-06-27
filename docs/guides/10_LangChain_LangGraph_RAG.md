# 10. LangChain · LangGraph · RAG 오케스트레이션 (제로베이스)

> 목표: 안건 1·2의 **AI 파이프라인을 "한 번 호출"이 아니라 "검증 가능한 그래프"로** 구성한다. (1) **LangChain**으로 Claude 호출·구조화 출력·검색기를 표준화하고, (2) **RAG**(검색증강생성)로 "근거 없는 생성"을 차단하며, (3) **LangGraph**로 *되묻기·재시도·근거검증 루프*가 있는 상태 그래프를 만든다.
> 선행: [06 저장(pgvector)](06_데이터저장_postgres_pgvector.md), [07 LLM(Claude)](07_LLM_Claude_분류감성인사이트.md), [09 스코어링·백테스팅](09_트렌드스코어링_백테스팅.md)
> 배경: `docs/02` §5(분석 파이프라인)·§10.1(난제 4 환각), `docs/03` §5~6(NLU·검색·랭킹)·§10.1(난제 1·2·5)
> 원칙(불변): **수치·사실 계산은 코드, 생성만 LLM** + **모든 주장에 출처(citation) 강제**. 프레임워크는 이 원칙을 *강제하는 골격*일 뿐, 차별화 R&D는 그 위에 얹는 도메인 로직(스코어링·LTR·하이브리드 가중·환각 통제)이다.

---

## 0. 왜 프레임워크인가 (TIPS 관점 한 줄)

LangChain/LangGraph는 **플러밍(호출·재시도·상태·도구 연결)을 표준화**해, 팀이 *재발명 대신 차별화 R&D*에 집중하게 한다. RAG는 `docs/02·03 §10.1`의 환각 KPI(**사실 오류율 ≤ 1%, 출처 추적률 100%**)를 *구조적으로* 달성하는 메커니즘이고, LangGraph의 상태 그래프는 *모호 질의 되묻기*(03 난제 1)·*근거 부족 시 재검색*(자기수정) 같은 **사이클**을 선형 파이프라인이 못 하는 방식으로 표현한다. → "단순 API 통합"이 아니라 **검증 가능한 추론 파이프라인 R&D**라는 서사를 코드로 뒷받침한다.

## 1. 설치

```bash
pip install langchain langchain-anthropic langgraph langchain-postgres
pip install sentence-transformers      # 임베딩(오픈소스·무료) — 06 §6과 동일
pip freeze > requirements.txt
```

> LLM 단가·모델 ID는 [07 §3](07_LLM_Claude_분류감성인사이트.md)·`docs/05`와 동일(Haiku `claude-haiku-4-5` / Sonnet `claude-sonnet-4-6` / Opus `claude-opus-4-8`). **프레임워크 자체는 오픈소스·무료** — 추가되는 운영비는 LLM 토큰뿐이다(`docs/05`).

## 2. LangChain + Claude — 호출·티어링·구조화 출력

`ChatAnthropic`로 [07](07_LLM_Claude_분류감성인사이트.md)의 호출을 표준화한다. 작업별 모델 티어링은 그대로 유지한다.

`src/llm/models.py`:

```python
from langchain_anthropic import ChatAnthropic

# 대량·단순(분류/감성) → 최저가 / 인사이트·근거생성 → 고급
haiku  = ChatAnthropic(model="claude-haiku-4-5",  max_tokens=512)
sonnet = ChatAnthropic(model="claude-sonnet-4-6", max_tokens=1500)
```

**슬롯 추출(03 난제 1)** 은 자유 텍스트 파싱 대신 스키마를 강제(`with_structured_output`)해 파싱 오류를 없앤다.

`src/nlu/slots.py`:

```python
from pydantic import BaseModel, Field
from .models import haiku

class Query(BaseModel):
    dessert: str | None = Field(None, description="디저트 종류(케이크/쿠키 등)")
    diet: list[str] = Field(default_factory=list, description="식이제약(비건·글루텐프리·저당)")
    max_price: int | None = None
    region: str | None = None
    occasion: str | None = Field(None, description="개인/선물/행사/B2B대량")
    missing: list[str] = Field(default_factory=list, description="되물어야 할 빠진 핵심 슬롯")

slot_extractor = haiku.with_structured_output(Query)

def extract(text: str) -> Query:
    return slot_extractor.invoke(
        f"다음 디저트 질의를 구조화하라. 명시 안 된 핵심값은 missing에 담아라.\n질의: {text}"
    )
```

## 3. RAG 빌딩블록 — 임베딩 → pgvector 검색기 → 하이브리드

RAG = **검색(Retrieve)으로 근거를 모아 → 생성(Generate)에 근거만 주입**. 안건 2의 후보 탐색(03 §5.2)·안건 1의 인사이트 근거(02 §5.4)가 모두 RAG다.

### 3.1 벡터 검색기 (pgvector)

[06 §6](06_데이터저장_postgres_pgvector.md)의 임베딩(오픈소스 `sentence-transformers`, 또는 Voyage AI)을 LangChain 검색기로 감싼다. **Anthropic은 임베딩 API를 제공하지 않으므로** 임베딩 모델은 별도다.

`src/rag/retriever.py`:

```python
from langchain_postgres import PGVector
from langchain_huggingface import HuggingFaceEmbeddings

emb = HuggingFaceEmbeddings(model_name="paraphrase-multilingual-MiniLM-L12-v2")  # 384차원
store = PGVector(
    embeddings=emb,
    collection_name="shops",          # 업체·메뉴(03) 또는 trend_docs(02)
    connection="postgresql+psycopg://trend:trend@localhost:5432/trend",
)
vector_retriever = store.as_retriever(search_kwargs={"k": 30})
```

### 3.2 하이브리드 검색 (의미 + 키워드/하드제약) — 03 난제 2

의미검색만으로는 "비건인데 우유 든 집" 같은 **하드제약 위반**을 못 막는다. 의미검색 후보에 **하드 필터(SQL)** 를 결합한다(03 §5.2). 이 융합 가중·LTR 튜닝이 차별화 R&D 지점이다.

```python
def hybrid_search(q):
    cands = vector_retriever.invoke(q.dessert or "")     # 의미 후보 top-K
    # 하드제약은 정확 필터(위반=치명적): 지역·식이·운영상태
    return [c for c in cands
            if (not q.region or q.region in c.metadata.get("region", ""))
            and all(d in c.metadata.get("diet_tags", []) for d in q.diet)
            and c.metadata.get("is_open", True)]
```

### 3.3 근거 기반 생성 + 출처 강제 (환각 통제) — 02 난제 4 / 03 난제 5

생성 단계는 **검색된 근거 안에서만** 답하고 **출처를 반드시 표기**하게 묶는다. 이 한 줄이 KPI(오류율 ≤ 1%·출처추적 100%)의 구조적 근거다.

`src/rag/generate.py`:

```python
from langchain_core.prompts import ChatPromptTemplate
from .models import sonnet

GROUNDED = ChatPromptTemplate.from_messages([
    ("system",
     "너는 디저트 추천 설명가다. 아래 '근거'에 있는 사실만 사용하라. "
     "근거에 없는 수치·영업상태·메뉴는 절대 지어내지 말고 '확인 불가'로 답하라. "
     "각 문장 끝에 [출처:id]를 붙여라."),
    ("human", "질문: {q}\n\n근거:\n{evidence}"),
])

def explain(q: str, docs) -> str:
    evidence = "\n".join(f"[{d.metadata['id']}] {d.page_content}" for d in docs)
    return (GROUNDED | sonnet).invoke({"q": q, "evidence": evidence}).content
```

## 4. LangGraph — 안건 2: 되묻기·재검색 루프가 있는 상태 그래프

선형 RAG는 "빠진 조건 되묻기"(03 난제 1)나 "근거 부족 시 재검색"(자기수정)을 못 한다. LangGraph로 **사이클이 있는 그래프**를 만든다.

`src/graph/agenda2.py`:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, END
from src.nlu.slots import extract
from src.rag.retriever import hybrid_search
from src.rag.generate import explain
# 랭킹은 코드(09 스코어링 / 03 §6) — LLM 아님
from src.ranking.score import rank

class S(TypedDict):
    text: str; query: object; candidates: list; answer: str; reask: str | None

def understand(s): return {"query": extract(s["text"])}

def route(s):                                   # 조건부 분기(되묻기 vs 진행)
    return "ask" if s["query"].missing else "retrieve"

def ask(s):                                     # 난제 1: 능동 되묻기
    return {"reask": f"{', '.join(s['query'].missing)} 정보를 알려주세요."}

def retrieve(s): return {"candidates": hybrid_search(s["query"])}

def check(s):                                   # 자기수정: 후보 부족 → 재검색
    return "rank" if s["candidates"] else "retrieve"

def do_rank(s): return {"candidates": rank(s["candidates"], s["query"])}  # 코드 랭킹
def answer(s):  return {"answer": explain(s["text"], s["candidates"][:5])} # 근거기반 생성

g = StateGraph(S)
for n, f in [("understand", understand), ("ask", ask),
             ("retrieve", retrieve), ("rank", do_rank), ("answer", answer)]:
    g.add_node(n, f)
g.set_entry_point("understand")
g.add_conditional_edges("understand", route, {"ask": "ask", "retrieve": "retrieve"})
g.add_edge("ask", END)                                   # 사용자 답을 받아 재진입
g.add_conditional_edges("retrieve", check, {"rank": "rank", "retrieve": "retrieve"})
g.add_edge("rank", "answer"); g.add_edge("answer", END)
agent = g.compile()
# agent.invoke({"text": "안 단 비건 생일케이크, 강남 당일배송"})
```

> 책임 분리 그대로: **이해(슬롯)=LLM, 검색=RAG, 랭킹=코드(09), 설명=근거기반 LLM**. 그래프는 이 단계들을 *검증 가능하게* 잇고, `route`/`check`가 난제 1(되묻기)·자기수정을 표현한다.

## 5. LangGraph — 안건 1: 분석 파이프라인을 그래프로

[07](07_LLM_Claude_분류감성인사이트.md)의 분류→감성→스코어→인사이트(02 §5)를 그래프 노드로 만들면 **단계별 재시도·관측·부분 실패 격리**가 쉬워진다. 핵심은 §5.4 인사이트를 **RAG로 근거 결박**하는 것:

- 노드 `extract`(NER/신조어, Haiku) → `sentiment`(Haiku) → **`score`(코드, 09 — LLM 아님)** → `insight`(RAG: 코드가 계산한 지표 + 원천 글을 근거로 주입해 Sonnet/Opus가 브리핑 생성, 출처 강제).
- 환각 통제(난제 4): 인사이트 노드는 §3.3의 `GROUNDED` 프롬프트를 재사용 — 지표는 코드값만, 문장마다 출처.

(전체 코드는 §4 패턴과 동일 구조이므로 생략 — `score` 노드만 `src/analysis`의 09 스코어러를 호출.)

## 6. 비용·운영

- **티어링 유지**: 대량(분류·감성)=Haiku, 인사이트·근거생성=Sonnet/Opus(02 §5.4, [07 §3](07_LLM_Claude_분류감성인사이트.md)).
- **캐싱**: 공통 시스템 프롬프트는 프롬프트 캐싱으로 입력비 절감([07 §6](07_LLM_Claude_분류감성인사이트.md)).
- **배치**: 실시간 불필요한 야간 분류는 배치 50% 할인([07 §5](07_LLM_Claude_분류감성인사이트.md)).
- **프레임워크 비용 = 0**: LangChain/LangGraph/pgvector/오픈소스 임베딩은 무료. 추가비는 LLM 토큰뿐 → `docs/05` 워크시트로 산정.
- **관측성**(확장): LangGraph 노드 단위 로깅으로 KPI(환각 오류율·되묻기 적중)를 단계별로 측정 → `docs/02·03 §10.1` 검증에 연결.

## 7. 안건 매핑

| 구성 | 안건 1(트렌드) | 안건 2(자연어 랭킹) |
|------|---------------|---------------------|
| LangChain 구조화 출력 | NER/신조어·감성 스키마(난제 2) | 슬롯 추출(난제 1) |
| RAG 검색 | 유사 트렌드/원천글 근거 | 하이브리드 후보 탐색(난제 2) |
| RAG 근거생성+출처 | 인사이트 브리핑(난제 4) | "왜 추천" 설명(난제 5) |
| LangGraph 그래프 | 분석 파이프라인(재시도·격리) | 되묻기·재검색 루프(난제 1·자기수정) |
| 코드(프레임워크 밖) | 트렌드 스코어(09) | 다신호 랭킹·LTR(03 §6) |

## 8. 다음 단계

- 03 §9 Phase 0(서울 일부 골든셋)에서 위 그래프로 end-to-end 1흐름 구성 → §10.1 KPI(슬롯 F1·nDCG·환각) 1차 측정.
- LTR 도입 시 `rank` 노드만 교체(그래프 구조 불변) — 프레임워크가 주는 *부품 교체 용이성*의 실증.
- 운영 피드백(클릭·전환) → 재학습 환류는 안건 1·2 공유([04](04_인터넷데이터_활용방법.md)).
