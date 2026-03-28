# TCP 소켓 상태 머신 — LISTEN에서 TIME_WAIT까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TCP 소켓이 거치는 모든 상태와 각 상태 전환 조건은 무엇인가?
- SYN_RCVD와 ESTABLISHED의 차이는 무엇인가?
- FIN_WAIT_1, FIN_WAIT_2, TIME_WAIT는 각각 어떤 상황인가?
- CLOSE_WAIT가 대량 누적되면 서버에 어떤 장애가 발생하는가?
- `ss -tanp`로 각 상태의 소켓을 어떻게 실시간 관찰하는가?
- Spring Boot 애플리케이션에서 CLOSE_WAIT가 누적되는 정확한 원인과 코드 레벨 해결책은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
장애 현장에서 가장 먼저 실행해야 하는 명령어:
  ss -tanp | sort | uniq -c -f 4 | sort -rn | head

  출력 예시:
  3000 CLOSE_WAIT   0   0  10.0.0.1:8080   10.0.0.2:54321
    50 ESTABLISHED  0   0  10.0.0.1:8080   ...
     1 LISTEN       0 100  *:8080          ...

  → CLOSE_WAIT 3000개: 서버 장애 시작점
  → 파일 디스크립터 고갈 → "Too many open files" 에러
  → 새 연결 수락 불가 → 완전 장애

상태를 모르면 장애 원인을 설명할 수 없음:
  "왜 서버가 응답을 안 하죠?" 
  → CLOSE_WAIT 3000개 → 소켓 누수 → FD 고갈
  → 코드의 어느 경로에서 close()를 안 하는지 추적

  "왜 재시작하면 잠깐 되다가 다시 멈추죠?"
  → 재시작 → CLOSE_WAIT 소멸 → 다시 누수 시작 → 반복
  → 코드 수정 없이는 해결 불가
```

---

## 😱 흔한 실수

```
Before — 상태 머신을 모를 때:

실수 1: CLOSE_WAIT를 서버 재시작으로 해결
  CLOSE_WAIT 3000개 → 서버 재시작 → 0개 → 안도
  수 시간 후 다시 3000개 → 다시 재시작
  → 코드 버그 미수정 → 무한 반복

실수 2: TIME_WAIT와 CLOSE_WAIT 혼동
  "TIME_WAIT가 많은데 줄여야 하지 않나요?"
  → TIME_WAIT는 정상 (Active Close 측의 안전 대기)
  → CLOSE_WAIT가 문제 (코드 버그)
  → 혼동하면 엉뚱한 조치 (tcp_fin_timeout 줄이기 등)

실수 3: FIN_WAIT_2 누적을 무시
  연결 후 서버가 FIN을 보냈지만 클라이언트가 FIN을 안 보냄
  → 서버: FIN_WAIT_2 상태로 대기
  → 기본 타임아웃(60초) 동안 소켓 점유
  → 대량 누적 시 FD 고갈 가능
  → tcp_fin_timeout 조정 또는 코드 수정 필요
```

---

## ✨ 올바른 접근

```
After — 상태 머신을 알면:

CLOSE_WAIT 발견 즉시 대응:
  # 1단계: 규모 파악
  ss -tan state close-wait | wc -l

  # 2단계: 어느 프로세스인지
  ss -tanp state close-wait | grep -oP 'pid=\d+' | sort | uniq -c

  # 3단계: 해당 프로세스의 FD 사용량
  PID=$(pgrep -f spring-app)
  ls /proc/$PID/fd | wc -l
  cat /proc/sys/fs/file-max  # 시스템 최대 FD

  # 4단계: 스택 트레이스로 소켓을 잡고 있는 스레드 확인
  jstack $PID | grep -A 20 "BLOCKED\|WAITING" | head -60

Spring Boot CLOSE_WAIT 체크리스트:
  □ RestTemplate: Bean으로 등록하고 Connection Pool 설정
  □ HttpClient: try-with-resources 또는 finally close()
  □ WebClient: 구독 취소 없이 종료하는 경로 확인
  □ DB 연결: HikariCP 타임아웃 설정 확인
  □ 외부 API 응답 타임아웃: ReadTimeout 설정 여부
