# 멀티모달 RAG ④ — FashionRAG (CLIP 이미지 검색 + GPT 비전) 정리

> 한국어로 질문하면 → **비슷한 패션 사진을 찾아(CLIP + Chroma)** → **사진을 보여주며 스타일링 조언(GPT 비전)** 을 해주는 챗봇을 만든다.

---

## 📌 오늘의 키워드

| 키워드 | 한 줄 설명 |
| --- | --- |
| **멀티모달 RAG** | 텍스트 대신 **이미지를 검색**해서, 이미지를 볼 수 있는 LLM(비전 모델)의 근거로 쓰는 RAG |
| **모달리티(modality)** | 데이터의 종류 (텍스트, 이미지, 음성 …) |
| **CLIP** | 텍스트와 이미지를 **같은 512차원 벡터 공간**에 놓는 모델 → 텍스트로 이미지 검색 가능 |
| **대조 학습** (Contrastive) | "사진–설명" 짝은 벡터를 **가깝게**, 짝이 아닌 것은 **멀게** 학습 |
| **Fashionpedia** | 패션 사진 데이터셋 (HF ID: `detection-datasets/fashionpedia`) |
| `streaming=True` | 데이터셋 전체(약 3.5GB)를 받지 않고 **필요한 만큼만** 읽어 옴 → `.take(n)` 으로 사용 |
| **Chroma** | 벡터 DB. `PersistentClient` = 디스크 저장 (껐다 켜도 유지) |
| `OpenCLIPEmbeddingFunction` | 데이터 → CLIP 벡터 변환 (임베딩 함수) |
| `ImageLoader` | 이미지 경로(URI) → 실제 이미지 배열(**NumPy**, 픽셀 숫자)로 읽어 줌 (데이터 로더) |
| `add(ids, uris)` / `query(query_texts)` | 이미지 저장 / 텍스트로 유사 이미지 검색 |
| **distance** | Chroma 기본 = 제곱 유클리드 거리(L2²). **작을수록 비슷함** |
| **LCEL** | `prompt \| model \| parser` 파이프로 체인 구성 |
| **템플릿 변수** `{변수}` | 프롬프트의 빈칸. `invoke(dict)` 할 때 채워짐 → **체인 재사용** |
| `temperature=0.0` | 무작위성 최소 → 번역처럼 정답에 가까운 출력이 필요할 때 |
| **base64** | 바이너리(이미지)를 문자 64개로 표현 → JSON 텍스트에 넣을 수 있음 (용량 약 1.33배) |
| **MIME 타입** | 파일 형식을 알려주는 표준 이름 (`image/jpeg`, `image/png`, `video/mp4` …) |
| **data URL** | `data:<MIME>;base64,<데이터>` → 파일을 인터넷에 안 올리고 URL 자리에 직접 넣음 |
| `__file__` | `.py` 파일에서만 존재 → 노트북에선 `NameError` → `os.getcwd()` 사용 |

> 📝 **추가 학습 필요:** 파이썬 **트리 / 그래프 자료구조** → 다음에 배울 **LangGraph** 이해에 필요.

---

## 1. 전체 구조 한눈에 보기

**사전 준비 (최초 1회)**
```
Fashionpedia ──(스트리밍, N장 저장)──▶ fashion_dataset/ ──CLIP 이미지 임베딩──▶ Chroma (img_vdb/)
```

**질문할 때마다**
```
① 한국어 질문 ──번역 체인──▶ 영어 질문                     (전처리)
② 영어 질문 ──CLIP 텍스트 임베딩──▶ Chroma 검색 → 사진 2장   (Retrieval)
③ 사진 → base64 data URL + 질문 → 프롬프트 입력 dict         (Augmentation)
④ 비전 체인(gpt-4o-mini)이 사진을 보고 영어로 조언           (Generation)
⑤ 영어 답변 ──번역 체인──▶ 한국어 답변 + 검색된 사진 출력    (후처리)
```

---

## 2. 환경 준비 & 설정값

