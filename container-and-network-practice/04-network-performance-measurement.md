# 네트워크 성능 측정 — RTT, 처리량, 지연시간 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ping으로 RTT와 Jitter를 어떻게 측정하고 해석하는가?
- iperf3로 TCP/UDP 처리량을 어떻게 정확하게 측정하는가?
- Wireshark TCP 분석 그래프에서 혼잡 제어 동작을 어떻게 읽는가?
- `ss --info`의 cwnd, rtt, rto 값은 무엇을 의미하는가?
- 지연이 "네트워크 문제"인지 "서버 처리 문제"인지 어떻게 구분하는가?
- 네트워크 성능 병목을 체계적으로 찾는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"API 응답이 느린데 서버 CPU는 정상입니다":
  CPU 정상 → 서버 처리는 빠름
  하지만 네트워크에서 지연 발생 가능
  
  측정으로 구분:
  ping target: RTT=50ms (순수 네트워크 지연)
  curl -w "%{time_connect}": 50ms (TCP 연결, RTT와 유사)
  curl -w "%{time_appconnect}": 100ms (TLS 추가)
  curl -w "%{time_total}": 250ms (서버 처리 포함)
  
  250ms - 100ms = 150ms → 서버 처리 시간
  → 서버 최적화 필요

"처리량이 이론값보다 낮습니다":
  이론: 1Gbps 링크
  실측: 500Mbps
  
  iperf3 측정:
  TCP: 950Mbps → 링크는 정상 (네트워크 계층 OK)
  HTTP(Nginx): 500Mbps → Nginx 또는 앱 병목
  → Nginx 설정 (sendfile, 버퍼) 또는 앱 처리 최적화
```

---

## 😱 흔한 실수

```
Before — 네트워크 성능 측정을 모를 때:

실수 1: ping 1번으로 RTT 판단
  ping -c 1 target → 49ms
  → 이 하나의 값으로 "네트워크 지연 49ms"로 결론
  → 실제: 40ms~150ms 사이에서 변동 (Jitter)
  → ping -c 100 → 평균, 최소, 최대, 표준편차로 판단

실수 2: iperf3 결과를 앱 처리량으로 혼동
  iperf3: 5.2 Gbps → 링크 OK
  앱 응답: 초당 100MB → 차이 발생!
  → iperf3: 순수 네트워크 처리량
  → 앱: 직렬화, 암호화, 처리 로직 포함
  → 각각 의미 다름, 혼용 금지

실수 3: Wireshark 없이 패킷 레벨 문제 진단
  "패킷이 왜 이렇게 늦나요?"
  → 추측만 함
  → tcpdump로 캡처 → Wireshark 분석 → 정확한 원인 확인
  
실수 4: 측정 환경 오염
  iperf3 서버와 클라이언트를 같은 서버에서 실행 (localhost)
  → 실제 네트워크를 거치지 않음 → 수십 Gbps (의미 없는 값)
  → 반드시 다른 호스트에서 측정
```

---

## ✨ 올바른 접근

```
After — 네트워크 성능 측정을 알면:

체계적 성능 측정 순서:
  1. RTT 측정 (ping) → 기본 네트워크 지연 파악
  2. 처리량 측정 (iperf3) → 링크 용량 확인
  3. HTTP 지연 분해 (curl -w) → 각 단계 시간 측정
  4. 소켓 상태 확인 (ss --info) → TCP 혼잡 상태
  5. 패킷 캡처 (tcpdump + Wireshark) → 세밀한 분석

curl로 HTTP 지연 단계별 측정:
  curl -w "
  namelookup: %{time_namelookup}s
  connect:    %{time_connect}s
  appconnect: %{time_appconnect}s
  pretransfer: %{time_pretransfer}s
  starttransfer: %{time_starttransfer}s
  total:      %{time_total}s
  " -o /dev/null -s https://api.example.com
  
  namelookup:   DNS 조회
  connect:      TCP 연결 (= RTT × 1.5)
  appconnect:   TLS 핸드쉐이크 완료
  pretransfer:  요청 전송 준비
  starttransfer: 첫 바이트 수신 (= TTFB)
  total:        전체 완료
  
  분석:
  connect - namelookup ≈ 1.5 RTT
  appconnect - connect ≈ TLS RTT
  starttransfer - appconnect ≈ 서버 처리 시간
  total - starttransfer ≈ 응답 다운로드 시간