```

---

## 🔬 내부 동작 원리

### 1. 전체 상태 다이어그램

```
TCP 소켓 전체 상태 전환:

                     ┌─────────────────────────────────────────────┐
                     │              CLOSED                         │
                     │  소켓 생성 직후 또는 완전 종료 후                  │
                     └─────────────────────────────────────────────┘
                            │                     │
                    [서버: listen()]        [클라이언트: connect()]
                            │                     │
                            ▼                     ▼
                     ┌────────────┐         ┌──────────┐
                     │  LISTEN    │         │          │
                     │  SYN 대기   │         │ SYN 전송  │
                     └────────────┘         └──────────┘
                            │                     │
                     [SYN 수신]              [SYN-ACK 수신]
                     [SYN-ACK 전송]          [ACK 전송]
                            │                     │
                            ▼                     │
                     ┌─────────────┐              │
                     │  SYN_RCVD   │              │
                     │  ACK 대기    │              │
                     └─────────────┘              │
                            │                     │
                     [ACK 수신]                    │
                            │                     │
                            └──────────┬──────────┘
                                       │
                                       ▼
                               ┌──────────────┐
                               │  ESTABLISHED │ ◄── 데이터 전송 단계
                               │  정상 통신 중   │
                               └──────────────┘
                              /                \
                  [내가 먼저 FIN]          [상대방 FIN 수신]
                  [Active Close]           [Passive Close]
                        /                         \
                       ▼                           ▼
              ┌─────────────────┐        ┌──────────────────┐
              │   FIN_WAIT_1    │        │   CLOSE_WAIT     │
              │   ACK 대기 중     │        │   내 FIN 전송 전   │
              └─────────────────┘        └──────────────────┘
                       │                           │
              [ACK 수신]                  [close() 호출]
                       │                   [FIN 전송]
                       ▼                           │
              ┌─────────────────┐                  ▼
              │   FIN_WAIT_2    │        ┌──────────────────┐
              │   상대 FIN 대기   │        │    LAST_ACK      │
              └─────────────────┘        │    ACK 대기       │
                       │                 └──────────────────┘
              [상대 FIN 수신]                        │
              [ACK 전송]                   [ACK 수신]
                       │                           │
                       ▼                           ▼
              ┌─────────────────┐           ┌──────────┐
              │    TIME_WAIT    │           │  CLOSED  │
              │  2 MSL 대기      │           │ 소켓 소멸  │
              └─────────────────┘           └──────────┘
                       │
              [2 MSL 타이머 만료]
                       │
                       ▼
                   ┌──────────┐
                   │  CLOSED  │
                   └──────────┘
```

### 2. 서버 측 상태 전환 상세

```
서버 소켓 생명주기:

1. CLOSED → LISTEN
   socket() + bind() + listen()
   
   int serverFd = socket(AF_INET, SOCK_STREAM, 0);
   bind(serverFd, &addr, sizeof(addr));
   listen(serverFd, backlog);  ← 이 시점에 LISTEN 상태
   
   ss -tlnp | grep 8080
   → LISTEN 0 100 *:8080 *:*

2. LISTEN → SYN_RCVD
   클라이언트 SYN 수신
   OS가 자동으로 SYN-ACK 전송
   Half-Open Queue에 추가
   
   ss -tanp | grep SYN_RCVD
   → SYN_RCVD 0 0 IP:8080 CLIENT:PORT

3. SYN_RCVD → ESTABLISHED
   클라이언트 ACK 수신
   Half-Open Queue → Accept Queue 이동
   애플리케이션의 accept() 호출 대기
   
   Tomcat/Netty가 accept() 호출 → 소켓 전달

4. ESTABLISHED → (종료 단계)
   클라이언트 먼저 FIN:
     서버: ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
   서버 먼저 FIN:
     서버: ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED

