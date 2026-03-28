# HTTP/1.1 내부 구조 — Keep-Alive와 파이프라이닝의 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HTTP 요청/응답 메시지는 정확히 어떤 포맷으로 구성되는가?
- `Connection: keep-alive`는 TCP 레벨에서 정확히 무엇을 하는가?
- 파이프라이닝은 어떻게 동작하고 왜 실패했는가?
- HOL Blocking이 HTTP/1.1에서 발생하는 정확한 지점은 어디인가?
- `curl -v`로 보이는 헤더 각각의 의미는 무엇인가?
- 브라우저가 도메인당 6개 연결을 병렬로 여는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"HTTP/1.1에서 API 응답이 느린 이유를 설명하라":
  모르면: "서버 처리 시간이 오래 걸려서요"
  알면:   "HOL Blocking 때문입니다.
           요청 A의 응답이 오기 전까지 같은 연결에서
           요청 B를 받을 수 없어, 직렬 처리가 강제됩니다.
           브라우저는 이를 6개 병렬 연결로 우회합니다."

실무 연결고리:
  Spring RestTemplate 기본 사용:
    HTTP/1.1 + Keep-Alive → 연결 재사용
    하지만 Connection Pool 안 쓰면 → Handshake 반복

  CDN 설정:
    Origin Keep-Alive 설정 미스 → 매 요청마다 새 TCP
    → RTT × 1.5 추가 지연

  Nginx → Tomcat 연결:
    upstream keepalive 설정 안 하면:
    Nginx가 Tomcat과 매 요청마다 새 TCP 연결
    → keepalive 설정으로 연결 재사용 → latency 감소
    
    upstream backend {
        server 127.0.0.1:8080;
        keepalive 32;  ← 이 한 줄의 의미를 알려면 HTTP/1.1을 알아야 함
    }
```

---

## 😱 흔한 실수

```
Before — HTTP/1.1 내부를 모를 때:

실수 1: Content-Type 없이 POST 전송
  POST /api/data HTTP/1.1
  [body: {"key": "value"}]
  → 서버: Content-Type 없어서 body 파싱 실패 → 400
  → "서버 버그"로 오판
  Content-Type: application/json 필수

실수 2: Connection: close를 습관적으로 추가
  매 응답에 Connection: close → TCP 연결 강제 종료
  → Keep-Alive 효과 없음 → Handshake 반복
  → 불필요하게 느린 API

실수 3: Content-Length 없이 스트리밍 응답
  Content-Length를 모르면 클라이언트가 연결 종료 시점을 모름
  → chunked transfer encoding 필요
  Transfer-Encoding: chunked → 마지막 chunk 크기 0으로 종료 알림

실수 4: 파이프라이닝으로 성능 해결 시도
  "파이프라이닝 켜면 빨라지지 않나요?"
  → 서버 구현 복잡성 + Proxy 호환성 문제
  → 실제로 대부분의 브라우저가 비활성화
  → HTTP/2 멀티플렉싱이 올바른 해결책
```

---

## ✨ 올바른 접근

```
After — HTTP/1.1 구조를 알면:

Nginx upstream keepalive 설정:
  upstream backend {
      server 127.0.0.1:8080;
      keepalive 32;            ← 최대 32개 idle 연결 유지
      keepalive_requests 1000; ← 연결당 최대 1000 요청 후 재생성
      keepalive_timeout 60s;   ← idle 연결 유지 시간
  }
  → Nginx-Tomcat 간 Handshake 비용 제거

Spring RestTemplate Keep-Alive 활용:
  @Bean
  public RestTemplate restTemplate() {
      PoolingHttpClientConnectionManager cm =
          new PoolingHttpClientConnectionManager();
      cm.setMaxTotal(100);
      cm.setDefaultMaxPerRoute(20);
      // Keep-Alive 설정
      cm.setDefaultConnectionConfig(
          ConnectionConfig.custom()
              .setSocketTimeout(Timeout.ofSeconds(30))
              .build());
      CloseableHttpClient client = HttpClients.custom()
          .setConnectionManager(cm)
          .setKeepAliveStrategy((response, context) ->
              TimeValue.ofSeconds(30))  // Keep-Alive 30초
          .build();
      return new RestTemplate(
          new HttpComponentsClientHttpRequestFactory(client));
  }

