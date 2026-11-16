# 멀티모달 RAG ⑤ — FashionRAG 웹 앱 (Streamlit UI/UX) 정리

> 지난 시간에 노트북으로 만든 FashionRAG 를 **누구나 브라우저로 쓰는 웹 앱**으로 바꾼다.
> 핵심은 **주방(로직, `fashion_rag_core.py`)과 홀(화면, `app.py`)을 분리**하고, Streamlit 의 **rerun 동작**을 이해하는 것.

---

## 📌 오늘의 키워드

| 키워드 | 한 줄 설명 |
| --- | --- |
| **Streamlit** | HTML·CSS·JS 없이 **파이썬만으로 웹 화면**을 만드는 도구. `st.xxx()` 를 부른 순서대로 화면에 그려짐 |
| **이벤트 드리븐(Event-driven)** | 이벤트(버튼 클릭, 입력)가 생기면 **약속된 함수가 실행**되는 방식 |
| **rerun** | 위젯을 건드릴 때마다 **`.py` 파일 전체를 1줄부터 다시 실행** |
| **MVC** | Model(비즈니스 로직·DB) / View(화면) / Controller(요청을 보고 Model 로 연결하는 분기 처리기, 라우터와 유사) |
| **관심사 분리 / 단일 책임 원칙(SRP)** | "1함수 1기능" — 역할을 나눠 독립시키면 **서로 영향을 적게** 준다 |
| `st.session_state` | rerun 돼도 남는 **사용자별 저장소** (비유: 내 사물함) |
| `@st.cache_resource` | 모델·DB 연결처럼 무거운 객체를 **한 번만 만들고 재사용** (비유: 모두 같이 쓰는 냉장고) |
| `@st.cache_data` | 표·리스트 같은 **계산 결과**를 입력값별로 캐시 (비유: 반찬통) |
| `st.form` | 입력할 때마다 실행되지 않고 **제출 버튼을 눌러야 한 번 실행** (비유: 편지 봉투) |
| **데코레이터 `@`** | 함수 위에 붙여 기능을 덧씌움. 여기선 "실행 결과를 캐시에 넣어 두자"는 약속 (수업에선 어노테이션이라고 부름) |
| `%%writefile 파일명` | 셀 내용을 **파일로 저장**(실행 X). 노트북에선 앱을 직접 못 켜서 `.py` 로 저장해 사용 |
| **콜백(callback)** | 함수에 "중간중간 알려 줘" 하고 **함수를 인자로 넘김** → 노트북은 `print`, 앱은 `st.progress` |
| `importlib.reload()` | 고친 `.py` 모듈을 **새 내용으로 다시 import** |
| `subprocess.Popen` | 파이썬에서 **다른 프로그램(streamlit 서버)을 백그라운드로 실행** |
| **포트(port)** | 서버의 "호수". 1~1024 는 시스템용, Streamlit 기본은 **8501**. 같은 포트에 서버 2개 불가 |
| **localhost** | "내 컴퓨터" 주소 (`http://localhost:8501`) |
| **웹 3대장** | HTML(문서 구조) / CSS(스타일, 정적) / JavaScript(동적 동작) |
| **HTTP** | 하나의 request 에는 반드시 response → **신뢰성 있는 통신** 약속 |
| **렌더링** | 브라우저는 캔버스 — 문서를 읽어서 컴포넌트를 원하는 위치에 그림 |

---

## 1. 웹 기초 개념

**웹 3대장**
- **HTML**: 문서를 구조화하는 약속된 규격 (제목, 버튼, 입력칸 …)
- **CSS**: 모양·배치를 정하는 정적 리소스 (한 번 정하면 포맷이 고정)
- **JavaScript**: 사용자 요청에 따라 **동적으로** 동작, 원하는 위치에 필요한 것을 끼워 넣음

**HTTP**: 요청(request) 1개 ↔ 응답(response) 1개 → 신뢰성 있는 통신