**설치 패키지:** `chromadb`, `open-clip-torch`(torch 포함, 오래 걸림), `pillow`, `datasets`, `langchain-core`, `langchain-openai`, `python-dotenv`
- `%pip` 사용 (`!pip` 은 다른 파이썬 환경에 설치될 수 있음)

**API 키:** `.env` → 없으면 `getpass()` 로 입력 (화면에 안 보임) → 키는 앞 6자리만 출력

**설정값을 맨 위에 모으는 이유**
- 경로·모델명·개수가 코드 곳곳에 흩어지면 바꾸기 어렵다.
- **설정 셀 하나만 고치면 전체 동작이 바뀌도록** → 값 확인·변경이 한 곳에서 끝남.
- 즉, 환경적인 부분(경로, 모델, 개수)은 **먼저 세팅한 뒤** 본 작업을 시작한다.

```python
BASE_DIR        = os.getcwd()                              # 노트북엔 __file__ 이 없음
DATASET_FOLDER  = os.path.join(BASE_DIR, "fashion_dataset")
VDB_PATH        = os.path.join(BASE_DIR, "img_vdb")
COLLECTION_NAME = "fashion_images"   # 3자 이상, 영문/숫자/._- 만 가능
NUM_IMAGES = 100                     # 가져올 이미지 수
N_RESULTS  = 2                       # 질문당 검색 이미지 수 (비전 프롬프트에 2장)
TRANSLATION_MODEL = VISION_MODEL = "gpt-4o-mini"
```

---

## 3. 보조 함수 (helper)

| 함수 | 하는 일 | 포인트 |
| --- | --- | --- |
| `list_image_files(folder)` | 폴더 속 이미지 파일명 **정렬 리스트** | `f.lower().endswith(IMAGE_EXTS)` / 폴더 없으면 `[]` |
| `show_image_grid(paths, captions)` | 썸네일을 가로로 이어 붙여 출력 | `Image.new()` 흰 캔버스 → `paste()` (matplotlib 없이 PIL만) |
| `guess_mime(path)` | 확장자로 MIME 추정 | `.png` → `image/png`, 나머지 → `image/jpeg` |

---

## 4. 데이터 준비 — Fashionpedia 스트리밍

```python
dataset = load_dataset("detection-datasets/fashionpedia", split="train", streaming=True)
for i, example in enumerate(dataset.take(num_images)):         # 앞에서 n개만 꺼냄
    path = os.path.join(dataset_folder, f"image_{i+1:04d}.jpg")  # 0 패딩
    example["image"].convert("RGB").save(path, format="JPEG", quality=90)
```

- `streaming=True` → 결과가 `IterableDataset` → `ds[0]` 대신 **for / `.take(n)`**
- `example["image"]` 는 자동으로 **PIL.Image** 객체
- `image_0001.jpg` **0 패딩** → 문자열 정렬 = 번호 순서 (`image_1, image_10, image_100 …` 문제 해결)
- **JPEG 저장** → PNG보다 작음 → base64 전송량도 줄어듦
- 이미 저장된 이미지 수 ≥ `NUM_IMAGES` 이면 다운로드 **건너뜀**

---

## 5. 임베딩 & 벡터 DB — CLIP + Chroma (가장 중요)

```python
image_vdb = chromadb.PersistentClient(VDB_PATH).get_or_create_collection(
    name=COLLECTION_NAME,
    embedding_function=OpenCLIPEmbeddingFunction(),  # ViT-B-32, 첫 실행 시 약 600MB 다운로드
    data_loader=ImageLoader(),                       # 경로 → 이미지 배열
)
```

**`add(uris=...)` 내부 순서 (꼭 기억)**
```
이미지 경로 → ImageLoader → 이미지 배열(NumPy) → CLIP 이미지 인코더 → 512차원 벡터 → img_vdb 에 저장
```

**`query(query_texts=...)` 내부 순서**
```
텍스트 → CLIP 텍스트 인코더 → 저장된 이미지 벡터들과 거리 계산 → 가까운 순 반환
```

