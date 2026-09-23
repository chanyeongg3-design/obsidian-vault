# 📚 TIL - 외부 LLM API 호출 안정화

## 1. 오늘 공부한 것

- **Timeout**
    
- **Retry**
    
- **외부 API 오류 처리**
    
- **HTTP 502 / 429**
    
- **FastAPI Middleware**
    
- **요청 로그**
    

---

## 2. 핵심 개념 정리

### 1) Timeout

**정의**  
외부 API가 너무 오래 응답하지 않을 때 기다리는 시간을 제한하는 기능이다.

```
async with httpx.AsyncClient(timeout=2.0) as client:
    response = await client.post(
        api_url,
        json={"prompt": prompt},
    )
```

**왜 필요한가?**

외부 LLM API가 멈추거나 느려졌는데 계속 기다리면 우리 FastAPI 서버도 요청을 끝내지 못한다.

즉,

```
외부 LLM 응답 지연
        ↓
우리 서버도 계속 대기
        ↓
사용자도 계속 기다림
```

Timeout을 설정하면 일정 시간이 지나면 실패로 처리할 수 있다.

**주의**

`timeout=2.0`은 이번 수업용 값이다. 실제 서비스에서는 정상 응답 시간을 측정한 뒤 결정해야 한다.

---

### 2) Retry

**정의**  
일시적인 오류가 발생했을 때 요청을 다시 보내는 것이다.

이번 강의에서는:

- Timeout → 한 번 재시도
    
- 5xx → 한 번 재시도
    
- 4xx → 재시도하지 않음
    
- 429 → 즉시 재시도하지 않음
    

으로 처리한다.

```
for attempt in range(2):
    try:
        response = await client.post(
            api_url,
            json={"prompt": prompt},
        )

    except httpx.TimeoutException:
        if attempt == 0:
            await asyncio.sleep(0.01)
            continue

        raise ExternalServiceError(
            "외부 API 응답 시간이 초과되었습니다."
        )
```

`range(2)`이므로:

```
첫 번째 호출
   ↓ 실패
한 번 재시도
```

최대 호출 횟수는 **2번**, 실제 재시도는 **1번**이다.

---

### 3) 외부 API 오류 처리

외부 서비스에서 발생한 문제를 하나의 예외로 묶는다.

```
class ExternalServiceError(RuntimeError):
    """외부 API 실패를 표현합니다."""
```

예를 들어:

- Timeout
    
- 네트워크 연결 실패
    
- 5xx
    
- 4xx
    
- 잘못된 JSON
    
- `answer`가 없는 응답
    

등을 `ExternalServiceError`로 바꾼다.

FastAPI에서는 이것을:

```
try:
    answer = await call_with_one_retry(
        client,
        api_url,
        prompt,
    )

except ExternalServiceError as error:
    raise HTTPException(
        status_code=502,
        detail=str(error),
    )
```

처럼 **502 Bad Gateway**로 변환한다.

### 상태 코드 구분

|상태 코드|의미|
|---|---|
|`422`|사용자가 보낸 요청 형식이 잘못됨|
|`502`|우리 서버가 외부 API에서 정상 답변을 받지 못함|
|`429`|너무 많은 요청으로 외부 서비스가 호출을 제한함|

---

### 4) Middleware와 요청 로그

**Middleware**는 요청이 Endpoint에 들어가기 전후에 공통 작업을 수행할 수 있는 기능이다.

이번에는 다음 세 가지만 기록한다.

- HTTP Method
    
- Path
    
- Status Code
    

```
@app.middleware("http")
async def request_log_middleware(
    request: Request,
    call_next,
):
    response = await call_next(request)

    make_request_log(
        request.method,
        request.url.path,
        response.status_code,
    )

    return response
```

로그 예:

```
POST /query -> 200
POST /query -> 502
```

### 왜 필요한가?

서비스에서 문제가 발생했을 때

```
어떤 API 요청이 들어왔는지
↓
어떤 상태 코드로 끝났는지
```

확인할 수 있다.

단, 다음 값은 로그에 남기지 않는다.

- 전체 `question`
    
- 전체 `context`
    
- API Key
    
- Authorization Header
    

민감한 정보가 포함될 수 있기 때문이다.

---

## 3. 전체 구조에서의 위치

이번 강의는 이전에 만든 LLM Query API에 **안정성 기능을 추가하는 단계**이다.