응답 헤더 설계:
  Content-Type: application/json; charset=UTF-8  ← 필수
  Content-Length: 1234                           ← 고정 크기 응답
  Transfer-Encoding: chunked                     ← 스트리밍 응답
  Cache-Control: no-cache                        ← 캐시 정책
  Connection: keep-alive                         ← 연결 유지 (기본값)
```

---

## 🔬 내부 동작 원리

### 1. HTTP 메시지 포맷

```
HTTP/1.1 요청 메시지:

┌─────────────────────────────────────────────────────────────────┐
│  Start Line (Request Line)                                      │
│  GET /api/users?page=1 HTTP/1.1                                 │
│  [메서드] [Request-URI] [HTTP 버전]                                │
├─────────────────────────────────────────────────────────────────┤
│  Headers (각 헤더: 이름: 값\r\n)                                   │
│  Host: api.example.com                 ← HTTP/1.1 필수           │
│  User-Agent: Mozilla/5.0 ...                                    │
│  Accept: application/json                                       │
│  Accept-Encoding: gzip, deflate, br                             │
│  Accept-Language: ko-KR,ko;q=0.9                                │
│  Connection: keep-alive                ← 연결 유지 요청            │
│  Authorization: Bearer eyJhbGci...    ← 인증 토큰                 │
├─────────────────────────────────────────────────────────────────┤
│  빈 줄 (\r\n)                          ← 헤더 종료 신호             │
├─────────────────────────────────────────────────────────────────┤
│  Body (GET은 비어있음)                                             │
└─────────────────────────────────────────────────────────────────┘

POST 요청 예시:
  POST /api/users HTTP/1.1
  Host: api.example.com
  Content-Type: application/json          ← Body 형식 필수
  Content-Length: 45                      ← Body 크기 (bytes)
  Connection: keep-alive
  [빈 줄]
  {"name": "Alice", "email": "alice@ex.com"}

─────────────────────────────────────────────────────────────────

HTTP/1.1 응답 메시지:

┌─────────────────────────────────────────────────────────────────┐
│  Status Line                                                    │
│  HTTP/1.1 200 OK                                                │
│  [HTTP 버전] [Status Code] [Reason Phrase]                       │
├─────────────────────────────────────────────────────────────────┤
│  Headers                                                        │
│  Date: Sat, 28 Mar 2026 10:00:00 GMT                            │
│  Content-Type: application/json; charset=UTF-8                  │
│  Content-Length: 82                                             │
│  Connection: keep-alive                                         │
│  Keep-Alive: timeout=60, max=1000      ← Keep-Alive 파라미터      │
│  Cache-Control: no-cache                                        │
│  X-Request-Id: f3d7a820-...            ← 요청 추적 ID             │
├─────────────────────────────────────────────────────────────────┤
│  빈 줄 (\r\n)                                                    │
├─────────────────────────────────────────────────────────────────┤
│  Body                                                           │
│  {"id": 1, "name": "Alice", "email": "alice@ex.com", "age": 30} │
└─────────────────────────────────────────────────────────────────┘

메시지 경계 결정:
  방법 1: Content-Length: N → N bytes 읽으면 종료
  방법 2: Transfer-Encoding: chunked → 청크 단위로 읽음
  방법 3: Connection: close → 연결 종료 시 종료 (HTTP/1.0 방식)

Host 헤더가 HTTP/1.1에서 필수인 이유:
  가상 호스팅 (Virtual Hosting):
    하나의 IP에 여러 도메인 호스팅
    IP만으로는 어느 사이트인지 알 수 없음
    → Host 헤더로 구분
    example.com → Nginx → Host 보고 → site-a 또는 site-b
