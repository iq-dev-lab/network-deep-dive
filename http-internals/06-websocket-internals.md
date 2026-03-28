# WebSocket — HTTP Upgrade와 프레임 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HTTP Upgrade 핸드쉐이크로 WebSocket 연결이 어떻게 수립되는가?
- `Sec-WebSocket-Key`와 `Sec-WebSocket-Accept`는 무슨 역할을 하는가?
- WebSocket 프레임 구조(FIN, Opcode, Mask, Payload)는 어떻게 구성되는가?
- Long Polling, SSE, WebSocket 각각의 동작 방식과 사용 기준은?
- Spring WebSocket과 STOMP는 어떻게 함께 동작하는가?
- WebSocket은 HTTP 방화벽과 Proxy를 어떻게 통과하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
실시간 기능 구현 시 선택 기준:

채팅 애플리케이션:
  Long Polling: 서버가 새 메시지를 기다렸다가 응답
                → 응답 받으면 즉시 다음 요청 → 유사 실시간
                → 매 요청마다 HTTP 오버헤드 (헤더 ~500bytes)
  
  SSE: 서버 → 클라이언트 단방향 스트리밍
       → 클라이언트가 서버에 메시지 보내려면 별도 POST API
       → 채팅에는 어색함 (양방향 필요)
  
  WebSocket: 완전 양방향, 단일 연결 유지
             → 클라이언트↔서버 모두 언제든 메시지 전송
             → 채팅, 게임, 실시간 협업에 최적

Spring 선택 시:
  단방향 서버→클라이언트 (알림, 피드):
    SSE + @SseEmitter → 간단, HTTP 기반, CDN 친화적
  
  양방향 실시간 (채팅, 게임, 협업):
    WebSocket + STOMP → 풍부한 기능, 채널 기반 라우팅
  
  레거시 환경 (WebSocket 차단 가능):
    SockJS → WebSocket 지원하면 WebSocket, 아니면 Long Polling 자동 선택

WebSocket이 왜 HTTP 위에서 시작하는가:
  방화벽: 80/443 포트만 허용하는 경우 많음
  HTTP로 시작 → 방화벽 통과
  Upgrade 핸드쉐이크 → WebSocket으로 전환
  이후: 같은 TCP 연결을 WebSocket 프로토콜로 사용
```

---

## 😱 흔한 실수

```
Before — WebSocket을 모를 때:

실수 1: 모든 실시간 기능에 WebSocket 사용
  단순 서버→클라이언트 알림:
  WebSocket 사용 → 양방향 연결 유지 → 서버 리소스 낭비
  → SSE가 더 적합 (단방향, HTTP, 방화벽 친화)

실수 2: WebSocket 연결 수를 HTTP 요청 수와 동일하게 처리
  HTTP: 요청마다 새 연결 → 동시 요청 수 = 단기 연결 수
  WebSocket: 연결이 유지됨 → 사용자 수 = 동시 연결 수
  1만 명 접속 = 1만 개 WebSocket 연결 상시 유지
  → Tomcat 기본 스레드 모델로는 감당 어려움
  → Netty 또는 Spring WebFlux 기반 논블로킹 서버 필요

실수 3: Nginx를 WebSocket 프록시로 미설정
  기본 Nginx: Upgrade 헤더를 전달하지 않음
  → WebSocket 핸드쉐이크 실패
  → proxy_set_header Upgrade $http_upgrade; 추가 필요

실수 4: STOMP 없이 WebSocket으로 채팅 직접 구현
  메시지 채널 관리, 구독 관리를 직접 구현
  → STOMP가 이미 이 기능 제공
  → /topic/chatroom, /user/queue/notifications 등 추상화
```

---

## ✨ 올바른 접근

```
After — WebSocket/SSE를 알고 나면:

