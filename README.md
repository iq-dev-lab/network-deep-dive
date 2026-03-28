<div align="center">

# 🌐 Network Deep Dive

**"HTTP를 쓰는 것과, TCP가 어떻게 연결을 맺고 끊는지 아는 것은 다르다"**

<br/>

> *"패킷을 보내는 것과, 패킷이 어떤 경로로 어떤 보장과 함께 전달되는지 아는 것은 다르다 — TLS 핸드쉐이크를 모른 채 HTTPS를 쓰고, TIME_WAIT을 모른 채 서버를 재시작하고, tcpdump를 한 번도 안 써본 채 네트워크 장애를 마주한다면 — 네트워크를 블랙박스로 두고 있는 것이다"*

TCP 3-Way Handshake가 왜 3번이어야 하는지, TLS 핸드쉐이크에서 공개키와 대칭키가 어떻게 결합되는지, HTTP/2 멀티플렉싱이 HOL Blocking을 어떻게 해결하는지, TIME_WAIT이 왜 필요하고 줄이면 안 되는지까지  
**왜 이렇게 설계됐는가** 라는 질문으로 TCP/IP 스택 전체를 패킷 레벨에서 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![TCP](https://img.shields.io/badge/TCP-RFC_793-blue?style=flat-square&logo=ietf&logoColor=white)](https://www.rfc-editor.org/rfc/rfc793)
[![HTTP/2](https://img.shields.io/badge/HTTP%2F2-RFC_7540-green?style=flat-square&logo=ietf&logoColor=white)](https://www.rfc-editor.org/rfc/rfc7540)
[![TLS](https://img.shields.io/badge/TLS_1.3-RFC_8446-orange?style=flat-square&logo=letsencrypt&logoColor=white)](https://www.rfc-editor.org/rfc/rfc8446)
[![Docs](https://img.shields.io/badge/Docs-37개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

네트워크에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "HTTPS를 쓰면 안전합니다" | TLS 핸드쉐이크에서 ClientHello → ServerHello → Certificate → Finished까지 각 패킷이 무엇을 교환하는지, 공개키 암호화가 세션 키 하나를 전달하고 이후 대칭키로 전환하는 이유 |
| "TCP는 신뢰성을 보장합니다" | Sequence Number와 ACK로 재전송 타이머를 설정하는 방식, Sliding Window가 수신 버퍼에 맞춰 전송 속도를 조절하는 원리, Slow Start에서 혼잡 윈도우가 지수적으로 증가하다 임계점에서 꺾이는 메커니즘 |
| "HTTP/2는 HTTP/1.1보다 빠릅니다" | HTTP/1.1의 HOL Blocking이 왜 파이프라이닝으로도 해결되지 않는지, HTTP/2 바이너리 프레임과 스트림 ID로 멀티플렉싱이 구현되는 방식, HPACK 정적 테이블이 헤더 크기를 어떻게 줄이는지 |
| "TIME_WAIT은 서버 재시작을 방해합니다" | TIME_WAIT 2MSL이 지연 패킷으로부터 새 연결을 보호하는 원리, SO_REUSEADDR을 무분별하게 쓰면 안 되는 이유, `netstat -an \| grep TIME_WAIT`로 상태를 진단하는 방법 |
| "tcpdump로 패킷을 캡처하세요" | `tcpdump -i any -w capture.pcap 'tcp port 443'`으로 TLS 핸드쉐이크를 캡처해 Wireshark에서 분석하는 전체 흐름, SYN/FIN/RST 패킷을 식별하고 RTT를 측정하는 방법 |
| 이론 나열 | 실행 가능한 tcpdump 캡처 + Wireshark 분석 + openssl 명령어 + Docker Compose 실험 환경 + Spring/MySQL 연결 지점 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Ch1-OSI_계층과_패킷_흐름-4479A1?style=for-the-badge&logo=cisco&logoColor=white)](./network-layers-and-packet-flow/01-osi-vs-tcp-ip-layers.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-TCP_3--Way_Handshake-4479A1?style=for-the-badge&logo=wireshark&logoColor=white)](./tcp-internals/01-three-way-handshake.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-HTTP/1.1_내부_구조-4479A1?style=for-the-badge&logo=http&logoColor=white)](./http-internals/01-http1-internals.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-TLS_핸드쉐이크-4479A1?style=for-the-badge&logo=letsencrypt&logoColor=white)](./tls-https-internals/01-crypto-basics.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-DNS_조회_흐름-4479A1?style=for-the-badge&logo=cloudflare&logoColor=white)](./dns-internals/01-dns-resolution-flow.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-L4_vs_L7_로드밸런서-4479A1?style=for-the-badge&logo=nginx&logoColor=white)](./load-balancing-and-proxy/01-l4-vs-l7-load-balancer.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-Docker_네트워킹-4479A1?style=for-the-badge&logo=docker&logoColor=white)](./container-and-network-practice/01-docker-networking.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 네트워크 계층과 패킷 흐름

> **핵심 질문:** 패킷은 출발지에서 목적지까지 어떤 경로로, 어느 계층이 무슨 책임을 지며 전달되는가?

<details>
<summary><b>OSI 7계층부터 네트워크 진단 도구까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. OSI 7계층 vs TCP/IP 4계층 — 계층 모델이 왜 필요한가](./network-layers-and-packet-flow/01-osi-vs-tcp-ip-layers.md) | 계층 모델이 존재하는 이유(관심사 분리, 독립적 발전), OSI 7계층과 TCP/IP 4계층의 각 계층이 실제로 하는 일과 담당하는 헤더, 패킷이 각 계층을 지나며 캡슐화(Encapsulation) / 역캡슐화(Decapsulation) 되는 원리, Spring MVC 요청이 어느 계층에서 처리되는가 |
| [02. IP와 라우팅 — 패킷이 목적지를 찾아가는 방법](./network-layers-and-packet-flow/02-ip-and-routing.md) | IP 헤더 구조와 TTL 감소로 라우팅 루프를 차단하는 원리, 라우터가 라우팅 테이블로 Next Hop을 결정하는 방식, 단편화(Fragmentation)가 발생하는 조건과 MTU, traceroute가 TTL을 1씩 늘려 경로를 추적하는 메커니즘 |
| [03. Ethernet과 ARP — 같은 네트워크 안에서 통신하는 원리](./network-layers-and-packet-flow/03-ethernet-and-arp.md) | IP 주소(L3)와 MAC 주소(L2)가 각각 필요한 이유, ARP 요청/응답으로 IP→MAC 매핑을 동적으로 해결하는 과정, ARP 캐시 테이블과 만료 주기, Gratuitous ARP와 ARP Spoofing 원리 |
| [04. NAT와 포트 포워딩 — 사설 IP가 인터넷에 나가는 방법](./network-layers-and-packet-flow/04-nat-and-port-forwarding.md) | NAPT(Network Address and Port Translation)가 출발지 IP:Port를 공인 IP:Port로 변환하는 원리, NAT 테이블이 응답 패킷을 역변환하는 방식, 포트 포워딩 설정이 외부에서 내부 서버에 접근하는 경로, Docker 컨테이너 포트 매핑이 iptables NAT 규칙으로 구현되는 방식 |
| [05. 네트워크 진단 도구 — tcpdump, Wireshark, ss 실전 활용](./network-layers-and-packet-flow/05-network-diagnostic-tools.md) | `tcpdump` 필터 문법으로 원하는 패킷만 캡처하는 방법, Wireshark에서 TCP Stream Follow로 전체 연결 흐름 분석, `ss -tanp`로 소켓 상태와 프로세스를 연결하는 방법, `ping`/`traceroute`로 경로 장애를 단계적으로 격리하는 진단 플로우 |