각 상태의 소켓 확인:
  ss -tan | grep -E "LISTEN|SYN_RCVD|ESTABLISHED|CLOSE_WAIT|TIME_WAIT"
```

### 3. CLOSE_WAIT 상세 — 버그의 신호

```
CLOSE_WAIT 발생 시나리오:

  Client (외부 or nginx)      Server (Spring Boot)
         │                           │
         │  GET /api/data            │
         │ ─────────────────────────►│  ESTABLISHED
         │                           │
         │  FIN (연결 종료 요청)        │
         │ ─────────────────────────►│
         │                           │  → OS: ACK 전송 자동
         │  ACK                      │  → 상태: CLOSE_WAIT
         │ ◄─────────────────────────│
         │                           │
         │  [서버 애플리케이션 내부]       │
         │                           │  응답 처리 중...
         │                           │  close() 호출을 안 함!
         │                           │  → FIN을 보내지 않음
         │                           │  → CLOSE_WAIT에서 영원히 대기

CLOSE_WAIT가 "영원히" 지속되는 이유:
  클라이언트가 FIN을 보내고 종료했음
  서버 OS는 ACK를 자동으로 보냄
  하지만 서버 애플리케이션이 close() 호출을 안 하면:
    → FIN 전송 없음
    → CLOSE_WAIT 유지
    → 소켓이 닫히지 않음 (FD 누수)

파일 디스크립터 고갈 과정:
  CLOSE_WAIT 1개 = FD 1개 점유
  
  계속 누적:
  CLOSE_WAIT 100개   → FD 100개 낭비
  CLOSE_WAIT 1000개  → FD 1000개 낭비
  CLOSE_WAIT 65535개 → FD 전체 고갈

  "Too many open files" (EMFILE):
    새 연결 accept() 실패
    새 파일 open() 실패
    → 서버 완전 불능

기본 FD 한계 확인:
  ulimit -n                    # 프로세스당 FD 한계 (기본 1024)
  cat /proc/sys/fs/file-max    # 시스템 전체 FD 한계
  ls /proc/<PID>/fd | wc -l    # 특정 프로세스 현재 FD 사용량
```

### 4. Spring Boot에서 CLOSE_WAIT 발생 패턴

```
패턴 1: RestTemplate 잘못된 사용

// ❌ 잘못된 패턴 1: 매번 새 RestTemplate 생성
@Service
public class ApiService {
    public String call() {
        RestTemplate rt = new RestTemplate();  // 커넥션 풀 없음
        // 응답 수신 후 클라이언트가 FIN을 보냄
        // RestTemplate이 제때 close()를 안 하면 CLOSE_WAIT
        return rt.getForObject(url, String.class);
    }
}

// ❌ 잘못된 패턴 2: HttpURLConnection 직접 사용
URL url = new URL("http://api.example.com/data");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
try {
    // 데이터 읽기
    String response = readResponse(conn);
    return response;
} catch (Exception e) {
    throw e;  // conn.disconnect() 호출 없이 예외 전파
              // → 원격이 FIN 보내면 CLOSE_WAIT
}

// ✅ 올바른 패턴: try-with-resources + 명시적 close
URL url = new URL("http://api.example.com/data");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
try {
    String response = readResponse(conn);
    return response;
} finally {
    conn.disconnect();  // 항상 호출
}

───────────────────────────────────────────────────────────────────

패턴 2: Apache HttpClient 응답 미처리

// ❌ 잘못된 패턴: response close 누락
CloseableHttpClient client = HttpClients.createDefault();
CloseableHttpResponse response = client.execute(request);
String body = EntityUtils.toString(response.getEntity());
// response.close() 누락!
// 커넥션이 풀로 반환 안 됨
// 클라이언트(서버 관점에서)가 FIN 보내면 CLOSE_WAIT

// ✅ 올바른 패턴: try-with-resources
try (CloseableHttpResponse response = client.execute(request)) {
    return EntityUtils.toString(response.getEntity());
}  // 자동으로 response.close() → 커넥션 반환 또는 종료

───────────────────────────────────────────────────────────────────