APM 연동:
  Prometheus + Grafana: 지연시간 분포 (P50, P95, P99)
  Grafana histogram_quantile: 백분위 레이턴시
  OpenTelemetry: 분산 추적으로 각 서비스 기여도 측정
```

---

## 🔬 내부 동작 원리

### 1. RTT와 Jitter — ping 분석

```
RTT (Round-Trip Time):
  패킷이 목적지까지 갔다가 돌아오는 시간
  네트워크 지연의 기본 단위
  
  모든 TCP 연결 비용은 RTT로 표현:
  TCP Handshake: 1.5 RTT
  TLS 1.2:       +2 RTT
  TLS 1.3:       +1 RTT
  HTTP 요청/응답: +0.5~1 RTT (전파 지연만)

ping으로 RTT 측정:

  ping -c 100 api.example.com
  
  --- api.example.com ping statistics ---
  100 packets transmitted, 100 received, 0% packet loss, time 99074ms
  rtt min/avg/max/mdev = 12.345/15.678/45.234/3.456 ms
  
  각 값 해석:
    min:  12.345ms → 네트워크 최솟값 (혼잡 없을 때)
    avg:  15.678ms → 평균 RTT
    max:  45.234ms → 최대 RTT (혼잡, 재전송 시)
    mdev: 3.456ms  → 표준편차 = Jitter (불안정성 지표)

Jitter (지터):
  RTT의 변동량 = 표준편차
  
  낮은 Jitter (< 1ms): 안정적 네트워크 (LAN, 전용선)
  보통 Jitter (1~10ms): 인터넷 (허용 가능)
  높은 Jitter (> 10ms): 혼잡, 불안정 (VoIP/화상통화 품질 저하)

ping 고급 분석:
  # 패킷 손실률 측정
  ping -c 1000 target | tail -3
  # 1000 packets transmitted, 995 received, 0.5% packet loss
  
  # MTU 탐지 (패킷 크기 변화)
  ping -M do -s 1400 target  # 1400 bytes, 분할 금지
  ping -M do -s 1472 target  # 더 큰 크기 시도
  # 특정 크기에서 실패 → MTU 병목 발견
  
  # 경로별 지연 분석
  traceroute target
  mtr --report target  # 경로 + 각 홉의 RTT/손실률
```

### 2. iperf3 — 처리량 측정

```
iperf3 아키텍처:
  서버: iperf3 -s (리스닝)
  클라이언트: iperf3 -c [서버IP] (측정 시작)

