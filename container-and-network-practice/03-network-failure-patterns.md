# 운영에서 만나는 네트워크 문제 패턴 — RST, Timeout, 포트 고갈

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Connection Reset(RST)이 발생하는 정확한 조건은 무엇인가?
- keep-alive timeout 불일치가 RST를 유발하는 메커니즘은?
- Connection Timeout, Read Timeout, Connection Refused는 각각 왜 발생하는가?
- Ephemeral Port 고갈은 어떻게 발생하고 어떻게 진단하는가?
- TIME_WAIT 소켓이 너무 많아지면 어떤 문제가 생기는가?
- `ss -s`로 소켓 통계를 어떻게 해석하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"간헐적으로 Connection Reset by peer가 납니다":
  원인 후보:
  ① Nginx keepalive_timeout vs Java RestTemplate keepalive 불일치
  ② AWS LB idle timeout vs 앱 keepalive timeout 불일치
  ③ 방화벽이 idle 연결 강제 종료 후 RST 전송
  ④ 서버가 응답 전 연결 종료 (서버 재시작, OOM)
  
  진단:
  tcpdump -nn port 8080 -w capture.pcap
  → Wireshark로 RST 패킷 찾기
  → RST 발신자가 서버인지 클라이언트인지 확인

"새벽에 갑자기 connection refused 에러 폭증":
  배치 작업이 새벽에 대량 요청 → 포트 고갈
  ephemeral port: 32768~60999 (2만 8천개)
  초당 1000 요청 × 30초 TIME_WAIT = 3만 소켓 → 고갈!
  
  확인: ss -s | grep TIME-WAIT
  해결: ip_local_port_range 확대
        tcp_tw_reuse=1
        Connection Pool 사용
```

---

## 😱 흔한 실수

```
Before — 네트워크 문제 패턴을 모를 때:

실수 1: RST를 서버 버그로 오해
  "서버가 갑자기 연결을 끊는다"
  → 실제: 서버 idle timeout < 클라이언트 keepalive 시도 간격
  → 서버가 연결 닫은 후 클라이언트가 재사용 시도 → RST
  → 해결: 클라이언트 keepalive 주기를 서버 timeout보다 짧게

실수 2: Connection Pool이 있으면 포트 고갈 없다고 생각
  DB Connection Pool: 연결 재사용 → DB 연결 포트 고갈 방지
  하지만 Pool 없이 HTTP 호출 반복:
  각 HTTP 요청마다 새 소켓 → TIME_WAIT 누적 → 포트 고갈

실수 3: Read Timeout과 Connection Timeout 혼동
  Connection Timeout: 서버에 연결 자체가 안 됨 (3초 → 서버 다운)
  Read Timeout: 연결됐지만 응답이 안 옴 (30초 → 서버 처리 중)
  → 다른 원인, 다른 해결책

실수 4: TIME_WAIT = 나쁜 것으로 오해
  TIME_WAIT: 정상적인 연결 종료 후 필요한 상태
  목적: 지연 패킷 처리, 중복 연결 방지
  문제가 되는 경우: 수가 너무 많아 포트 고갈
  → 적당한 TIME_WAIT는 정상
```

---

## ✨ 올바른 접근

```
After — 네트워크 문제 패턴을 알면:

keep-alive 불일치 방지:
  서버 (Nginx):
    keepalive_timeout 65s;  # 65초 idle 후 연결 종료
    keepalive_requests 1000;
  
  클라이언트 (Spring RestTemplate):
    # 클라이언트 keepalive 간격 < 서버 timeout
    # keepalive 60s < nginx 65s → 안전
    CloseableHttpClient client = HttpClients.custom()
        .setConnectionManager(cm)
        .setKeepAliveStrategy((response, context) ->
            TimeValue.ofSeconds(55))  # 65초보다 짧게!
        .build();
  
  AWS ALB (기본 60초):
    → 앱 keepalive를 55초 이하로 설정

