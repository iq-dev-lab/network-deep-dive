# HTTP 메서드와 상태 코드 — 멱등성과 안전성의 의미

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 멱등성(Idempotent)과 안전성(Safe)의 정확한 정의는 무엇인가?
- GET, POST, PUT, PATCH, DELETE 각각의 멱등성/안전성은?
- 멱등성이 클라이언트 재시도 전략에 어떤 영향을 미치는가?
- 200, 201, 204의 사용 상황은 각각 언제인가?
- 400, 422, 409의 차이는 무엇인가?
- 올바른 상태 코드 선택이 왜 API 계약(Contract)의 일부인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
멱등성이 실무에서 중요한 이유:

재시도 안전성:
  네트워크 타임아웃 발생 시 클라이언트는 재시도
  
  PUT /orders/123 (주문 수정):
    타임아웃 → 재시도 → 같은 결과 (안전)
    멱등성이 있으므로 재시도 OK
  
  POST /orders (주문 생성):
    타임아웃 → 재시도 → 중복 주문 생성 위험!
    멱등성이 없으므로 재시도 위험

  Kafka Consumer 재처리:
    at-least-once 전달 보장 → 중복 메시지 가능
    → 소비자 로직이 멱등해야 중복 처리 무해
    → 주문 ID로 중복 체크 후 처리

상태 코드가 잘못되면:
  잘못된 예:
    POST /users → 200 OK (사용자 생성)
    → 클라이언트: "이미 있는 건가? 새로 만든 건가?" 혼란
  
  올바른 예:
    POST /users → 201 Created + Location: /users/123
    → 명확: 새 리소스 생성됨, 위치 알 수 있음
  
  REST 클라이언트가 상태 코드를 기반으로 동작:
    201 → 응답 헤더의 Location으로 이동
    409 → "중복 요청이니 재시도 불필요"
    503 → "서버 문제, 나중에 재시도"
    429 → "Retry-After 헤더 확인 후 재시도"
```

---

## 😱 흔한 실수

```
Before — 메서드와 상태 코드를 모를 때:

실수 1: POST를 모든 변경 작업에 사용
  POST /users/{id}/update (수정)
  POST /users/{id}/delete (삭제)
  → REST 설계 위반, 멱등성 불명확
  → PUT/PATCH/DELETE 메서드가 이 목적에 존재

실수 2: 모든 응답에 200 OK
  사용자 생성 → 200 OK (201 Created 대신)
  삭제 성공  → 200 OK + body {"result": "deleted"} (204 대신)
  → 클라이언트가 생성/수정/삭제 구분 어려움

실수 3: 유효성 검사 오류에 500 Internal Server Error
  요청 파라미터 누락 → 500 (400 대신)
  → 클라이언트: "서버 버그다" 오판
  → 실제: 잘못된 요청 (클라이언트 수정 필요)
  → 400 Bad Request 또는 422 Unprocessable Entity

실수 4: 인증 오류에 403 사용 (401 대신)
  로그인 안 한 사용자 → 403 Forbidden (401 Unauthorized 대신)
  → 403: 인증됐지만 권한 없음
  → 401: 인증 자체가 안 됨 (로그인 필요)
```

---

## ✨ 올바른 접근

```
After — 메서드와 상태 코드를 알고 나면:

Spring REST Controller 올바른 설계:
  @RestController
  @RequestMapping("/api/users")
  public class UserController {

      @GetMapping("/{id}")
      public ResponseEntity<User> getUser(@PathVariable Long id) {
          return userService.findById(id)
              .map(ResponseEntity::ok)          // 200 OK
              .orElse(ResponseEntity.notFound().build()); // 404
      }

      @PostMapping
      public ResponseEntity<User> createUser(@Valid @RequestBody UserDto dto) {
          User created = userService.create(dto);
          URI location = URI.create("/api/users/" + created.getId());
          return ResponseEntity.created(location).body(created); // 201 Created
      }

      @PutMapping("/{id}")
      public ResponseEntity<User> updateUser(
              @PathVariable Long id, @RequestBody UserDto dto) {
          return ResponseEntity.ok(userService.update(id, dto)); // 200 OK
      }

      @DeleteMapping("/{id}")
      public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
          userService.delete(id);
          return ResponseEntity.noContent().build(); // 204 No Content
      }

      @PatchMapping("/{id}")
      public ResponseEntity<User> partialUpdate(
              @PathVariable Long id, @RequestBody Map<String, Object> updates) {
          return ResponseEntity.ok(userService.partialUpdate(id, updates));
      }
  }