</details>

<br/>

### 🔹 Chapter 2: TCP 완전 분해

> **핵심 질문:** TCP는 신뢰성을 어떻게 보장하고, TIME_WAIT은 왜 필요하며, 혼잡 제어는 어떻게 동작하는가?

<details>
<summary><b>3-Way Handshake부터 소켓 상태 머신까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. TCP 연결 수립 — 3-Way Handshake가 왜 3번인가](./tcp-internals/01-three-way-handshake.md) | SYN/SYN-ACK/ACK 각 패킷이 실제로 교환하는 정보(ISN, 윈도우 사이즈, MSS), 2-Way로는 충분하지 않은 이유(양방향 ISN 동기화), `tcpdump`로 Handshake 패킷 캡처하고 각 필드 해석하기, Spring RestTemplate 연결 시 Handshake 비용과 Connection Pool의 관계 |
| [02. TCP 연결 종료 — 4-Way Handshake와 TIME_WAIT](./tcp-internals/02-four-way-handshake-time-wait.md) | FIN/FIN-ACK/FIN/ACK 4단계가 필요한 이유(Half-Close와 데이터 잔존), TIME_WAIT 2MSL(Maximum Segment Lifetime)이 지연 패킷으로부터 새 연결을 보호하는 원리, `netstat -an \| grep TIME_WAIT`으로 포트 고갈 현상 진단, SO_REUSEADDR/SO_REUSEPORT의 올바른 사용 기준 |
| [03. TCP 신뢰성 보장 — Sequence Number, ACK, 재전송](./tcp-internals/03-tcp-reliability.md) | Sequence Number와 ACK Number로 손실/중복/순서 뒤바뀜을 감지하는 방식, RTO(Retransmission Timeout) 계산 알고리즘(Karn's Algorithm, RTT 측정), Cumulative ACK vs Selective ACK(SACK), `tcpdump`에서 재전송 패킷을 식별하는 방법 |
| [04. TCP 흐름 제어 — Sliding Window와 수신 버퍼](./tcp-internals/04-flow-control-sliding-window.md) | 수신자의 버퍼 여유(rwnd)를 ACK에 실어 송신 속도를 조절하는 Sliding Window 프로토콜, Window Size가 0이 되는 Zero Window 상태와 Window Probe, Wireshark에서 `tcp.window_size`로 흐름 제어 동작 확인하기 |
| [05. TCP 혼잡 제어 — Slow Start, AIMD, Fast Recovery](./tcp-internals/05-congestion-control.md) | 혼잡 윈도우(cwnd)가 Slow Start에서 지수 증가 → ssthresh 도달 후 선형 증가로 전환되는 AIMD 메커니즘, 패킷 손실 감지 시 Fast Retransmit/Fast Recovery로 cwnd를 절반으로 줄이는 원리, BBR vs CUBIC 알고리즘 차이 |
| [06. TCP vs UDP — 구조 차이와 UDP를 선택하는 조건](./tcp-internals/06-tcp-vs-udp.md) | TCP 헤더(20바이트)와 UDP 헤더(8바이트) 구조 비교, UDP가 신뢰성 없이 속도를 택하는 이유(게임, DNS, 스트리밍), QUIC이 UDP 위에서 신뢰성을 직접 구현하는 방식, DNS가 UDP를 쓰면서도 재전송을 구현하는 방법 |
| [07. TCP 소켓 상태 머신 — LISTEN에서 TIME_WAIT까지](./tcp-internals/07-tcp-socket-state-machine.md) | CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → FIN_WAIT_1/2 → TIME_WAIT 전체 상태 전환 조건, CLOSE_WAIT 누적이 서버 장애로 이어지는 시나리오, `ss -tanp`로 각 상태의 소켓을 실시간 관찰하기, Spring Boot 애플리케이션에서 CLOSE_WAIT 누적 원인과 해결 방법 |

</details>

<br/>

### 🔹 Chapter 3: HTTP 완전 분해

> **핵심 질문:** HTTP/1.1의 어떤 문제를 HTTP/2가 어떻게 해결했고, HTTP/3는 왜 UDP를 선택했는가?

<details>
<summary><b>HTTP/1.1 구조부터 WebSocket까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. HTTP/1.1 내부 구조 — Keep-Alive와 파이프라이닝의 한계](./http-internals/01-http1-internals.md) | HTTP 요청/응답 메시지 포맷(Start Line, Headers, Body), Keep-Alive가 TCP 연결을 재사용하는 방식과 `Connection: keep-alive` 헤더, 파이프라이닝이 HOL Blocking을 해결하지 못하는 이유(순서 보장 문제), `curl -v`로 요청/응답 헤더 전체를 보는 방법 |
| [02. HTTP/2 — 바이너리 프레이밍과 멀티플렉싱](./http-internals/02-http2-multiplexing.md) | HTTP/2 바이너리 프레임 구조(Length, Type, Flags, Stream ID), 동일 TCP 연결에서 여러 Stream이 병렬로 교환되어 HOL Blocking을 해결하는 멀티플렉싱 원리, HPACK 정적/동적 테이블로 헤더 크기를 줄이는 방식, `curl --http2 -v`로 HTTP/2 협상 과정 확인 |
| [03. HTTP/3와 QUIC — UDP 위에서 신뢰성을 구현하는 방법](./http-internals/03-http3-quic.md) | HTTP/2의 TCP HOL Blocking(패킷 손실 시 전체 스트림 차단) 문제, QUIC이 UDP 위에서 스트림별 독립 재전송을 구현하는 방식, 0-RTT 핸드쉐이크가 이전 세션 정보로 즉시 데이터를 보내는 원리와 재전송 공격 위험, `curl --http3`로 QUIC 연결 확인 |
| [04. HTTP 캐싱 완전 분해 — Cache-Control과 캐시 무효화](./http-internals/04-http-caching.md) | `Cache-Control: max-age`, `no-cache`, `no-store`, `must-revalidate` 각 지시어의 정확한 의미 차이, ETag와 `If-None-Match`로 조건부 요청을 구현하는 방식(304 Not Modified), CDN 캐시와 브라우저 캐시의 계층, 배포 시 캐시 무효화(Cache Busting) 전략 |
| [05. HTTP 메서드와 상태 코드 — 멱등성과 안전성의 의미](./http-internals/05-http-methods-status-codes.md) | GET/POST/PUT/PATCH/DELETE의 멱등성(Idempotent)과 안전성(Safe) 분류와 그 의미, 올바른 상태 코드 선택 기준(200 vs 201 vs 204, 400 vs 422 vs 409), REST API 설계에서 상태 코드가 클라이언트 재시도 전략에 미치는 영향 |
| [06. WebSocket — HTTP Upgrade와 프레임 구조](./http-internals/06-websocket-internals.md) | HTTP Upgrade 핸드쉐이크로 WebSocket 연결이 수립되는 과정(`Sec-WebSocket-Key` 검증), WebSocket 프레임 구조(FIN, Opcode, Mask, Payload), Long Polling vs SSE(Server-Sent Events) vs WebSocket 비교, Spring WebSocket과 STOMP가 TCP 연결 위에서 동작하는 방식 |