포트 고갈 예방:
  # /etc/sysctl.conf
  net.ipv4.ip_local_port_range = 1024 65000  # 64000개
  net.ipv4.tcp_tw_reuse = 1     # TIME_WAIT 소켓 재사용
  net.ipv4.tcp_fin_timeout = 30 # FIN_WAIT2 단축
  
  sysctl -p  # 적용
  
  Connection Pool 반드시 사용:
  HikariCP (DB), Apache HttpClient Pool (HTTP)

소켓 상태 모니터링:
  # 실시간 소켓 통계
  watch -n 1 'ss -s'
  
  # TIME_WAIT 수 모니터링
  while true; do
    echo -n "$(date +%H:%M:%S) TIME_WAIT: "
    ss -s | grep "TIME-WAIT"
    sleep 5
  done
  
  # 알람 설정 (Prometheus)
  node_sockstat_TCP_tw > 10000 → 경보
```

---

## 🔬 내부 동작 원리

### 1. TCP RST — 강제 연결 종료

```
RST (Reset) 패킷:
  TCP 정상 종료 (FIN): 4-way Handshake
  RST: 즉각 강제 종료, 상대방에게 통보

RST가 발생하는 조건:

조건 1: 없는 포트로 연결 시도
  클라이언트 → SYN (서버:9999)
  서버: 9999 포트 리스닝 없음
  서버 OS → RST (연결 거부)
  결과: Connection Refused (RST와 같은 효과)

조건 2: 닫힌 연결에 데이터 전송
  서버가 연결을 FIN으로 종료
  클라이언트가 아직 모르고 데이터 전송
  서버 → RST (이미 닫힌 연결)
  결과: Connection Reset by peer

조건 3: SO_LINGER = 0으로 소켓 닫기
  서버가 강제로 소켓 닫음 (사용자가 설정)
  FIN 대신 RST 전송
  → 연결 끊기는 빠르지만 상대방이 데이터 잃을 수 있음

조건 4: 방화벽이 연결을 끊고 RST 전송
  방화벽: idle 연결 타임아웃 → 상태 테이블에서 제거
  이후 해당 연결의 패킷 도착 → RST 전송
  결과: 앱이 RST 받음 → Connection Reset

keep-alive 불일치로 RST 발생:

  서버: keepalive_timeout=30s (30초 후 연결 종료)
  클라이언트: keepalive=60s (60초마다 요청 가능성)
  
  t=0:   클라이언트 → 서버 연결 수립
  t=30:  서버: 30초 idle → FIN 전송 → 연결 종료
  t=35:  클라이언트: 기존 연결 재사용 → GET 요청
  t=35:  서버 OS: 이미 닫힌 연결 → RST 응답
  클라이언트 앱: "Connection reset by peer" 오류

해결:
  클라이언트 keepalive < 서버 keepalive_timeout
  예: 서버 65초, 클라이언트 55초 → 안전 마진 10초

Wireshark로 RST 찾기:
  필터: tcp.flags.reset == 1
  → RST 패킷의 출발지 확인 (서버 vs 클라이언트)
  → RST 직전 패킷 확인 (어떤 데이터가 트리거했는지)
```

### 2. Timeout 종류별 원인과 진단

```
Connection Timeout (연결 자체 실패):
  
  증상: "Connection timed out" (예: 3초 후)
  
  원인:
  ① 서버 다운 (SYN 패킷에 응답 없음)
  ② 방화벽이 SYN 차단 (패킷 드롭)
  ③ 잘못된 IP/포트 (라우팅 없음)
  ④ 서버 백로그 큐 가득 참 (SYN 드롭)
  
  진단:
    # SYN 패킷이 가는지 확인
    tcpdump -nn host target-ip and port 8080
    # SYN만 보이고 SYN-ACK 없음 → 방화벽 또는 서버 다운
    
    # 서버의 포트 바인딩 확인
    ss -tlnp | grep 8080
    
    # 방화벽 규칙 확인
    sudo iptables -L -n | grep target-ip

Read Timeout (응답 지연):
  
  증상: "Read timed out" (예: 30초 후)
  
  원인:
  ① 서버 처리가 오래 걸림 (DB 슬로우 쿼리)
  ② 서버 스레드 풀 고갈 (큐잉)
  ③ 서버가 응답을 보내기 시작했지만 중간에 멈춤
  ④ 네트워크 패킷 손실로 재전송 반복
  
  진단:
    # 서버 응답 시간 확인
    curl -w "%{time_total}" http://server/api
    
    # 서버 스레드 상태
    jstack [PID] | grep "in Object.wait\|BLOCKED"
    
    # 서버 로그에서 슬로우 쿼리 확인
    grep "took [0-9]" application.log | tail -20

