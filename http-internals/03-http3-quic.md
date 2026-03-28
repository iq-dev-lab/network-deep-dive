# HTTP/3와 QUIC — UDP 위에서 신뢰성을 구현하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HTTP/2가 TCP HOL Blocking을 해결하지 못하는 이유는?
- QUIC은 UDP 위에서 어떻게 스트림별 독립 재전송을 구현하는가?
- 0-RTT 핸드쉐이크는 어떻게 동작하고 어떤 보안 위험이 있는가?
- Connection Migration은 무엇이고 어떤 환경에서 중요한가?
- QUIC Packet Number가 TCP Sequence Number와 어떻게 다른가?
- `curl --http3`로 QUIC 연결을 어떻게 확인하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"HTTP/2로 이미 성능 최적화했는데 HTTP/3도 필요한가?":
  안정적인 고속 LAN: HTTP/2로 충분
  
  모바일/Wi-Fi: HTTP/3가 의미 있음
    패킷 손실률 1%에서 비교:
    HTTP/2: TCP 손실 → 전체 스트림 블로킹 → 추가 RTT 대기
    HTTP/3: QUIC 손실 → 해당 스트림만 재전송 → 다른 스트림 계속

  Wi-Fi → LTE 전환 시 (모바일):
    HTTP/2(TCP): IP 변경 → 연결 끊김 → 재연결 (1.5 RTT + TLS 1 RTT)
    HTTP/3(QUIC): Connection ID로 연결 유지 → 중단 없음

실제 도입 현황 (2026):
  Google: 자사 서비스 전체 HTTP/3 지원
  Cloudflare: 기본 HTTP/3 제공
  AWS CloudFront: HTTP/3 지원
  Nginx 1.25+: HTTP/3 지원 (실험적)
  Spring Boot: 내장 Netty 통해 HTTP/3 지원 (실험적)

도입 고려 시:
  방화벽이 UDP 443을 차단하는 경우 fallback 자동 처리
  브라우저는 Alt-Svc: h3=":443"으로 HTTP/3 감지
  서버가 HTTP/3를 제공해도 클라이언트가 HTTP/2로 fallback 가능
```

---

## 😱 흔한 실수

```
Before — HTTP/3을 모를 때:

실수 1: HTTP/2가 모든 HOL Blocking을 해결했다고 생각
  HTTP/2 멀티플렉싱: HTTP 레벨 HOL 해결 ✅
  HTTP/2 TCP:       TCP 레벨 HOL 잔존 ❌
  
  패킷 1개 손실 → TCP가 해당 패킷 재전송 대기
  → 그 동안 모든 HTTP/2 스트림 차단
  → HTTP/1.1 병렬 6연결이 이 경우엔 더 유리할 수도

실수 2: "QUIC = UDP = 신뢰성 없다"
  QUIC은 UDP 위에서 TCP보다 더 정교한 신뢰성 구현
  스트림별 독립 재전송, ACK, SACK, 흐름 제어 모두 포함
  → UDP 위에 있다는 이유만으로 신뢰성 없다는 것은 오해

실수 3: 0-RTT를 모든 요청에 사용
  0-RTT 재전송 공격 (Replay Attack):
    동일한 요청을 공격자가 재전송 가능
    → POST/DELETE 등 비멱등 요청에는 0-RTT 사용 금지
    → GET 등 멱등 요청에만 사용
```

---

## ✨ 올바른 접근

```
After — HTTP/3와 QUIC을 알고 나면:

HTTP/3 도입 결정 기준:
  ① 모바일 트래픽 비율이 높은가? → 효과 큼
  ② 패킷 손실이 잦은 환경인가? → 효과 큼
  ③ 네트워크 전환이 잦은가? → Connection Migration 효과
  ④ 레이턴시가 핵심 지표인가? → 0-RTT 효과

