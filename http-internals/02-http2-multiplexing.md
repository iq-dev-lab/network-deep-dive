# HTTP/2 — 바이너리 프레이밍과 멀티플렉싱

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HTTP/2 바이너리 프레임은 어떤 구조이고 HTTP/1.1 텍스트와 어떻게 다른가?
- 멀티플렉싱은 어떻게 HOL Blocking을 해결하는가?
- Stream ID는 무엇이고 어떻게 요청과 응답을 매핑하는가?
- HPACK의 정적 테이블과 동적 테이블은 어떻게 헤더 크기를 줄이는가?
- HTTP/2 서버 푸시는 어떤 원리이고 왜 실패했는가?
- `curl --http2 -v`로 HTTP/2 협상 과정에서 무엇을 확인할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
HTTP/2 도입 효과를 수치로 이해:
  같은 서버, 같은 페이지 로딩
  HTTP/1.1: 175ms (병렬 6연결)
  HTTP/2:   80ms  (단일 연결 + 멀티플렉싱)
  → 2배 이상 빠름

  Spring Boot + HTTP/2 활성화:
  server.http2.enabled=true  (이 한 줄 추가)
  + HTTPS 필수 (브라우저가 HTTPS에서만 HTTP/2 허용)

실무 결정 포인트:
  "CDN 앞단에 HTTP/2를 적용해야 하나요?":
    대부분의 CDN은 이미 HTTP/2 지원
    브라우저 ↔ CDN: HTTP/2
    CDN ↔ Origin: HTTP/1.1 또는 HTTP/2 선택 가능

  "도메인 샤딩을 계속 써야 하나요?":
    HTTP/2에서는 역효과
    단일 도메인 + 단일 연결이 최적
    도메인 샤딩 → 불필요한 TLS 핸드쉐이크 증가

  "번들링을 제거해도 될까요?":
    HTTP/2 멀티플렉싱으로 다수의 작은 파일도 효율적
    하지만 파일 수가 수백 개면 여전히 번들링 유리
    (스트림 오버헤드, 압축 비율 등 고려)
```

---

## 😱 흔한 실수

```
Before — HTTP/2를 모를 때:

실수 1: HTTP/2에서도 도메인 샤딩 유지
  HTTP/1.1 시대 최적화를 HTTP/2에 그대로 적용
  → cdn1.example.com, cdn2.example.com 유지
  → 각 도메인마다 새 TLS 핸드쉐이크
  → HTTP/2의 멀티플렉싱 이점 소멸
  → 오히려 더 느려질 수 있음

실수 2: HTTP/2를 HTTP로만 사용 시도
  server.http2.enabled=true + HTTP
  → 브라우저는 HTTPS에서만 HTTP/2 사용 (h2)
  → HTTP에서는 h2c (cleartext) 로 명시 필요
  → 실무에서는 HTTPS + HTTP/2 세트로 구성

실수 3: 서버 푸시를 대규모로 사용
  서버 푸시 → 클라이언트가 이미 캐시된 리소스도 수신
  → 불필요한 대역폭 낭비
  → Chrome이 2022년 서버 푸시 지원 제거
  → 대신 103 Early Hints 사용
```

---

## ✨ 올바른 접근

```
After — HTTP/2를 알고 나면:

Spring Boot HTTP/2 활성화:
  # application.properties
  server.http2.enabled=true
  # HTTPS 필수 (TLS 없이는 브라우저 HTTP/2 미지원)
  server.ssl.enabled=true
  server.ssl.key-store=classpath:keystore.p12
  server.ssl.key-store-password=password

  # 또는 Nginx에서 HTTP/2 처리 (권장)
  server {
      listen 443 ssl http2;  ← http2 추가
      ssl_certificate /etc/nginx/cert.pem;
      ssl_certificate_key /etc/nginx/key.pem;
      # Nginx → Tomcat은 HTTP/1.1로 충분
      proxy_pass http://backend;
  }