Connection Refused (연결 거부):
  
  증상: "Connection refused" (즉시)
  
  원인:
  ① 해당 포트에 프로세스 없음 (서버 미실행)
  ② OS가 RST로 거부 (포트 없음)
  
  특징: Timeout이 없음 (즉시 거부)
  
  진단:
    ss -tlnp | grep 8080  # 포트 리스닝 여부
    systemctl status my-service  # 서비스 상태

Connection Reset (연결 초기화):
  
  증상: "Connection reset by peer"
  
  원인:
  ① keep-alive 불일치 (위 설명 참조)
  ② 서버 재시작 중 연결 처리
  ③ 방화벽 세션 만료 후 RST
  ④ 서버 OOM으로 프로세스 종료
  
  진단:
    tcpdump으로 RST 발신자 확인
    서버 로그에서 비정상 종료 흔적
    방화벽 로그 확인

오류 구분 요약:
  Connection Refused: 포트 없음 → 즉시 오류
  Connection Timeout: 응답 없음 → N초 대기 후 오류
  Read Timeout:       연결됐지만 응답 지연 → N초 대기 후 오류
  Connection Reset:   연결 강제 종료 → 즉시 오류 (RST 수신)
```

### 3. Ephemeral Port 고갈

```
TCP 연결의 4-Tuple:
  (출발지 IP, 출발지 Port, 목적지 IP, 목적지 Port)
  
  목적지 IP와 포트가 고정이면:
  출발지 포트로만 연결 구분 가능
  
  클라이언트가 같은 서버에 반복 연결:
  각 연결마다 새 출발지 포트 필요
  → Ephemeral Port (임시 포트) 범위에서 할당

기본 포트 범위:
  cat /proc/sys/net/ipv4/ip_local_port_range
  # 32768  60999  → 28,232개

고갈 시나리오:
  배치 서버 → 외부 API (같은 서버 IP:443)
  초당 1,000 요청
  TIME_WAIT 상태 지속 시간: 60초 (2MSL)
  
  동시 TIME_WAIT 소켓 수:
  1,000/초 × 60초 = 60,000개
  
  가용 포트: 28,232개 < 60,000개
  → 포트 고갈! → Connection refused (포트 할당 실패)

진단:
  ss -s
  # Total: 89432 (커널 최대: 131072)
  # TCP: 62341 (established 1200, TIME-WAIT 60000, ...)
  
  ss -tn state time-wait | wc -l
  # TIME-WAIT 소켓 수 확인
  
  # 특정 목적지로의 TIME_WAIT 확인
  ss -tn state time-wait 'dst 203.0.113.10'

해결 방법:

1. 포트 범위 확대:
  echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range
  # 64,511개 → 2배 이상

2. tcp_tw_reuse 활성화:
  echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
  # TIME_WAIT 소켓 재사용 가능
  # 같은 4-Tuple로 새 연결 시 TIME_WAIT 소켓 재사용
  # 단, tcp_timestamps = 1 필요

3. Connection Pool 사용:
  HTTP Keep-Alive Pool:
  닫지 않고 재사용 → TIME_WAIT 발생 안 함
  DB Pool: 연결 재사용 → 포트 재사용

4. TIME_WAIT 단축 (주의):
  # 기본 60초 → 짧게
  echo 30 > /proc/sys/net/ipv4/tcp_fin_timeout
  # 주의: 너무 짧으면 지연 패킷 처리 문제

5. SO_REUSEADDR 소켓 옵션:
  서버 재시작 시 기존 포트 즉시 재사용
  Java: serverSocket.setReuseAddress(true)
```

### 4. ss 명령어로 소켓 통계 해석

```
ss -s (소켓 요약 통계):
  Total: 89432           ← 총 소켓 수
  TCP:   62341           ← TCP 소켓
    orphaned: 0          ← 프로세스 없이 남은 소켓
    TIME-WAIT: 8241      ← ← 이 숫자가 중요!
    timewait: 8241 (reuse:0)
    
  UDP:   125
  RAW:   0
  FRAG:  0 0