</details>

<br/>

### 🔹 Chapter 4: TLS/HTTPS 완전 분해

> **핵심 질문:** TLS 핸드쉐이크에서 공개키와 대칭키는 어떻게 결합되고, TLS 1.3은 어떻게 1-RTT를 달성하는가?

<details>
<summary><b>암호화 기초부터 mTLS까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 암호화 기초 — 대칭키/비대칭키/해시가 각각 어디에 쓰이는가](./tls-https-internals/01-crypto-basics.md) | 대칭키(AES) 암호화의 빠른 이유와 키 교환 문제, 비대칭키(RSA, ECDH)로 키를 안전하게 교환하는 원리, 해시 함수(SHA-256)가 무결성 검증에 쓰이는 방식, MAC(Message Authentication Code)과 디지털 서명의 차이 |
| [02. TLS 1.2 핸드쉐이크 — ClientHello부터 Finished까지](./tls-https-internals/02-tls12-handshake.md) | ClientHello(지원 Cipher Suite 목록) → ServerHello(선택한 Cipher Suite) → Certificate(서버 인증서) → ServerKeyExchange → ClientKeyExchange → ChangeCipherSpec → Finished 각 패킷이 교환하는 정보, `openssl s_client -connect host:443 -tls1_2 -msg`로 전체 핸드쉐이크 추적 |
| [03. TLS 1.3 — 1-RTT 핸드쉐이크와 Forward Secrecy](./tls-https-internals/03-tls13-improvements.md) | TLS 1.3이 2-RTT → 1-RTT로 줄인 방법(ClientHello에 KeyShare 포함), RSA 키 교환 제거와 ECDHE 필수화로 Forward Secrecy를 보장하는 원리, 0-RTT Early Data의 동작 방식과 재전송 공격(Replay Attack) 위험, `curl --tlsv1.3 -v`로 핸드쉐이크 확인 |
| [04. 인증서와 PKI — X.509 구조와 CA 체인 검증](./tls-https-internals/04-certificate-pki.md) | X.509 인증서 필드 구조(Subject, Issuer, Validity, Public Key, Signature), Root CA → Intermediate CA → End-Entity 인증서 체인 검증 과정, Self-Signed 인증서의 한계, OCSP Stapling이 인증서 폐기 확인 지연을 줄이는 방식, `openssl x509 -text`로 인증서 파싱 |
| [05. HTTPS 성능 최적화 — Session Resumption과 OCSP Stapling](./tls-https-internals/05-https-performance.md) | Session ID 기반 재개(서버 메모리 의존)와 Session Ticket 기반 재개(상태 없는 서버) 비교, TLS 핸드쉐이크 RTT 비용을 측정하는 방법(`curl -w "%{time_connect} %{time_appconnect}"`), Nginx에서 `ssl_session_tickets`, `ssl_stapling` 설정 |
| [06. mTLS — 양방향 인증과 Zero Trust](./tls-https-internals/06-mtls-zero-trust.md) | 서버만 인증하는 일반 TLS vs 클라이언트도 인증하는 mTLS 핸드쉐이크 차이, 마이크로서비스 간 통신에서 mTLS로 Zero Trust 보안을 구현하는 방식, Kubernetes에서 Istio가 사이드카로 mTLS를 투명하게 적용하는 원리, `openssl`로 직접 클라이언트/서버 인증서를 생성하고 mTLS 연결 실험 |