Spring SSE (단방향, 서버→클라이언트):
  @GetMapping(value = "/notifications", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
  public SseEmitter subscribeNotifications(@RequestParam Long userId) {
      SseEmitter emitter = new SseEmitter(60_000L);  // 60초 타임아웃
      sseService.register(userId, emitter);
      
      emitter.onCompletion(() -> sseService.remove(userId));
      emitter.onTimeout(() -> sseService.remove(userId));
      return emitter;
  }
  
  // 이벤트 발생 시 (별도 스레드/이벤트 리스너)
  public void notifyUser(Long userId, String message) {
      SseEmitter emitter = emitters.get(userId);
      if (emitter != null) {
          emitter.send(SseEmitter.event()
              .name("notification")
              .data(message));
      }
  }

Spring WebSocket + STOMP (양방향):
  @Configuration
  @EnableWebSocketMessageBroker
  public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
      
      @Override
      public void registerStompEndpoints(StompEndpointRegistry registry) {
          registry.addEndpoint("/ws")  // WebSocket 엔드포인트
                  .setAllowedOrigins("*")
                  .withSockJS();       // SockJS 폴백
      }
      
      @Override
      public void configureMessageBroker(MessageBrokerRegistry registry) {
          registry.enableSimpleBroker("/topic", "/queue");  // 메모리 브로커
          registry.setApplicationDestinationPrefixes("/app"); // 앱 메시지 prefix
      }
  }
  
  @Controller
  public class ChatController {
      @MessageMapping("/chat.send")   // /app/chat.send 로 수신
      @SendTo("/topic/chatroom")      // /topic/chatroom 구독자에게 전송
      public ChatMessage sendMessage(ChatMessage message) {
          return message;
      }
  }

Nginx WebSocket 프록시 설정:
  location /ws {
      proxy_pass http://backend;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;   ← 필수!
      proxy_set_header Connection "upgrade";    ← 필수!
      proxy_read_timeout 3600s;                 ← 연결 유지 시간
      proxy_send_timeout 3600s;
  }
```

---

## 🔬 내부 동작 원리

### 1. HTTP Upgrade 핸드쉐이크

```
WebSocket 연결 수립 (HTTP → WebSocket 전환):

Client                              Server
│                                        │
│  GET /ws HTTP/1.1                      │
│  Host: api.example.com                 │
│  Upgrade: websocket                    │ ← WebSocket으로 업그레이드 요청
│  Connection: Upgrade                   │ ← 연결 업그레이드 의향
│  Sec-WebSocket-Key: dGhlIHNhbXBsZ...   │ ← 무작위 16바이트 Base64
│  Sec-WebSocket-Version: 13             │ ← WebSocket 버전
│  Sec-WebSocket-Protocol: chat, stomp   │ ← 서브프로토콜 선호
│  Origin: https://www.example.com       │
│ ─────────────────────────────────────► │
│                                        │
│  HTTP/1.1 101 Switching Protocols      │ ← 업그레이드 수락
│  Upgrade: websocket                    │
│  Connection: Upgrade                   │
│  Sec-WebSocket-Accept: s3pPLMBiTxaQ9k  │ ← 서버 응답 키
│  Sec-WebSocket-Protocol: chat          │ ← 선택된 서브프로토콜
│ ◄───────────────────────────────────── │
│                                        │
│  [이제 WebSocket 프레임 통신]              │
│ ════════════════════════════════════►  │ (클라이언트→서버)
│ ◄════════════════════════════════════  │ (서버→클라이언트)

Sec-WebSocket-Key 검증 과정:
  클라이언트: 무작위 16바이트를 Base64 인코딩
              "dGhlIHNhbXBsZSBub25jZQ=="
  
  서버 계산:
    1. Key + GUID 연결:
       "dGhlIHNhbXBsZSBub25jZQ==" + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
    2. SHA-1 해시
    3. Base64 인코딩
    → "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
  
  목적:
    HTTP 캐시나 Proxy가 WebSocket 핸드쉐이크를 캐싱하지 못하도록
    (일반 HTTP 요청으로 착각하는 오동작 방지)
    보안 검증이 아닌 프로토콜 오동작 방지용