패턴 3: WebClient 구독 취소

// ❌ 문제가 될 수 있는 패턴
webClient.get()
    .uri("/api/data")
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(5))  // timeout 시 구독 취소
    // 구독 취소 시 연결이 깔끔하게 종료되지 않으면 CLOSE_WAIT

// ✅ 명시적 연결 관리
ConnectionProvider provider = ConnectionProvider.builder("custom")
    .maxConnections(500)
    .maxIdleTime(Duration.ofSeconds(20))
    .maxLifeTime(Duration.ofSeconds(60))
    .pendingAcquireTimeout(Duration.ofSeconds(10))
    .evictInBackground(Duration.ofSeconds(120))
    .build();

───────────────────────────────────────────────────────────────────

패턴 4: Nginx keepalive 타임아웃 < Tomcat keepalive

# Nginx 설정
keepalive_timeout 10s;  ← 10초 후 FIN

# Tomcat 설정 (application.properties)
server.tomcat.keep-alive-timeout=20000  ← 20초

→ Nginx가 먼저 FIN을 보냄
→ Tomcat: CLOSE_WAIT
→ Tomcat 애플리케이션이 keepalive 연결을 재사용하려 하면 이미 종료
→ CLOSE_WAIT 대량 발생

해결: Nginx keepalive_timeout > Tomcat keepalive 불가
      반대로 Tomcat을 더 짧게 설정해야 함
      server.tomcat.keep-alive-timeout=8000 (Nginx의 10s보다 짧게)
```

### 5. FIN_WAIT_2와 LAST_ACK

```
FIN_WAIT_2:
  내가 FIN을 보내고 ACK는 받았지만
  상대방의 FIN을 아직 못 받은 상태

  위험: 상대방이 FIN을 보내지 않으면 영원히 대기
  
  Linux 보호: tcp_fin_timeout (기본 60초)
    60초 동안 FIN 없으면 소켓 강제 종료 (CLOSED)
    → 좀비 소켓 방지

FIN_WAIT_2가 대량 발생하는 시나리오:
  서버가 먼저 연결 종료 (HTTP/1.0 기본)
  클라이언트가 FIN을 늦게 보내는 경우:
    클라이언트가 느린 네트워크에서 접속
    클라이언트 애플리케이션이 버그로 FIN 안 보냄
  → 60초 동안 FD 점유
  → 대량 접속 환경에서 FD 고갈 가능

진단:
  ss -tan state fin-wait-2 | wc -l
  # 정상: 수십 개 이하
  # 비정상: 수백~수천 개

───────────────────────────────────────────────────────────────────

LAST_ACK:
  내가 FIN을 보내고 상대방의 ACK를 기다리는 상태
  (Passive Close 측의 마지막 단계)

  일반적으로 매우 짧음 (1 RTT 이내에 종료)
  LAST_ACK가 오래 유지되면:
    ACK가 손실됨 (드문 경우)
    상대방이 응답하지 않는 경우
  → 서버가 FIN 재전송 후 타임아웃으로 CLOSED
```

### 6. ss로 전체 상태 실시간 관찰

```
ss 명령어 완전 가이드:

# 모든 상태의 TCP 소켓 (숫자, 프로세스 정보 포함)
ss -tanp

# 특정 상태만 필터링
ss -tan state established
ss -tan state close-wait
ss -tan state time-wait
ss -tan state listen

# 특정 포트 필터링
ss -tanp 'sport = :8080'   # Source Port 8080
ss -tanp 'dport = :5432'   # Destination Port 5432 (PostgreSQL)

# 통계 요약 (가장 빠른 진단)
ss -s
# 출력:
# Total: 1580
# TCP:   1200 (estab 50, closed 0, orphaned 0, timewait 800)
# → timewait:800 = 정상
# → 이 줄에 close-wait가 없으면 별도 확인

# CLOSE_WAIT 개수
ss -tan state close-wait | wc -l