재시도 전략 설계:
  @Retryable(
      include = {HttpServerErrorException.ServiceUnavailable.class}, // 503
      exclude = {HttpClientErrorException.BadRequest.class},          // 400 재시도 X
      maxAttempts = 3,
      backoff = @Backoff(delay = 1000, multiplier = 2)
  )
  public ResponseEntity<Data> callExternalApi() { ... }
```

---

## 🔬 내부 동작 원리

### 1. 안전성과 멱등성 정의

```
안전성(Safe):
  메서드 호출이 서버 상태를 변경하지 않음
  → 읽기 전용 (Side-effect free)
  → 얼마든지 호출 가능 (캐시, 프리페치 등)

멱등성(Idempotent):
  같은 요청을 여러 번 보내도 결과가 동일
  → 1번 실행 = 100번 실행의 서버 상태 결과
  → 재시도 안전 (단, 응답 자체는 다를 수 있음)
  → 수학적: f(f(x)) = f(x)

메서드별 분류:
┌─────────────────────────────────────────────────────────────────────┐
│  메서드     │  안전성  │  멱등성   │  설명                                │
├─────────────────────────────────────────────────────────────────────┤
│  GET      │  ✅ 안전 │  ✅ 멱등 │  읽기 전용, 여러 번 호출 OK              │
│  HEAD     │  ✅ 안전 │  ✅ 멱등 │  GET과 같지만 body 없음                │
│  OPTIONS  │  ✅ 안전 │  ✅ 멱등 │  지원 메서드 조회                       │
│  PUT      │  ❌ 위험 │  ✅ 멱등 │  전체 교체, 동일 결과 보장               │
│  DELETE   │  ❌ 위험 │  ✅ 멱등 │  삭제 (이미 없어도 결과 동일)             │
│  POST     │  ❌ 위험 │  ❌ 비멱등│  생성, 호출마다 새 리소스                │
│  PATCH    │  ❌ 위험 │  △ 조건부 │  부분 수정 (구현에 따라 다름)             │
└─────────────────────────────────────────────────────────────────────┘

멱등성 예시:

PUT /users/123 { "name": "Alice", "age": 30 }
  1번 호출: name=Alice, age=30
  5번 호출: name=Alice, age=30 (동일)
  → 멱등 ✅

DELETE /users/123
  1번 호출: users/123 삭제됨
  2번 호출: users/123 없음 (이미 삭제) → 404 or 204
  서버 상태는 동일 (없음) → 멱등 ✅
  응답 코드는 다를 수 있음 (204 or 404)

POST /orders { "product": "laptop", "qty": 1 }
  1번 호출: 주문 #100 생성
  2번 호출: 주문 #101 생성 (새 주문!)
  → 비멱등 ❌ (재시도 시 중복 주문)

PATCH /account/balance { "operation": "add", "amount": 100 }
  1번: balance = 1000 + 100 = 1100
  2번: balance = 1100 + 100 = 1200 (다름!)
  → 비멱등 ❌
  
  PATCH /account/balance { "balance": 1100 } (절대값으로)
  1번: balance = 1100
  2번: balance = 1100 (동일)
  → 멱등 ✅ (구현 방식에 따라 결정됨)
```

### 2. 멱등성과 재시도 전략

```
재시도 안전 판단:

네트워크 타임아웃 발생 시:
  서버가 요청을 처리했는지 알 수 없음
  → 처리O + 응답 손실 가능
  → 처리X + 연결 오류 가능