</details>

<br/>

### 🔹 Chapter 5: DNS 완전 분해

> **핵심 질문:** DNS 조회는 어떤 서버를 거치고, TTL은 캐싱과 배포에 어떤 영향을 주는가?

<details>
<summary><b>DNS 조회 흐름부터 DNS 보안까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. DNS 조회 흐름 — Resolver, Root NS, TLD NS, Authoritative NS](./dns-internals/01-dns-resolution-flow.md) | 재귀 쿼리(Recursive)와 반복 쿼리(Iterative)의 차이, Stub Resolver → Recursive Resolver → Root NS → TLD NS → Authoritative NS 각 단계가 응답하는 내용, `dig +trace example.com`으로 전체 조회 경로 추적, Spring 애플리케이션의 DNS 캐시 TTL 설정(`networkaddress.cache.ttl`) |
| [02. DNS 레코드 완전 가이드 — A, CNAME, MX, TXT, SRV](./dns-internals/02-dns-records.md) | A(IPv4), AAAA(IPv6), CNAME(별칭), MX(메일), TXT(검증/SPF/DKIM), SRV(서비스 위치), PTR(역방향) 레코드별 용도와 형식, CNAME이 A 레코드를 직접 대체할 수 없는 경우(Zone Apex), `dig` 명령어로 각 레코드 타입 조회 |
| [03. DNS 캐싱과 전파 — TTL이 배포에 미치는 영향](./dns-internals/03-dns-caching-and-propagation.md) | TTL(Time To Live)이 Resolver 캐시에 저장되는 시간과 전파 지연의 관계, 서비스 마이그레이션 전 TTL을 낮추는 이유와 롤백 안전망 확보 전략, Negative Caching(NXDOMAIN 캐시), `dig +norecurse`로 캐시 상태 확인 |
| [04. DNS 보안과 고급 — DNSSEC, DoH, DNS 기반 로드밸런싱](./dns-internals/04-dns-security-advanced.md) | DNSSEC이 전자 서명으로 DNS 응답 위변조를 방지하는 원리, DNS over HTTPS(DoH) / DNS over TLS(DoT)가 평문 DNS 쿼리를 암호화하는 방식, DNS 기반 라운드로빈 로드밸런싱의 한계(클라이언트 캐시로 불균등 분산), GeoDNS로 지역별 IP를 다르게 응답하는 방식 |