# 상태별 카운트 한 번에
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
# 출력:
#   800 TIME-WAIT
#    50 ESTABLISHED
#    20 CLOSE-WAIT      ← 있으면 코드 확인
#     1 LISTEN

# 특정 프로세스의 소켓만
ss -tanp | grep "pid=<PID>"

# 자세한 연결 정보 (RTT, cwnd 등)
ss -ti state established 'dport = :80'
```

---

## 💻 실전 실험

### 실험 1: 각 상태 직접 관찰

```bash
# 터미널 1: 서버 (Python)
python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 9090))
s.listen(5)
print('[서버] LISTEN 상태')
conn, addr = s.accept()
print('[서버] ESTABLISHED')
data = conn.recv(1024)
print(f'[서버] 수신: {data}')
conn.send(b'HTTP/1.0 200 OK\r\n\r\nHello')
print('[서버] 응답 전송 완료 - FIN 대기 중')
time.sleep(30)  # CLOSE_WAIT 관찰을 위해 close() 지연
conn.close()
"

# 터미널 2: 상태 모니터링
watch -n 0.3 "ss -tan 'sport = :9090 or dport = :9090'"

# 터미널 3: 클라이언트
python3 -c "
import socket, time
s = socket.socket()
s.connect(('127.0.0.1', 9090))
print('[클라이언트] ESTABLISHED')
s.send(b'GET / HTTP/1.0\r\n\r\n')
data = s.recv(1024)
print(f'[클라이언트] 수신: {data}')
s.close()  # FIN 전송 → 서버는 CLOSE_WAIT
print('[클라이언트] 연결 종료')
time.sleep(20)
"

# 관찰 순서:
# 1. 서버 시작 → LISTEN
# 2. 클라이언트 연결 → ESTABLISHED (양쪽)
# 3. 클라이언트 close() → 서버: CLOSE_WAIT, 클라이언트: TIME_WAIT
# 4. 서버가 30초 후 close() → 서버: LAST_ACK → CLOSED
```

### 실험 2: CLOSE_WAIT 재현 및 FD 고갈 시뮬레이션

```bash
# Spring Boot CLOSE_WAIT 시뮬레이션
# (RestTemplate without connection pool)

# 1. 현재 FD 사용량 확인
PID=$(pgrep -f spring-boot)
echo "현재 FD 수: $(ls /proc/$PID/fd | wc -l)"
echo "FD 한계: $(cat /proc/$PID/limits | grep 'open files' | awk '{print $4}')"

# 2. CLOSE_WAIT 현재 수
ss -tan state close-wait | grep ":8080" | wc -l

# 3. 부하 시 모니터링 (5초 간격)
while true; do
  CWAIT=$(ss -tan state close-wait | grep ":8080" | wc -l)
  FD=$(ls /proc/$PID/fd 2>/dev/null | wc -l)
  echo "$(date +%H:%M:%S) CLOSE_WAIT=$CWAIT FD=$FD"
  sleep 5
done &

# 4. 부하 발생
ab -n 10000 -c 100 http://localhost:8080/api/endpoint

# 5. CLOSE_WAIT가 누적되면 FD도 증가함을 확인
```

### 실험 3: ss -s로 빠른 진단

```bash
# 시스템 전체 소켓 상태 요약
ss -s
# 출력:
# Total: 2050 (kernel 2080)
# TCP:   1800 (estab 100, closed 50, orphaned 0, timewait 1500)
#
# Transport  Total  IP   IPv6
# RAW        0      0    0
# UDP        10     10   0
# TCP        1800   1800 0
# INET       1810   1810 0
# FRAG       0      0    0

# 주목할 수치:
# timewait: 정상 (Connection Pool 없으면 높을 수 있음)
# closed: 소켓은 할당됐지만 아직 미사용 (정상)
# orphaned: 프로세스가 없는 고아 소켓 (비정상이면 문제)

# 상태별 전체 카운트
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# 특정 포트로 CLOSE_WAIT 추적
watch -n 2 "ss -tan state close-wait | grep ':8080' | wc -l"
```

### 실험 4: jstack으로 CLOSE_WAIT 유발 스레드 찾기

```bash
# Spring Boot가 실행 중일 때