안전한 재시도 (멱등 메서드):
  GET:  재시도 → 같은 데이터 반환 (OK)
  PUT:  재시도 → 같은 상태로 수렴 (OK)
  DELETE: 재시도 → 이미 없으면 404 (허용 가능)

위험한 재시도 (비멱등 메서드):
  POST /orders: 재시도 → 중복 주문 생성 (위험!)
  해결책:
    ① Idempotency Key 패턴:
       클라이언트가 고유 키 생성 (UUID)
       POST /orders + Idempotency-Key: uuid-123
       서버: 동일 키 재전송 시 이전 응답 반환 (중복 처리 안 함)
    
    ② 서버측 중복 체크:
       주문 ID 또는 고유 제약으로 중복 방지
       두 번째 요청 → 409 Conflict

HTTP Retry-After 헤더:
  429 Too Many Requests:
    Retry-After: 60      ← 60초 후 재시도
    Retry-After: Sat, 28 Mar 2026 10:00:00 GMT
  
  503 Service Unavailable:
    Retry-After: 30      ← 30초 후 재시도

Spring @Retryable 설계:
  멱등 호출:
    @Retryable(include = {IOException.class, RestClientException.class})
    public User getUser(Long id) { ... }  // GET, 재시도 OK
  
  비멱등 호출:
    // 재시도 안 함 or Idempotency Key 적용
    public Order createOrder(OrderDto dto) { ... }  // POST
```

### 3. 상태 코드 완전 분류

```
2xx — 성공:

  200 OK:          요청 성공, body에 결과
  201 Created:     리소스 생성 성공, Location 헤더에 새 URL
  202 Accepted:    요청 수락됨, 처리는 비동기 (아직 완료 안 됨)
  204 No Content:  성공, 반환할 body 없음 (DELETE, 일부 PUT)
  206 Partial Content: Range 요청 성공 (파일 다운로드 중간부터)

3xx — 리다이렉션:

  301 Moved Permanently: 영구 이동 (검색 엔진이 새 URL 인덱싱)
  302 Found:              임시 이동 (POST → GET 리다이렉트에 사용)
  303 See Other:          POST 완료 후 GET으로 리다이렉트 (PRG 패턴)
  304 Not Modified:       캐시 유효 (ETag/Last-Modified 재검증)
  307 Temporary Redirect: 임시 이동, 메서드 유지 (POST → POST)
  308 Permanent Redirect: 영구 이동, 메서드 유지

4xx — 클라이언트 오류:

  400 Bad Request:         요청 형식 오류 (파싱 불가, 필수 파라미터 누락)
  401 Unauthorized:        인증 없음 (로그인 필요) — 이름이 Unauthorized지만 Authentication
  403 Forbidden:           인증됐지만 권한 없음 (Authorization)
  404 Not Found:           리소스 없음
  405 Method Not Allowed:  지원하지 않는 HTTP 메서드
  409 Conflict:            현재 상태와 충돌 (중복 생성, 버전 충돌)
  410 Gone:                리소스가 영구적으로 삭제됨 (404 vs 410)
  422 Unprocessable Entity: 형식은 맞지만 의미적 오류 (비즈니스 규칙 위반)
  429 Too Many Requests:   Rate Limit 초과

5xx — 서버 오류:

  500 Internal Server Error: 서버 내부 오류 (버그, 예외)
  501 Not Implemented:       서버가 해당 기능 미구현
  502 Bad Gateway:           업스트림 서버 오류 (Nginx → Tomcat 연결 실패)
  503 Service Unavailable:   서버 과부하 또는 점검 중
  504 Gateway Timeout:       업스트림 서버 타임아웃
  507 Insufficient Storage:  저장 공간 부족