</details>

<br/>

### 🔹 Chapter 6: 로드 밸런싱과 프록시 아키텍처

> **핵심 질문:** L4와 L7 로드 밸런서는 어느 계층에서 무엇을 보고 분산하며, Nginx는 upstream 연결을 어떻게 관리하는가?

<details>
<summary><b>L4/L7 로드 밸런서부터 서킷 브레이커까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. L4 vs L7 로드 밸런서 — 계층별 분산 방식과 한계](./load-balancing-and-proxy/01-l4-vs-l7-load-balancer.md) | L4 로드 밸런서가 TCP/UDP 헤더(IP:Port)만 보고 NAT로 분산하는 방식, L7 로드 밸런서가 HTTP 헤더/URL/쿠키를 파싱해 라우팅하는 방식, DSR(Direct Server Return)이 응답 트래픽에서 로드 밸런서를 우회하는 원리, AWS NLB(L4) vs ALB(L7) 선택 기준 |
| [02. 리버스 프록시 — Nginx 내부 동작과 upstream 연결 관리](./load-balancing-and-proxy/02-reverse-proxy-nginx.md) | Nginx가 클라이언트 연결과 upstream 연결을 분리해서 관리하는 방식, `keepalive` 지시어로 upstream TCP 연결을 재사용하는 풀 관리, 버퍼링(Buffering)이 upstream 서버를 빠른 클라이언트에게 맞춰 보호하는 원리, `access_log`에서 `upstream_response_time`과 `request_time` 차이 |
| [03. 연결 유지 전략 — Sticky Session vs Stateless 설계](./load-balancing-and-proxy/03-sticky-session-vs-stateless.md) | Sticky Session(IP Hash, Cookie 기반 Session Affinity)이 서버 쏠림과 단일 장애점을 만드는 이유, JWT/Redis를 사용한 Stateless 세션 설계로 수평 확장하는 방법, Spring Session + Redis로 세션을 공유하면서 Sticky Session 의존을 제거하는 패턴 |
| [04. Rate Limiting — Token Bucket, Leaky Bucket, Sliding Window](./load-balancing-and-proxy/04-rate-limiting-algorithms.md) | Fixed Window의 경계 시점 burst 문제, Sliding Window Log/Counter로 경계 문제를 해결하는 방식, Token Bucket이 순간 burst를 허용하면서 평균 처리율을 제한하는 원리, Leaky Bucket의 엄격한 출력 속도 제한, Nginx `limit_req_zone`과 Spring의 Bucket4j 구현 비교 |
| [05. 서킷 브레이커 — 장애 전파 방지와 Half-Open 상태](./load-balancing-and-proxy/05-circuit-breaker.md) | Closed → Open → Half-Open 상태 전환 조건과 각 상태에서의 요청 처리 방식, 실패율(Failure Rate) 기반 vs 슬로우 콜(Slow Call) 기반 임계값 설정, Resilience4j가 슬라이딩 윈도우로 실패율을 계산하는 내부 구조, Fallback 전략(캐시 응답, 기본값 반환) 설계 |