# 1. PID 확인
PID=$(pgrep -f spring-boot)

# 2. CLOSE_WAIT 소켓의 원격 주소 확인
ss -tanp state close-wait | grep "pid=$PID" | head -5

# 3. 스레드 덤프 (BLOCKED/WAITING 스레드 확인)
jstack $PID > /tmp/thread-dump.txt

# 4. 어느 스레드가 HttpConnection 관련 작업 중인지
grep -A 30 "HttpURLConnection\|CloseableHttpClient\|RestTemplate" /tmp/thread-dump.txt

# 5. Actuator로 활성 스레드 확인
curl http://localhost:8080/actuator/metrics/jvm.threads.live
curl http://localhost:8080/actuator/metrics/jvm.threads.states

# 6. 연결 풀 상태 (HikariCP)
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
```

---

## 📊 성능/비용 비교

```
소켓 상태별 리소스 비용:

┌────────────────────────────────────────────────────────────────────┐
│  상태           │  메모리      │  FD 점유 │  포트 점유  │  지속 시간       │
├────────────────────────────────────────────────────────────────────┤
│  LISTEN        │  ~500 bytes│  예      │  고정포트   │  영구           │
│  SYN_RCVD      │  ~280 bytes│  예      │  예       │  수십 ms        │
│  ESTABLISHED   │  ~1KB+버퍼  │  예      │  예       │  사용 중         │
│  CLOSE_WAIT    │  ~280 bytes│  예 (누수)│  예 (누수) │  무한(버그)       │
│  TIME_WAIT     │  ~280 bytes│  아니오   │  예       │  60~120초       │
│  FIN_WAIT_2    │  ~280 bytes│  예      │  예       │  60초(제한)      │
│  LAST_ACK      │  ~280 bytes│  예      │  예       │  수 RTT         │
└────────────────────────────────────────────────────────────────────┘

CLOSE_WAIT 1000개의 비용:
  메모리: 280 bytes × 1000 = 280 KB (미미)
  FD:     1000개 점유 (ulimit -n 기본 1024 → 거의 고갈!)
  포트:   1000개 점유

CLOSE_WAIT의 진짜 위협:
  FD 고갈 → accept() 실패 → 새 연결 불가
  FD 고갈 → 파일 open() 실패 → 로그 기록 불가
  FD 고갈 → DB 연결 불가 → 완전 장애

ulimit 확인 및 조정:
  # 현재 FD 한계
  ulimit -n
  # Spring Boot 서비스 권장: 최소 65536
  
  # /etc/security/limits.conf
  * soft nofile 65536
  * hard nofile 65536
  
  # systemd 서비스
  [Service]
  LimitNOFILE=65536
```

---

## ⚖️ 트레이드오프

```
소켓 상태 관련 설정 트레이드오프:

tcp_fin_timeout (FIN_WAIT_2 타임아웃):
  줄이면: FIN_WAIT_2 소켓이 빨리 해제됨
          비정상 클라이언트로 인한 FD 고갈 방지
  늘리면: 느린 클라이언트에게 FIN 보낼 시간 더 줌
  권장: 기본 60초 유지, 고부하 서버에서 15~30초로 감소

tcp_max_orphans (고아 소켓 최대 수):
  고아 소켓: 프로세스는 없지만 OS가 관리 중인 소켓
  기본: 8192
  줄이면: 고아 소켓 빠르게 제거 (RST 전송)
  늘리면: 더 많은 소켓을 유지 (메모리 증가)

Accept Queue (listen backlog):
  줄이면: 연결 수락 대기열 짧음 → 고부하 시 연결 거부
  늘리면: 더 많은 연결 대기 가능 → 메모리 증가
  
  Spring Boot:
  server.tomcat.accept-count=100  (기본)
  고부하 서버: 200~500