Nginx HTTP/3 설정 (1.25+):
  server {
      listen 443 ssl;
      listen 443 quic reuseport;   ← UDP 443 추가
      ssl_certificate /etc/nginx/cert.pem;
      ssl_certificate_key /etc/nginx/key.pem;
      
      # HTTP/3 광고
      add_header Alt-Svc 'h3=":443"; ma=86400';
      
      # QUIC 설정
      http3 on;
      quic_retry on;               ← Address Validation
      quic_gso on;                 ← Generic Segmentation Offload
  }

0-RTT 안전한 사용:
  멱등 요청만: GET, HEAD, OPTIONS
  Spring Security: 0-RTT 요청에 CSRF 토큰 검증
  
  // Spring에서 POST에 0-RTT 방지
  // 서버 측에서 early_data 헤더 확인
  // 또는 0-RTT 완전 비활성화로 안전성 선택
```

---

## 🔬 내부 동작 원리

### 1. HTTP/2 TCP HOL Blocking의 정확한 위치

```
HTTP/2의 스트림 레벨 HOL Blocking 해결:

Stream 1: [HEADERS][DATA 1][DATA 2]
Stream 3: [HEADERS][DATA 1]
Stream 5: [HEADERS]
                    ↕ 인터리빙, 독립 처리

그러나 TCP 레벨에서:

전송 순서:
TCP seg 1: [Stream1 HEADERS][Stream3 HEADERS]
TCP seg 2: [Stream1 DATA-1]
TCP seg 3: [Stream3 DATA-1][Stream5 HEADERS]  ← 이 세그먼트가 손실!
TCP seg 4: [Stream1 DATA-2]
TCP seg 5: [Stream5 DATA-1]

세그먼트 3 손실 시:
  TCP: 세그먼트 3 재전송 대기
  세그먼트 4, 5는 수신됐지만 대기 (순서 보장)
  
  → Stream1의 DATA-2, Stream5의 DATA-1이
    실제로는 도착했는데 전달 불가
  → 두 스트림 모두 차단!

  이것이 HTTP/2에서도 잔존하는 TCP HOL Blocking

왜 이것이 HTTP/1.1 병렬 연결보다 나쁠 수도 있나:
  HTTP/1.1 병렬 6연결:
    6개 TCP 중 1개에서 손실 → 그 연결만 블로킹
    나머지 5개 연결은 계속 동작
  
  HTTP/2 단일 연결:
    1개 TCP에서 손실 → 모든 스트림 블로킹
  
  1% 손실 환경에서 HTTP/2 > HTTP/1.1 비교:
    항상 HTTP/2가 빠른 것은 아님!
```

### 2. QUIC의 스트림별 독립 재전송

```
QUIC 패킷 구조:
  ┌────────────────────────────────────────────────────────────────┐
  │  Short Header (1 byte flags + Connection ID + Packet Number)   │
  ├────────────────────────────────────────────────────────────────┤
  │  QUIC Frame (타입별 다양한 프레임 포함 가능)                          │
  │    STREAM Frame:                                               │
  │      Stream ID (가변 길이)                                       │
  │      Offset (스트림 내 위치)                                      │
  │      Length                                                    │
  │      Data                                                      │
  │    ACK Frame, CRYPTO Frame, PING Frame...                      │
  └────────────────────────────────────────────────────────────────┘

핵심: 하나의 QUIC 패킷에 여러 스트림의 프레임 포함 가능

스트림 독립 재전송 원리:

QUIC 패킷 1: [Stream1 offset=0-1000][Stream3 offset=0-500]
QUIC 패킷 2: [Stream1 offset=1001-2000]
QUIC 패킷 3: [Stream3 offset=501-1000][Stream5 offset=0-200]  ← 손실!
QUIC 패킷 4: [Stream1 offset=2001-3000]

패킷 3 손실 시:
  QUIC: 패킷 3의 내용 (Stream3 501-1000, Stream5 0-200) 재전송
  Stream1의 패킷 4는? → 정상 처리! (다른 스트림)
  
  차이:
    TCP: 스트림 구분 없이 바이트 스트림 → 손실 위치 이후 전체 차단
    QUIC: 스트림별 Offset 추적 → 다른 스트림은 영향 없음