</details>

<br/>

### 🔹 Chapter 7: 컨테이너와 실전 네트워크

> **핵심 질문:** Docker 컨테이너는 어떻게 격리된 네트워크를 갖고, Kubernetes의 Pod는 어떻게 서로 통신하는가?

<details>
<summary><b>Docker 네트워킹부터 네트워크 성능 측정까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Docker 네트워킹 — bridge, host, overlay 모드와 veth](./container-and-network-practice/01-docker-networking.md) | Docker bridge 네트워크에서 veth(virtual ethernet) 페어로 컨테이너가 격리된 네임스페이스를 갖는 원리, iptables MASQUERADE 규칙으로 컨테이너 트래픽이 NAT되는 과정, overlay 네트워크가 VXLAN으로 여러 호스트의 컨테이너를 연결하는 방식, `docker network inspect`와 `ip link show`로 veth 구조 확인 |
| [02. Kubernetes 네트워킹 — Pod IP, kube-proxy, Service 타입](./container-and-network-practice/02-kubernetes-networking.md) | Pod가 고유 IP를 갖고 NAT 없이 통신할 수 있는 Kubernetes 네트워킹 모델, kube-proxy가 iptables/ipvs 규칙으로 Service ClusterIP를 Pod IP로 로드밸런싱하는 원리, NodePort/LoadBalancer/Ingress의 트래픽 흐름 차이, CNI(Container Network Interface) 플러그인이 Pod 네트워크를 구성하는 역할 |
| [03. 운영에서 만나는 네트워크 문제 패턴 — RST, Timeout, 포트 고갈](./container-and-network-practice/03-network-failure-patterns.md) | Connection Reset(RST)이 발생하는 조건(keep-alive timeout 불일치, 방화벽 세션 만료), Connection Timeout vs Read Timeout vs Connection Refused의 원인 차이와 진단 방법, Ephemeral Port 고갈이 발생하는 조건(`/proc/sys/net/ipv4/ip_local_port_range`)과 TIME_WAIT 누적, `ss -s`로 소켓 통계 실시간 모니터링 |
| [04. 네트워크 성능 측정 — RTT, 처리량, 지연시간 분석](./container-and-network-practice/04-network-performance-measurement.md) | `ping`으로 RTT(Round-Trip Time)를 측정하고 지터(Jitter) 분석, `iperf3`로 TCP/UDP 처리량과 대역폭을 측정하는 방법, Wireshark TCP 분석 그래프(Time-Sequence, Throughput, RTT)로 혼잡 제어 동작 시각화, `ss --info`로 개별 소켓의 cwnd/rtt/rto 값 확인 |

</details>

---

## 🔭 핵심 분석 대상 — HTTP 요청의 전체 여정