**중복 없이 등록하기**
- `image_vdb.get(include=[])["ids"]` → id 만 가볍게 조회 → `set` 으로 이미 있는 것 제외
- `batch_size=50` 장씩 나눠 `add` → 진행 상황 확인 + 메모리 절약

**검색 결과 구조 — 2중 리스트**
```python
{'ids':       [['image_0042', 'image_0310']],
 'uris':      [['.../image_0042.jpg', '.../image_0310.jpg']],
 'distances': [[1.52, 1.55]]}
# 바깥 = 검색어 순서, 안쪽 = 순위 순서 → r["uris"][0] 으로 첫 검색어 결과 꺼냄
```

**거리 읽는 법:** CLIP 벡터는 길이 1로 정규화 → `L2² = 2 − 2 × 코사인유사도` → 순위는 코사인 유사도와 같다.

> 💡 **한국어로 검색하면?** `"빨간 드레스"` 로 검색하면 결과가 엉뚱해진다. CLIP 은 주로 **영어**로 학습됐기 때문 → 그래서 질문을 **영어로 번역 후 검색**한다.

---

## 6. LangChain 체인 ① — 번역 체인

```python
translation_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a professional translator. Translate the following text to {target_lang}. "
               "Keep markdown formatting. Output only the translation."),
    ("user", "{text}"),
])
translation_chain = translation_prompt | ChatOpenAI(model=TRANSLATION_MODEL, temperature=0.0) | StrOutputParser()
translation_chain.invoke({"text": "데이트할 때 입을 빨간 원피스", "target_lang": "English"})
```

**LCEL 단계별 출력 타입**
```
dict ─▶ ChatPromptTemplate ─▶ [SystemMessage, HumanMessage] ─▶ ChatOpenAI ─▶ AIMessage ─▶ StrOutputParser ─▶ str
```

### 💡 템플릿 변수 처리 → 왜 재사용성이 높아지나?

- `{target_lang}`, `{text}` 는 **빈칸(자리표시자)**. 체인을 만들 때는 비워 두고, `invoke()` 할 때 dict 로 채운다.
- **원본 코드:** `translate()` 를 부를 때마다 모델·프롬프트·체인을 **새로 생성** + 목표 언어도 고정
- **개선 코드:** 체인을 **한 번만** 만들고, 언어까지 변수로 받음 → 같은 체인 하나로
  - 한→영: `{"text": 질문, "target_lang": "English"}`
  - 영→한: `{"text": 답변, "target_lang": "Korean"}`
- 장점
  1. **코드 중복 제거** — 언어마다 프롬프트를 따로 만들 필요 없음
  2. **효율** — 객체를 매번 새로 만들지 않음
  3. **관리 쉬움** — 프롬프트 문구를 고칠 곳이 한 군데
  4. **안전** — f-string 으로 직접 끼워 넣는 것보다 구조(역할·메시지)가 유지됨
- 비유: **양식지(템플릿)** 를 한 번 만들어 두고 **빈칸만 바꿔 쓰는 것**.

---

## 7. LangChain 체인 ② — 비전 체인 (텍스트 + 이미지 2장)

비전 모델은 user 메시지에 **콘텐츠 블록 리스트**를 받는다.

```python
("user", [
    {"type": "text",      "text": "What are some ideas for styling {user_query}"},
    {"type": "image_url", "image_url": {"url": "data:{image_mime_1};base64,{image_data_1}"}},
    {"type": "image_url", "image_url": {"url": "data:{image_mime_2};base64,{image_data_2}"}},
])
```

- 이미지 URL 도 **템플릿 변수**로 → MIME 까지 변수 처리 (원본은 PNG인데 `image/jpeg` 고정 → 수정)
- system: 패션 스타일링 보조자, **이미지의 특정 부분을 직접 언급**, 대화체, 마크다운 사용

**base64 & data URL 만들기**
```python
with open(path, "rb") as f:                                  # 'rb' = 바이너리 읽기
    b64 = base64.b64encode(f.read()).decode("utf-8")         # bytes → base64 bytes → str
url = f"data:image/jpeg;base64,{b64}"
```