101 Switching Protocols:
  HTTP 1xx 정보 응답 중 하나
  "이 연결을 다른 프로토콜로 전환함"
  → 이후 TCP 연결은 WebSocket 프로토콜 사용
  → HTTP 요청/응답 사이클 종료, WebSocket 프레임 시작
```

### 2. WebSocket 프레임 구조

```
WebSocket 프레임 (RFC 6455):

  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 ┌─┬─┬─┬─┬───────┬─┬──────────────────────────────────────────────┐
 │F│R│R│R│Opcode │M│    Payload Length                            │
 │I│S│S│S│(4bits)│A│    (7 bits)                                  │
 │N│V│V│V│       │S│                                              │
 │ │1│2│3│       │K│                                              │
 ├─┴─┴─┴─┴───────┴─┴──────────────────────────────────────────────┤
 │  Masking Key (if MASK=1, 32 bits)                              │
 ├────────────────────────────────────────────────────────────────┤
 │  Payload Data (마스킹 적용 또는 미적용)                              │
 └────────────────────────────────────────────────────────────────┘

각 필드 의미:

FIN (1 bit):
  1 = 이 프레임이 메시지의 마지막
  0 = 더 이어지는 프레임 있음 (대용량 메시지 분할)

RSV1/2/3 (각 1 bit):
  확장용 예약 비트 (기본 0)
  압축 확장(permessage-deflate)에서 RSV1 사용

Opcode (4 bits):
  0x0: Continuation Frame (분할 메시지 계속)
  0x1: Text Frame (UTF-8 텍스트)
  0x2: Binary Frame (바이너리 데이터)
  0x8: Connection Close (연결 종료)
  0x9: Ping (연결 살아있는지 확인)
  0xA: Pong (Ping에 대한 응답)

MASK (1 bit):
  클라이언트→서버: 반드시 1 (마스킹 필수!)
  서버→클라이언트: 반드시 0 (마스킹 없음)
  
  왜 클라이언트만 마스킹하는가:
    중간 캐시 Proxy가 WebSocket 프레임을 HTTP 요청으로 오해할 위험
    마스킹으로 Proxy 공격 방지 (Cache Poisoning 등)

Payload Length (7 + 추가 bits):
  0~125:    7비트로 직접 표현
  126:      다음 2 bytes에 실제 길이 (최대 65535 bytes)
  127:      다음 8 bytes에 실제 길이 (최대 2^64 bytes)

Masking Key (4 bytes, MASK=1일 때):
  무작위 4바이트
  페이로드 XOR 마스킹: payload[i] = payload[i] XOR key[i % 4]

텍스트 프레임 예시 ("Hello"):
  FIN=1, RSV=000, Opcode=0x1 (Text), MASK=1
  Payload Length=5
  Masking Key=[0x37, 0xFA, 0x21, 0x3D]
  Masked Payload = "Hello" XOR Masking Key
```

### 3. Long Polling vs SSE vs WebSocket

```
Long Polling:

Client                              Server
│  GET /events?lastId=99            │
│ ─────────────────────────────────►│
│                                   │  [새 이벤트 없음... 대기]
│                                   │  [새 이벤트 발생!]
│  HTTP/1.1 200 OK                  │
│  {events: [{id:100, ...}]}        │
│ ◄───────────────────────────────  │
│  GET /events?lastId=100           │  ← 즉시 다시 요청
│ ─────────────────────────────────►│

특징:
  프로토콜: 일반 HTTP
  방향: 클라이언트→서버 요청 기반
  오버헤드: 매 요청마다 HTTP 헤더 (~500 bytes)
  호환성: 모든 환경 작동
  적합: 드문 이벤트, 단순 구현 필요 시

────────────────────────────────────────────────────────────────────

SSE (Server-Sent Events):