TIME-WAIT 기준:
  < 1,000:   정상
  1,000~5,000: 주시
  > 10,000:   주의 (포트 고갈 위험 확인)
  > 포트 범위: 위험!

ss 상세 옵션:
  ss -tn                # TCP 연결, 이름 해석 없이 (빠름)
  ss -tlnp             # TCP 리스닝, 프로세스 포함
  ss -tn state established  # ESTABLISHED 연결만
  ss -tn state time-wait    # TIME-WAIT만
  
  ss --info            # 각 소켓의 상세 정보 (cwnd, rtt, rto)
  # State   Send-Q  Recv-Q  Local    Peer
  # ESTAB   0       0       ...      ...
  # cwnd:10 rto:200 rtt:15.3/7.5 ato:40 mss:1448

Send-Q와 Recv-Q:
  Send-Q: 전송 버퍼에 쌓인 데이터 (응답 대기 중)
          > 0 이면: 상대방이 느리거나 네트워크 혼잡
  Recv-Q: 수신 버퍼에 쌓인 데이터 (앱이 읽지 않음)
          > 0 이면: 앱이 데이터를 처리 못하고 있음
          → 스레드 풀 고갈, 앱 처리 지연 신호

리스닝 소켓의 Recv-Q:
  ss -tlnp | grep :8080
  # State    Recv-Q  Send-Q  Local
  # LISTEN   0       128     0.0.0.0:8080
  # Recv-Q=0: 대기 연결 없음 (정상)
  # Recv-Q>0: accept() 대기 중인 연결 (backlog)
  # Recv-Q=Send-Q: backlog 가득 참 → SYN 드롭 시작
```

---

## 💻 실전 실험

### 실험 1: RST 발생 시뮬레이션

```bash
# 서버: 짧은 keepalive 설정 (10초)
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import socket

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Connection', 'keep-alive')
        self.end_headers()
        self.wfile.write(b'OK')
    def log_message(self, *args): pass

class ShortKeepaliveHTTPServer(HTTPServer):
    def server_bind(self):
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
        self.socket.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 5)
        super().server_bind()

ShortKeepaliveHTTPServer(('127.0.0.1', 9999), Handler).serve_forever()
" &

# tcpdump으로 RST 캡처
sudo tcpdump -nn -i lo port 9999 -w /tmp/rst.pcap &

# 클라이언트: keep-alive 연결 후 서버 timeout 후에 재사용
python3 -c "
import requests
import time

session = requests.Session()
adapter = requests.adapters.HTTPAdapter()
session.mount('http://', adapter)

session.get('http://127.0.0.1:9999/')
print('첫 요청 성공')
time.sleep(15)  # 서버 keepalive(10초) 초과 대기
try:
    session.get('http://127.0.0.1:9999/')  # RST 받을 수 있음
    print('두 번째 요청 성공')
except Exception as e:
    print(f'오류 발생: {e}')  # Connection reset by peer
"

# RST 패킷 확인
sudo tcpdump -r /tmp/rst.pcap -nn | grep RST
```

### 실험 2: Ephemeral Port 고갈 관찰

```bash
# 현재 포트 범위 확인
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768 60999

# 빠른 연결로 TIME_WAIT 생성
python3 -c "
import socket
import time

# 짧은 연결을 반복해서 TIME_WAIT 누적
target = ('httpbin.org', 80)
for i in range(1000):
    try:
        s = socket.socket()
        s.connect(target)
        s.send(b'GET / HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n')
        s.recv(100)
        s.close()  # → TIME_WAIT 상태로 전환
        if i % 100 == 0:
            print(f'연결 {i}회')
    except Exception as e:
        print(f'포트 고갈: {e}')
        break
"

# TIME_WAIT 증가 확인
ss -s | grep "TIME-WAIT"
watch -n 1 'ss -tn state time-wait | wc -l'
```

### 실험 3: 소켓 상태 실시간 모니터링

```bash
# 전체 TCP 상태 요약
ss -s