**MIME 타입**
- 파일 형식을 알려주는 이름 (이미지·영상·음성 등) — `image/png`, `image/jpeg`, `audio/mpeg`, `video/mp4`
- 통신 규약(인터넷 표준)으로 **이름이 고정**되어 있다 → 실제 파일 형식과 **맞춰야** 함

**`format_prompt_inputs(results, query)`** — 검색 결과 상위 2장 → `{"user_query", "image_data_1", "image_mime_1", ...}` dict
- 결과가 2장 미만이면 `IndexError` 대신 **친절한 `ValueError`**

---

## 8. 전체 파이프라인 조립

| 단계 | 함수 | RAG 역할 |
| --- | --- | --- |
| ① 질문 번역 | `translate(query_ko, "English")` | 전처리 |
| ② 이미지 검색 | `query_db(image_vdb, query_en)` | **R**etrieval |
| ③ 입력 구성 | `format_prompt_inputs(results, query_en)` | **A**ugmentation |
| ④ 답변 생성 | `vision_chain.invoke(prompt_input)` | **G**eneration |
| ⑤ 답변 번역 | `translate(response_en, "Korean")` | 후처리 |

- **번역을 두 번 하는 이유:** CLIP 검색 품질 & LLM 이해도는 영어가 좋고, 사용자에겐 한국어로 보여줘야 하니까.
- 대화형 모드(`while` + `input()`)는 `RUN_INTERACTIVE = False` 기본값 → 셀 전체 실행 시 멈춘 것처럼 보이지 않게.
- `try / except` 로 API 오류가 나도 루프가 죽지 않게 처리.

> ※ Part 6(실습 문제 1~4, 퀴즈)는 수업 중 진행하지 않음.

---

## 9. 더 나아가기

- **다국어 CLIP** (`xlm-roberta-base-ViT-B-32`) → 번역 없이 한국어로 바로 검색
- **메타데이터 필터** → 카테고리를 `metadatas` 로 저장, `where={"category": "dress"}`
- **대화 기억** → `RunnableWithMessageHistory`
- **웹 UI** → Gradio / Streamlit 으로 `ask_fashion_rag()` 감싸기

---

## 🧭 오늘의 전체 흐름 — 한 번에 보는 코드