```
HTTP 요청 전체 흐름:

애플리케이션 (Spring RestTemplate / WebClient)
  │
  ▼ DNS 조회 (Ch5)
  │  Stub Resolver → Recursive Resolver → Root NS → TLD NS → Authoritative NS
  │  결과: example.com → 93.184.216.34
  ▼
IP 주소 확인
  │
  ▼ TCP 연결 수립 (Ch2-01)
  │  [Client] SYN (seq=x)         →
  │                               ← SYN-ACK (seq=y, ack=x+1)
  │  [Client] ACK (ack=y+1)       →   [ESTABLISHED]
  │  비용: 1.5 RTT (약 30~100ms, 서버 위치에 따라 다름)
  ▼
TLS 핸드쉐이크 (Ch4, HTTPS인 경우)
  │  TLS 1.3: 1 RTT
  │    ClientHello (KeyShare 포함) →
  │                               ← ServerHello + Certificate + Finished
  │    Finished                   →
  │  TLS 1.2: 2 RTT (레거시)
  ▼
HTTP 요청 전송 (Ch3)
  │  HTTP/1.1: 요청 직렬 처리 → HOL Blocking
  │  HTTP/2:   스트림 멀티플렉싱 → 병렬 처리
  │  HTTP/3:   QUIC (UDP) → 스트림별 독립 재전송
  ▼
응답 수신
  │
  ▼ TCP 연결 종료 또는 Keep-Alive 유지 (Ch2-02)
  │  FIN → FIN-ACK → FIN → ACK
  │  → TIME_WAIT 2MSL (약 60초) — 지연 패킷 보호

장애 진단 플로우 (Ch1-05, Ch2-07, Ch7-03):
  증상: 특정 API 응답 없음
  ├── ping target-host      → ICMP 도달 여부 (L3 연결 확인)
  ├── telnet target 443     → TCP 포트 오픈 여부 (L4 확인)
  ├── curl -v https://...   → TLS 핸드쉐이크 성공 여부 (L7 확인)
  ├── tcpdump 'host target' → 패킷 수준 SYN/RST/FIN 확인
  └── ss -tanp              → CLOSE_WAIT/TIME_WAIT 누적 여부
```

---

## 🐳 실험 환경 (Docker Compose)

모든 챕터의 실험은 아래 환경에서 재현 가능합니다.

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    cap_add:
      - NET_ADMIN

  app:
    image: eclipse-temurin:21-jre
    depends_on:
      - nginx
    environment:
      - JAVA_OPTS=-Djava.net.preferIPv4Stack=true

  wireshark:
    image: linuxserver/wireshark
    cap_add:
      - NET_ADMIN
    network_mode: host
    ports:
      - "3000:3000"         # 브라우저에서 Wireshark GUI 접속
    environment:
      - PUID=1000
      - PGID=1000
```

```bash
# 실험용 공통 명령어 세트

# TCP 3-Way Handshake 캡처 (Chapter 2)
tcpdump -i any -w capture.pcap 'tcp port 80 and host example.com'

# TLS 핸드쉐이크 전체 메시지 추적 (Chapter 4)
openssl s_client -connect example.com:443 -tls1_3 -msg 2>&1 | grep -E ">>|<<"

# TLS 연결 시간 측정 (Chapter 4)
curl -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | Total: %{time_total}s\n" \
     -o /dev/null -s https://example.com

# HTTP/2 협상 확인 (Chapter 3)
curl -v --http2 https://example.com 2>&1 | grep -E "< HTTP|ALPN|h2"

# 소켓 상태 실시간 모니터링 (Chapter 2, 7)
watch -n 1 "ss -tanp | grep -E 'ESTABLISHED|TIME_WAIT|CLOSE_WAIT' | wc -l"

# TIME_WAIT 개수 확인 (Chapter 2)
ss -tan | grep TIME-WAIT | wc -l

# DNS 조회 경로 추적 (Chapter 5)
dig +trace +additional example.com

# 패킷 처리량 측정 (Chapter 7)
iperf3 -s          # 서버
iperf3 -c server_ip -t 30 -P 4   # 클라이언트 (4개 스트림)
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 원리를 모를 때의 접근과 그 결과 |
| ✨ **올바른 접근** | After — 원리를 알고 난 후의 설계/운영 방식 |
| 🔬 **내부 동작 원리** | 패킷 레벨 분석 + 프로토콜 내부 구조도 |
| 💻 **실전 실험** | tcpdump 캡처, Wireshark 분석, openssl, curl 재현 시나리오 |
| 📊 **성능/비용 비교** | HTTP/1.1 vs HTTP/2, TLS 버전별 RTT, 알고리즘 비교 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "네트워크 장애가 생겼는데 아무것도 모르겠다" — 긴급 진단 투입 (2일)</b></summary>

<br/>

```
Day 1  Ch1-05  tcpdump, ss, ping, traceroute — 도구 먼저 익히기
       Ch2-07  소켓 상태 머신 — CLOSE_WAIT/TIME_WAIT 의미 파악
Day 2  Ch7-03  운영에서 만나는 문제 패턴 — RST, Timeout 진단 플로우
       Ch7-04  네트워크 성능 측정 — RTT, 처리량 수치화
```

</details>