400 vs 422 vs 409 상세 비교:
┌─────────────────────────────────────────────────────────────────────┐
│  상태 코드   │  상황                    │  예시                         │
├─────────────────────────────────────────────────────────────────────┤
│  400       │  요청 형식 자체가 잘못됨     │  JSON 파싱 오류,                │
│            │  (서버가 이해 불가)        │  필수 필드 누락                  │
├─────────────────────────────────────────────────────────────────────┤
│  422       │  형식은 맞지만             │  나이 -1, 만료된 날짜,           │
│            │  의미적으로 처리 불가        │  존재하지 않는 카테고리 ID         │
├─────────────────────────────────────────────────────────────────────┤
│  409       │  현재 서버 상태와 충돌       │  중복 이메일 회원가입,            │
│            │  (형식/의미는 맞음)         │  낙관적 락 버전 불일치            │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. PRG 패턴과 상태 코드

```
Post-Redirect-Get (PRG) 패턴:
  문제: POST 후 새로고침 → 중복 제출
  
  잘못된 흐름:
    Client → POST /orders (주문 생성)
    Server ← 200 OK + 주문 확인 페이지
    사용자: 새로고침
    Client → POST /orders (다시!) → 중복 주문!

  PRG 해결:
    Client → POST /orders (주문 생성)
    Server ← 303 See Other, Location: /orders/100
    Client → GET /orders/100 (리다이렉트)
    Server ← 200 OK + 주문 확인 페이지
    사용자: 새로고침
    Client → GET /orders/100 (GET은 멱등 → 중복 주문 없음)

상태 코드 선택 기준:
  동일 URL을 클라이언트가 북마크해야 하는가?
    예 → 200 OK
    아니오 (리다이렉트) → 3xx
  
  영구 이동인가?
    예 → 301 (검색엔진도 이동)
    아니오 → 302/307
  
  POST 후 GET 페이지로 이동?
    → 303 See Other

HATEOAS와 상태 코드:
  201 Created 응답 예시:
    HTTP/1.1 201 Created
    Location: /api/orders/123
    Content-Type: application/json
    
    {
      "id": 123,
      "status": "pending",
      "_links": {
        "self": "/api/orders/123",
        "cancel": "/api/orders/123/cancel",
        "payment": "/api/payments?orderId=123"
      }
    }
    
    → Location: 새 리소스 URL (클라이언트가 GET으로 확인 가능)
    → _links: 다음 가능한 행동 (HATEOAS)
```

### 5. Spring 예외-상태 코드 매핑

```
Spring @ExceptionHandler:

  @RestControllerAdvice
  public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)  // 400
    public ErrorResponse handleValidation(MethodArgumentNotValidException e) {
        List<String> errors = e.getBindingResult().getFieldErrors()
            .stream().map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.toList());
        return new ErrorResponse(400, "Validation failed", errors);
    }
    
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)  // 422
    public ErrorResponse handleConstraint(ConstraintViolationException e) {
        return new ErrorResponse(422, "Business rule violation", ...);
    }
    
    @ExceptionHandler(DuplicateKeyException.class)
    @ResponseStatus(HttpStatus.CONFLICT)  // 409
    public ErrorResponse handleDuplicate(DuplicateKeyException e) {
        return new ErrorResponse(409, "Resource already exists");
    }
    
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)  // 404
    public ErrorResponse handleNotFound(EntityNotFoundException e) {
        return new ErrorResponse(404, e.getMessage());
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)  // 403
    public ErrorResponse handleForbidden(AccessDeniedException e) {
        return new ErrorResponse(403, "Access denied");
    }
  }

문제 명세 (RFC 7807 — Problem Details):
  표준화된 에러 응답 형식:
  {
    "type": "https://example.com/problems/insufficient-stock",
    "title": "재고 부족",
    "status": 422,
    "detail": "요청 수량 5개 중 3개만 재고 있음",
    "instance": "/api/orders/123"
  }
  
  Spring Boot 3.x: ProblemDetail 클래스 내장 지원
```

---

## 💻 실전 실험

### 실험 1: HTTP 메서드 멱등성 직접 확인