**브라우저 = 캔버스**: 버튼·텍스트박스 같은 컴포넌트를 원하는 위치에 붙이고, **렌더링**(읽어서 그리기)해서 보여 준다.
→ Streamlit 은 이 과정을 파이썬 함수(`st.button`, `st.text_input` …)로 대신 해 준다.

**MVC 패턴 — 왜 나눌까?**

| | 역할 | 오늘 앱에서 |
| --- | --- | --- |
| **M**odel | 비즈니스 로직, DB 연동, 실제 요구 처리 | `fashion_rag_core.py` (주방) |
| **V**iew | 사용자가 보는 화면 | `app.py` 의 `st.xxx` 화면 |
| **C**ontroller | 요청을 보고 판단해서 Model 로 연결 (라우터, 분기 처리기) | `app.py` 의 `if submitted: core.ask(...)` |

→ 각자 독립적인 역할을 가지면 **하나를 바꿔도 다른 쪽에 영향이 적다.**

---

## 2. 전체 구조 — 식당 비유

| 식당 | 우리 앱 | 하는 일 |
| --- | --- | --- |
| 손님 | 웹 브라우저 | 질문 |
| 홀 직원 | `app.py` | 주문 받고 결과를 화면에 표시 (**대문 파일**: 처음 접속 시 보이는 화면) |
| 주방 | `fashion_rag_core.py` | 이미지 검색·AI 답변 (진짜 로직) |
| 맛보기 | 노트북 | 화면 없이 주방 함수 테스트 |
| 창고 | `fashion_dataset/`, `img_vdb/` | 이미지 + 벡터 DB |

```
              주문                 요리 부탁
 손님(브라우저) ───▶ 홀 직원(app.py) ───────▶ 주방(fashion_rag_core.py) ◀──▶ 창고(사진 폴더 + DB)
                      ▲                          │        ▲
                      └──────── 완성! ───────────┘        │
                                                          ▼
                                                맛보기(노트북) : 손님 오기 전 미리 테스트
```

```
작업 폴더/
├── .env                 ← OPENAI_API_KEY
├── 노트북.ipynb          ← 테스트 장소
├── fashion_rag_core.py  ← 주방 (백엔드 모듈)
├── hello_app.py         ← Streamlit 원리 체험 미니 앱
├── app.py               ← FashionRAG 웹 앱
├── app_practice.py      ← 실습 앱 (CLIP 검색기)
├── fashion_dataset/     ← 이미지 (자동 생성)
└── img_vdb/             ← Chroma DB (자동 생성)
```

**주방/홀 분리 장점**
1. 화면 없이 노트북에서 **주방 함수만 테스트** 가능
2. 화면을 Gradio·FastAPI 등으로 바꿔도 **주방 코드는 그대로 재사용**

---

## 3. 주방 파일 만들기 — `fashion_rag_core.py`

지난 시간 노트북 코드를 **함수 단위로 정리해서 `.py` 파일**로 저장 (`%%writefile`).

| 함수 | 하는 일 |
| --- | --- |
| `check_api_key()` | 키 없으면 `ValueError` → 앱에서 친절한 안내로 표시 |
| `ensure_dataset(progress=)` | Fashionpedia 스트리밍으로 이미지 저장 (**콜백**으로 진행 알림) |
| `get_collection()` | CLIP 임베딩 + Chroma 컬렉션 (약 600MB → 앱에서 캐시) |
| `index_images(collection, progress=)` | DB 에 **없는 이미지만** 50장씩 등록 |
| `query_db(collection, query, n)` | 텍스트로 유사 이미지 검색 |
| `build_chains(vision_model=)` | (번역 체인, 비전 체인) 반환 |
| `ask(collection, chains, query_ko)` | ① 번역 → ② 검색(R) → ③ 입력 구성(A) → ④ 비전 답변(G) → ⑤ 한국어 번역, **dict 반환** |