HTTP/2 환경에서 최적화 변경:
  HTTP/1.1 최적화 (제거 가능):
    도메인 샤딩 → 불필요 (단일 도메인으로)
    JS/CSS 번들 → 더 작은 단위로 분리 가능

  HTTP/2 환경 최적화:
    HPACK 활용: 반복 헤더는 동적 테이블로 자동 압축
    서버 응답 헤더 줄이기: 불필요한 헤더 제거
    HTTPS 전용: HTTP/2는 HTTPS 위에서만 브라우저 지원

연결 상태 확인:
  curl -v --http2 https://your-site.com 2>&1 | head -20
  # "Using HTTP2, server supports HTTP/2" 확인
```

---

## 🔬 내부 동작 원리

### 1. 바이너리 프레임 구조

```
HTTP/1.1 텍스트 vs HTTP/2 바이너리:

HTTP/1.1 (텍스트):
  GET /api/users HTTP/1.1\r\n
  Host: example.com\r\n
  Accept: application/json\r\n
  \r\n

  → 사람이 읽을 수 있음
  → 파싱이 복잡 (경계가 \r\n으로 구분, 예외 처리 필요)
  → 파싱 오버헤드 큼

HTTP/2 바이너리 프레임:
  ┌──────────────────────────────────────────────────────────────┐
  │  Length (24 bits): 프레임 페이로드 크기                           │
  ├──────────────────────────────────────────────────────────────┤
  │  Type (8 bits): 프레임 종류                                     │
  │    0x0 DATA, 0x1 HEADERS, 0x2 PRIORITY                       │
  │    0x3 RST_STREAM, 0x4 SETTINGS, 0x5 PUSH_PROMISE            │
  │    0x6 PING, 0x7 GOAWAY, 0x8 WINDOW_UPDATE, 0x9 CONTINUATION │
  ├──────────────────────────────────────────────────────────────┤
  │  Flags (8 bits): 프레임별 의미 다름                              │
  │    HEADERS: END_HEADERS, END_STREAM, PADDED, PRIORITY        │
  │    DATA: END_STREAM, PADDED                                  │
  ├──────────────────────────────────────────────────────────────┤
  │  Reserved (1 bit) + Stream ID (31 bits)                      │
  │  → 어느 스트림(요청)의 프레임인지                                   │
  ├──────────────────────────────────────────────────────────────┤
  │  Payload (Length bytes): 실제 데이터                            │
  └──────────────────────────────────────────────────────────────┘

  총 헤더: 9 bytes (고정)
  → 파싱: Length 읽기 → 그만큼 읽기 → 끝
  → 단순하고 빠름

프레임 종류별 역할:
  HEADERS: HTTP 헤더 전달 (HPACK 압축)
  DATA:    HTTP 바디 전달
  SETTINGS: 연결 설정 협상 (초기 + 갱신)
  WINDOW_UPDATE: 흐름 제어 (스트림/연결 레벨)
  PING:    연결 살아있는지 확인 (RTT 측정)
  GOAWAY:  연결 종료 알림 (마지막 처리 Stream ID 포함)
  RST_STREAM: 특정 스트림만 취소
```

### 2. 멀티플렉싱 — HOL Blocking 해결

```
HTTP/1.1의 문제:

  연결 1: [요청A] → [응답A, 느림] → [요청B] → [응답B]
                    ↑ B가 기다림

  연결 2: [요청C] → [응답C]
  연결 3: [요청D] → [응답D]
  ...
  (6개 병렬 연결로 우회)

HTTP/2 멀티플렉싱:

  단일 TCP 연결에서:

  Stream 1 (GET /api/users):
    ── HEADERS(stream=1) ─────────────────────────────────────►
    ◄─ HEADERS(stream=1) + DATA(stream=1) ───────────────────
  
  Stream 3 (GET /api/products):
    ── HEADERS(stream=3) ─────────────────────────────────────►
    ◄─ HEADERS(stream=3) + DATA(stream=3) ───────────────────
  
  Stream 5 (POST /api/orders):
    ── HEADERS(stream=5) ─────────────────────────────────────►
    ── DATA(stream=5) ───────────────────────────────────────►
    ◄─ HEADERS(stream=5) + DATA(stream=5) ───────────────────

  실제 전송 순서 (인터리빙):
    HEADERS(1) → HEADERS(3) → HEADERS(5) →
    DATA(5,body) → DATA(1,response) → DATA(3,response) →
    DATA(5,response)
  
  → 각 프레임에 Stream ID가 있으므로 섞여도 재조합 가능
  → 느린 응답이 다른 스트림을 차단하지 않음