# ESTABLISHED 연결 상세
ss -tn state established | head -20

# 특정 포트의 연결 확인
ss -tn sport = :8080 or dport = :8080

# 소켓 상세 정보 (cwnd, rtt)
ss --info -tn state established | head -10
# Recv-Q Send-Q ...
# cwnd:10 rto:200 rtt:15/7 ato:40 mss:1448

# 연결 수 추이 모니터링
while true; do
  echo -n "$(date +%H:%M:%S) "
  ss -s | grep "TCP:"
  sleep 2
done

# Prometheus node_exporter로 자동 메트릭화
# node_sockstat_TCP_tw: TIME_WAIT 수
# node_sockstat_TCP_alloc: 할당된 TCP 소켓
```

### 실험 4: 다양한 오류 시뮬레이션

```bash
# Connection Refused 시뮬레이션
time curl -v http://localhost:9876  # 없는 포트
# curl: (7) Failed to connect to localhost port 9876: Connection refused
# 즉시 실패 (Timeout 없음)

# Connection Timeout 시뮬레이션
# iptables로 패킷 드롭 (SYN 차단)
sudo iptables -A INPUT -p tcp --dport 9876 -j DROP
python3 -c "
import socket
s = socket.socket()
s.settimeout(5)
try:
    s.connect(('127.0.0.1', 9876))  # SYN 보냈지만 응답 없음
except socket.timeout:
    print('Connection Timeout (5초)')
"
sudo iptables -D INPUT -p tcp --dport 9876 -j DROP

# Read Timeout 시뮬레이션
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import time

class SlowHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        time.sleep(60)  # 60초 지연 후 응답
        self.send_response(200)
        self.end_headers()
    def log_message(self, *args): pass

HTTPServer(('127.0.0.1', 9999), SlowHandler).serve_forever()
" &

curl --max-time 5 http://127.0.0.1:9999/
# curl: (28) Operation timed out after 5000 milliseconds (Read Timeout)
```

---

## 📊 성능/비용 비교

```
포트 고갈 대응별 효과:

기본 설정:
  포트 범위: 32768~60999 = 28,232개
  초당 1000 연결, TIME_WAIT 60초
  동시 TIME_WAIT: 60,000 > 28,232 → 고갈!

포트 범위 확대 후:
  포트 범위: 1024~65535 = 64,511개
  동시 TIME_WAIT: 60,000 < 64,511 → 안전

tcp_tw_reuse 활성화:
  TIME_WAIT 소켓 재사용
  실질적 가용 포트: 훨씬 증가
  주의: tcp_timestamps 함께 활성화 필요

Connection Pool 사용:
  Pool 크기 100: 항상 100개 연결 유지
  새 연결 없음 → TIME_WAIT 발생 안 함
  포트 고갈 완전 방지 (연결 수 = Pool 크기)

소켓 상태별 메모리 사용:
  ESTABLISHED: ~2KB (send/recv 버퍼 포함)
  TIME_WAIT: ~100 bytes (최소화됨)
  10,000 TIME_WAIT: 약 1MB (메모리 영향 미미)
  → 포트 고갈이 메모리 고갈보다 먼저 발생
```

---

## ⚖️ 트레이드오프

```
tcp_tw_reuse 활성화:
  장점: TIME_WAIT 소켓 포트 재사용 → 포트 고갈 완화
  단점: 이전 연결의 지연 패킷이 새 연결과 섞일 위험
        timestamps 없으면 위험 (timestamps가 구 패킷 필터)
  안전 조건: net.ipv4.tcp_timestamps=1 (기본값 1)

tcp_tw_recycle (사용 금지):
  더 공격적인 TIME_WAIT 재활용
  NAT 뒤 클라이언트와 통신 시 연결 거부 발생
  리눅스 4.12에서 제거됨

keepalive timeout 설정:
  짧게: 빠른 연결 해제 → TIME_WAIT 빨리 증가
  길게: 연결 재사용 → TIME_WAIT 감소 but 서버 리소스 점유
  → 적정값: 사용 패턴 분석 후 결정 (30~60초)