```bash
# PUT 멱등성 확인 (같은 요청 3번)
for i in 1 2 3; do
  curl -X PUT https://httpbin.org/put \
    -H "Content-Type: application/json" \
    -d '{"name": "Alice"}' \
    -s | jq '.json'
done
# 3번 모두 동일한 결과

# DELETE 멱등성 확인 (없는 리소스 삭제)
curl -X DELETE https://httpbin.org/delete -s | jq '.url'
# 200 OK (httpbin은 항상 OK 반환)

# POST 비멱등성 확인 (서로 다른 요청 ID 생성)
for i in 1 2 3; do
  curl -X POST https://httpbin.org/post \
    -H "Content-Type: application/json" \
    -d '{"action": "create"}' \
    -s | jq '.headers["X-Request-Id"]' 2>/dev/null
done
```

### 실험 2: 상태 코드별 동작 확인

```bash
# 다양한 상태 코드 직접 반환받기
for code in 200 201 204 400 401 403 404 409 422 429 500 503; do
  response=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://httpbin.org/status/$code")
  echo "요청한 코드: $code, 받은 코드: $response"
done

# 201 Created + Location 헤더 확인
curl -v -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{}' 2>&1 | grep -iE "201|location|content-type"

# 304 Not Modified 조건부 요청
ETAG=$(curl -s -I https://httpbin.org/etag/myresource | \
  grep -i etag | awk '{print $2}' | tr -d '\r')
curl -v "https://httpbin.org/etag/myresource" \
  -H "If-None-Match: $ETAG" 2>&1 | grep "304\|200"
```

### 실험 3: Idempotency Key 패턴 시뮬레이션

```bash
# UUID 생성
IDEMPOTENCY_KEY=$(uuidgen)

# 같은 키로 두 번 POST (두 번째는 같은 응답 반환해야 함)
for i in 1 2; do
  echo "=== 요청 $i ==="
  curl -X POST https://httpbin.org/post \
    -H "Content-Type: application/json" \
    -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
    -d '{"amount": 1000, "recipient": "alice"}' \
    -s | jq '.headers["Idempotency-Key"]'
done
# 실제 서버에서는 같은 Key면 동일 응답 반환해야 함
```

### 실험 4: Spring Boot 상태 코드 테스트

```bash
# 정상 조회 (200)
curl -s http://localhost:8080/api/users/1 | jq .
# {"id": 1, "name": "Alice"}

# 존재하지 않는 리소스 (404)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/users/99999
# 404

# 유효성 검사 실패 (400)
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": ""}' \
  -s | jq .
# {"status": 400, "errors": ["name: 이름은 필수입니다"]}

# 중복 이메일 (409)
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Bob", "email": "alice@example.com"}' \
  -s | jq .
# {"status": 409, "message": "이미 사용 중인 이메일"}
```

---

## 📊 성능/비용 비교

```
잘못된 상태 코드의 비용:

클라이언트 재시도 동작 비교:

올바른 상태 코드 사용 시:
  500 Internal Server Error → 재시도 (서버 문제)
  503 Service Unavailable   → Retry-After 후 재시도
  400 Bad Request           → 재시도 안 함 (내 요청 문제)
  409 Conflict              → 재시도 안 함 (중복)
  429 Too Many Requests     → Retry-After 후 재시도

잘못된 상태 코드 사용 시:
  모든 오류에 500 → 클라이언트가 항상 재시도
  → 잘못된 요청(400)도 재시도 → 서버 불필요 부하
  
  모든 성공에 200 → 클라이언트가 생성/수정/삭제 구분 불가
  → 추가 API 호출 필요 (Location 헤더 없어서)

Idempotency Key 사용 비용:
  추가 Redis 조회: ~1ms
  Vs. 중복 주문 처리 비용: 수백ms + DB 오염 + 환불 처리
  → Idempotency Key 비용은 미미함
```

---

## ⚖️ 트레이드오프