Stream ID 규칙:
  클라이언트 발신: 홀수 (1, 3, 5, 7...)
  서버 발신 (Push): 짝수 (2, 4, 6...)
  Stream 0: 연결 레벨 제어 (SETTINGS, PING 등)
  
  한 연결에서 Stream ID는 단조 증가 (재사용 없음)
  → Stream 1이 끝나도 다음은 Stream 3 (1로 돌아가지 않음)
```

### 3. HPACK — 헤더 압축

```
HTTP/1.1 헤더 중복 문제:

  요청 1:
    User-Agent: Mozilla/5.0 (Macintosh; Intel...) [500 bytes]
    Accept: text/html,application/xhtml+xml... [200 bytes]
    Cookie: session=abc123;pref=dark;user=alice [300 bytes]
    → 총 1KB 헤더 매 요청마다 전송

  요청 2:
    User-Agent: Mozilla/5.0 (Macintosh; Intel...) [500 bytes] ← 동일!
    Accept: text/html,application/xhtml+xml... [200 bytes]    ← 동일!
    Cookie: session=abc123;pref=dark;user=alice [300 bytes]   ← 동일!
    → 또 1KB 전송 (완전히 낭비)

HPACK 두 가지 테이블:

1. 정적 테이블 (Static Table, 고정 61개 항목):
   인덱스 1:  :authority
   인덱스 2:  :method GET
   인덱스 3:  :method POST
   인덱스 4:  :path /
   인덱스 5:  :path /index.html
   인덱스 6:  :scheme http
   인덱스 7:  :scheme https
   인덱스 8:  :status 200
   인덱스 19: :status 404
   ...
   인덱스 61: www-authenticate
   
   "GET /" → 인덱스 2 + 인덱스 4 = 2 bytes!
   (원래 "GET" + "/" = 4 bytes + 헤더 이름)

2. 동적 테이블 (Dynamic Table, 연결별 관리):
   처음 등장하는 헤더 → 전체 이름+값 전송 + 동적 테이블 추가
   이후 동일 헤더 → 인덱스 번호만 전송 (1~2 bytes)

   예시:
   첫 요청:
     Authorization: Bearer eyJhbGc...  (전체 전송, 동적 테이블 추가)
   
   다음 요청:
     [인덱스 62]  (1 byte로 Authorization 헤더 전체 표현)

HPACK 압축 효과:
  첫 요청: 원본 대비 30~40% 감소 (정적 테이블)
  반복 요청: 원본 대비 80~90% 감소 (동적 테이블)
  
  실제 측정 예:
    HTTP/1.1: 평균 헤더 1,400 bytes
    HTTP/2:   평균 헤더 200 bytes (85% 감소)

HPACK 주의:
  HPACK Crime 취약점 (BREACH 공격 변형):
    TLS 압축 + HPACK 조합 시 압축 오라클 공격 가능
    → HTTP/2 over TLS: TLS 압축 비활성화 권장 (ssl_comp off)
```

### 4. HTTP/2 연결 수립

```
ALPN (Application-Layer Protocol Negotiation):
  TLS 핸드쉐이크 중에 HTTP 버전 협상
  클라이언트: "h2, http/1.1 지원해요"
  서버: "h2 쓸게요"
  → TLS 완료와 동시에 HTTP/2 확정
  → 추가 RTT 없음