**`ask()` — 질문 하나가 답이 되기까지**
```
① 한국어 질문 ─▶ ② 영어로 번역 ─▶ ③ 사진 찾기 (R) ─▶ ④ 조언 만들기 (G) ─▶ ⑤ 한국어로 번역
"가을 트렌치코트     AI 번역가        사진 창고에서        AI가 사진 2장을       다시 한국어로
 어떻게 입어?"                       제일 닮은 2장         보고 설명             보여 줌
```
- **RAG = 오픈북 시험**: 먼저 자료(사진)를 찾고(R) → 질문에 붙여서(A) → AI 가 답을 만든다(G). 책을 펴 보고 답하니까 더 정확하다.

**CLIP = 글과 사진을 같은 '지도' 위에 찍어 주는 모델**
- 사진 한 장 = 지도 위 점, 내가 쓴 글(`"a red dress"`)도 같은 지도 위 점
- 글과 **가까이 있는 사진 = 내용이 닮은 사진** → `distance` 가 **작을수록** 더 닮음
- 비슷한 사진끼리 모인다 (빨간 원피스끼리, 청바지끼리, 코트끼리 …)

**포인트**
- `.py` 안에서는 `__file__` 사용 가능 → `BASE_DIR = os.path.dirname(os.path.abspath(__file__))`
- `ask()` 는 화면 출력 없이 **결과 dict 만 반환** → 노트북·앱 어디서든 사용 가능

**콜백 — 같은 함수, 다른 연락 방법** ("중간중간 알려 줘!" 하고 맡기는 전화번호)
```
                 ensure_dataset()  ← 사진 500장 저장하는 심부름꾼, 한 장 끝날 때마다 progress() 로 연락
                 ┌───────┴────────┐
        노트북이 맡기면          웹 앱이 맡기면
        print "... 50/500"      진행 막대 ████░░░░ 40%
```
```python
# 노트북: 글자로 출력
def print_progress(done, total):
    if done % 50 == 0 or done == total:
        print(f"  ... {done}/{total}")
core.ensure_dataset(progress=print_progress)

# 앱: 진행 막대 채우기
core.ensure_dataset(progress=lambda d, t: bar.progress(d / t, text=f"이미지 저장 {d}/{t}"))
```

**노트북에서 모듈 불러오기**
```python
import sys, os, importlib
sys.path.insert(0, os.getcwd())   # 현재 폴더를 import 검색 경로 맨 앞에
import fashion_rag_core as core
importlib.reload(core)            # 파일 수정 후에도 새 내용 반영
```

---

## 4. Streamlit 의 4가지 특징

| 고민 | 해결 도구 | 비유 |
| --- | --- | --- |
| 위젯 조작마다 파일 전체 재실행 | **rerun** (기본 동작) | 요리책을 1쪽부터 다시 읽기 |
| 일반 변수는 rerun 때 초기화 | `st.session_state` | 내 사물함 (사용자별) |
| 무거운 일을 매번 반복 | `@st.cache_resource` / `@st.cache_data` | 냉장고 / 반찬통 (모두 공유) |
| 글자 칠 때마다 실행됨 | `st.form` | 편지 봉투 → 제출해야 실행 |

**① rerun — 버튼을 누르면 처음부터 다시 읽는다**
```
버튼 클릭 ─▶ app.py 1줄: 제목 쓰기
             2줄: 버튼 그리기
             3줄: 사진 보여 주기 ...
             마지막 줄 ─▶ 새 화면 (완전히 새로 그려짐)
버튼을 또 누르면 ─▶ 또 1줄부터!
```

**② 일반 변수 vs `session_state`**

| 일반 변수 = 교실 칠판 | `session_state` = 내 사물함 |
| --- | --- |
| 다시 실행될 때마다 싹 지워짐 → 항상 0 아니면 1 | 다시 실행돼도 그대로 남음 → 누를수록 계속 커짐 |

**③ `@st.cache_resource` — 한 번 만들어서 냉장고에**
```
처음 1번: CLIP(600MB) 만들기, 오래 걸림 ─넣기─▶ [냉장고(캐시)] ─꺼내기─▶ 2번째부터: 기다림 0초
```
- **냉장고(cache_resource)는 모든 손님이 같이** 쓰고, **사물함(session_state)은 손님마다 따로** 있다.

