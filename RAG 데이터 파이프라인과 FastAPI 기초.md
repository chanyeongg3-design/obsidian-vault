# 📚 TIL - 2026-09-22

## 1. 오늘 공부한 것

- 데이터 정제
    
- RAG 문서 구조
    
- FastAPI / Uvicorn
    
- REST API와 Routing
    
- Path / Query / Body
    
- HTTP 상태 코드
    
- Pydantic 데이터 검증
    

---

## 2. 핵심 개념 정리

### 1) 데이터 정제

**정의**  
수집한 데이터의 **결측값, 중복, HTML, 불필요한 공백, 날짜 형식** 등을 일정한 규칙으로 정리하는 과정이다.

**왜 필요한가?**  
수집에 성공한 데이터라도 그대로 검색이나 RAG에 사용하기에는 형식이 일정하지 않을 수 있기 때문이다.

**어디에 쓰이는가?**  
RAG 문서, 학습 데이터, 크롤링 데이터, API 수집 데이터 등을 사용하기 전에 수행한다.

**역할**  
→ **원천 데이터를 AI 시스템에서 사용할 수 있는 데이터로 만드는 단계**

---

### 2) RAG 문서 구조

정제한 데이터를 RAG에서 사용하기 위해 다음처럼 나눈다.

|구성|역할|
|---|---|
|`document_id`|문서 하나를 구별하는 ID|
|`content`|의미 검색에 사용할 실제 본문|
|`metadata`|제목, 날짜, 출처, URL 등의 설명 정보|

예:

```
{
  "document_id": "doc-news-001",
  "content": "데이터를 안전하게 정제합니다.",
  "metadata": {
    "title": "첫 번째 소식",
    "source_url": "...",
    "published_at": "2026-06-01"
  }
}
```

**왜 필요한가?**  
검색할 내용과 문서를 설명하는 정보를 분리하기 위해서이다.

**RAG에서의 역할**

```
content
  ↓
Chunking
  ↓
Embedding
  ↓
Vector DB 검색
```

`metadata`는 검색 결과의 **출처 표시, 날짜 필터, 제목 표시** 등에 사용한다.

---

### 3) FastAPI와 Uvicorn

#### FastAPI

Python으로 **API를 만드는 프레임워크**이다.

```
@app.get("/health")
def read_health():
    return {"status": "ok"}
```

`GET /health` 요청이 들어오면 `read_health()` 함수를 실행하도록 연결한다.

#### Uvicorn

FastAPI 앱이 실제 네트워크에서 요청을 받을 수 있도록 실행하는 **ASGI 서버**이다.

|개념|역할|
|---|---|
|FastAPI|API의 경로와 처리 로직 정의|
|Uvicorn|HTTP 요청을 받아 FastAPI에 전달|

즉,

> **FastAPI는 API를 만들고, Uvicorn은 그 API를 실행한다.**

---

### 4) REST API와 Routing

REST API에서는 주로

- **URL Path → 어떤 자원인가**
    
- **HTTP Method → 그 자원에 무엇을 할 것인가**
    

를 나타낸다.

예:

|요청|의미|
|---|---|
|`GET /documents`|문서 조회|
|`POST /documents`|문서 생성|
|`GET /health`|서버 상태 확인|

**Routing**은 이런 `Method + Path` 조합을 Python 함수와 연결하는 과정이다.

```
GET /health
      ↓
read_health()
```

---

### 5) Path / Query / Body

FastAPI에서는 요청 데이터를 어디서 받는지 구분해야 한다.

|종류|역할|예|
|---|---|---|
|Path Parameter|어떤 자원인지 선택|`/documents/doc-news-001`|
|Query Parameter|조회 조건 조절|`?limit=12`|
|Request Body|구조화된 데이터 전달|JSON|

예:

```
GET /documents/doc-news-001?limit=12
```

여기서

- `doc-news-001` → **Path**
    
- `limit=12` → **Query**
    

그리고

```
{
  "title": "Pydantic 입문",
  "content": "..."
}
```

처럼 여러 값을 전달할 때는 **Body**를 사용한다.

---

### 6) HTTP 상태 코드

서버는 요청 결과를 상태 코드로 알려준다.

|코드|의미|
|---|---|
|`200`|정상 처리|
|`404`|요청한 자원이 없음|
|`422`|입력값이 API 규칙과 맞지 않음|

예:

```
존재하지 않는 문서
→ 404

limit=2
최소값이 5
→ 422
```

즉,

> **404는 자원 문제, 422는 입력값 문제**

---

### 7) Pydantic

**Pydantic**은 클라이언트가 보낸 JSON이 서버가 원하는 데이터 구조와 조건에 맞는지 검사한다.

예:

```
class DocumentCreate(BaseModel):
    title: str = Field(min_length=2)
    content: str = Field(min_length=10)
    source_url: str
```

이 모델은 다음을 검사할 수 있다.