HTTP/2 연결 수립 전체 흐름:
  Client                              Server
  │  TCP SYN                           │
  │ ─────────────────────────────────► │
  │  TCP SYN-ACK                       │
  │ ◄───────────────────────────────── │
  │  TCP ACK                           │
  │ ─────────────────────────────────► │
  │                                    │
  │  TLS ClientHello                   │
  │  (ALPN: h2, http/1.1)              │ ← HTTP/2 지원 광고
  │ ─────────────────────────────────► │
  │  TLS ServerHello                   │
  │  (ALPN: h2 선택)                    │ ← HTTP/2 확정
  │ ◄───────────────────────────────── │
  │  TLS 완료                           │
  │                                    │
  │  PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n  │ ← HTTP/2 Magic (연결 Preface)
  │ ─────────────────────────────────► │
  │  SETTINGS 프레임 (클라이언트)          │ ← 초기 설정
  │ ─────────────────────────────────► │
  │  SETTINGS 프레임 (서버)               │
  │ ◄───────────────────────────────── │
  │  SETTINGS_ACK                      │
  │ ─────────────────────────────────► │
  │                                    │
  │  이제 HTTP/2 요청 가능!               │

SETTINGS 프레임에서 협상하는 것:
  HEADER_TABLE_SIZE:      동적 테이블 크기 (기본 4096 bytes)
  ENABLE_PUSH:            서버 푸시 허용 여부
  MAX_CONCURRENT_STREAMS: 동시 스트림 최대 수 (기본 무제한)
  INITIAL_WINDOW_SIZE:    스트림 초기 윈도우 크기
  MAX_FRAME_SIZE:         최대 프레임 크기 (기본 16384 bytes)
  MAX_HEADER_LIST_SIZE:   최대 헤더 목록 크기

h2c (HTTP/2 over cleartext):
  HTTPS 없이 HTTP/2 사용
  Upgrade: h2c 헤더로 업그레이드 협상
  → 브라우저는 지원하지 않음 (보안 이유)
  → 서버 간 통신(gRPC)에서 사용
```

### 5. 서버 푸시 (Server Push)

```
HTTP/2 서버 푸시 개념:
  클라이언트 요청 없이 서버가 먼저 리소스 전송

  클라이언트: GET /index.html
  서버: "index.html에 style.css, script.js가 필요할 것"
         → PUSH_PROMISE(stream=2, /style.css) 전송
         → PUSH_PROMISE(stream=4, /script.js) 전송
         → index.html 응답 (stream=1)
         → style.css 내용 (stream=2)
         → script.js 내용 (stream=4)

  클라이언트가 /style.css, /script.js를 요청하기 전에 수신

왜 실패했는가:
  문제 1: 클라이언트 캐시 무시
    브라우저에 style.css가 캐시돼 있어도 서버는 모름
    → 이미 있는 리소스를 push → 대역폭 낭비
  
  문제 2: RST_STREAM으로 거부 가능하지만 이미 전송 중
    클라이언트: "이 리소스 이미 있어요" → RST_STREAM
    하지만 서버는 이미 전송 시작 → 취소 불가
  
  문제 3: Cache-Digest 기술이 표준화 안 됨
    클라이언트 캐시 상태를 서버에 알려주는 메커니즘
    → Draft 상태에서 폐기
  
  결과: Chrome 2022년 서버 푸시 지원 제거
        Firefox, Safari도 제한적 지원

대안 — 103 Early Hints (RFC 8297):
  HTTP 상태 코드 103: 헤더 먼저 전송
  
  클라이언트: GET /index.html
  서버: 103 Early Hints
        Link: </style.css>; rel=preload; as=style
        Link: </script.js>; rel=preload; as=script
        [처리 계속...]
  
  브라우저: 103을 받자마자 style.css, script.js를 fetch 시작
  
  → 클라이언트가 직접 요청하므로 캐시 활용 가능
  → 서버는 단순히 힌트만 제공
  → Spring Boot 6.1+: 103 Early Hints 지원
```

---

## 💻 실전 실험

### 실험 1: curl로 HTTP/2 협상 확인

```bash
# HTTP/2 협상 과정 전체 확인
curl -v --http2 https://httpbin.org/get 2>&1 | head -30

