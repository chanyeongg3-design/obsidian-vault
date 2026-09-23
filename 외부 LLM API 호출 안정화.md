# 📚 TIL - 2026-09-23

## 1. 오늘 공부한 것

- **문서 등록·조회 API**
    
- **Query API와 Prompt 구성**
    
- **Mock / Ollama / 외부 LLM 호출**
    
- **비동기 처리**
    
- **Timeout / Retry / 오류 처리**
    
- **Backoff / Fallback**
    

---

## 2. 핵심 개념 정리

### 1) 문서 API와 메모리 저장소

문서를 API로 등록하고 다시 조회하는 흐름을 학습했다.

주요 Endpoint는 다음과 같다.

|Method|Path|역할|
|---|---|---|
|`POST`|`/documents`|문서 등록|
|`GET`|`/documents`|전체 문서 조회|
|`GET`|`/documents/{document_id}`|특정 문서 조회|

문서는 Python 사전에 저장한다.

```
DOCUMENTS = {
    "doc-news-001": {
        "title": "첫 번째 소식",
        "content": "데이터를 안전하게 정제합니다."
    }
}
```

이번 메모리 저장소는 **학습용**이다.

- 코드가 단순해서 흐름을 이해하기 쉽다.
    
- 서버를 종료하면 데이터가