```

### 2. Keep-Alive — TCP 연결 재사용

```
HTTP/1.0 vs HTTP/1.1 연결 방식:

HTTP/1.0 (기본: 연결 종료):
  Client                Server
  ─ TCP Connect ──────► │
  ─ GET /index.html ───►│
  ◄─ 200 OK ──────────  │
  ─ TCP Close ─────────►│
  [다음 요청]
  ─ TCP Connect ──────► │  ← 다시 Handshake!
  ─ GET /style.css ────►│
  ◄─ 200 OK ──────────  │
  ─ TCP Close ─────────►│

  10개 리소스 → 10번 Handshake
  RTT=50ms: 10 × 75ms = 750ms 순수 연결 비용

HTTP/1.1 (기본: Keep-Alive):
  Client                Server
  ─ TCP Connect ──────► │  ← 1번만!
  ─ GET /index.html ───►│
  ◄─ 200 OK ──────────  │
  ─ GET /style.css ────►│  ← 같은 연결 재사용
  ◄─ 200 OK ──────────  │
  ─ GET /script.js ────►│  ← 계속 재사용
  ◄─ 200 OK ──────────  │
  ─ TCP Close ─────────►│  ← Keep-Alive timeout 초과 시

  10개 리소스 → 1번 Handshake
  RTT=50ms: 1 × 75ms = 75ms 연결 비용

Keep-Alive 동작 원리:
  클라이언트: Connection: keep-alive 헤더 전송
  서버: Connection: keep-alive + Keep-Alive: timeout=60 응답
  → 60초 동안 연결 유지
  → 60초 내 새 요청 없으면 서버가 FIN 전송

  Keep-Alive: timeout=60, max=1000
    timeout: idle 상태 유지 시간 (초)
    max: 이 연결에서 처리할 최대 요청 수

HTTP/1.1에서 Keep-Alive 비활성화:
  응답 헤더에 Connection: close 포함 시
  → 해당 응답 후 TCP 연결 종료
  → 다음 요청에서 새 TCP 연결 필요
```

### 3. 파이프라이닝과 HOL Blocking

```
파이프라이닝 시도:
  아이디어: ACK를 기다리지 않고 여러 요청을 연속 전송

  Client                Server
  ─ GET /a ────────────►│
  ─ GET /b ────────────►│  ← 응답 기다리지 않고 전송
  ─ GET /c ────────────►│  ← 연속 전송

  이상적 기대:
  ◄─ 200 /a (1ms) ────── │
  ◄─ 200 /b (1ms) ────── │
  ◄─ 200 /c (1ms) ────── │

  실제 문제 — HOL Blocking:
  서버는 요청 순서대로 응답해야 함 (RFC 요구사항)

  /a 처리: 100ms (느린 DB 쿼리)
  /b 처리: 1ms   (빠른 캐시 히트)
  /c 처리: 1ms

  Client                Server
  ─ GET /a ────────────►│  처리 시작 (100ms 소요)
  ─ GET /b ────────────►│  큐에서 대기
  ─ GET /c ────────────►│  큐에서 대기
  
  [100ms 대기 중...]
  
  ◄─ 200 /a ─────────── │  100ms 후 응답
  ◄─ 200 /b ─────────── │  즉시 (이미 완료됐지만 대기 중이었음)
  ◄─ 200 /c ─────────── │  즉시

  /b와 /c는 1ms면 됐는데 100ms를 기다림
  → Head-of-Line Blocking

왜 순서 보장이 필요한가:
  클라이언트가 어느 응답이 어느 요청에 대한 것인지 알려면
  요청 순서 = 응답 순서여야 함
  (HTTP/1.1에는 응답을 요청에 매핑하는 ID 없음)