# 출력:
# *   Trying 34.227.113.240:443...
# * Connected to httpbin.org port 443
# * ALPN: curl offers h2,http/1.1    ← HTTP/2 제안
# * SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
# * ALPN: server accepted h2          ← HTTP/2 선택됨!
# * Using HTTP2, server supports HTTP/2
# * h2 [:method: GET]                 ← 바이너리 헤더 (가상 헤더)
# * h2 [:path: /get]
# * h2 [:scheme: https]
# * h2 [:authority: httpbin.org]
# * h2 [accept: */*]
#
# > GET /get HTTP/2                   ← HTTP/2 요청
# < HTTP/2 200                        ← HTTP/2 응답
# < content-type: application/json
```

### 실험 2: HTTP/1.1 vs HTTP/2 성능 비교

```bash
# 여러 리소스 병렬 로딩 시간 비교
# HTTP/1.1
time for i in $(seq 1 10); do
  curl -s --http1.1 https://nghttp2.org/httpbin/get -o /dev/null
done

# HTTP/2 (단일 연결로 10개 병렬)
time curl -s --http2 \
  https://nghttp2.org/httpbin/get \
  https://nghttp2.org/httpbin/get \
  https://nghttp2.org/httpbin/get \
  https://nghttp2.org/httpbin/get \
  https://nghttp2.org/httpbin/get \
  -o /dev/null -o /dev/null -o /dev/null -o /dev/null -o /dev/null

# HTTP/2가 유의미하게 빠름 (멀티플렉싱 효과)

# nghttp 도구로 상세 분석
# apt install nghttp2
nghttp -v https://nghttp2.org/
# 프레임별 전송 순서와 스트림 ID 확인
```

### 실험 3: HPACK 헤더 압축 효과 확인

```bash
# HTTP/1.1 헤더 크기 확인
curl -v --http1.1 https://httpbin.org/get 2>&1 | \
  grep "^>" | awk '{total += length($0)} END {print "요청 헤더:", total, "bytes"}'

# HTTP/2 헤더 크기 (nghttp로 정확한 측정)
nghttp -v https://httpbin.org/get 2>&1 | \
  grep "HEADERS\|frame length"
# HEADERS frame의 length 필드 확인
# → HTTP/1.1 대비 훨씬 작음

# 반복 요청에서 압축 효과 (동적 테이블)
for i in 1 2 3; do
  nghttp -v https://httpbin.org/get 2>&1 | grep "HEADERS.*length"
done
# 첫 번째: 큰 HEADERS 프레임
# 두 번째/세 번째: 동적 테이블 활용 → 더 작은 프레임
```

### 실험 4: Stream 인터리빙 관찰

```bash
# Wireshark에서 HTTP/2 스트림 확인
# 필터: http2
# Statistics > HTTP2 Streams
# → 여러 Stream ID가 인터리빙되는 것 확인

# 또는 nghttp로 프레임 순서 확인
nghttp -v --multiply=5 https://httpbin.org/get 2>&1 | \
  grep -E "stream_id|HEADERS|DATA" | head -40
# stream_id=1,3,5,7,9 가 섞여서 전송됨
```

---

## 📊 성능/비용 비교

```
HTTP/1.1 vs HTTP/2 상세 비교:

┌─────────────────────────────────────────────────────────────────────┐
│  항목                │  HTTP/1.1              │  HTTP/2              │
├─────────────────────────────────────────────────────────────────────┤
│  연결 수립            │  1.5 RTT + TLS(2 RTT)  │  1.5 RTT + TLS(1 RTT)│
│  (TLS 1.3 기준)      │  = 2.5 RTT             │  = 2.5 RTT (동일)     │
├─────────────────────────────────────────────────────────────────────┤
│  동시 요청            │  6개 연결 × 1            │  1개 연결 × N 스트림    │
├─────────────────────────────────────────────────────────────────────┤
│  HOL Blocking       │  연결 내 직렬 처리         │  스트림별 독립          │
│                     │  (파이프라이닝 미지원)      │  (단, TCP 수준 HOL)    │
├─────────────────────────────────────────────────────────────────────┤
│  헤더 크기            │  반복 전송 (~1KB/요청)     │  HPACK 압축 (~200B)   │
├─────────────────────────────────────────────────────────────────────┤
│  서버 푸시            │  없음                    │  있음 (실질적 사용 ↓)   │
├─────────────────────────────────────────────────────────────────────┤
│  스트림 우선순위        │  없음                    │  있음 (PRIORITY)     │
└─────────────────────────────────────────────────────────────────────┘