QUIC Packet Number vs TCP Sequence Number:
  TCP Sequence Number:
    재전송 시 동일한 번호 사용
    → ACK가 원본 응답인지 재전송 응답인지 모호 (Karn's Algorithm 필요)
  
  QUIC Packet Number:
    재전송 시 새로운 번호 사용 (단조 증가)
    → 어느 전송에 대한 ACK인지 명확
    → RTT 측정 정확도 향상 (Karn's Algorithm 불필요)
    → 재전송 데이터는 동일 (STREAM Frame의 offset/data)
       하지만 다른 Packet Number로 감싸서 전송
```

### 3. QUIC 연결 수립과 0-RTT

```
QUIC 연결 수립 (최초 연결, 1-RTT):

Client                                    Server
│                                              │
│  Initial Packet                              │
│  (CRYPTO: TLS ClientHello)                   │ ← TLS 1.3 + QUIC Transport Params
│ ─────────────────────────────────────────── ►│
│                                              │
│  Initial + Handshake Packets                 │
│  (CRYPTO: ServerHello, Certificate, Fin)     │
│ ◄─────────────────────────────────────────── │
│                                              │
│  Handshake Packet                            │
│  (CRYPTO: Client Finished)                   │
│  [즉시 HTTP/3 요청 데이터도 포함!]                 │ ← 1-RTT에서 이미 데이터 가능
│ ─────────────────────────────────────────── ►│
│                                              │
  소요: 1 RTT

  비교:
  TCP + TLS 1.2: 1.5 RTT (TCP) + 2 RTT (TLS) = 3.5 RTT
  TCP + TLS 1.3: 1.5 RTT (TCP) + 1 RTT (TLS) = 2.5 RTT
  QUIC (최초):   1 RTT (UDP + TLS 통합)

───────────────────────────────────────────────────────────────────

0-RTT 연결 재개:

  조건: 이전에 같은 서버와 통신한 기록 있음
        클라이언트가 Session Ticket 저장

Client                                    Server
│                                              │
│  Initial Packet                              │
│  (CRYPTO: ClientHello + Session Ticket)      │ ← 이전 세션 재개
│  0-RTT Packet                                │
│  (HTTP/3 요청 데이터!)                          │ ← 핸드쉐이크 전에 데이터!
│ ─────────────────────────────────────────── ►│
│                                              │  → Session Ticket 검증
│                                              │  → 요청 처리 시작
│  Initial + Handshake Packets (응답)           │
│ ◄─────────────────────────────────────────── │
  
  효과: 핸드쉐이크 완료 전에 데이터 도달
        → 이론적 0 RTT 지연

  0-RTT Replay Attack 위험:
    공격자가 캡처한 0-RTT 패킷을 재전송
    → 서버가 같은 요청을 두 번 처리
    → 멱등 요청(GET)은 괜찮지만 POST/DELETE는 위험
    
    방어:
    ① 0-RTT 요청에 단회성 토큰 포함 (서버 추적)
    ② GET 등 안전한 메서드만 0-RTT 허용
    ③ 0-RTT 완전 비활성화 (보안 최우선 시)
```

### 4. Connection Migration

```
TCP의 IP 변경 문제:
  TCP 연결 = 4-Tuple (SrcIP, SrcPort, DstIP, DstPort)
  
  Wi-Fi → LTE 전환 시:
    SrcIP 변경 (Wi-Fi IP → LTE IP)
    → TCP 4-Tuple 변경 → 연결 끊김
    → 새 TCP + TLS 핸드쉐이크 필요
    → 진행 중이던 다운로드/업로드 중단

QUIC Connection Migration:
  Connection ID로 연결 식별 (IP/Port 무관)
  
  Wi-Fi → LTE 전환 시:
    새 IP에서 동일 Connection ID로 패킷 전송
    서버: "같은 Connection ID, IP가 바뀌었군" → 수락
    → 연결 유지, 전송 계속
  
  Connection ID 예시:
    이전: srcIP=192.168.1.100:54321, Connection ID=0xABCD1234
    이후: srcIP=10.0.0.5:12345,     Connection ID=0xABCD1234  ← 동일
    
    서버: Connection ID=0xABCD1234 → 기존 세션 찾아서 계속

QUIC Path Validation:
  새 경로에서 패킷 도착 시 서버가 즉시 수락하지 않음
  PATH_CHALLENGE 전송 → 클라이언트가 PATH_RESPONSE 반환
  → 경로가 실제로 클라이언트에게 속함을 검증 (IP 위조 방지)

실무 영향:
  동영상 스트리밍 중 지하철 탑승 (Wi-Fi → LTE):
    HTTP/2(TCP): 재연결 → 재버퍼링
    HTTP/3(QUIC): 중단 없이 스트리밍 계속
  
  화상통화 중 이동:
    HTTP/2: 통화 끊김 → 재연결
    HTTP/3: 잠깐 끊기더라도 자동 재개
```

### 5. QPACK — HTTP/3의 헤더 압축

```
HTTP/2 HPACK의 문제:
  HPACK 동적 테이블이 순서 의존적
  
  Stream 1: 헤더 A를 동적 테이블에 추가 (인덱스 62)
  Stream 3: 인덱스 62 사용 (헤더 A)
  
  하지만 Stream 1의 HEADERS가 Stream 3보다 늦게 도착하면?
  → Stream 3이 아직 동적 테이블에 없는 인덱스 62 사용 → 에러
  → 따라서 HPACK에서는 헤더 프레임 순서 보장 필요
  → HTTP/2에서 HEADERS 프레임은 순서대로 처리 (암묵적 직렬화)

QPACK (HTTP/3 헤더 압축):
  동적 테이블을 별도 단방향 스트림으로 분리
  
  구성:
  ① 양방향 스트림: HTTP 요청/응답 데이터
  ② 단방향 스트림 (Encoder → Decoder): 동적 테이블 업데이트
  ③ 단방향 스트림 (Decoder → Encoder): 테이블 확인 ACK
  
  장점:
    헤더 압축 테이블 업데이트와 요청/응답이 독립
    순서 보장 없이 동작 가능
    HOL Blocking 없는 헤더 압축

  동적 테이블 없이도 사용 가능 (Literal Without Indexing):
    스트림별 독립성이 최우선일 때
    → 압축률 희생, HOL Blocking 제거
```

---

## 💻 실전 실험

### 실험 1: curl로 HTTP/3 연결 확인

```bash
# curl HTTP/3 지원 확인 (최신 curl 필요)
curl --version | grep "HTTP3\|http3"

# HTTP/3 요청
curl -v --http3 https://cloudflare.com 2>&1 | head -20
# "Using HTTP/3" 확인

# HTTP/3 지원 여부 확인 (Alt-Svc 헤더)
curl -v https://cloudflare.com 2>&1 | grep "alt-svc\|Alt-Svc"
# alt-svc: h3=":443"; ma=86400  ← HTTP/3 지원 광고

# HTTP/2 vs HTTP/3 응답 시간 비교
curl -w "HTTP version: %{http_version}\nConnect: %{time_connect}\nTotal: %{time_total}\n" \
     --http2 -o /dev/null -s https://cloudflare.com
curl -w "HTTP version: %{http_version}\nConnect: %{time_connect}\nTotal: %{time_total}\n" \
     --http3 -o /dev/null -s https://cloudflare.com
```

### 실험 2: QUIC 패킷 캡처

```bash
# UDP 443 포트 캡처 (QUIC 트래픽)
sudo tcpdump -nn -i any 'udp port 443 and host 1.1.1.1' &

# HTTP/3 요청
curl --http3 https://1.1.1.1 -o /dev/null

# 캡처 확인
# UDP 패킷이 보임 (TCP 패킷 없음)
# QUIC 패킷은 TLS 암호화됨 → 내용 읽기 어려움

# Wireshark에서 QUIC 복호화 (키 추출 필요):
SSLKEYLOGFILE=/tmp/ssl-keys.txt curl --http3 https://cloudflare.com
# Wireshark → Preferences → TLS → (Pre)-Master-Secret log filename
# → /tmp/ssl-keys.txt 입력 후 QUIC 패킷 복호화
```

### 실험 3: 패킷 손실 환경에서 HTTP/2 vs HTTP/3

```bash
# 패킷 손실 주입 (UDP와 TCP 모두)
sudo tc qdisc add dev eth0 root netem loss 3%

# HTTP/2 성능 측정
time curl -w "%{http_version} %{time_total}\n" \
     --http2 -o /dev/null -s https://target-server/large-file

# HTTP/3 성능 측정
time curl -w "%{http_version} %{time_total}\n" \
     --http3 -o /dev/null -s https://target-server/large-file

# 손실 환경에서 HTTP/3이 우위를 보임 확인

sudo tc qdisc del dev eth0 root
```

### 실험 4: 0-RTT 동작 확인

```bash
# openssl을 이용한 QUIC 연결 (최신 openssl)
# 첫 연결 (1-RTT)
openssl s_client -connect cloudflare.com:443 \
  -quic -sess_out /tmp/session.pem 2>&1 | grep -E "TLS|RTT|Early"

# 세션 재개로 0-RTT
openssl s_client -connect cloudflare.com:443 \
  -quic -sess_in /tmp/session.pem -early_data /dev/null 2>&1 | \
  grep -E "Early data|0-RTT"
# "Early data was accepted" 확인 시 0-RTT 성공
```

---

## 📊 성능/비용 비교

```
프로토콜별 연결 수립 비용 비교:

┌──────────────────────────────────────────────────────────────────────┐
│  프로토콜                 │  최초 연결          │  재연결 (세션 재개)        │
├──────────────────────────────────────────────────────────────────────┤
│  HTTP/1.1 + TLS 1.2     │  3.5 RTT          │  2.5 RTT               │
│  HTTP/1.1 + TLS 1.3     │  2.5 RTT          │  1.5 RTT               │
│  HTTP/2   + TLS 1.3     │  2.5 RTT          │  1.5 RTT               │
│  HTTP/3   + QUIC (1-RTT)│  1 RTT            │  1 RTT                 │
│  HTTP/3   + QUIC (0-RTT)│  1 RTT            │  0 RTT (데이터)          │
└──────────────────────────────────────────────────────────────────────┘

패킷 손실 환경에서 처리량 비교 (RTT=50ms, 손실률 2%):
  HTTP/2:   TCP HOL → 처리량 약 40% 감소
  HTTP/3:   스트림 독립 → 처리량 약 10% 감소 (훨씬 탄력적)

모바일 환경 (Wi-Fi → LTE 전환):
  HTTP/2 재연결 시간: 2.5 RTT = 375ms (RTT=150ms 기준)
  HTTP/3 마이그레이션: 1-2 RTT 정도 끊김 → 자동 재개

Google 내부 데이터 (2020):
  QUIC vs TCP: 검색 지연 8% 감소
  YouTube 버퍼링: 18% 감소
  손실 환경에서 더 큰 개선

Cloudflare 측정:
  HTTP/3 활성화 시 TTFB(Time To First Byte) 12% 감소
  패킷 손실 환경에서 50% 이상 성능 차이
```

---

## ⚖️ 트레이드오프

```
HTTP/3 도입의 트레이드오프:

장점:
  ① TCP HOL Blocking 완전 제거 (스트림별 독립)
  ② 빠른 연결 수립 (1 RTT, 0 RTT)
  ③ Connection Migration (모바일 환경)
  ④ 고손실/고지연 환경에서 강인함

단점:
  ① UDP 차단:
     일부 방화벽/기업 네트워크가 UDP 443 차단
     → HTTP/2로 자동 fallback (클라이언트가 처리)
     → 실제로 약 5-10% 환경에서 UDP 차단
  
  ② CPU 오버헤드:
     QUIC은 사용자 공간(User Space) 구현
     TCP는 커널이 처리 → QUIC보다 효율적
     고트래픽 서버에서 CPU 사용량 증가
  
  ③ 디버깅 어려움:
     UDP 패킷 + TLS 암호화 → 캡처/분석 복잡
     Wireshark에 키 파일 필요
  
  ④ 생태계 성숙도:
     TCP에 비해 미들박스(방화벽, 로드밸런서) 지원 미흡
     QoS(Quality of Service) 적용 어려움

언제 HTTP/3가 HTTP/2보다 느릴 수 있나:
  LAN 환경 (패킷 손실 거의 없음):
    QUIC의 사용자 공간 처리 오버헤드가 더 클 수 있음
  
  CPU 집약적 서버:
    QUIC 처리로 인한 CPU 증가 → 다른 처리에 영향
```

---

## 📌 핵심 정리

```
HTTP/3와 QUIC 핵심 요약:

HTTP/2의 잔존 문제:
  TCP HOL Blocking: 패킷 손실 시 모든 스트림 차단
  Connection Migration 불가: IP 변경 시 재연결 필요

QUIC의 해결책:
  UDP 기반: TCP 커널 스택 우회, 사용자 공간 구현
  스트림별 독립: 패킷 손실이 해당 스트림만 영향
  Packet Number 단조 증가: RTT 측정 정확도 향상
  Connection ID: IP 변경 시 연결 유지

연결 수립:
  최초 1-RTT: UDP + TLS 1.3 통합 핸드쉐이크
  재연결 0-RTT: Session Ticket으로 즉시 데이터 전송
  0-RTT 주의: 멱등 요청(GET)에만 사용 (Replay Attack 방지)

Connection Migration:
  Connection ID로 연결 식별 (IP/Port 무관)
  Wi-Fi → LTE: 중단 없는 연결 유지
  Path Validation: 새 경로 검증 (IP 위조 방지)

QPACK:
  별도 스트림으로 동적 테이블 관리
  헤더 압축에서도 HOL Blocking 없음

도입 판단:
  모바일 트래픽 많음 → 효과 큼
  패킷 손실 잦음 → 효과 큼
  기업 방화벽 환경 → 차단 가능성 고려

진단 도구:
  curl --http3 -v: HTTP/3 연결 확인
  tcpdump udp port 443: QUIC 패킷 캡처
  Chrome DevTools: h3 프로토콜 표시
```

---

## 🤔 생각해볼 문제

**Q1.** UDP 기반인 QUIC이 기업 방화벽에서 차단되면 사용자는 어떤 경험을 하는가? 브라우저와 서버는 이를 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**사용자 경험:** 자동으로 HTTP/2로 fallback되므로 사용자는 차이를 느끼지 못합니다. 다만 HTTP/3의 성능 이점은 받지 못합니다.

**브라우저의 처리 과정:**

1. 서버가 `Alt-Svc: h3=":443"` 헤더를 이전 연결에서 전달
2. 브라우저가 다음 연결에서 UDP 443으로 QUIC 시도
3. 방화벽이 UDP 차단 → 타임아웃 (약 300ms~3초)
4. 브라우저: HTTP/2로 fallback

**Happy Eyeballs 유사 메커니즘 (Chrome):**
- TCP와 UDP를 병렬로 시도
- 먼저 성공하는 것 사용
- QUIC이 실패해도 TCP가 대기 중 → 빠른 fallback

**서버 측 대응:**
```nginx
# Alt-Svc로 HTTP/3 광고
add_header Alt-Svc 'h3=":443"; ma=86400';

# QUIC이 차단된 클라이언트는 자동으로 HTTP/2 사용
# 서버는 둘 다 지원하면 됨
listen 443 ssl;      # TCP (HTTP/2용)
listen 443 quic;     # UDP (HTTP/3용)
```

**실제 통계:** 전체 인터넷 트래픽의 약 5-10%에서 UDP 443 차단. 대부분의 소비자 인터넷은 차단 없음. 기업 방화벽이 주요 차단 원인.

</details>

---

**Q2.** QUIC의 Connection Migration 기능이 보안 관점에서 갖는 위험은 무엇이고 어떻게 방어하는가?

<details>
<summary>해설 보기</summary>

**위험 1: IP 스푸핑 기반 세션 탈취**

공격자가 피해자의 Connection ID를 알고, 자신의 IP로 해당 Connection ID를 사용해 패킷을 보내면 서버가 연결을 마이그레이션할 수 있습니다.

**QUIC의 방어: Path Validation**
```
새 경로에서 패킷 수신 시:
서버 → PATH_CHALLENGE (무작위 8바이트)
클라이언트 → PATH_RESPONSE (같은 8바이트)
서버: 올바른 응답 → 경로 유효성 확인
공격자: 응답 불가 → 마이그레이션 거부
```

**위험 2: Connection ID 유추**

Connection ID가 예측 가능하면 공격자가 다른 사용자의 세션을 탈취 시도 가능.

**방어:** Connection ID는 암호학적 무작위 값 사용. 또한 QUIC은 마이그레이션 시 새로운 Connection ID로 변경을 권장 (Preferred Address).

**위험 3: 서비스 거부 (DoS)**

공격자가 대량의 IP에서 피해자의 Connection ID로 PATH_CHALLENGE를 유발 → 서버가 검증 트래픽 폭증.

**방어:** QUIC에서 PATH_CHALLENGE 횟수 제한 및 Rate Limiting.

**Spring Boot/Netty의 처리:** 대부분 자동. 커스텀 QUIC 서버 구현 시 Path Validation 로직 직접 구현 필요.

</details>

---

**Q3.** QUIC이 커널이 아닌 사용자 공간에서 구현된다는 것의 의미는 무엇이고, 이것이 성능에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**커널 공간 vs 사용자 공간:**

- **TCP**: 리눅스 커널 내부 구현. 직접 하드웨어/네트워크 스택 접근. 컨텍스트 스위칭 없음.
- **QUIC**: 사용자 공간(애플리케이션 레벨)에서 UDP 소켓 위에 구현.

**사용자 공간 구현의 비용:**

1. **시스템 콜 오버헤드**: 패킷 송수신마다 `sendmsg()`, `recvmsg()` 시스템 콜 → 커널-사용자 공간 전환
2. **복사 비용**: 커널 버퍼 → 사용자 공간 버퍼 데이터 복사
3. **CPU 사용량**: TCP 처리는 커널이 하지만 QUIC은 애플리케이션 CPU 사용

**최적화 기술:**

- **GSO (Generic Segmentation Offload)**: 여러 QUIC 패킷을 하나의 시스템 콜로 전송
- **GRO (Generic Receive Offload)**: 수신 시 패킷 묶음 처리
- **io_uring**: 비동기 I/O로 시스템 콜 최소화
- **XDP (eXpress Data Path)**: 커널 우회 고속 패킷 처리

**현실적 성능:**

Google 측정: QUIC 처리가 TCP 대비 CPU 사용량 3.5배 많음 (초기). 최적화 후 약 1.5~2배로 감소.

고트래픽 서버(1Gbps+)에서: QUIC의 CPU 오버헤드가 전체 처리량에 영향. TCP로도 충분한 경우 굳이 HTTP/3으로 전환할 필요 없음.

**미래**: QUIC 커널 구현 진행 중 (Linux 6.x). 커널에서 처리되면 TCP와 유사한 성능 기대.

</details>

---

<div align="center">

**[⬅️ 이전: HTTP/2](./02-http2-multiplexing.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP 캐싱 ➡️](./04-http-caching.md)**

</div>