**④ `st.form` — 편지를 다 쓰고 우체통에 넣을 때만 배달**
```
편지 쓰는 중 (글자 칠 때마다 배달 X, AI 호출 X, 비용 X) ─제출 버튼─▶ 우체통 ─딱 1번─▶ AI에게 배달
```

**미니 앱 `hello_app.py` 핵심**
```python
count = 0                                   # 일반 변수: rerun 때마다 0
if "count" not in st.session_state:         # 처음 한 번만 초기화
    st.session_state.count = 0
if st.button("+1 누르기"):                   # 누른 그 실행에서만 True
    count += 1
    st.session_state.count += 1             # 계속 누적

@st.cache_resource                          # 첫 실행만 3초, 이후 캐시에서 꺼냄
def load_heavy_model():
    time.sleep(3)
    return {"loaded_at": time.strftime("%H:%M:%S")}

with st.form("demo_form"):                  # 제출 버튼을 눌러야 실행
    name = st.text_input("이름")
    submitted = st.form_submit_button("제출")
if submitted:
    st.success(f"안녕하세요, {name}님!")
```

**자주 쓰는 명령**

| 종류 | 명령 |
| --- | --- |
| 페이지 | `st.set_page_config` (**반드시 제일 먼저**) |
| 글 | `st.title`, `st.header`, `st.subheader`, `st.write`, `st.markdown`, `st.caption` |
| 입력 | `st.text_input`, `st.button`, `st.selectbox`, `st.slider`, `st.form` / `st.form_submit_button` |
| 배치 | `st.sidebar`, `st.columns`, `st.expander`, `st.divider` |
| 상태 | `st.spinner`, `st.progress`, `st.status`, `st.info` / `st.success` / `st.error`, `st.stop` |
| 이미지 | `st.image` (파일 경로 그대로 OK) |

---

## 5. 앱 실행하기 (서버 열기/닫기)

Streamlit 앱은 **별도의 웹 서버** → 노트북 셀 안에서 못 돌림 → `.py` 파일 + `streamlit run` 필요.

**터미널에서**
```powershell
cd C:\workspaces\fashionRAG-assistant
.\.venv\Scripts\Activate.ps1
streamlit run app.py          # 종료: Ctrl + C
```

**노트북에서 (도우미 함수)**
```python
def start_app(script, port=8501, wait=60):
    cmd = [sys.executable, "-m", "streamlit", "run", script,
           "--server.port", str(port), "--server.headless", "true"]
    proc = subprocess.Popen(cmd, cwd=WORK_DIR,
                            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    # 1초마다 http://localhost:{port}/_stcore/health 가 "ok" 인지 확인 후 proc 반환
    ...

def stop_app(proc):
    if proc and proc.poll() is None:   # 아직 실행 중이면
        proc.terminate(); proc.wait(timeout=10)
```

- `start_app()` = 가게 문 열기(영업 중), `stop_app()` = 가게 문 닫기. **포트 번호(8501) = 가게 호수** → 같은 호수엔 가게를 두 개 못 연다 → 다 쓰면 꼭 닫기
- **포트 사용 중** 오류 → 이전 서버를 안 닫음 → `stop_app` 또는 `port=8502` 로 변경
- `app.py` 수정 후 저장 → 브라우저의 **Rerun** 으로 반영 / `core` 수정 시엔 서버 재시작이 안전
- 노트북 방식은 오류 로그를 숨김 → 오류 확인은 **터미널 실행**으로

---

## 6. FashionRAG 웹 앱 — `app.py`

**화면 구성**
```
┌─ localhost:8501 ───────────────────────────────────────────────────────┐
│ ┌ 사이드바(옆 서랍) ┐   FashionRAG: 패션 및 스타일링 어시스턴트           │
│ │ AI 모델 고르기    │   [ 질문 입력 칸 (form)            ] [조언 받기]   │
│ │ 기록 지우기 버튼  │   Q. 하객룩 원피스 코디                             │
│ │ 사진 몇 장 있는지 │   [닮은 사진 1] [닮은 사진 2]  [AI 스타일링 조언]   │
│ │                   │    (columns = 가로로 나란히)                       │
│ └───────────────────┘   ▶ 이전 질문들 (expander = 접었다 폈다 하는 상자)  │
└────────────────────────────────────────────────────────────────────────┘
```