Keep-Alive Timeout:
  짧게: 연결 빠르게 종료 → ESTABLISHED 소켓 수 감소
        단, 자주 새 연결 = Handshake 비용 증가
  길게: 연결 재사용 증가 → Handshake 감소
        단, ESTABLISHED 소켓 많아짐 (메모리)
  
  Spring Boot: server.tomcat.keep-alive-timeout=20000 (20초 기본)
  Nginx: keepalive_timeout 65s (기본)
```

---

## 📌 핵심 정리

```
TCP 소켓 상태 머신 핵심 요약:

11개 상태:
  CLOSED, LISTEN, SYN_SENT, SYN_RCVD
  ESTABLISHED
  FIN_WAIT_1, FIN_WAIT_2, TIME_WAIT  ← Active Close 측
  CLOSE_WAIT, LAST_ACK               ← Passive Close 측
  CLOSING (동시 종료, 드문 경우)

서버 관점 정상 상태:
  LISTEN:       포트 오픈 중 (항상 있어야 함)
  ESTABLISHED:  활성 연결
  TIME_WAIT:    정상 종료 후 대기 (Active Close 측)
  CLOSE_WAIT:   비정상! 코드 버그

CLOSE_WAIT가 위험한 이유:
  FD를 무한정 점유 (close() 호출 없는 한 영원)
  FD 고갈 → "Too many open files" → 서버 불능

CLOSE_WAIT 원인:
  close() / disconnect() 미호출
  예외 발생 시 finally에서 close() 누락
  try-with-resources 미사용

CLOSE_WAIT 해결:
  try-with-resources로 모든 Closeable 관리
  Connection Pool 사용 (풀이 연결 생명주기 관리)
  Nginx ↔ Tomcat keepalive 타임아웃 정합성

진단 명령어:
  ss -s                         → 전체 요약
  ss -tan | awk ... | sort -rn  → 상태별 카운트
  ss -tan state close-wait      → CLOSE_WAIT 목록
  ls /proc/<PID>/fd | wc -l     → FD 사용량
  jstack <PID>                  → 스레드 덤프
```

---

## 🤔 생각해볼 문제

**Q1.** `ss -s` 출력에서 `orphaned: 150`을 발견했다. 고아 소켓이란 무엇이고 어떤 상황에서 발생하는가?

<details>
<summary>해설 보기</summary>

**고아 소켓(Orphaned Socket):** 연결된 프로세스가 없는 소켓입니다. 프로세스는 종료됐지만 OS가 아직 TCP 종료 절차를 완료하지 못한 소켓입니다.

**발생 상황:**

1. **프로세스가 close() 없이 종료됨**: OS가 종료되는 프로세스의 소켓을 대신 처리하지만, 아직 FIN 전송 중 또는 TIME_WAIT 대기 중인 경우

2. **SO_LINGER(0)으로 RST 종료**: RST는 즉시 전송되지만 OS는 잠깐 소켓 상태를 유지

3. **애플리케이션 크래시**: JVM 크래시, SIGKILL 등으로 강제 종료 시 OS가 남은 소켓 정리

**150개가 의미하는 것:**
- 최근에 프로세스 크래시나 강제 종료가 있었을 가능성
- 정상적이라면 수 초 내에 감소해야 함
- 150개가 지속된다면 소켓이 제대로 종료되지 않고 있음

**진단:**
```bash
# 고아 소켓 상태 지속 모니터링
watch -n 1 "ss -s | grep orphaned"

# 고아 소켓이 있는 FIN_WAIT_1 또는 CLOSE_WAIT 확인
ss -tan | grep -v ESTABLISHED | grep -v LISTEN