Connection Pool 크기:
  작게: 고갈 시 대기 발생, 낮은 리소스 사용
  크게: 항상 빠른 연결, 높은 리소스 사용
  → P99 지연과 리소스 사용량의 균형

타임아웃 설정:
  Connection Timeout:
    짧게: 빠른 장애 감지, 빠른 Failover
    길게: 일시적 지연 허용
    권장: 3~5초 (서버 응답 없으면 다운으로 판단)
  
  Read Timeout:
    API 성격에 따라 다름
    일반 API: 10~30초
    보고서/배치: 60~120초
    → 너무 길면 스레드 점유 오래됨 (서킷 브레이커와 조합)
```

---

## 📌 핵심 정리

```
네트워크 문제 패턴 핵심 요약:

RST 발생 조건:
  닫힌 연결에 데이터 전송
  keepalive 불일치 (서버가 먼저 닫은 후 클라이언트 재사용)
  방화벽 세션 만료 후 RST
  → 클라이언트 keepalive < 서버 keepalive_timeout 설정

Timeout 종류:
  Connection Timeout: SYN에 응답 없음 (서버 다운, 방화벽)
  Read Timeout: 연결됐지만 응답 지연 (처리 중, 스레드 고갈)
  Connection Refused: RST 즉시 (포트 없음)
  Connection Reset: RST 수신 (keepalive 불일치 등)

Ephemeral Port 고갈:
  짧은 연결 반복 → TIME_WAIT 누적
  포트 범위 (기본 28,232개) 초과 → 고갈
  해결: 포트 범위 확대, tcp_tw_reuse, Connection Pool

TIME_WAIT:
  정상적인 연결 종료 후 상태 (2MSL = 60초)
  소켓은 존재하지만 리소스 적음 (~100 bytes)
  문제: 포트 고갈 (수 > 포트 범위)

소켓 진단:
  ss -s: 전체 소켓 통계 요약
  ss -tn state time-wait | wc -l: TIME_WAIT 수
  ss --info -tn: cwnd, rtt, rto 상세
  Send-Q > 0: 상대방이 느림 또는 혼잡
  Recv-Q > 0: 앱이 데이터 처리 못함

tcpdump로 RST 찾기:
  tcpdump -nn -w capture.pcap
  Wireshark: tcp.flags.reset == 1 필터
  RST 발신자 확인 → 원인 분석
```

---

## 🤔 생각해볼 문제

**Q1.** AWS ALB의 기본 idle timeout이 60초인데 Spring Boot 앱이 `Connection reset by peer` 오류를 간헐적으로 받는다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**원인:** ALB가 60초 idle 후 연결을 종료합니다. 그런데 Spring Boot의 HTTP 클라이언트(또는 RestTemplate)가 이전 연결을 Pool에 유지하다가 ALB가 종료한 연결을 재사용하면 RST를 받습니다.

**해결 방법:**

1. **HTTP 클라이언트 keepalive를 ALB timeout보다 짧게:**
   ```java
   // Apache HttpClient
   ConnectionConfig connConfig = ConnectionConfig.custom()
       .setSocketTimeout(Timeout.ofSeconds(30))
       .build();
   
   // Keep-Alive 전략: 55초 (ALB 60초보다 짧게)
   CloseableHttpClient client = HttpClients.custom()
       .setKeepAliveStrategy((response, context) -> 
           TimeValue.ofSeconds(55))
       .evictExpiredConnections()  // 만료된 연결 제거
       .evictIdleConnections(TimeValue.ofSeconds(50))  // idle 50초 후 제거
       .build();
   ```

2. **ALB idle timeout 증가:**
   AWS 콘솔 → ALB → Attributes → Idle timeout → 120초로 증가
   단, ALB 비용 약간 증가 (연결 유지 시간 증가)

3. **Retry 로직 추가:**
   RST 받으면 자동으로 새 연결로 재시도
   ```java
   @Retryable(include = IOException.class, maxAttempts = 2)
   public ResponseEntity<Data> callApi() {
       return restTemplate.getForEntity(url, Data.class);
   }
   ```

4. **Root 원인 확인:**
   - ALB Access Log에서 연결 타임아웃 로그 확인
   - `curl -v --keepalive-time 55 http://alb-endpoint`로 재현