**rerun 마다 위에서부터 실행되는 순서**

| 순서 | 코드 | 의미 |
| --- | --- | --- |
| 1 | `st.set_page_config` | 탭 이름·와이드 레이아웃 |
| 2 | `check_api_key()` 실패 → `st.error` + `st.stop()` | 키 없으면 안내 후 중단 |
| 3 | 사이드바 `selectbox` | 비전 모델 선택 (gpt-4o-mini / gpt-4o) |
| 4 | `load_collection()` (`@st.cache_resource`) | CLIP + DB 는 **한 번만** 로드 |
| 5 | `session_state.data_ready` 없으면 준비 | `st.status` + `st.progress` + 콜백 |
| 6 | `load_chains(vision_model)` | **인자(모델)별로** 캐시 칸이 따로 |
| 7 | `st.form` 제출 → `core.ask()` | `try/except` 로 오류 안내 |
| 8 | `history` 맨 앞에 저장 → 출력 | 최신은 크게, 예전은 `st.expander` |

**원래 코드의 문제 → 개선**
- 질문마다 600MB CLIP 을 다시 로드 → `@st.cache_resource` 로 **한 번만**
- 입력칸에 글자가 남아 다른 버튼만 눌러도 AI 재호출(비용!) → `st.form(clear_on_submit=True)`
- 첫 실행 시 이미지 저장·색인이 오래 걸려 멈춘 것처럼 보임 → **콜백 + 진행 막대**로 진행 상황 표시

---

## 7. 실습 — CLIP 이미지 검색기 (`app_practice.py`)

OpenAI 없이 **CLIP + Chroma 만으로** 검색 (비용 0원)

```python
with st.sidebar:
    n = st.slider("검색할 이미지 수", min_value=1, max_value=6, value=3)        # TODO 1

if "searches" not in st.session_state:
    st.session_state.searches = []                                            # TODO 2

query = st.text_input("영어 검색어를 입력하세요 (예: a red dress)")
if query:
    results = core.query_db(collection, query, n_results=n)
    cols = st.columns(n)
    for col, pid, uri, dist in zip(cols, results["ids"][0], results["uris"][0], results["distances"][0]):
        col.image(uri, caption=f"{pid} (거리 {dist:.3f})", width=200)        # TODO 3
```

> 📝 **다음 시간:** 웹 브라우저 ↔ 서버의 틀이 어떤 식으로 연동되는지 확인

---

## 🧭 오늘의 전체 흐름 — 한 번에 보는 코드