파이프라이닝이 실패한 추가 이유:
  Proxy 서버 호환성:
    일부 Proxy가 파이프라인 요청을 올바르게 처리 못함
    → 응답이 섞이거나 유실
  
  서버 구현 복잡성:
    요청 취소(Cancel) 처리 어려움
    에러 복구 어려움
  
  결론: 대부분의 브라우저가 파이프라이닝을 비활성화

브라우저의 우회책 — 병렬 연결:
  같은 도메인에 최대 6개 TCP 연결 동시 사용
  → 6개 연결 × 1개 요청 = 6개 병렬 요청 가능
  → 그래도 각 연결 내부는 여전히 직렬 처리
  → 이것도 근본적 해결책이 아님 (HTTP/2가 해결)
```

### 4. Chunked Transfer Encoding

```
스트리밍 응답에서 Content-Length를 모를 때:

Transfer-Encoding: chunked 형식:
  HTTP/1.1 200 OK
  Transfer-Encoding: chunked
  Content-Type: text/plain
  
  1a\r\n               ← 청크 크기 (16진수, 26바이트)
  This is the first ch\r\n  ← 청크 데이터
  \r\n                 ← 청크 종료
  
  0d\r\n               ← 다음 청크 크기 (13바이트)
  unk of data.\r\n
  \r\n
  
  0\r\n                ← 크기 0 = 마지막 청크 (스트림 종료)
  \r\n

사용 사례:
  Spring Boot @RestController:
    응답 크기를 미리 알면: Content-Length 자동 설정
    ResponseEntity<StreamingResponseBody>:
      → chunked 자동 사용
  
  Server-Sent Events (SSE):
    Transfer-Encoding: chunked (또는 HTTP/2 스트림)
    → 서버에서 클라이언트로 이벤트 스트리밍

  LLM 응답 스트리밍:
    ChatGPT 등이 토큰별로 스트리밍
    Transfer-Encoding: chunked로 구현
    각 청크 = 생성된 토큰 일부
```

### 5. 주요 헤더 완전 분석

```
요청 헤더:
  Host: api.example.com
    → HTTP/1.1 필수. 가상 호스팅 지원
  
  Accept: application/json, text/html;q=0.9, */*;q=0.8
    → 클라이언트가 받을 수 있는 콘텐츠 타입 (q=품질값)
  
  Accept-Encoding: gzip, deflate, br
    → 압축 방식 지원 여부
    → 서버가 gzip으로 압축 → Content-Encoding: gzip
  
  If-Modified-Since: Sat, 28 Mar 2026 00:00:00 GMT
    → 이 시간 이후 변경됐으면 새 응답, 아니면 304
  
  If-None-Match: "etag-value-123"
    → 서버 ETag와 다르면 새 응답, 같으면 304

응답 헤더:
  Content-Type: application/json; charset=UTF-8
    → Body 형식 + 인코딩
    → charset 없으면 클라이언트가 추측 → 화둔깨짐
  
  Content-Encoding: gzip
    → Body가 gzip으로 압축됨 (클라이언트가 압축 해제)
  
  ETag: "abc123"
    → 리소스 버전 식별자 (조건부 요청에 사용)
  
  Vary: Accept-Encoding
    → 이 헤더값에 따라 캐시를 다르게 저장
    → Accept-Encoding이 다르면 다른 캐시 항목
  
  X-RateLimit-Remaining: 95
    → 남은 API 호출 횟수
  
  Retry-After: 30
    → 429 Too Many Requests 시 30초 후 재시도
```

---

## 💻 실전 실험

### 실험 1: curl -v로 HTTP/1.1 메시지 전체 확인

```bash
# HTTP/1.1 요청/응답 전체 헤더 확인
curl -v --http1.1 https://httpbin.org/get 2>&1

# 출력:
# *   Trying 54.157.211.80:443...
# * Connected to httpbin.org (54.157.211.80) port 443
# * SSL/TLS 협상 ...
#
# > GET /get HTTP/1.1            ← 요청 라인
# > Host: httpbin.org            ← 자동 추가
# > User-Agent: curl/7.88.1
# > Accept: */*                  ← 기본값
# >                              ← 빈 줄 (헤더 끝)
#
# < HTTP/1.1 200 OK              ← 상태 라인
# < Date: ...
# < Content-Type: application/json
# < Content-Length: 282
# < Connection: keep-alive       ← Keep-Alive 활성화 확인
```

### 실험 2: Keep-Alive 연결 재사용 확인

```bash
# 같은 연결에서 여러 요청 (Keep-Alive)
curl -v --http1.1 https://httpbin.org/get https://httpbin.org/headers 2>&1 | \
  grep -E "Connected|Re-using|Closing|HTTP/1"