- 필수 필드가 있는가?
    
- 타입이 맞는가?
    
- 문자열 길이가 맞는가?
    
- 기본값은 무엇인가?
    

잘못된 입력은 FastAPI에서 일반적으로 **422**로 처리된다.

### Request Model과 Response Model

|구분|역할|
|---|---|
|Request Model|클라이언트가 서버에 보내는 데이터|
|Response Model|서버가 클라이언트에 반환하는 데이터|

예를 들어

```
Request
title
content
source_url
```

서버가 처리한 뒤

```
Response
document_id
title
content
source_url
created_at
```

처럼 서버가 만든 값을 추가할 수 있다.

---

## 3. 전체 구조에서의 위치

|단계|구성 요소|역할|
|---|---|---|
|1|원천 데이터|데이터 수집|
|2|데이터 정제|결측·중복·HTML·날짜 정리|
|3|RAG Document|`content`, `metadata` 구조 생성|
|4|FastAPI|외부 요청을 받을 API 제공|
|5|Path / Query / Body|클라이언트 입력 전달|
|6|Pydantic|입력 데이터 검증|
|7|Endpoint|실제 Python 로직 실행|
|8|Response|결과를 JSON으로 반환|

```
[원천 데이터]
      ↓
[데이터 정제]
      ↓
[RAG Document]
content + metadata
      ↓
[FastAPI]
      ↓
[Path / Query / Body]
      ↓
[Pydantic 검증]
      ↓
[Python 로직]
      ↓
[JSON 응답]
```

---

## 4. 개념 간 차이

|개념|역할|
|---|---|
|FastAPI|API 구조와 기능 정의|
|Uvicorn|FastAPI 서버 실행|
|Path|특정 자원 선택|
|Query|조회 조건 설정|
|Body|구조화된 데이터 전달|
|content|RAG 검색 대상 본문|
|metadata|문서 설명 및 필터 정보|
|Request Model|들어오는 데이터 계약|
|Response Model|나가는 데이터 계약|

---

## 5. 실제 LLM 프로젝트에서는 어떻게 사용되는가?

오늘 배운 내용은 실제 RAG 서비스에서 다음 위치에 사용된다.

```
[사용자]
   ↓
[FastAPI]
   ↓
[Pydantic 검증]
   ↓
[Embedding]
   ↓
[Vector DB 검색]
   ↓
[관련 content 검색]
   ↓
[metadata로 출처 확인]
   ↓
[Prompt 구성]
   ↓
[LLM]
   ↓
[Response]
```

즉, **FastAPI와 Pydantic은 LLM 모델 자체보다는 LLM/RAG 기능을 실제 서비스로 연결하는 API 계층**에서 사용한다.

---

## 6. 같이 알아두면 좋은 관련 개념

- **HTTP**: 클라이언트와 서버가 통신하는 규칙
    
- **JSON**: API에서 많이 사용하는 데이터 전달 형식
    
- **OpenAPI**: API의 구조를 표현하는 표준
    
- **Swagger UI**: `/docs`에서 API를 확인하고 테스트하는 화면
    
- **Chunking**: 긴 문서를 작은 단위로 분리하는 과정
    
- **Embedding**: 텍스트를 의미를 가진 벡터로 변환
    
- **Vector DB**: Embedding을 저장하고 유사 문서를 검색
    
- **RAG**: 관련 문서를 검색한 뒤 LLM 답변 생성에 활용
    

---

## 7. 복습용 핵심 정리

**데이터 정제**  
→ 데이터를 사용할 수 있는 상태로 정리  
→ RAG 데이터 준비 단계

**RAG Document**  
→ `document_id + content + metadata` 구조  
→ 검색 가능한 문서 준비

**FastAPI**  
→ Python API 서버 프레임워크  
→ LLM/RAG 기능을 외부에 제공

**Uvicorn**  
→ FastAPI 실행 서버  
→ HTTP 요청 수신

**Path / Query / Body**  
→ API 입력 위치 구분  
→ 자원 선택 / 조건 설정 / 데이터 전달

**Pydantic**  
→ 요청 데이터 검증  
→ 잘못된 입력을 Endpoint 실행 전에 차단

---

## 8. 내가 설명할 수 있어야 하는 질문

1. 데이터 정제가 RAG 전에 필요한 이유는 무엇인가?
    
2. `content`와 `metadata`의 역할 차이는 무엇인가?
    
3. FastAPI와 Uvicorn의 차이는 무엇인가?
    
4. Path, Query, Body는 각각 언제 사용하는가?
    
5. Pydantic은 FastAPI에서 어떤 역할을 하는가?
    

---

## 9. 오늘의 한 줄

> **원천 데이터를 정제해 RAG 문서 구조로 만들고, FastAPI로 요청을 받은 뒤 Pydantic으로 검증하여 안전하게 처리하는 전체 흐름을 학습했다.**