실제 개선 효과 (고지연 네트워크, RTT=100ms):
  HTTP/1.1 (6 병렬 연결): 총 로딩 200ms
  HTTP/2 (단일 연결):      총 로딩 90ms (2.2배)

  패킷 손실 1% 환경:
  HTTP/2:   패킷 1개 손실 → 전체 연결 블로킹 (TCP HOL)
  HTTP/3:   패킷 1개 손실 → 해당 스트림만 블로킹
  → 고손실 환경에서 HTTP/3이 HTTP/2보다 유리
```

---

## ⚖️ 트레이드오프

```
HTTP/2 도입의 트레이드오프:

장점:
  ① 멀티플렉싱: HOL Blocking 해결 (HTTP 레벨)
  ② HPACK: 헤더 크기 80~90% 감소
  ③ 단일 연결: 서버 연결 수 감소 → 리소스 절약
  ④ 바이너리: 파싱 오버헤드 감소

단점:
  ① TCP HOL Blocking 잔존:
     단일 TCP 연결이므로 패킷 손실 시 전체 스트림 블로킹
     HTTP/1.1의 6개 병렬 연결이 이 점에서 유리한 경우도 있음
  
  ② HPACK 상태 유지:
     동적 테이블이 연결에 묶여 있음
     연결이 끊기면 테이블 초기화 → 첫 요청 비용 다시 발생
  
  ③ 디버깅 어려움:
     바이너리 프로토콜 → curl -v로 읽기 어려움
     Wireshark 또는 nghttp 필요

  ④ 구현 복잡성:
     서버 구현이 HTTP/1.1보다 복잡
     잘못 구현하면 HTTP/1.1보다 느릴 수 있음

HTTP/2가 HTTP/1.1보다 느린 경우:
  LAN 환경 (RTT < 1ms):
    멀티플렉싱 이점이 거의 없음
    연결 수립 오버헤드만 추가
  
  단일 대용량 파일 전송:
    멀티플렉싱 불필요, 단일 스트림
    HTTP/1.1 Keep-Alive와 차이 없음
```

---

## 📌 핵심 정리

```
HTTP/2 핵심 요약:

바이너리 프레이밍:
  9 bytes 고정 헤더 + 가변 Payload
  Length + Type + Flags + Stream ID
  → 파싱 단순화, 오류 가능성 감소

멀티플렉싱:
  단일 TCP 연결에서 여러 Stream 동시 처리
  Stream ID로 요청/응답 매핑
  HOL Blocking 해결 (HTTP 레벨)
  단, TCP HOL Blocking은 잔존

HPACK 헤더 압축:
  정적 테이블 (61개 공통 헤더): 항상 사용
  동적 테이블 (연결별 학습): 반복 헤더 1~2 bytes로
  효과: 최초 40%, 반복 요청 80~90% 감소

연결 수립:
  ALPN으로 TLS 핸드쉐이크 중 HTTP/2 협상
  추가 RTT 없이 HTTP/2 시작
  SETTINGS 프레임으로 연결 파라미터 협상

서버 푸시:
  2022년 Chrome에서 제거
  대안: 103 Early Hints (Link 헤더로 리소스 힌트)

실무 적용:
  Spring Boot: server.http2.enabled=true + HTTPS
  Nginx: listen 443 ssl http2;
  HTTP/2 환경: 도메인 샤딩 제거, 번들링 완화

진단 도구:
  curl --http2 -v: 협상 과정 확인
  nghttp: HTTP/2 프레임 상세 분석
  Chrome DevTools: Network 탭 → Protocol 컬럼