```python
# ============================================================
# [노트북] 0. 준비
# ============================================================
# %pip install -q streamlit ipywidgets chromadb open-clip-torch pillow datasets langchain-core langchain-openai python-dotenv
import os, sys, importlib, subprocess, time, urllib.request

WORK_DIR = os.getcwd()
if WORK_DIR not in sys.path:
    sys.path.insert(0, WORK_DIR)       # 현재 폴더의 .py 를 import 할 수 있게


# ============================================================
# [노트북] 1. 주방(Model) 파일 저장 — 화면 코드 없이 "로직만"
#   (지난 시간 노트북 코드를 함수로 묶은 것. 핵심 함수만 표시)
# ============================================================
%%writefile fashion_rag_core.py
import os, base64
from dotenv import load_dotenv
import chromadb
from chromadb.utils.embedding_functions import OpenCLIPEmbeddingFunction
from chromadb.utils.data_loaders import ImageLoader
from datasets import load_dataset
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

load_dotenv()
BASE_DIR       = os.path.dirname(os.path.abspath(__file__))   # .py 에서는 __file__ 사용 가능
DATASET_FOLDER = os.path.join(BASE_DIR, "fashion_dataset")
VDB_PATH       = os.path.join(BASE_DIR, "img_vdb")
NUM_IMAGES, N_RESULTS = 500, 2

def check_api_key():
    if not os.getenv("OPENAI_API_KEY"):
        raise ValueError("OPENAI_API_KEY 가 없습니다.")          # 앱에서 st.error 로 안내

def ensure_dataset(num_images=NUM_IMAGES, progress=None):    # progress = 콜백 함수 (done, total)
    ...  # Fashionpedia 스트리밍 → image_0001.jpg 저장, 저장할 때마다 progress(saved, total) 호출

def get_collection():                                       # 무거움(CLIP 600MB) → 앱에서 캐시
    client = chromadb.PersistentClient(path=VDB_PATH)
    return client.get_or_create_collection(name="fashion_images",
                                           embedding_function=OpenCLIPEmbeddingFunction(),
                                           data_loader=ImageLoader())

def index_images(collection, batch_size=50, progress=None):
    ...  # DB 에 없는 이미지만 50장씩 add(ids, uris), 배치마다 progress 호출

def query_db(collection, query, n_results=N_RESULTS):
    return collection.query(query_texts=[query], n_results=n_results, include=["uris", "distances"])

def build_chains(translation_model="gpt-4o-mini", vision_model="gpt-4o-mini"):
    ...  # (번역 체인, 비전 체인) = prompt | ChatOpenAI | StrOutputParser

def ask(collection, chains, query_ko, n_results=N_RESULTS):
    translation_chain, vision_chain = chains
    query_en  = translation_chain.invoke({"text": query_ko, "target_lang": "English"})  # ① 번역
    results   = query_db(collection, query_en, n_results)                               # ② 검색 (R)
    inputs    = format_prompt_inputs(results, query_en)                                 # ③ 입력 구성 (A)
    answer_en = vision_chain.invoke(inputs)                                             # ④ 생성 (G)
    answer_ko = translation_chain.invoke({"text": answer_en, "target_lang": "Korean"})  # ⑤ 번역
    return {"query_ko": query_ko, "query_en": query_en,                                 # 화면 출력 X
            "ids": results["ids"][0], "uris": results["uris"][0],                       # → dict 만 반환
            "distances": results["distances"][0], "answer_ko": answer_ko}


# ============================================================
# [노트북] 2. 주방 맛보기 — 화면 없이 테스트 + 데이터 미리 준비
# ============================================================
import fashion_rag_core as core
importlib.reload(core)                       # 수정한 파일 다시 불러오기
core.check_api_key()

def print_progress(done, total):             # 노트북용 콜백: print
    if done % 50 == 0 or done == total:
        print(f"  ... {done}/{total}")

core.ensure_dataset(progress=print_progress)                 # 이미지 저장
collection = core.get_collection()                           # CLIP + Chroma
core.index_images(collection, progress=print_progress)      # 임베딩 색인
result = core.ask(collection, core.build_chains(), "가을에 입기 좋은 트렌치코트 스타일링 알려줘")
print(result["query_en"], result["ids"])


# ============================================================
# [노트북] 3. 홀(View + Controller) 파일 저장 — 화면만 담당, 요리는 core 에 부탁
# ============================================================
%%writefile app.py
import streamlit as st
import fashion_rag_core as core

st.set_page_config(page_title="FashionRAG", layout="wide")    # ① 반드시 제일 먼저
st.title("FashionRAG: 패션 및 스타일링 어시스턴트")

@st.cache_resource(show_spinner="CLIP 모델과 벡터 DB를 불러오는 중...")
def load_collection():                       # 냉장고: 앱 전체에서 한 번만 생성
    return core.get_collection()

@st.cache_resource(show_spinner=False)
def load_chains(vision_model):               # 인자(모델명)별로 캐시가 따로 생김
    return core.build_chains(vision_model=vision_model)

try:                                         # ② 키 없으면 안내만 하고 멈춤
    core.check_api_key()
except ValueError as e:
    st.error(str(e)); st.stop()

with st.sidebar:                             # ③ 사이드바: 설정
    vision_model = st.selectbox("비전 모델", ["gpt-4o-mini", "gpt-4o"])
    if st.button("대화 기록 지우기"):
        st.session_state.history = []

collection = load_collection()               # ④ 캐시에서 꺼내기

if not st.session_state.get("data_ready"):   # ⑤ 사용자별 최초 1회만 데이터 준비
    with st.status("데이터 준비 상태 확인 중...", expanded=True) as status:
        bar = st.progress(0.0)
        # 같은 주방 함수에 "진행 막대 채우기"를 콜백으로 넘김
        core.ensure_dataset(progress=lambda d, t: bar.progress(d / t, text=f"이미지 저장 {d}/{t}"))
        core.index_images(collection, progress=lambda d, t: bar.progress(d / t, text=f"임베딩 색인 {d}/{t}"))
        status.update(label="준비 완료", state="complete", expanded=False)
    st.session_state.data_ready = True       # 사물함에 "준비 끝" 표시 → rerun 때 건너뜀

chains = load_chains(vision_model)           # ⑥ 선택한 모델의 체인
if "history" not in st.session_state:
    st.session_state.history = []            # 질문 기록장 (rerun 돼도 유지)

with st.form("ask_form", clear_on_submit=True):   # ⑦ 편지 봉투: 제출 시에만 실행 (AI 비용 절약)
    query_ko = st.text_input("스타일링에 대한 질문을 입력하세요:")
    submitted = st.form_submit_button("조언 받기")

if submitted and query_ko.strip():           # Controller: 이벤트(제출) → Model(core.ask) 호출
    try:
        with st.spinner("번역 -> 이미지 검색 -> 스타일링 조언 생성 중..."):
            result = core.ask(collection, chains, query_ko.strip())
        st.session_state.history.insert(0, result)    # 최신 결과를 맨 앞에
    except Exception as e:
        st.error(f"처리 중 오류가 발생했습니다: {e}")   # 앱 전체가 죽지 않게

def render(result):                          # ⑧ 결과 그리기 (View)
    st.caption(f"번역된 질문: {result['query_en']}")
    cols = st.columns(len(result["uris"]))   # 이미지 수만큼 가로 배치
    for col, pid, uri, dist in zip(cols, result["ids"], result["uris"], result["distances"]):
        col.image(uri, caption=f"{pid} (거리 {dist:.3f})", width=300)
    st.markdown(result["answer_ko"])

if st.session_state.history:
    latest, *older = st.session_state.history          # 최신 1개 + 나머지
    st.subheader(f"Q. {latest['query_ko']}"); render(latest)
    for past in older:
        with st.expander(f"Q. {past['query_ko']}"):    # 예전 질문은 접어서
            render(past)
else:
    st.info("질문을 입력하면 비슷한 패션 사진 2장과 함께 스타일링 조언을 드립니다.")


# ============================================================
# [노트북] 4. 서버 열기/닫기 (터미널이면: streamlit run app.py)
# ============================================================
def start_app(script, port=8501, wait=60):
    cmd = [sys.executable, "-m", "streamlit", "run", script,
           "--server.port", str(port), "--server.headless", "true"]
    proc = subprocess.Popen(cmd, cwd=WORK_DIR,                       # 백그라운드로 서버 실행
                            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    url = f"http://localhost:{port}"
    for _ in range(wait):                                            # 1초마다 서버 상태 확인
        try:
            if urllib.request.urlopen(url + "/_stcore/health", timeout=1).read() == b"ok":
                print("실행 중:", url); return proc
        except Exception:
            time.sleep(1)
    return proc

def stop_app(proc):
    if proc and proc.poll() is None:         # 실행 중이면 종료 (포트 반납)
        proc.terminate(); proc.wait(timeout=10)

app_proc = start_app("app.py", port=8501)    # 브라우저에서 http://localhost:8501 접속
# stop_app(app_proc)                         # 다 쓰면 반드시 닫기
```