Client                              Server
│  GET /stream HTTP/1.1             │
│  Accept: text/event-stream        │
│ ─────────────────────────────────►│
│                                   │
│  HTTP/1.1 200 OK                  │
│  Content-Type: text/event-stream  │
│  Transfer-Encoding: chunked       │
│                                   │
│  data: {"msg": "hello"}\n\n       │  ← 서버에서 이벤트 발생 시
│ ◄───────────────────────────────  │
│  data: {"msg": "world"}\n\n       │  ← 계속 스트리밍
│ ◄───────────────────────────────  │

SSE 이벤트 포맷:
  event: notification   ← 이벤트 타입 (선택)
  id: 123              ← 이벤트 ID (재연결 시 Last-Event-ID로 재개)
  data: {"message": "주문 완료"}  ← 데이터 (여러 줄 가능)
  retry: 3000          ← 재연결 대기 시간 (ms)
  [빈 줄]              ← 이벤트 구분

특징:
  프로토콜: HTTP (Chunked Transfer)
  방향: 서버→클라이언트 단방향
  오버헤드: 연결 1개로 지속 스트리밍
  자동 재연결: 브라우저 내장 (EventSource API)
  방화벽: HTTP 포트 사용 → 친화적
  적합: 알림, 피드, 실시간 로그, 주식 시세

────────────────────────────────────────────────────────────────────

WebSocket:

Client                              Server
│  HTTP Upgrade 핸드쉐이크             │
│ ─────────────────────────────────►│
│ ◄───────────────────────────────  │ 101 Switching Protocols
│                                   │
│  WS Frame: "Hello"                │ ← 클라이언트→서버 (언제든)
│ ═══════════════════════════════►  │
│  WS Frame: "World"                │ ← 서버→클라이언트 (언제든)
│ ◄═══════════════════════════════  │
│  WS Frame: Close                  │ ← 종료
│ ═══════════════════════════════►  │

특징:
  프로토콜: WebSocket (ws:// 또는 wss://)
  방향: 완전 양방향 (Full Duplex)
  오버헤드: 프레임 헤더 2~10 bytes (HTTP보다 훨씬 작음)
  방화벽: HTTP 포트 시작 후 업그레이드 (대체로 통과)
  적합: 채팅, 게임, 실시간 협업, 트레이딩

비교표:
┌──────────────────────────────────────────────────────────────────────┐
│  방식          │ 방향    │ 오버헤드  │ 방화벽 │ 재연결   │ 적합 사례           │
├──────────────────────────────────────────────────────────────────────┤
│  Long Polling │ 양방향  │  높음    │ 최고   │ 자동    │ 단순, 레거시         │
│  SSE          │ 단방향  │  낮음    │ 높음   │ 자동    │ 알림, 피드          │
│  WebSocket    │ 양방향  │  최저    │ 보통   │ 수동    │ 채팅, 게임          │
└──────────────────────────────────────────────────────────────────────┘
```

### 4. STOMP — WebSocket 위의 메시징 프로토콜

```
왜 STOMP가 필요한가:
  WebSocket: 단순 바이트 전송 채널
  → "이 메시지가 어느 채팅방으로 가는가?" → WebSocket에 없음
  → "특정 사용자에게만 전송" → WebSocket에 없음
  
  STOMP (Simple Text Oriented Messaging Protocol):
    메시지 채널(Destination) 개념 추가
    구독(Subscribe)/발행(Publish) 패턴
    ActiveMQ, RabbitMQ 등 메시지 브로커와 연동 가능

STOMP 프레임 구조:
  COMMAND\n
  header1: value1\n
  header2: value2\n
  \n
  body (선택)\0

STOMP 명령어:
  CONNECT/CONNECTED: 연결 수립
  SEND:             서버로 메시지 전송
  SUBSCRIBE:        채널 구독
  UNSUBSCRIBE:      구독 해제
  MESSAGE:          서버→클라이언트 메시지
  ACK/NACK:         메시지 처리 확인
  DISCONNECT:       연결 종료

Spring WebSocket + STOMP 메시지 흐름:

  클라이언트                   Spring Server
  
  CONNECT
  ────────────────────────────►
  CONNECTED
  ◄────────────────────────────
  
  SUBSCRIBE /topic/chatroom
  ────────────────────────────►
  
  SEND /app/chat.send
  {"content": "안녕하세요"}
  ────────────────────────────► @MessageMapping("/chat.send")
                                 메서드 호출
                                 @SendTo("/topic/chatroom")
  MESSAGE /topic/chatroom
  {"content": "안녕하세요", ...}
  ◄────────────────────────────  ← /topic/chatroom 구독자 전체에게

사용자별 메시지:
  /user/queue/notifications:
    특정 사용자에게만 전송
    @SendToUser("/queue/notifications")
    → /user/{username}/queue/notifications 로 자동 변환
```

### 5. WebSocket 연결 관리

```
Heartbeat (연결 유지):
  문제: 유휴 WebSocket 연결을 방화벽/LB가 끊을 수 있음
        일부 NAT: 30초 이상 활동 없으면 세션 삭제
  
  WebSocket Ping/Pong:
    서버: Ping 프레임 전송 (Opcode=0x9)
    클라이언트: Pong 프레임 응답 (Opcode=0xA)
    → 연결 유지 확인 + 방화벽/NAT 세션 갱신
  
  STOMP Heartbeat:
    CONNECT 프레임: heart-beat:10000,10000
    → 10초마다 하트비트 교환
    → 끊김 감지 + 재연결

  Spring WebSocket 설정:
    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration reg) {
        reg.setMessageSizeLimit(64 * 1024);  // 64KB 메시지 제한
        reg.setSendBufferSizeLimit(512 * 1024);
        reg.setSendTimeLimit(20 * 1000);     // 20초 이내 전송 실패 시 종료
    }
    
    // STOMP 핸드쉐이크 인터셉터에서 heartbeat 설정
    // registry.addEndpoint("/ws").addInterceptors(new HttpSessionHandshakeInterceptor());