```

---

## 🤔 생각해볼 문제

**Q1.** HTTP/2에서 `MAX_CONCURRENT_STREAMS`를 너무 작게 설정하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

`MAX_CONCURRENT_STREAMS`는 하나의 HTTP/2 연결에서 동시에 처리할 수 있는 스트림 수의 상한입니다.

**너무 작게 설정 시 (예: 10):**

1. **HOL Blocking 재발생**: 11번째 요청은 앞선 스트림 중 하나가 끝날 때까지 대기. HTTP/1.1의 파이프라이닝 문제와 유사.

2. **클라이언트 동작 변화**: 스트림 한도 초과 시 클라이언트가 추가 TCP 연결을 열어 HTTP/2의 단일 연결 이점을 잃음.

3. **서버 측 문제**: Spring Boot 내장 Tomcat의 `maxConcurrentStreams` 기본값은 100. gRPC 서버의 경우 `max_concurrent_streams` 설정에 주의.

**설정 예시:**
```properties
# Tomcat HTTP/2
server.tomcat.max-concurrent-streams=100

# Nginx
http2_max_concurrent_streams 128;
```

**너무 크게 설정 시**: 각 스트림이 메모리를 사용하므로 메모리 고갈 가능. 특히 대용량 요청이 많은 환경에서 OOM 위험.

**권장**: 대부분 기본값(100~200)으로 충분. 측정 후 조정.

</details>

---

**Q2.** HPACK 동적 테이블이 HTTP/2 보안에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**CRIME/BREACH 공격 변형:**

HPACK 압축은 압축 오라클(Compression Oracle) 공격에 이론적으로 취약합니다.

원리:
1. TLS로 암호화됐지만 공격자가 암호문 크기를 측정 가능
2. HPACK이 반복 패턴을 압축하므로, 압축 후 크기가 작아지면 추측이 맞음
3. 공격자가 요청에 추측 문자열을 포함시키고 응답 크기로 쿠키 등 비밀을 유추

**완화 방법:**
- TLS 압축 비활성화 (이미 TLS 1.3에서 제거됨)
- 비밀 정보(쿠키, 토큰)와 공격자 제어 데이터를 같은 스트림에 혼합하지 않기
- 쿠키에 무작위 padding 추가

**현실적 위험도**: 낮음. 실제 공격은 매우 복잡하고 제한된 환경에서만 가능. 하지만 민감한 금융/결제 시스템에서는 추가 방어 필요.

**Spring Boot 설정:**
```yaml
server:
  ssl:
    # TLS 1.3은 기본적으로 TLS 압축 없음
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2
```

</details>

---

**Q3.** HTTP/2 스트림 우선순위(Priority)는 어떻게 동작하고, 실무에서 어떻게 활용하는가?

<details>
<summary>해설 보기</summary>

**HTTP/2 우선순위 (RFC 7540):**

각 스트림은 다음을 가집니다:
- **의존성(Dependency)**: 어느 스트림에 의존하는지 (트리 구조)
- **가중치(Weight)**: 1~256, 같은 레벨에서 자원 배분 비율

```
Stream 1 (weight: 12) ─── Stream 3 (weight: 4)
                    └──── Stream 5 (weight: 8)
```
→ Stream 3과 5는 1:2 비율로 자원 배분

**실무 활용:**
1. **HTML 먼저, CSS 다음, JS 나중**: 브라우저가 자동으로 렌더링 블로킹 리소스에 높은 우선순위
2. **위의 이미지 먼저**: Viewport 내 이미지에 높은 우선순위
3. **API 요청 > 분석 요청**: 사용자 경험에 직접 영향을 미치는 요청 우선

**현실:**
- HTTP/2의 Priority 구현은 서버마다 다름
- HTTP/3 (RFC 9218)에서 Priority 구조가 단순화됨 (의존성 트리 제거)
- 대부분의 서버는 기본 우선순위 로직만 구현

**Spring Boot**: 애플리케이션 수준에서 직접 제어하기 어려움. 프레임워크가 자동 처리. 세밀한 제어가 필요하면 Netty 기반의 WebFlux + 커스텀 핸들러 필요.

</details>

---

<div align="center">

**[⬅️ 이전: HTTP/1.1](./01-http1-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP/3와 QUIC ➡️](./03-http3-quic.md)**

</div>