# 시스템 최대 고아 소켓
sysctl net.ipv4.tcp_max_orphans  # 기본 8192
# 초과 시 RST로 강제 종료 + "Out of socket memory" 경고
```

**해결:** 고아 소켓은 일반적으로 OS가 자동 정리합니다. 지속적으로 많다면 애플리케이션의 비정상 종료 원인을 찾아야 합니다.

</details>

---

**Q2.** Spring Boot 서버에서 특정 시간대(예: 새벽 2시~4시)에만 CLOSE_WAIT가 누적됐다가 자연 소멸된다. 어떤 상황을 의심해야 하는가?

<details>
<summary>해설 보기</summary>

**의심해야 할 상황들:**

**1. 배치 작업과의 연관성:**
- 새벽 2시에 스케줄된 배치 작업이 대용량 DB 쿼리 실행
- 쿼리 완료 후 DB 연결이 제대로 반환되지 않으면 CLOSE_WAIT
- DB 서버가 타임아웃으로 연결을 끊음 → Spring이 CLOSE_WAIT 상태로 남김

```bash
# 스케줄 확인
crontab -l
cat /etc/cron.d/*
```

**2. Nginx/로드밸런서의 keepalive_timeout:**
- Nginx가 주기적으로 유휴 연결을 정리하는 시간
- 새벽에 트래픽이 낮아지면 Nginx가 idle 연결에 FIN 전송
- Tomcat이 CLOSE_WAIT → 배치가 끝나면 자연 소멸

**3. DB의 wait_timeout (MySQL 기본 8시간):**
- 유휴 DB 연결이 8시간 후 서버에서 끊김
- 새벽 2시 = 어젯밤 6시에 마지막으로 사용된 연결의 wait_timeout 도래
- HikariCP가 `testOnBorrow` 또는 `keepaliveTime` 설정 없으면 stale 연결 문제

**HikariCP 설정 확인:**
```yaml
spring:
  datasource:
    hikari:
      connection-test-query: SELECT 1
      keepalive-time: 300000   # 5분마다 keepalive
      max-lifetime: 1800000    # 30분 후 연결 교체
      connection-timeout: 30000
```

**4. 진단:**
```bash
# 특정 시간대의 로그 확인
grep "02:0[0-9]\|03:" /var/log/app.log | grep -i "connection\|timeout\|close"

# DB wait_timeout 확인
mysql -e "SHOW VARIABLES LIKE 'wait_timeout'"
```

</details>

---

**Q3.** TCP 상태 머신에서 CLOSING 상태는 언제 발생하는가? 일반적으로 왜 거의 보기 어려운가?

<details>
<summary>해설 보기</summary>

**CLOSING 상태 발생 조건: 동시 종료 (Simultaneous Close)**

두 쪽이 거의 동시에 FIN을 전송할 때 발생합니다.

```
Client                    Server
  │  FIN → → → → → → → → →│
  │← ← ← ← ← ← ← FIN ←  │
  │  (두 FIN이 교차!)      │
  
Client 관점:
  FIN 전송 → FIN_WAIT_1
  FIN_WAIT_1에서 ACK 아닌 FIN 수신 → CLOSING
  자신의 FIN에 대한 ACK 수신 → TIME_WAIT
  
Server도 동일한 CLOSING 거침
```

**CLOSING이 거의 안 보이는 이유:**

1. **타이밍의 문제**: 두 쪽이 나노초 단위로 동시에 FIN을 보내야 함. 실제로 이 타이밍이 맞을 확률은 매우 낮음.

2. **TCP 구현의 경향**: 대부분의 HTTP 통신에서 서버나 클라이언트 중 한 쪽이 명확하게 먼저 종료 결정을 내림. 동시 종료는 설계상 드문 패턴.

3. **Delayed ACK**: ACK를 40ms 지연해서 보내는 Delayed ACK로 인해, FIN을 받으면 ACK를 먼저 보내는 경향이 있어 "동시" 타이밍이 더 어려움.

**실무에서의 의미:** `ss -tan | grep CLOSING`을 보면 거의 0개일 것입니다. 만약 CLOSING이 지속적으로 관찰된다면 비정상적인 동시 종료 패턴이 발생 중이라는 신호로 볼 수 있으나, 실질적인 문제를 일으키는 경우는 드뭅니다.

</details>

---

<div align="center">

**[⬅️ 이전: TCP vs UDP](./06-tcp-vs-udp.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — HTTP 완전 분해 ➡️](../http-internals/01-http1-internals.md)**

</div>