# 첫 번째 URL: "Connected to httpbin.org"
# 두 번째 URL: "Re-using existing connection"  ← 연결 재사용!

# Keep-Alive 없이 (매번 새 연결)
curl -v --http1.1 -H "Connection: close" \
  https://httpbin.org/get https://httpbin.org/headers 2>&1 | \
  grep -E "Connected|Closing"
# 두 번 모두 "Connected" → 새 연결
```

### 실험 3: HOL Blocking 직접 측정

```bash
# 느린 응답을 먼저 요청 → 빠른 응답이 기다리는 현상 측정

# 서버: 응답 지연 시뮬레이션 (Python)
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import time

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/slow':
            time.sleep(2)  # 2초 지연
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b'slow response')
        else:
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b'fast response')
    def log_message(self, *args): pass

HTTPServer(('127.0.0.1', 8888), Handler).serve_forever()
" &

# HTTP/1.1 파이프라이닝 효과 시뮬레이션
# 느린 요청 후 빠른 요청: 총 2초+ 소요
time curl -s http://127.0.0.1:8888/slow http://127.0.0.1:8888/fast
# real: 2.003s → /fast가 /slow를 기다림 (HOL Blocking)

# 병렬 연결 (HOL Blocking 우회):
time (curl -s http://127.0.0.1:8888/slow &
      curl -s http://127.0.0.1:8888/fast &
      wait)
# real: 2.001s → 각자 독립 연결 → /fast는 즉시 완료
```

### 실험 4: 헤더 압축 효과

```bash
# Accept-Encoding으로 gzip 압축 요청
curl -v -H "Accept-Encoding: gzip" https://httpbin.org/get -o /tmp/compressed 2>&1 | \
  grep -E "Content-Encoding|Content-Length"
# Content-Encoding: gzip
# Content-Length: 215  ← 압축된 크기

# 압축 없이 요청
curl -v -H "Accept-Encoding: identity" https://httpbin.org/get 2>&1 | \
  grep "Content-Length"
# Content-Length: 350  ← 원본 크기

# 절감 비율 계산
echo "압축률: $(echo 'scale=1; (1 - 215/350) * 100' | bc)%"
# 압축률: 38.6%
```

---

## 📊 성능/비용 비교

```
HTTP/1.0 vs HTTP/1.1 vs 병렬 연결 성능:

시나리오: 웹 페이지 로딩, 리소스 10개, 각 50ms, RTT=50ms

HTTP/1.0 (매 요청 새 연결):
  각 요청: Handshake(75ms) + 응답(50ms) = 125ms
  총 시간: 10 × 125ms = 1,250ms

HTTP/1.1 Keep-Alive (직렬):
  첫 요청: Handshake(75ms) + 응답(50ms) = 125ms
  이후 요청: 응답(50ms)씩 × 9 = 450ms
  총 시간: 125 + 450 = 575ms

HTTP/1.1 병렬 연결 6개:
  6개 병렬: 처음 6개 → 125ms (Handshake+응답 병렬)
  나머지 4개 → 50ms (기존 연결 재사용)
  총 시간: 125 + 50 = 175ms