TCP 처리량 측정:

  # 서버 (측정 대상 서버에서)
  iperf3 -s
  
  # 클라이언트 (측정하는 측에서)
  iperf3 -c server-ip -t 30  # 30초간 측정
  
  결과:
  [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
  [  5]   0.00-10.00 sec  1.10 GBytes   941 Mbits/sec    0   1.41 MBytes
  [  5]  10.00-20.00 sec  1.09 GBytes   936 Mbits/sec    2   1.12 MBytes
  [  5]  20.00-30.00 sec  1.11 GBytes   951 Mbits/sec    0   1.41 MBytes
  
  Bitrate: 실제 처리량 (941 Mbps)
  Retr: 재전송 수 (0이 이상적)
  Cwnd: 혼잡 윈도우 (크면 클수록 좋음)

UDP 처리량 및 지터 측정:
  # 10 Mbps UDP 스트림 (VoIP, 스트리밍 시뮬레이션)
  iperf3 -c server-ip -u -b 10M -t 30
  
  결과:
  [ ID] Interval           Transfer     Bitrate         Jitter  Lost/Total
  [  5]   0.00-30.00 sec  35.8 MBytes  10.0 Mbits/sec  0.082ms 0/25658 (0%)
  
  Jitter: 0.082ms → 안정적 (< 1ms)
  Lost: 0/25658 (0%) → 패킷 손실 없음

병렬 스트림 (멀티 커넥션):
  iperf3 -c server-ip -P 10 -t 30  # 10개 병렬 스트림
  
  → 단일 TCP 흐름의 혼잡 윈도우 한계를 여러 연결로 우회
  → 더 높은 처리량 달성 가능
  → HTTP/2 멀티플렉싱 vs HTTP/1.1 병렬 연결과 유사

역방향 측정:
  iperf3 -c server-ip -R  # 서버 → 클라이언트 방향 측정
  → 업로드와 다운로드 비대칭 대역폭 확인

윈도우 크기 조정:
  iperf3 -c server-ip -w 4M  # 수신 버퍼 4MB
  → RTT가 높은 환경(해외 서버)에서 처리량 향상
  → 높은 BDP(Bandwidth-Delay Product)에서 기본 윈도우 부족
  BDP = 대역폭 × RTT = 100Mbps × 0.1s = 10Mbit = 1.25MB
```

### 3. Wireshark TCP 분석 그래프

```
Time-Sequence Graph (Stevens 방식):
  Y축: Sequence Number (전송된 데이터 누적)
  X축: 시간
  
  기울기 = 처리량
  급격한 기울기: 빠른 데이터 전송
  평탄한 구간: 전송 정지 (혼잡, 재전송 대기)
  
  혼잡 제어 패턴:
  /  /  /  /____/  /  /  /
  (Slow Start) (혼잡 감지) (재시작)
  
  재전송 확인:
  Statistics → TCP Stream Graphs → Time Sequence (Stevens)

Throughput Graph:
  Y축: 처리량 (bps)
  X축: 시간
  
  안정적 처리량: 일정한 높이 유지
  버스트: 급격히 높아졌다 낮아짐
  혼잡으로 인한 감소: 계단식 감소

RTT Graph:
  Y축: RTT (ms)
  X축: 시간
  
  일정한 RTT: 안정적 네트워크
  증가하는 RTT: 혼잡 (버퍼 블로트 신호)
  갑자기 높은 RTT: 재전송 (RTT = 첫 전송 + 재전송 시간)

Window Scaling Graph:
  Y축: 수신 윈도우 크기
  
  윈도우가 작음: 수신 측 버퍼 부족 또는 느린 처리
  윈도우가 0 (Zero Window): 수신 버퍼 가득 참 → 전송 멈춤

패킷 분석 필터:
  # 재전송 찾기
  tcp.analysis.retransmission
  
  # 중복 ACK 찾기 (혼잡 신호)
  tcp.analysis.duplicate_ack
  
  # Zero Window 찾기
  tcp.window_size == 0
  
  # 특정 스트림만 보기
  tcp.stream == 5
```

### 4. ss --info — 개별 소켓 TCP 상태

```
ss --info -tn state established | head -20

출력 예:
State   Send-Q  Recv-Q  Local      Peer
ESTAB   0       0       10.0.0.1:443 10.0.0.2:54321
         cubic wscale:7,7 rto:204 rtt:2.5/1.25 ato:40
         mss:1448 pmtu:1500 rcvmss:1448 advmss:1448
         cwnd:10 bytes_sent:1234567 bytes_acked:1234567
         bytes_received:456789 segs_out:1000 segs_in:800
         data_segs_out:900 data_segs_in:700

각 필드 의미:
  cubic:        혼잡 제어 알고리즘 (CUBIC)
  wscale:7,7:   윈도우 스케일 (2^7=128배)
  rto:204:      재전송 타임아웃 ms (현재 204ms)
  rtt:2.5/1.25: 평균RTT/RTT표준편차 (ms)
  ato:40:       ACK 지연 타임아웃 (ms)
  mss:1448:     최대 세그먼트 크기 (bytes)
  pmtu:1500:    경로 MTU (bytes)
  cwnd:10:      혼잡 윈도우 (세그먼트 수)
                cwnd × mss = 현재 비행 중 최대 데이터
                10 × 1448 = 14,480 bytes
  bytes_sent:   전송된 총 바이트
  bytes_acked:  ACK 받은 총 바이트
  (bytes_sent - bytes_acked = 전송 중인 데이터)

cwnd 분석:
  cwnd=1: Slow Start 초기 상태
  cwnd가 증가 중: 정상 Slow Start
  cwnd 감소: 혼잡 감지 (패킷 손실 또는 ECN)
  cwnd 일정: 안정 상태
  
  최적 cwnd:
  BDP = 대역폭 × RTT
  = 1Gbps × 0.01s = 10,000,000 bits = 1.25MB
  최적 cwnd = 1.25MB / 1448 ≈ 863 세그먼트
  
  cwnd=10 (14KB)이면 대역폭 활용률:
  14KB / 1.25MB ≈ 1.1% → 매우 낮음
  → Slow Start 초기 또는 혼잡으로 감소된 상태

rto 분석:
  rto = rtt × 1.5 + (4 × rtt_variance) 정도
  rto가 높으면: 재전송 발생 시 오래 기다림
  rto 초과 → 재전송 → cwnd 감소 → 처리량 급락
```

### 5. 지연 병목 진단 프레임워크

```
단계별 지연 분석:

1. 물리 계층 / 링크:
   측정: ping (ICMP)
   정상: LAN < 1ms, 국내 < 10ms, 해외 < 200ms
   이상: ping loss, RTT 급등 → 링크 문제

2. 네트워크 용량:
   측정: iperf3 (TCP/UDP)
   정상: 링크 이론값의 90% 이상
   이상: 이론값보다 낮음 → 중간 노드 병목, QoS

3. 애플리케이션 연결:
   측정: curl -w (time_connect, time_appconnect)
   time_connect ≈ 1.5 RTT → 정상이면 TCP OK
   time_appconnect >> 1.5 RTT → TLS 문제

4. 서버 처리 시간:
   측정: curl -w (starttransfer - appconnect)
   정상: < 50ms (빠른 API)
   이상: > 500ms → DB 슬로우 쿼리, 스레드 대기

5. 응답 전송 시간:
   측정: curl -w (total - starttransfer)
   대용량 응답: 정상
   소용량 응답에서 느림: 클라이언트 네트워크 문제

6. 개별 소켓:
   측정: ss --info
   cwnd가 작음 → 혼잡 제어 동작
   rtt 높음 → 네트워크 지연
   재전송 → 패킷 손실

7. 패킷 레벨:
   측정: tcpdump + Wireshark
   재전송, Zero Window, 중복 ACK 확인
   → 가장 세밀한 분석

진단 순서 (빠른 것부터):
  1. ping (1초) → RTT, 손실 확인
  2. curl -w (5초) → HTTP 단계별 시간
  3. iperf3 (30초) → 처리량 병목
  4. ss --info → 소켓 TCP 상태
  5. Wireshark → 패킷 레벨 분석
```

---

## 💻 실전 실험

### 실험 1: RTT와 Jitter 측정

```bash
# 기본 RTT 측정 (100 패킷)
ping -c 100 google.com
# 최소/평균/최대/편차(Jitter) 확인

# 패킷 크기별 RTT (MTU 탐지)
for size in 64 512 1024 1400 1472 1500; do
  result=$(ping -c 5 -s $size -M do google.com 2>&1 | grep "avg" | awk -F'/' '{print $5}')
  echo "패킷 크기 $size bytes: RTT avg = ${result:-BLOCKED}ms"
done

# mtr로 경로별 지연 분석
mtr --report --report-cycles 100 google.com
# 각 홉의 최소/평균/최대/Jitter/손실률

# 손실률에 따른 TCP 처리량 영향 추정
# TCP 처리량 ≈ MSS × sqrt(3/2) / (RTT × sqrt(loss_rate))
python3 -c "
import math
rtt = 0.020  # 20ms
mss = 1448
for loss in [0, 0.001, 0.01, 0.05, 0.1]:
    if loss == 0:
        print(f'손실률 0%: 최대 처리량')
    else:
        throughput = mss * math.sqrt(3/2) / (rtt * math.sqrt(loss))
        print(f'손실률 {loss*100:.1f}%: 처리량 ≈ {throughput/1e6:.1f} Mbps')
"
```

### 실험 2: iperf3 처리량 측정

```bash
# 서버 모드 (다른 호스트에서)
iperf3 -s

# 기본 TCP 처리량 측정 (30초)
iperf3 -c server-ip -t 30 -i 5  # 5초마다 중간 결과

# 높은 RTT 환경 시뮬레이션 (tc로 지연 추가)
sudo tc qdisc add dev eth0 root netem delay 100ms
iperf3 -c localhost -t 10
# RTT 100ms에서 기본 TCP 처리량 확인

# 윈도우 크기 증가로 BDP 맞춤
iperf3 -c localhost -t 10 -w 4M  # 4MB 버퍼
# 처리량 향상 확인 (BDP = 100Mbps × 0.2s = 20Mbit = 2.5MB)

sudo tc qdisc del dev eth0 root

# 패킷 손실 시뮬레이션
sudo tc qdisc add dev eth0 root netem loss 1%
iperf3 -c server-ip -t 30
# 1% 손실에서 처리량 감소 확인 (Retr 칼럼)
sudo tc qdisc del dev eth0 root

# UDP 스트림 (VoIP 시뮬레이션)
iperf3 -c server-ip -u -b 10M -t 30
# Jitter, 손실률 확인
```

### 실험 3: curl로 HTTP 지연 분해

```bash
# HTTP/HTTPS 지연 단계별 측정
cat << 'EOF' > measure_http.sh
#!/bin/bash
URL="${1:-https://www.google.com}"
echo "=== HTTP 지연 분해: $URL ==="
curl -w "\nDNS조회:       %{time_namelookup}s
TCP연결:       %{time_connect}s
TLS핸드쉐이크: %{time_appconnect}s
첫바이트(TTFB): %{time_starttransfer}s
전체:          %{time_total}s
응답크기:      %{size_download} bytes\n" \
-o /dev/null -s "$URL"
EOF
chmod +x measure_http.sh

./measure_http.sh https://google.com
./measure_http.sh http://httpbin.org/get

# 여러 번 측정해서 안정성 확인
for i in $(seq 1 5); do
  TTFB=$(curl -w "%{time_starttransfer}" -o /dev/null -s https://api.example.com)
  echo "측정 $i: TTFB=${TTFB}s"
done | awk '{sum+=$3; count++} END {print "평균 TTFB:", sum/count, "s"}'
```

### 실험 4: ss --info로 소켓 상태 분석

```bash
# ESTABLISHED 소켓 TCP 상태 조회
ss --info -tn state established | head -30

# cwnd 크기 분포 확인
ss --info -tn state established | grep cwnd | \
  awk '{for(i=1;i<=NF;i++) if($i~/cwnd/) print substr($i,6)}' | \
  sort -n | uniq -c | head -20

# RTT 높은 연결 찾기 (100ms 이상)
ss --info -tn state established | grep -oP 'rtt:\K[0-9.]+' | \
  awk '{if($1>100) print "고RTT 소켓: "$1"ms"}'

# 재전송 많은 소켓 찾기
ss --info -tn state established | grep "retrans:" | \
  awk '{for(i=1;i<=NF;i++) if($i~/retrans/) print $0}' | head -10

# 실시간 cwnd 모니터링 (혼잡 제어 관찰)
watch -n 0.5 'ss --info -tn state established | grep "dst server-ip" | grep -oP "cwnd:\K[0-9]+"'
```

---

## 📊 성능/비용 비교

```
네트워크 측정 도구 비교:

┌──────────────────────────────────────────────────────────────────────┐
│  도구        │  측정 항목            │  정확도  │  사용 난이도               │
├──────────────────────────────────────────────────────────────────────┤
│  ping       │  RTT, 손실률, Jitter │  높음    │  매우 쉬움                │
│  mtr        │  경로별 RTT/손실      │  높음    │  쉬움                    │
│  iperf3     │  처리량, 지터(UDP)    │  높음    │  중간 (서버 필요)          │
│  curl -w    │  HTTP 단계별 지연     │  높음    │  쉬움                    │
│  ss --info  │  소켓 TCP 상태       │  매우 높음 │  중간                   │
│  Wireshark  │  패킷 레벨 분석       │  완전     │  어려움                  │
│  ab/wrk     │  HTTP 부하 테스트     │  중간     │  쉬움                   │
└──────────────────────────────────────────────────────────────────────┘

RTT별 서비스 영향:
  RTT < 1ms:    LAN, 같은 데이터센터 → 거의 영향 없음
  RTT 1~20ms:   국내 IDC 간 → 미미한 영향
  RTT 20~100ms: 해외 서버 → 체감 가능 (특히 TLS 협상)
  RTT > 100ms:  고지연 → 사용자 경험 저하, 최적화 필요

처리량 한계 계산:
  TCP 단일 연결 이론 최대:
  = window_size / RTT
  = 65,535 bytes / 0.020s = 3.2 MB/s ≈ 26 Mbps (최대 윈도우 없이)
  
  Window Scaling (최대 2^30 bytes):
  = 1GB / 0.020s = 50 GB/s (이론값)
  
  실제는 혼잡 윈도우, 수신 버퍼 한계로 제한됨

패킷 손실과 처리량 관계:
  손실 0%:   이론 최대
  손실 0.1%: 약 10% 처리량 감소
  손실 1%:   약 60% 처리량 감소  
  손실 5%:   약 85% 처리량 감소
  → 손실이 처리량에 기하급수적 영향
```

---

## ⚖️ 트레이드오프

```
측정 정확도 vs 비용:

ping (ICMP):
  정확도: RTT만 측정 (TCP와 다를 수 있음)
  비용: 거의 없음 (ICMP 패킷)
  한계: 방화벽이 ICMP 차단 시 동작 안 함

iperf3:
  정확도: 높음 (실제 데이터 전송)
  비용: 측정 대상 서버에 iperf3 서버 실행 필요
  한계: 실제 앱 트래픽 패턴과 다를 수 있음

Wireshark:
  정확도: 완전 (모든 패킷 분석)
  비용: CPU/디스크 (패킷 캡처), 분석 시간
  한계: 암호화(TLS)된 내용 읽으려면 키 필요

프로덕션에서의 상시 모니터링:
  RTT: Prometheus blackbox_exporter (probe_duration)
  처리량: node_exporter (network_transmit_bytes_total)
  소켓: node_exporter (node_sockstat_*)
  HTTP 지연: APM (Datadog, New Relic, Jaeger)
  → 지속적 측정으로 이상 감지 자동화

네트워크 성능과 비즈니스 영향:
  P50 레이턴시: 일반 사용자 경험
  P95 레이턴시: 느린 사용자 (개선 집중)
  P99 레이턴시: 가장 느린 1% (SLA 위반 위험)
  → 평균만 보지 말고 백분위로 분석

TCO (총소유비용) 관점:
  CDN: RTT를 수십ms에서 수ms로 단축
       비용: CDN 서비스료
       절감: 서버 응답 개선 불필요 → 서버 비용 절감
  
  지역별 서버: 각 지역에 가까운 서버 배치
              비용: 다중 인프라
              절감: RTT 최소화 → 사용자 경험 대폭 향상
```

---

## 📌 핵심 정리

```
네트워크 성능 측정 핵심 요약:

RTT 측정 (ping):
  ping -c 100: 평균, 최대, Jitter(mdev) 확인
  mtr: 경로별 RTT와 손실률
  Jitter > 10ms: 불안정 (VoIP/게임에 영향)
  손실 > 0.1%: TCP 처리량에 심각한 영향

처리량 측정 (iperf3):
  TCP: 순수 링크 용량 측정
  UDP: Jitter, 손실률 측정 (스트리밍 시뮬레이션)
  높은 RTT + 낮은 처리량: 윈도우 크기 늘리기 (-w)
  재전송 있으면: 패킷 손실 또는 혼잡

HTTP 지연 분해 (curl -w):
  time_connect: TCP (≈ 1.5 RTT)
  time_appconnect: TLS 추가 (≈ 1 RTT more for 1.3)
  starttransfer: TTFB (서버 처리 포함)
  total - starttransfer: 응답 다운로드

소켓 TCP 상태 (ss --info):
  cwnd: 혼잡 윈도우 (클수록 많은 데이터 비행 중)
  rtt: 평균 RTT
  rto: 재전송 타임아웃 (rtt 기반 동적 조정)
  retrans: 재전송 수 (0이 이상적)

Wireshark 분석:
  Time-Sequence: 처리량 패턴, 혼잡 감지
  tcp.analysis.retransmission: 재전송 찾기
  tcp.window_size == 0: Zero Window (수신 버퍼 고갈)

진단 순서:
  ping → curl -w → iperf3 → ss → Wireshark
  (간단한 것부터 시작)
```

---

## 🤔 생각해볼 문제

**Q1.** `iperf3`로 측정한 처리량이 이론 대역폭의 60%밖에 안 나온다. 어디서 병목이 발생하고 있을 가능성이 높은가?

<details>
<summary>해설 보기</summary>

**단계적 병목 진단:**

1. **RTT 확인 (BDP 계산):**
   ```bash
   ping server-ip -c 10  # RTT 확인
   # RTT = 50ms, 대역폭 = 1Gbps
   # BDP = 1Gbps × 0.05s = 50Mbit = 6.25MB
   # 기본 TCP 소켓 버퍼가 BDP보다 작으면 처리량 제한
   ```

2. **TCP 소켓 버퍼 확인:**
   ```bash
   cat /proc/sys/net/ipv4/tcp_rmem  # 수신 버퍼
   cat /proc/sys/net/ipv4/tcp_wmem  # 전송 버퍼
   # min default max
   # 4096 87380 6291456  → default 85KB, max 6MB
   
   # BDP = 6.25MB > default 85KB → 자동 확장에 의존
   # sysctl -w net.ipv4.tcp_rmem="4096 4194304 16777216"
   ```

3. **재전송 확인:**
   ```bash
   iperf3 -c server-ip -t 30 | grep Retr
   # Retr > 0 → 패킷 손실 → 혼잡 제어 → 처리량 감소
   ```

4. **CPU 병목 (소프트웨어 IRQ):**
   ```bash
   # iperf3 실행 중
   top → CPU si (소프트웨어 인터럽트) 확인
   # si > 50% → NIC 인터럽트 처리가 병목
   # 해결: RSS/RPS 설정, 멀티큐 NIC
   ```

5. **윈도우 크기 실험:**
   ```bash
   iperf3 -c server-ip -w 4M -t 30  # 큰 버퍼
   # 처리량 향상 → BDP 미스매치가 원인
   ```

6. **병렬 스트림:**
   ```bash
   iperf3 -c server-ip -P 4 -t 30  # 4개 스트림
   # 단일 스트림보다 높으면 단일 TCP 흐름의 한계
   ```

</details>

---

**Q2.** Wireshark에서 `tcp.analysis.retransmission` 필터로 재전송 패킷을 찾았다. 재전송이 처리량에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**재전송의 처리량 영향:**

1. **직접적 영향: 중복 데이터 전송**
   재전송 패킷은 동일 데이터를 다시 보냄 → 실제 전송 데이터 비율 감소.
   재전송 비율 5% → 실제 처리량의 5%가 낭비.

2. **더 큰 영향: 혼잡 제어 반응**
   - **패킷 손실 감지 방법 1 (3 중복 ACK):**
     cwnd를 절반으로 감소 → 처리량 즉시 50% 감소
   - **패킷 손실 감지 방법 2 (RTO 타임아웃):**
     cwnd를 1로 감소 → Slow Start 재시작 → 처리량 급락

3. **RTO의 지수적 증가:**
   재전송 실패 → 다음 RTO = 2배 (200ms → 400ms → 800ms...)
   여러 번 재전송 실패 → 수 초 대기 → 처리량 거의 0

**진단:**
```
Wireshark:
  tcp.analysis.retransmission → 재전송 패킷 찾기
  tcp.analysis.fast_retransmission → 빠른 재전송 (3 중복 ACK)
  tcp.analysis.retransmission AND NOT tcp.analysis.fast_retransmission
    → RTO 타임아웃 재전송 (더 심각)
```

**해결:**
- 패킷 손실 원인: 링크 품질 점검 (이더넷 오류 카운터, 광 파워)
- 네트워크 혼잡: QoS 설정, 대역폭 업그레이드
- 버퍼 블로트: 작은 버퍼로 지연 최소화 (CoDel, FQ-CoDel)

</details>

---

**Q3.** 해외 서버(RTT=150ms)와 통신할 때 단일 TCP 연결의 이론 최대 처리량은 얼마이고, 이를 높이는 방법은?

<details>
<summary>해설 보기</summary>

**이론 최대 처리량 계산:**

```
TCP 처리량 ≈ 혼잡 윈도우(cwnd) × MSS / RTT

패킷 손실 없는 이상적 상황:
cwnd_max = min(수신 윈도우, 혼잡 윈도우)
수신 윈도우 기본값: 65,535 bytes (16 bits, Window Scaling 없이)
RTT: 150ms = 0.15s

이론 처리량 = 65,535 / 0.15 = 436,900 bytes/s ≈ 3.5 Mbps
```

**3.5 Mbps??** 1Gbps 링크인데도 RTT만으로 이렇게 제한될 수 있습니다.

**Window Scaling 적용 시:**
최신 OS는 자동으로 Window Scaling을 협상합니다 (최대 2^30 = 1GB).
실제로는 소켓 버퍼 크기가 한계:

```
net.ipv4.tcp_rmem max = 16MB
이론 처리량 = 16MB / 0.15s ≈ 853 Mbps
```

**실제 처리량을 높이는 방법:**

1. **TCP 소켓 버퍼 증가:**
   ```bash
   sysctl -w net.ipv4.tcp_rmem="4096 4194304 33554432"  # max 32MB
   sysctl -w net.ipv4.tcp_wmem="4096 4194304 33554432"
   ```

2. **병렬 연결 사용:**
   단일 TCP가 느리면 → 다중 연결로 우회
   HTTP/1.1 6개 병렬 > HTTP/2 단일 (고지연 환경에서)

3. **HTTP/2 우선순위 + 멀티플렉싱:**
   단일 연결에서 여러 스트림 → cwnd 공유 → 빠른 성장

4. **QUIC (HTTP/3):**
   UDP 기반 → 연결 성립이 빠름
   패킷 손실 시 해당 스트림만 재전송 (다른 스트림 차단 없음)

5. **CDN 사용:**
   가장 효과적: RTT 150ms → 20ms (가까운 PoP)
   = 처리량 7.5배 향상 (150/20)

</details>

---

<div align="center">

**[⬅️ 이전: 네트워크 문제 패턴](./03-network-failure-patterns.md)** | **[홈으로 🏠](../README.md)**

*🎉 network-deep-dive 완주! 총 37개 문서*

</div>