**권장:** 방법 1 (클라이언트 keepalive 단축) + 방법 3 (Retry)가 가장 안정적.

</details>

---

**Q2.** `ss -s` 출력에서 `Send-Q`가 계속 0이 아닌 값으로 유지된다. 무엇을 의미하고 어떻게 조치하는가?

<details>
<summary>해설 보기</summary>

**Send-Q가 0이 아닌 경우:**

`ss -tn`으로 연결 목록에서:
```
ESTAB  0  1024  10.0.0.1:8080  10.0.0.2:54321
```
여기서 Send-Q(1024)는 전송 버퍼에 ACK를 받지 못한 1024 bytes가 있다는 의미입니다.

**원인:**
1. **클라이언트가 느리게 수신:** 클라이언트 recv 버퍼가 가득 참 → 서버 Send-Q 증가
2. **네트워크 혼잡:** 패킷 손실 → 재전송 대기 → Send-Q 증가
3. **클라이언트 응답 안 함:** ACK 없음 → Send-Q 계속 누적

**`LISTEN` 소켓의 Send-Q:**
```
LISTEN  128  0  *:8080
```
- Send-Q = listen backlog 크기 (128)
- Recv-Q = 현재 대기 중인 연결 수 (accept 안 된 연결)
- Recv-Q = 128 (= Send-Q) → backlog 가득 참 → SYN 드롭 시작!

**조치:**
```bash
# 1. 특정 연결의 상세 확인
ss --info -tn | grep "10.0.0.2:54321"
# cwnd, rtt, retrans 값 확인

# 2. 클라이언트 측 확인 (Recv-Q)
# 클라이언트 서버에서: ss -tn
# Recv-Q > 0 → 앱이 데이터 읽지 않음

# 3. 네트워크 패킷 손실 확인
ping -c 100 client-ip | tail -3

# 4. 느린 클라이언트라면 → 타임아웃 설정으로 연결 정리
# send_timeout 30s; (Nginx)
```

</details>

---

**Q3.** 서버 재시작 후 즉시 같은 포트를 다시 바인딩하면 "Address already in use" 오류가 발생한다. 왜 그런가?

<details>
<summary>해설 보기</summary>

**원인: TIME_WAIT 상태의 소켓**

서버가 종료될 때 기존 연결들이 FIN을 통해 정상 종료됩니다. 이때 서버 측 소켓은 TIME_WAIT 상태로 2MSL(약 60초)동안 유지됩니다. 이 상태에서 같은 주소:포트를 바인딩하려 하면 "Address already in use" 오류가 발생합니다.

**해결 방법:**

1. **SO_REUSEADDR 소켓 옵션 (가장 일반적):**
   ```java
   ServerSocket serverSocket = new ServerSocket();
   serverSocket.setReuseAddress(true);
   serverSocket.bind(new InetSocketAddress(8080));
   ```
   ```python
   sock = socket.socket()
   sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
   sock.bind(('0.0.0.0', 8080))
   ```
   Spring Boot, Netty, Nginx는 기본으로 SO_REUSEADDR 설정.

2. **기다리기 (2MSL ≈ 60초):**
   TIME_WAIT이 자연스럽게 만료될 때까지 대기.

3. **net.ipv4.tcp_fin_timeout 단축:**
   ```bash
   echo 15 > /proc/sys/net/ipv4/tcp_fin_timeout  # 30초 → 15초
   ```
   주의: 너무 짧으면 지연 패킷 처리 문제.

4. **SO_REUSEPORT (다중 프로세스 동일 포트 공유):**
   Nginx worker 등이 사용. 여러 프로세스가 같은 포트에 바인딩 가능.

**왜 TIME_WAIT이 필요한가:**
이 대기 시간이 없으면 새 연결이 이전 연결의 지연된 패킷을 받을 수 있습니다. TIME_WAIT은 이를 방지하는 안전장치입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Kubernetes 네트워킹](./02-kubernetes-networking.md)** | **[홈으로 🏠](../README.md)** | **[다음: 네트워크 성능 측정 ➡️](./04-network-performance-measurement.md)**

</div>