HTTP/2 (멀티플렉싱):
  1 TCP 연결: Handshake(75ms) + TLS(75ms) = 150ms
  10개 요청 병렬: max(50ms) = 50ms
  총 시간: 150 + 50 = 200ms
  (단, 패킷 손실 없는 환경에서)

Keep-Alive 유무:
  Keep-Alive 없음: 1,250ms
  Keep-Alive 있음: 575ms (2.2배 빠름)
  병렬 연결:       175ms (7.1배 빠름)
```

---

## ⚖️ 트레이드오프

```
Keep-Alive 설정:

Keep-Alive timeout 길게:
  장점: 연결 재사용 증가 → Handshake 감소
  단점: 유휴 연결이 서버 FD 점유
        대용량 트래픽 서버에서 FD 고갈 가능

Keep-Alive timeout 짧게:
  장점: 빠른 연결 해제 → FD 절약
  단점: 잦은 Handshake → 지연 증가

max 설정 (연결당 최대 요청 수):
  작게: 자주 연결 교체 → 메모리 누수 방지 (연결 상태 초기화)
  크게: 연결 재사용 극대화 → Handshake 최소화

HTTP/1.1의 근본적 한계:
  Keep-Alive + 병렬 연결로도 해결 안 되는 것:
  ① HOL Blocking: 연결당 직렬 처리
  ② 헤더 중복: 매 요청마다 동일한 헤더 반복 전송
     User-Agent, Accept, Cookie 등 수백 bytes 반복
  ③ 서버 푸시 없음: 클라이언트 요청 없이 데이터 전송 불가
  → 이 3가지를 해결하는 것이 HTTP/2
```

---

## 📌 핵심 정리

```
HTTP/1.1 핵심 요약:

메시지 구조:
  Start Line: 요청(메서드 URI 버전) / 응답(버전 상태코드 문구)
  Headers: 이름: 값\r\n 반복
  빈 줄: \r\n (헤더/바디 구분)
  Body: 콘텐츠 (POST/PUT/응답)

Keep-Alive:
  HTTP/1.1 기본 활성화
  TCP 연결을 여러 요청에 재사용
  Connection: close로 비활성화
  Keep-Alive: timeout=N, max=M으로 파라미터 제어

HOL Blocking:
  연결당 요청-응답이 직렬 처리
  앞 요청이 느리면 뒤 요청이 기다림
  파이프라이닝으로 해결 시도 → 실패
  브라우저 우회: 도메인당 6개 병렬 연결

Content-Length vs Chunked:
  Content-Length: 크기 알 때 (고정 응답)
  Transfer-Encoding: chunked: 크기 모를 때 (스트리밍)

한계:
  HOL Blocking, 헤더 중복, 서버 푸시 없음
  → HTTP/2가 해결

진단 명령어:
  curl -v --http1.1: 헤더 전체 확인
  curl -w "%{time_connect} %{time_total}": 연결/총 시간
  wireshark: HTTP 필터로 요청/응답 분석
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 서버가 `Connection: close`를 응답 헤더에 포함하고 있다. 어떤 상황에서 이것이 자동으로 발생하며, 성능에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**`Connection: close`가 자동 삽입되는 상황:**

1. **서버 셧다운 중**: Spring Boot가 Graceful Shutdown을 진행하면 새 연결을 받지 않기 위해 `Connection: close` 삽입

2. **Keep-Alive 한도 초과**: Tomcat의 `maxKeepAliveRequests`(기본 100) 초과 시 자동으로 `Connection: close` 추가

3. **HTTP/1.0 클라이언트 요청**: HTTP/1.0은 Keep-Alive가 기본값이 아님 → 명시적 `Connection: keep-alive` 없으면 close

4. **응답 크기 불명확 + 스트리밍**: Content-Length도 없고 chunked도 아닌 경우, 연결 종료로 응답 끝을 알림