```
상태 코드 선택의 트레이드오프:

400 vs 422:
  400 사용:   구현 단순, 의미 구분 없음
  422 사용:   의미 명확 (비즈니스 로직 오류 구분)
  RFC 9110 (HTTP 시맨틱스): 422는 WebDAV에서 유래, 일반 REST에서도 수용됨
  실무 권장: 400은 형식 오류, 422는 비즈니스 규칙 위반으로 구분

401 vs 403:
  401: 인증 없음 → 로그인 페이지로 리다이렉트 신호
  403: 권한 없음 → 다른 계정으로 로그인해도 소용없음
  잘못 쓰면: 보안 정보 노출 (존재하는 리소스인지 알 수 있음)
  극단적 보안: 모든 경우에 404 반환 (존재 여부 숨김)

멱등성 보장의 비용:
  Idempotency Key 저장: Redis/DB 추가 조회
  낙관적 락(Optimistic Lock): 충돌 시 409 반환 → 클라이언트 재시도 로직 필요
  at-least-once vs exactly-once: exactly-once는 구현 복잡도 높음

POST vs PUT 새 리소스 생성:
  POST /resources:  서버가 ID 생성 → 클라이언트가 ID 모름
  PUT /resources/id: 클라이언트가 ID 제공 → 멱등성 확보
  → UUID 클라이언트 생성 + PUT → 멱등한 리소스 생성
  → 재시도 안전 (서버가 멱등하게 처리)
```

---

## 📌 핵심 정리

```
HTTP 메서드와 상태 코드 핵심 요약:

안전성과 멱등성:
  안전(Safe): 서버 상태 변경 없음 (GET, HEAD, OPTIONS)
  멱등(Idempotent): 여러 번 호출해도 같은 결과 (GET, PUT, DELETE)
  POST: 비멱등 → 재시도 위험 → Idempotency Key 패턴 사용

메서드별 사용 기준:
  GET:    조회 (안전 + 멱등)
  POST:   생성 (비멱등)
  PUT:    전체 교체 (멱등)
  PATCH:  부분 수정 (멱등성은 구현 의존)
  DELETE: 삭제 (멱등)

주요 상태 코드:
  200 OK:        조회/수정 성공 (body 있음)
  201 Created:   생성 성공 + Location 헤더
  204 No Content: 성공 (body 없음, 삭제 등)
  304 Not Modified: 캐시 유효 (ETag 재검증)
  400 Bad Request:  요청 형식 오류
  401 Unauthorized: 인증 없음 (로그인 필요)
  403 Forbidden:    권한 없음 (로그인해도 불가)
  404 Not Found:    리소스 없음
  409 Conflict:     현재 상태와 충돌
  422 Unprocessable: 의미적 오류 (비즈니스 규칙)
  429 Too Many:     Rate Limit
  500 Server Error: 서버 버그
  503 Unavailable:  서버 다운/과부하

재시도 전략:
  5xx → 재시도 가능 (서버 문제)
  4xx → 재시도 불필요 (클라이언트 수정 필요)
  멱등 메서드(GET/PUT/DELETE) → 자유 재시도
  POST → Idempotency Key 없으면 재시도 금지
```

---

## 🤔 생각해볼 문제

**Q1.** REST API에서 `DELETE /users/123`을 호출했는데 해당 사용자가 없다. 어떤 상태 코드를 반환해야 하는가? 404와 204 중 어느 것이 더 올바른가?

<details>
<summary>해설 보기</summary>

**정답은 관점에 따라 다릅니다.** 두 가지 모두 허용되지만, 상황에 따라 선택합니다.

**204 No Content 반환 (멱등성 강조):**
- DELETE는 멱등해야 하므로, "이미 없는" 상태가 "삭제된" 상태와 동일하다고 보면 204
- 클라이언트가 "삭제 성공"으로 처리 가능
- 재시도 시 혼란 없음

**404 Not Found 반환 (리소스 존재 여부 강조):**
- 삭제하려는 리소스가 없다는 것을 명시
- 클라이언트가 "없는 걸 삭제하려 했다"는 피드백 수신
- 잘못된 ID 입력 등 클라이언트 버그 발견에 도움