|단계|구성 요소|역할|
|---|---|---|
|1|사용자|질문과 context 전송|
|2|FastAPI|`/query` 요청 수신|
|3|Pydantic|입력 검증|
|4|Prompt|질문 + 근거 구성|
|5|AsyncClient|외부 LLM API 호출|
|6|Timeout|너무 긴 대기 차단|
|7|Retry|일시적 실패 한 번 재시도|
|8|Error Handling|외부 실패를 502로 변환|
|9|Middleware|요청 결과 로그 기록|
|10|사용자|최종 답변 또는 오류 수신|

```
[사용자]
   ↓
[FastAPI /query]
   ↓
[Pydantic 검증]
   ↓
[Prompt 생성]
   ↓
[외부 LLM API 호출]
   ↓
┌─────────────────────┐
│ Timeout 설정        │
│ 5xx/Timeout Retry   │
│ 오류 처리           │
└─────────┬───────────┘
          ↓
     [LLM Answer]
          ↓
     [QueryResponse]
          ↓
       [사용자]

Middleware
→ 요청 Method / Path / Status 기록
```

---

## 4. Retry 판단 기준

이 부분은 꼭 기억하면 좋다.

|상황|재시도|이유|
|---|---|---|
|Timeout|O|일시적인 지연일 수 있음|
|500 / 503|O|서버의 일시적 오류일 수 있음|
|400|X|요청 자체가 잘못됨|
|401 / 403|X|인증·권한 문제|
|404|X|주소 또는 자원이 없음|
|429|X|호출 제한 정책 문제|

특히 **429는 서버 장애가 아니다.**

```
429
→ 너무 많은 요청
→ Rate Limit
```

실제 서비스에서는 `Retry-After` 같은 공급자 정책을 확인해야 한다.

---

## 5. 실제 LLM 프로젝트에서는 어떻게 사용되는가?

LLM API는 일반 API보다 응답 시간이 길거나 외부 서비스 상태에 영향을 받을 수 있다.

따라서:

```
사용자 질문
    ↓
FastAPI
    ↓
LLM API 호출
    ↓
Timeout
    ↓
필요하면 Retry
    ↓
성공 → 답변 반환
실패 → 502 반환
```

같은 구조가 필요하다.

즉, 이번 강의는 **LLM을 호출하는 방법**보다 한 단계 더 나아가,

> **LLM API가 느리거나 실패하더라도 우리 서비스가 예측 가능하게 동작하도록 만드는 방법**

을 배운 것이다.

---

## 6. 같이 알아두면 좋은 개념

- **5xx**: 외부 서버 측 오류
    
- **4xx**: 요청·인증·자원 등에 문제가 있는 상태
    
- **429**: 요청 횟수 제한
    
- **Exponential Backoff**: 재시도할수록 대기 시간을 늘리는 방식
    
- **Middleware**: 여러 Endpoint에 공통 기능을 적용하는 계층
    
- **Logging**: 실행 상태와 오류를 기록하는 과정
    

---

## 7. 복습용 핵심 정리

**Timeout**  
→ 외부 API를 무한정 기다리지 않음  
→ LLM 호출 안정성 확보

**Retry**  
→ Timeout이나 5xx를 한 번 다시 시도  
→ 일시적 오류 대응

**ExternalServiceError**  
→ 외부 서비스 오류를 하나의 예외로 통일  
→ FastAPI에서 502로 변환

**429**  
→ 너무 많은 요청으로 인한 Rate Limit  
→ 무조건 바로 재시도하지 않음

**Middleware**  
→ 요청 전후의 공통 작업  
→ Method, Path, Status Code 로그 기록

---

## 8. 내가 설명할 수 있어야 하는 질문

1. 외부 LLM API를 호출할 때 Timeout이 필요한 이유는 무엇인가?
    
2. Timeout과 5xx는 왜 재시도하지만 4xx는 바로 재시도하지 않는가?
    
3. 재시도 1번이라는 것은 실제 HTTP 호출이 최대 몇 번이라는 뜻인가?
    
4. 외부 LLM API가 실패했을 때 왜 502를 반환하는가?
    
5. 로그에 question, context, API Key를 남기지 않는 이유는 무엇인가?
    

---

## 9. 오늘의 한 줄

> **외부 LLM API 호출에는 Timeout, 제한된 Retry, 명확한 오류 처리, 최소한의 요청 로그를 적용해 서비스가 실패 상황에서도 안정적으로 동작하도록 해야 한다.**