**성능 영향:**
- `Connection: close`가 있으면 매 요청마다 새 TCP 연결
- RTT=50ms 환경: 요청당 +75ms (Handshake 비용)
- 1000 req/s 서버: 초당 1000번의 Handshake = 엄청난 오버헤드

**진단:**
```bash
curl -v https://your-api.com/endpoint 2>&1 | grep "Connection:"
# Connection: close 발견 시 → 원인 추적 필요

# Tomcat 설정 확인
# server.tomcat.max-keep-alive-requests=200  (기본 100에서 증가)
```

</details>

---

**Q2.** `Content-Length`와 `Transfer-Encoding: chunked`를 동시에 보내면 어떻게 되는가? HTTP 스펙은 이 경우 어떻게 처리하도록 정의하는가?

<details>
<summary>해설 보기</summary>

**RFC 7230(HTTP/1.1)에 따르면:** `Transfer-Encoding`이 있으면 `Content-Length`는 무시해야 합니다.

구체적으로:
- `Transfer-Encoding: chunked`가 있으면 메시지 크기는 청크 종료 신호(크기 0)로 결정
- `Content-Length` 헤더가 동시에 있어도 무시
- 일부 서버는 보안을 위해 이런 경우 요청을 거부 (HTTP Request Smuggling 방어)

**HTTP Request Smuggling:**
이 모호성을 악용하는 공격입니다. 프론트엔드(Nginx)와 백엔드(Tomcat)가 Content-Length와 Transfer-Encoding 중 어느 것을 우선시하는지 다르면, 공격자가 두 요청 사이에 악의적인 요청을 끼워넣을 수 있습니다.

**Spring Boot의 처리:**
- Spring MVC는 일반적으로 둘 다 있으면 Transfer-Encoding을 우선
- 최신 Tomcat은 이런 모호한 요청을 400으로 거부하는 설정 가능

**실무:** 두 헤더를 동시에 보내는 것은 피해야 합니다. 크기를 알면 Content-Length만, 모르면 Transfer-Encoding: chunked만 사용합니다.

</details>

---

**Q3.** 브라우저가 도메인당 6개 TCP 연결을 맺는 전략의 한계는 무엇이고, 개발자가 이를 어떻게 우회했는가?

<details>
<summary>해설 보기</summary>

**6개 연결 제한의 한계:**

1. **여전히 HOL Blocking**: 각 연결 내부는 여전히 직렬 처리
2. **Handshake 6배**: 6개 TCP + TLS = 최대 18 RTT 연결 비용
3. **서버 부하**: 동접 사용자 × 6 = 서버 연결 수 폭증

**개발자들의 우회 전략:**

1. **도메인 샤딩(Domain Sharding):**
   같은 CDN을 `cdn1.example.com`, `cdn2.example.com` 등 여러 도메인으로 서비스. 도메인당 6개 × 도메인 수 = 더 많은 병렬 연결.
   ```html
   <img src="https://cdn1.example.com/a.png">
   <img src="https://cdn2.example.com/b.png">
   ```
   단점: 각 도메인마다 DNS 조회 + TCP Handshake + TLS

2. **스프라이트(CSS Sprite):**
   여러 이미지를 하나의 큰 이미지로 합쳐서 HTTP 요청 수 자체를 줄임

3. **파일 번들링:**
   JS/CSS를 하나의 파일로 합침 (webpack, rollup)

4. **인라인 리소스:**
   작은 이미지를 Base64로 CSS에 직접 포함

**HTTP/2의 결론:** 이 우회 전략들은 HTTP/2에서는 오히려 역효과. HTTP/2는 단일 연결에 여러 스트림을 쓰므로, 도메인 샤딩은 불필요한 연결 수립 비용만 추가합니다. HTTP/2 환경에서는 번들링도 불필요해질 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: HTTP/2 멀티플렉싱 ➡️](./02-http2-multiplexing.md)**

</div>