연결 종료 코드 (Close Frame):
  1000: 정상 종료
  1001: 서버/클라이언트 떠남 (페이지 이동)
  1002: 프로토콜 오류
  1003: 지원하지 않는 데이터 타입
  1006: 비정상 종료 (Close 프레임 없음)
  1007: 인코딩 오류 (UTF-8)
  1008: 정책 위반
  1009: 메시지 너무 큼
  1011: 서버 내부 오류

WebSocket 확장 — permessage-deflate:
  WebSocket 프레임 Payload를 zlib로 압축
  RSV1=1로 압축 적용 표시
  Sec-WebSocket-Extensions: permessage-deflate 협상
  → 대용량 텍스트 메시지에서 60~80% 크기 감소
  → CPU 사용량 증가 트레이드오프
```

---

## 💻 실전 실험

### 실험 1: WebSocket Upgrade 핸드쉐이크 캡처

```bash
# WebSocket 핸드쉐이크 패킷 캡처
sudo tcpdump -nn -i any 'port 8080 and tcp' -A 2>/dev/null | \
  grep -E "Upgrade|Switching|Sec-WebSocket" | head -10

# wscat으로 WebSocket 연결 테스트
npm install -g wscat
wscat -c ws://echo.websocket.org
# 연결 후 메시지 입력 → 에코 확인

# curl로 WebSocket Upgrade 헤더 확인 (연결은 안 되지만 핸드쉐이크 헤더 확인)
curl -v -H "Upgrade: websocket" \
     -H "Connection: Upgrade" \
     -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
     -H "Sec-WebSocket-Version: 13" \
     http://echo.websocket.org 2>&1 | \
     grep -E "101|Upgrade|Sec-WebSocket"
```

### 실험 2: SSE vs WebSocket 비교

```bash
# SSE 서버 (Python)
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import time, json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/sse':
            self.send_response(200)
            self.send_header('Content-Type', 'text/event-stream')
            self.send_header('Cache-Control', 'no-cache')
            self.send_header('Connection', 'keep-alive')
            self.end_headers()
            for i in range(5):
                event = json.dumps({'msg': f'Event {i}', 'time': time.time()})
                self.wfile.write(f'data: {event}\n\n'.encode())
                self.wfile.flush()
                time.sleep(1)
    def log_message(self, *args): pass