**실무 권장:**
- 멱등성을 중시하는 API: 204 반환 (재시도 친화적)
- 명확한 피드백이 중요한 API: 404 반환 (개발자 경험)

**중요:** 어느 것을 선택하든 **문서화**가 핵심입니다. 클라이언트가 예측 가능하도록 일관성 있게 적용해야 합니다.

</details>

---

**Q2.** 낙관적 락(Optimistic Lock)을 사용하는 API에서 버전 충돌이 발생했을 때 어떤 상태 코드와 응답을 반환해야 하는가?

<details>
<summary>해설 보기</summary>

**409 Conflict**가 적절합니다. 현재 서버 상태(높은 버전)와 요청(낮은 버전)이 충돌하기 때문입니다.

**응답 예시 (RFC 7807 Problem Details):**
```json
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/optimistic-lock-failure",
  "title": "낙관적 락 충돌",
  "status": 409,
  "detail": "요청의 버전(2)이 서버 버전(3)과 다릅니다",
  "instance": "/api/products/123",
  "serverVersion": 3,
  "requestedVersion": 2
}
```

**클라이언트 처리 로직:**
```javascript
try {
  await api.put('/products/123', { ...data, version: currentVersion });
} catch (e) {
  if (e.status === 409) {
    // 최신 버전 다시 조회
    const fresh = await api.get('/products/123');
    // 변경사항 병합 후 재시도
    await api.put('/products/123', { ...merge(data, fresh), version: fresh.version });
  }
}
```

**Spring JPA에서:**
```java
@Version
private Long version;  // JPA가 자동으로 OptimisticLockException 발생

@ExceptionHandler(OptimisticLockException.class)
@ResponseStatus(HttpStatus.CONFLICT)
public ProblemDetail handleOptimisticLock(OptimisticLockException e) {
    ProblemDetail pd = ProblemDetail.forStatus(409);
    pd.setTitle("낙관적 락 충돌");
    pd.setDetail("다른 사용자가 먼저 수정했습니다. 최신 데이터를 조회 후 재시도하세요.");
    return pd;
}
```

</details>

---

**Q3.** PATCH 메서드가 "조건부 멱등"이라는 것의 의미는 무엇인가? 멱등한 PATCH와 비멱등한 PATCH의 예시를 들라.

<details>
<summary>해설 보기</summary>

**PATCH는 구현 방식에 따라 멱등할 수도 있고 아닐 수도 있습니다.**

**비멱등한 PATCH (연산 기반):**
```json
PATCH /account/123
{ "operation": "increment", "amount": 100 }
```
- 1번 호출: balance 1000 → 1100
- 2번 호출: balance 1100 → 1200
- **비멱등** ❌

**멱등한 PATCH (절대값 기반):**
```json
PATCH /users/123
{ "name": "Alice Kim" }
```
- 1번 호출: name → "Alice Kim"
- 2번 호출: name → "Alice Kim" (변경 없음)
- **멱등** ✅

**JSON Patch (RFC 6902) — 연산 기반, 대부분 비멱등:**
```json
PATCH /document/1
[
  { "op": "add", "path": "/tags/-", "value": "new-tag" }
]
```
- 1번: tags = ["tag1", "new-tag"]
- 2번: tags = ["tag1", "new-tag", "new-tag"] (중복!)
- **비멱등** ❌

**JSON Merge Patch (RFC 7396) — 대부분 멱등:**
```json
PATCH /users/123
Content-Type: application/merge-patch+json
{ "name": "Alice Kim", "email": null }
```
- null 필드는 삭제, 나머지는 설정
- 여러 번 적용해도 같은 결과
- **멱등** ✅ (대부분)

**실무 결론:** PATCH의 멱등성은 문서에 명시해야 합니다. 재시도가 필요한 환경에서는 멱등한 PATCH 설계를 선택하거나, Idempotency Key를 사용합니다.

</details>

---

<div align="center">

**[⬅️ 이전: HTTP 캐싱](./04-http-caching.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebSocket ➡️](./06-websocket-internals.md)**

</div>
