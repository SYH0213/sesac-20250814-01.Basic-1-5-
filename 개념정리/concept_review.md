# 오늘의 개념 복습 (LangChain & 비동기) 🚀

> **목표:** GitHub에 올릴 수 있도록, 오늘 다룬 핵심 개념을 *간단 요약 + 예시 코드*로 정리했습니다.  
> 키워드: **LangSmith, Runnable, invoke/stream/batch/await, RunnableParallel, 동시성 vs 병렬성**

---

## TL;DR
- **LangSmith**: LLM 앱의 *로깅/트레이싱/평가* 플랫폼
- **Runnable(LCEL)**: LangChain의 표준 실행 인터페이스 — `invoke / ainvoke / stream / astream / batch / abatch`
- **invoke vs stream vs batch**: **단건** / **스트리밍** / **다건(묶음)** 처리
- **await**: 비동기 함수의 결과를 *기다리는* 키워드(파이썬 `asyncio`)
- **RunnableParallel**: 여러 Runnable을 **동시에(동시성)** 실행해 *맵*으로 결과 반환
- **동시성 vs 병렬성**: *겹쳐 처리* vs *진짜 동시에 계산*

---

## 1) LangSmith 한눈에 보기
LLM 애플리케이션의 품질을 높이기 위한 **트레이싱/디버깅/평가** 도구.
- **Tracing**: 프롬프트/응답/토큰/지연시간 등 실행 경로 기록
- **Datasets & Evaluations**: 데이터셋 기반 자동 평가(정확도/포맷/독성 등)
- **Compare & Iterate**: 프롬프트/모델 변경에 따른 성능 비교

> 시작 3요소: (1) **Dataset** (입력/정답), (2) **Target**(평가 대상 함수/체인), (3) **Evaluators**(채점기)

---

## 2) Runnable & LCEL 핵심
**Runnable**은 LangChain 컴포넌트를 *일관된 방법*으로 실행하는 공통 인터페이스입니다.  
LCEL(LangChain Expression Language) 파이프 구성을 통해 **동기/비동기/배치/스트리밍**을 *자동 지원*합니다.

### 공통 메서드 치트시트
| 메서드 | 설명 | 반환 |
|---|---|---|
| `invoke(input)` | 단건 동기 실행 | 결과(모델은 보통 `AIMessage`) |
| `ainvoke(input)` | 단건 비동기 실행 | `await` 가능한 결과 |
| `stream(input)` | 동기 스트리밍(청크 단위) | 이터레이터 |
| `astream(input)` | 비동기 스트리밍 | async 이터레이터 |
| `batch(list)` | 여러 입력 **동시에** 처리 | 결과 리스트 |
| `abatch(list)` | 비동기 배치 | `await` 가능한 리스트 |

### 예시: 프롬프트 → 모델 → 문자열 파서
```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("한국 라면 5종류 추천해줘.")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
chain = prompt | llm | StrOutputParser()

print(chain.invoke({}))             # 동기 단건
# await chain.ainvoke({})          # 비동기 단건
# for chunk in chain.stream({}):   # 스트리밍
#     print(chunk, end="")
```

---

## 3) stream / batch 간단 이해
- **stream**: 모델의 출력을 **조각(토큰/문장)** 단위로 *바로바로* 받음 → 실시간 UI에 적합
- **batch**: 입력 리스트를 **묶음 처리** → 네트워크 대기 시간을 숨겨 **처리량↑**

```python
# batch 예시
inputs = [{"q": f"item {i}"} for i in range(5)]
results = chain.batch(inputs, max_concurrency=5)
```

---

## 4) RunnableParallel (여러 작업 동시 실행)
여러 Runnable을 **동시에 실행**하고, 결과를 *맵(dict)* 형태로 받습니다.

```python
from langchain_core.runnables import RunnableParallel
from operator import itemgetter

qa = ChatPromptTemplate.from_template("Q: {q}\nA:")
summ = ChatPromptTemplate.from_template("Summarize: {text}")

parallel = RunnableParallel(
    answer = ({"q": itemgetter("q")} | qa | llm | StrOutputParser()),
    summary = ({"text": itemgetter("context")} | summ | llm | StrOutputParser()),
)

out = parallel.invoke({"q": "라면 역사", "context": "라면은 1958년 일본에서..."})

# out -> {"answer": "...", "summary": "..."}
```

> 내부적으로 입력을 동일하게 브로드캐스트하고, 각 브랜치를 **동시성**으로 실행합니다.

---

## 5) `await` (파이썬 비동기)
파이썬 `asyncio`에서 **비동기 함수의 완료를 기다리는** 키워드.

```python
import asyncio, aiohttp

async def fetch(url):
    async with aiohttp.ClientSession() as s:
        async with s.get(url) as r:
            return await r.text()

async def main():
    html = await fetch("https://example.com")
    print(len(html))

asyncio.run(main())
```

- **동시성(Concurrency)**: 여러 I/O 작업을 *겹쳐* 처리(한 코어에서도 가능)  
- **병렬성(Parallelism)**: 여러 코어가 *진짜 동시에* 계산(예: `multiprocessing`)

---

## 6) 작은 패턴 모음
- **전/후처리**: `RunnableLambda` + `RunnablePassthrough.assign(...)`
- **분기 실행**: `RunnableBranch` (입력 값/상태에 따라 다른 서브체인 선택)
- **Jupyter 팁**: 최상위 `await` 지원 → `await chain.ainvoke(...)`

---

## 참고 링크
- LangSmith 개요 & 평가: https://docs.smith.langchain.com/evaluation  
- Runnable 인터페이스: https://python.langchain.com/docs/concepts/runnables/  
- Streaming 가이드: https://python.langchain.com/docs/how_to/streaming/  
- RunnableParallel: https://python.langchain.com/docs/how_to/parallel/  
- Python `asyncio`: https://docs.python.org/3/library/asyncio.html  
- 동시성 vs 병렬성(파이썬 문서/가이드):  
  - https://docs.python.org/3/library/concurrency.html  
  - https://docs.python.org/3/library/multiprocessing.html