HTTPServer(('127.0.0.1', 8765), Handler).serve_forever()
" &

# SSE 수신
curl -N http://127.0.0.1:8765/sse
# data: {"msg": "Event 0", "time": ...}
# data: {"msg": "Event 1", "time": ...}
# ...
```

### 실험 3: Spring WebSocket STOMP 연결

```bash
# Spring Boot WebSocket 앱이 실행 중이라 가정

# wscat으로 STOMP 연결 (수동)
wscat -c ws://localhost:8080/ws

# STOMP CONNECT 프레임 전송
CONNECT
accept-version:1.2
heart-beat:10000,10000

^@

# SUBSCRIBE 프레임
SUBSCRIBE
id:sub-0
destination:/topic/chatroom

^@

# SEND 프레임
SEND
destination:/app/chat.send
content-type:application/json

{"content":"안녕하세요","sender":"테스트"}^@

# 다른 클라이언트가 구독하면 MESSAGE 프레임 수신
```

### 실험 4: WebSocket 프레임 분석

```bash
# Wireshark에서 WebSocket 프레임 분석
# 필터: websocket
# 각 프레임에서 확인:
#   FIN 비트
#   Opcode (Text=0x1, Binary=0x2, Ping=0x9, Pong=0xA)
#   MASK 비트 (클라이언트→서버: 1)
#   Payload Length

# websocat으로 WebSocket 프레임 분석
# cargo install websocat
websocat ws://echo.websocket.org -v
# --dump-frames: 프레임 구조 출력
```

---

## 📊 성능/비용 비교

```
메시징 방식별 오버헤드 비교:

메시지 1개 전송 비용 (payload: 50 bytes):

Long Polling:
  HTTP 요청 헤더: ~500 bytes
  HTTP 응답 헤더: ~400 bytes
  Payload:         50 bytes
  총: ~950 bytes (payload의 19배!)
  서버 처리: 연결 수립 + 종료 (Handshake 포함)

SSE (스트리밍 중):
  청크 헤더: ~10 bytes
  Payload:   50 bytes  (+ "data: \n\n" ~8 bytes)
  총: ~68 bytes
  서버 처리: 청크 작성만 (연결 유지)

WebSocket:
  프레임 헤더: 2~10 bytes
  Payload:     50 bytes
  총: ~56 bytes
  서버 처리: 프레임 작성만

초당 1000 메시지 시나리오:
  Long Polling: 950 × 1000 = 0.95 MB/s 헤더 트래픽
  SSE:          68 × 1000  = 68 KB/s
  WebSocket:    56 × 1000  = 56 KB/s

서버 연결 관리 비용:
  Long Polling: 연결 수립/해제 반복
                Tomcat 스레드: 요청 중 블로킹
  SSE:          연결 유지, 스레드는 이벤트 발생 시만 사용 (Reactive 권장)
  WebSocket:    연결 유지 (상태 있음)
                10,000 동시 연결 = 스레드 10,000개 (Tomcat 기본)
                → Netty 기반 논블로킹 필요 (10,000+ 연결)
```

---

## ⚖️ 트레이드오프

```
WebSocket의 트레이드오프:

장점:
  완전 양방향 실시간 통신
  낮은 프레임 오버헤드 (2~10 bytes)
  지연 최소화 (서버→클라이언트 즉시 푸시)
  단일 연결 유지 (Handshake 최소화)

단점:
  상태 유지: 연결을 서버가 메모리에 보관
  수평 확장 어려움: 클라이언트A가 서버1에, 서버1에서 서버2의 클라이언트B에게 메시지?
                    → Redis Pub/Sub 또는 메시지 브로커 필요
  방화벽: 일부 기업 환경에서 ws:// 차단 → SockJS 폴백 필요
  로드밸런서: sticky session 또는 WebSocket 지원 LB 필요