<details>
<summary><b>🟡 "TCP와 TLS를 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch2-01  TCP 3-Way Handshake → tcpdump로 직접 캡처
Day 2  Ch2-02  4-Way Handshake + TIME_WAIT → netstat로 상태 확인
       Ch2-03  Sequence Number, ACK, 재전송 원리
Day 3  Ch2-04  흐름 제어 — Sliding Window
       Ch2-05  혼잡 제어 — Slow Start, AIMD
Day 4  Ch4-01  암호화 기초 — 대칭키/비대칭키/해시
       Ch4-02  TLS 1.2 핸드쉐이크 — 패킷별 분석
Day 5  Ch4-03  TLS 1.3 — 1-RTT 달성 방법
       Ch4-04  인증서와 PKI — CA 체인 검증
Day 6  Ch3-01  HTTP/1.1 내부 구조 → Keep-Alive
       Ch3-02  HTTP/2 멀티플렉싱 → HOL Blocking 해결
Day 7  Ch3-03  HTTP/3 QUIC → UDP 위의 신뢰성
```

</details>

<details>
<summary><b>🔴 "TCP/IP 스택 전체를 패킷 레벨에서 완전히 이해하고 싶다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 네트워크 계층과 패킷 흐름
        → tcpdump로 ARP → IP → TCP 계층 캡슐화 직접 관찰

2주차  Chapter 2 전체 — TCP 완전 분해
        → Wireshark TCP Stream으로 Handshake, 재전송, 윈도우 변화 시각화

3주차  Chapter 3 전체 — HTTP 완전 분해
        → curl로 HTTP/1.1 → HTTP/2 → HTTP/3 각각 연결해 헤더 비교

4주차  Chapter 4 전체 — TLS/HTTPS 완전 분해
        → openssl로 TLS 1.2 vs 1.3 핸드쉐이크 메시지 직접 추적

5주차  Chapter 5 전체 — DNS 완전 분해
        → dig +trace로 전체 조회 경로 확인, TTL 변화 실험

6주차  Chapter 6 전체 — 로드 밸런싱과 프록시
        → Nginx upstream 연결 풀 설정 실험, Rate Limiting 알고리즘 비교

7주차  Chapter 7 전체 — 컨테이너와 실전 네트워크
        → Docker veth 구조 직접 확인, Kubernetes Service iptables 규칙 분석
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-mvc-deep-dive](https://github.com/dev-book-lab/spring-mvc-deep-dive) | DispatcherServlet, 필터/인터셉터, 요청 처리 흐름 | Ch3(HTTP 메서드/상태 코드가 Spring MVC 응답과 연결되는 지점), Ch4(Spring Security의 TLS 설정) |
| [mysql-deep-dive](https://github.com/dev-book-lab/mysql-deep-dive) | MySQL 사용자 관리, 인증, SSL 설정 | Ch4(MySQL SSL/TLS 연결 설정, `require_secure_transport`), Ch2(DB Connection Pool이 TCP Keep-Alive를 사용하는 방식) |
| [database-internals](https://github.com/dev-book-lab/database-internals) | InnoDB 스토리지, 인덱스, MVCC | Ch2(DB 연결이 소켓 상태 머신을 거치는 방식, HikariCP의 CLOSE_WAIT 처리) |

> 💡 이 레포는 **TCP/IP 스택 내부 동작**에 집중합니다. Spring/MySQL/Docker를 몰라도 네트워크 관점으로 학습 가능합니다. 단, Chapter 3의 Spring WebSocket 연결, Chapter 6의 Resilience4j 서킷 브레이커, Chapter 7의 Spring Boot 컨테이너 설정은 Spring 선행 지식이 있을 때 더 깊이 연결됩니다.

---

## 🙏 Reference

- [TCP/IP Illustrated, Volume 1 — W. Richard Stevens](https://www.oreilly.com/library/view/tcpip-illustrated-volume/9780132808200/)
- [Computer Networks: A Systems Approach — Peterson & Davie](https://book.systemsapproach.org/)
- [High Performance Browser Networking — Ilya Grigorik](https://hpbn.co)
- [RFC 793 — Transmission Control Protocol (TCP)](https://www.rfc-editor.org/rfc/rfc793)
- [RFC 7540 — Hypertext Transfer Protocol Version 2 (HTTP/2)](https://www.rfc-editor.org/rfc/rfc7540)
- [RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000)
- [Cloudflare Blog — How we built network internals](https://blog.cloudflare.com)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"HTTP를 쓰는 것과, TCP가 어떻게 연결을 맺고 끊는지 아는 것은 다르다"*

</div>