```python
# ============================================================
# 0. 준비: 라이브러리 + API 키
# ============================================================
# %pip -q install chromadb open-clip-torch pillow datasets langchain-openai langchain-core python-dotenv
import os, base64
from getpass import getpass
from dotenv import load_dotenv

import chromadb
from chromadb.utils.embedding_functions import OpenCLIPEmbeddingFunction  # 텍스트/이미지 → CLIP 벡터
from chromadb.utils.data_loaders import ImageLoader                       # 경로 → 이미지 배열(NumPy)
from datasets import load_dataset                                         # Hugging Face 데이터셋

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

from PIL import Image
from IPython.display import display, Markdown

load_dotenv()                                           # .env → 환경변수
if not os.getenv("OPENAI_API_KEY"):                     # 없으면 직접 입력 (화면에 안 보임)
    os.environ["OPENAI_API_KEY"] = getpass("OpenAI API 키: ").strip()


# ============================================================
# 1. 설정값: 여기만 고치면 전체 동작이 바뀐다
# ============================================================
BASE_DIR        = os.getcwd()                               # 노트북엔 __file__ 없음 → 현재 폴더 기준
DATASET_FOLDER  = os.path.join(BASE_DIR, "fashion_dataset") # 이미지 저장 폴더
VDB_PATH        = os.path.join(BASE_DIR, "img_vdb")         # Chroma DB 저장 폴더
COLLECTION_NAME = "fashion_images"                          # 컬렉션(=테이블) 이름
NUM_IMAGES = 100                                            # 가져올 이미지 수
N_RESULTS  = 2                                              # 질문당 검색할 이미지 수
TRANSLATION_MODEL = "gpt-4o-mini"                           # 번역용
VISION_MODEL      = "gpt-4o-mini"                           # 이미지를 이해하는 비전 모델
IMAGE_EXTS = (".jpg", ".jpeg", ".png")


# ============================================================
# 2. 보조 함수: 파일 목록 / 썸네일 출력 / MIME 추정
# ============================================================
def list_image_files(folder):
    """폴더 속 이미지 파일명을 정렬된 리스트로 (폴더 없으면 [])"""
    if not os.path.isdir(folder):
        return []
    return sorted(f for f in os.listdir(folder) if f.lower().endswith(IMAGE_EXTS))

def show_image_grid(paths, captions=None, thumb=220):
    """여러 이미지를 가로로 이어 붙여 한 장으로 출력"""
    thumbs = []
    for p in paths:
        img = Image.open(p).convert("RGB")
        img.thumbnail((thumb, thumb))                   # 비율 유지하며 축소
        thumbs.append(img)
    gap = 10
    width  = sum(t.width for t in thumbs) + gap * (len(thumbs) - 1)
    height = max(t.height for t in thumbs)
    canvas = Image.new("RGB", (width, height), "white") # 흰 도화지
    x = 0
    for t in thumbs:
        canvas.paste(t, (x, 0))                         # 왼쪽부터 차례로 붙이기
        x += t.width + gap
    display(canvas)
    if captions:
        print(" | ".join(captions))

def guess_mime(path):
    """확장자 → MIME 타입 (data URL 에 사용)"""
    return "image/png" if os.path.splitext(path)[1].lower() == ".png" else "image/jpeg"


# ============================================================
# 3. 데이터 준비: Fashionpedia 를 스트리밍으로 N장만 저장
# ============================================================
if len(list_image_files(DATASET_FOLDER)) < NUM_IMAGES:             # 이미 있으면 건너뜀
    os.makedirs(DATASET_FOLDER, exist_ok=True)
    dataset = load_dataset("detection-datasets/fashionpedia",
                           split="train", streaming=True)          # 전체(3.5GB) 대신 필요한 만큼만
    for i, example in enumerate(dataset.take(NUM_IMAGES)):         # 앞에서 N개만
        path = os.path.join(DATASET_FOLDER, f"image_{i+1:04d}.jpg")  # 0 패딩 → 정렬 = 번호 순
        example["image"].convert("RGB").save(path, format="JPEG", quality=90)

image_files = list_image_files(DATASET_FOLDER)
show_image_grid([os.path.join(DATASET_FOLDER, f) for f in image_files[:3]], image_files[:3])


# ============================================================
# 4. 벡터 DB: CLIP 임베딩 함수 + 이미지 로더를 연결한 Chroma 컬렉션
# ============================================================
image_vdb = chromadb.PersistentClient(VDB_PATH).get_or_create_collection(  # 디스크 저장 / 있으면 로드
    name=COLLECTION_NAME,
    embedding_function=OpenCLIPEmbeddingFunction(),   # ViT-B-32, 512차원
    data_loader=ImageLoader(),                        # uris → 실제 이미지
)

# 이미지 등록: 경로 → ImageLoader → NumPy 배열 → CLIP 이미지 인코더 → 512차원 벡터 → 저장
ids  = [os.path.splitext(f)[0] for f in image_files]             # 'image_0001.jpg' → 'image_0001'
uris = [os.path.join(DATASET_FOLDER, f) for f in image_files]    # 전체 경로
existing_ids = set(image_vdb.get(include=[])["ids"])             # id 만 가볍게 조회
new_items = [(i, u) for i, u in zip(ids, uris) if i not in existing_ids]  # 새 이미지만

for start in range(0, len(new_items), 50):                       # 50장씩 나눠서 등록
    batch = new_items[start:start + 50]
    image_vdb.add(ids=[i for i, _ in batch], uris=[u for _, u in batch])
print("저장된 이미지 수:", image_vdb.count())


# ============================================================
# 5. 검색: 텍스트 → CLIP 텍스트 인코더 → 가장 가까운 이미지
# ============================================================
def query_db(image_vdb, query, results=N_RESULTS):
    return image_vdb.query(
        query_texts=[query],                  # 리스트로 → 결과도 2중 리스트
        n_results=results,
        include=["uris", "distances"],        # 경로 + 거리 (ids 는 항상 포함)
    )

def print_results(results):
    uris = results["uris"][0]                 # [0] = 첫 번째 검색어의 결과
    for rank, uri in enumerate(uris):
        print(f"[{rank+1}위] {results['ids'][0][rank]}  distance={results['distances'][0][rank]:.4f}")
    show_image_grid(uris, [f"{r+1}위" for r in range(len(uris))])

print_results(query_db(image_vdb, "a red dress", results=3))   # 영어로 검색해야 잘 됨


# ============================================================
# 6. 번역 체인 (LCEL): 한 번 만들고 변수만 바꿔 재사용
# ============================================================
translation_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a professional translator. "
               "Translate the following text to {target_lang}. "      # ← 언어도 변수
               "Keep markdown formatting. Output only the translation."),
    ("user", "{text}"),                                               # ← 번역할 글도 변수
])
translation_chain = (translation_prompt                               # dict → 메시지 목록
                     | ChatOpenAI(model=TRANSLATION_MODEL, temperature=0.0)  # → AIMessage
                     | StrOutputParser())                             # → str

def translate(text, target_lang):
    return translation_chain.invoke({"text": text, "target_lang": target_lang})


# ============================================================
# 7. 비전 체인: 질문 텍스트 + 이미지 2장(data URL)
# ============================================================
image_prompt = ChatPromptTemplate.from_messages([
    ("system",
     "You are a helpful fashion and styling assistant. "
     "Answer the user's question using the given image context with direct references "
     "to parts of the images provided. Maintain a more conversational tone, "
     "don't make too many lists. Use markdown formatting for highlights, emphasis, and structure."),
    ("user", [                                                         # 콘텐츠 블록 리스트
        {"type": "text", "text": "What are some ideas for styling {user_query}"},
        {"type": "image_url", "image_url": {"url": "data:{image_mime_1};base64,{image_data_1}"}},
        {"type": "image_url", "image_url": {"url": "data:{image_mime_2};base64,{image_data_2}"}},
    ]),
])
vision_chain = image_prompt | ChatOpenAI(model=VISION_MODEL, temperature=0.0) | StrOutputParser()

def encode_image(path):
    """이미지 파일(bytes) → base64 문자열(str)"""
    with open(path, "rb") as f:                                        # 바이너리 읽기
        return base64.b64encode(f.read()).decode("utf-8")

def format_prompt_inputs(data, user_query):
    """검색 결과 상위 2장 + 질문 → 비전 프롬프트의 빈칸을 채울 dict"""
    uris = data["uris"][0]
    if len(uris) < 2:
        raise ValueError(f"이미지가 최소 2장 필요합니다. (검색 결과 {len(uris)}장)")
    inputs = {"user_query": user_query}
    for n, path in enumerate(uris[:2], start=1):                       # n = 1, 2
        inputs[f"image_data_{n}"] = encode_image(path)
        inputs[f"image_mime_{n}"] = guess_mime(path)
    return inputs


# ============================================================
# 8. 전체 파이프라인: 번역 → 검색(R) → 입력 구성(A) → 생성(G) → 번역
# ============================================================
def ask_fashion_rag(query_ko, n_results=N_RESULTS):
    query_en     = translate(query_ko, "English")                  # ① 한 → 영 (CLIP 은 영어에 강함)
    results      = query_db(image_vdb, query_en, n_results)        # ② Retrieval
    prompt_input = format_prompt_inputs(results, query_en)         # ③ Augmentation
    response_en  = vision_chain.invoke(prompt_input)               # ④ Generation
    response_ko  = translate(response_en, "Korean")                # ⑤ 영 → 한

    print("번역된 검색어:", query_en)
    print_results(results)                                         # 검색된 사진
    display(Markdown(response_ko))                                 # 마크다운 렌더링
    return {"query_en": query_en, "results": results,
            "response_en": response_en, "response_ko": response_ko}

answer = ask_fashion_rag("가을에 입기 좋은 트렌치코트 스타일링 알려줘")
```