SSE의 트레이드오프:
  장점: HTTP 기반 (방화벽 친화), 자동 재연결, CDN 가능
  단점: 단방향만 (클라이언트→서버는 별도 POST API)
        HTTP/1.1: 도메인당 연결 수 제한 (6개)
        HTTP/2: 제한 없음 (멀티플렉싱)

SockJS:
  WebSocket이 차단되거나 지원 안 될 때 자동 폴백
  폴백 우선순위: WebSocket → iframe-eventsource → iframe-htmlfile → xhr-polling
  → 모든 환경 지원하지만 폴백 시 Long Polling 비용
```

---

## 📌 핵심 정리

```
WebSocket 핵심 요약:

HTTP Upgrade 핸드쉐이크:
  GET + Upgrade: websocket → 101 Switching Protocols
  Sec-WebSocket-Key → Sec-WebSocket-Accept (SHA-1 + Base64)
  이후 같은 TCP 연결을 WebSocket 프레임으로 사용

WebSocket 프레임:
  FIN:     마지막 프레임 여부
  Opcode:  Text(0x1), Binary(0x2), Close(0x8), Ping(0x9), Pong(0xA)
  MASK:    클라이언트→서버 필수, 서버→클라이언트 없음
  Payload: 2~10 bytes 헤더 오버헤드

방식 비교:
  Long Polling: 양방향, HTTP, 높은 오버헤드, 모든 환경 지원
  SSE:          단방향(서버→클라이언트), HTTP, 자동 재연결
  WebSocket:    완전 양방향, 낮은 오버헤드, 방화벽 주의

STOMP (Spring):
  WebSocket 위의 메시징 프로토콜
  /topic/...: 브로드캐스트
  /user/queue/...: 특정 사용자
  @MessageMapping + @SendTo로 라우팅

운영 고려사항:
  Nginx: proxy_set_header Upgrade/Connection 설정 필수
  수평 확장: Redis Pub/Sub으로 서버 간 메시지 전달
  연결 수: 10,000+ → Netty/Spring WebFlux 필요
  Heartbeat: 방화벽/NAT 세션 유지

진단 도구:
  wscat: WebSocket 클라이언트
  Wireshark: websocket 필터
  Chrome DevTools: Network → WS 탭
```

---

## 🤔 생각해볼 문제

**Q1.** Spring WebSocket 서버를 2개 이상의 인스턴스로 수평 확장할 때 어떤 문제가 생기고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**문제: 메시지 라우팅**

클라이언트 A가 서버 1에, 클라이언트 B가 서버 2에 연결된 경우, 서버 1에서 클라이언트 B에게 메시지를 보내려면 어떻게 해야 할까요?

**해결 방법 1: Redis Pub/Sub (가장 일반적)**

```java
// application.yml
spring:
  redis:
    host: redis-server
    port: 6379

// WebSocket 설정에서 외부 브로커 사용
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    // 메모리 브로커 대신 Redis 기반 브로커
    registry.enableStompBrokerRelay("/topic", "/queue")
        .setRelayHost("redis-server")
        .setRelayPort(61613)  // Redis STOMP 어댑터
        .setClientLogin("guest")
        .setClientPasscode("guest");
}
```

실제로는 Redis STOMP 어댑터나 RabbitMQ STOMP 플러그인 사용.

**해결 방법 2: RabbitMQ STOMP Plugin**

RabbitMQ가 STOMP 브로커 역할을 함. 모든 Spring 인스턴스가 같은 RabbitMQ에 연결.

```java
registry.enableStompBrokerRelay("/topic", "/queue")
    .setRelayHost("rabbitmq-server")
    .setRelayPort(61613);
```

**해결 방법 3: 로드밸런서 Sticky Session**

같은 사용자의 요청이 항상 같은 서버로 → 근본 해결이 아님 (서버 장애 시 연결 끊김)

**권장:** RabbitMQ STOMP + 각 서버가 /topic 을 RabbitMQ에 위임. 서버 장애 시에도 메시지 전달 가능.

</details>

---

**Q2.** WebSocket 연결이 HTTPS를 통해야 하는 이유는 무엇이고, ws://와 wss://의 차이는?

<details>
<summary>해설 보기</summary>

**ws:// (WebSocket without TLS):**
- HTTP 80 포트를 통해 업그레이드
- 데이터가 평문으로 전송
- 중간자(Man-in-the-Middle)가 메시지를 도청/변조 가능

**wss:// (WebSocket Secure with TLS):**
- HTTPS 443 포트를 통해 업그레이드
- TLS 핸드쉐이크 후 암호화된 터널에서 WebSocket 업그레이드
- 메시지가 암호화됨

**wss://를 사용해야 하는 이유:**

1. **보안:** 채팅, 인증 토큰, 개인정보가 포함된 메시지 보호

2. **방화벽 통과:** 많은 기업 방화벽이 80 포트 평문 트래픽보다 443 트래픽을 더 자유롭게 허용

3. **브라우저 정책:** HTTPS 페이지에서 ws://는 mixed content로 차단됨. wss://만 사용 가능.

4. **HTTP/2와 호환:** HTTP/2는 TLS 필수. HTTP/2 페이지에서 ws://는 불가.

**Spring Boot 설정:**
```yaml
server:
  ssl:
    enabled: true
    key-store: keystore.p12
```
→ WebSocket endpoint도 자동으로 wss://로 변경됨.

**Nginx 뒤에 배치 시:**
- Nginx: 클라이언트와 wss:// 종단점
- Nginx → Tomcat: 내부 ws:// 가능 (신뢰 네트워크 내부)
- Nginx에서 TLS 처리, Tomcat은 평문 WebSocket

</details>

---

**Q3.** 10만 명이 동시에 WebSocket으로 연결하는 서비스를 구축한다면 어떤 아키텍처를 설계해야 하는가?

<details>
<summary>해설 보기</summary>

**핵심 도전과제:**
- 10만 연결 = 10만 개의 열린 TCP 소켓
- Tomcat 기본 스레드 200개 → 200개 연결만 처리

**아키텍처 설계:**

**1. 논블로킹 I/O (필수)**
```java
// Spring WebFlux + Reactive WebSocket
@Configuration
public class ReactiveWebSocketConfig {
    @Bean
    public HandlerMapping webSocketHandlerMapping() {
        // Netty 기반 → 스레드 수 = CPU 코어 수로 10만 연결 처리
    }
}
```
Netty는 이벤트 루프 기반으로 수십만 연결을 적은 스레드로 처리.

**2. 연결 서버 분리 (Gateway 패턴)**
```
클라이언트 10만 명
        ↓
WebSocket Gateway (Netty, 순수 연결 관리)
        ↓
Message Broker (RabbitMQ, Kafka)
        ↓
Application Server (비즈니스 로직)
```
- WebSocket Gateway: 연결 유지, 메시지 라우팅만
- Application Server: 비즈니스 로직, 수평 확장

**3. OS 튜닝**
```bash
# 파일 디스크립터 한계 증가 (소켓 = FD)
ulimit -n 1000000
sysctl -w fs.file-max=1000000

# TCP 버퍼 최적화
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

**4. 실제 사례:**
- Slack: Erlang 기반 (10만+ 연결 per 서버)
- Discord: 처음 Go, 이후 Rust (250만+ 연결 per 서버)
- Spring WebFlux + Netty: 단일 서버에서 수만 연결 처리 가능

**실무 시작점:**
10만 연결이라면 서버 5~10대로 분산 + Spring WebFlux + Netty + Redis Pub/Sub 조합이 현실적.

</details>

---

<div align="center">

**[⬅️ 이전: HTTP 메서드와 상태 코드](./05-http-methods-status-codes.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — TLS/HTTPS ➡️](../tls-https-internals/01-crypto-basics.md)**

</div>
