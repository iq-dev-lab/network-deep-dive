# DNS 보안과 고급 — DNSSEC, DoH, DNS 기반 로드밸런싱

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DNS Cache Poisoning 공격은 어떻게 동작하고 DNSSEC은 이를 어떻게 방어하는가?
- DNSSEC의 신뢰 체인(DNSKEY → DS → Root Key)은 어떻게 구성되는가?
- DNS over HTTPS(DoH)와 DNS over TLS(DoT)의 차이와 각각의 장단점은?
- DNS 라운드로빈 로드밸런싱이 실제로 불균등하게 동작하는 이유는?
- GeoDNS는 어떤 기준으로 지역을 판단하는가?
- DNS가 Kubernetes 서비스 디스커버리에서 어떻게 사용되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"DNS Cache Poisoning이 실제로 가능한가?":
  Kaminsky Attack (2008년 발견):
  공격자가 Resolver의 DNS 캐시에 가짜 IP 삽입
  example.com → 공격자 서버 → 피싱/중간자 공격
  
  방어: DNSSEC (암호학적 서명)
  현실: Let's Encrypt, 대형 도메인들은 DNSSEC 미사용
  → DNSSEC 설정의 복잡성 vs 보안 이점

"직원 PC에서 8.8.8.8 대신 회사 DNS를 써야 하는 이유":
  평문 UDP DNS → 중간자가 쿼리 내용 열람 가능
  → 어느 사이트에 접속하는지 ISP/공격자가 알 수 있음
  → DoH/DoT로 DNS 쿼리 암호화 필요

"DNS 로드밸런싱으로 부하 분산이 안 된다":
  A 레코드 3개 (라운드로빈):
  Resolver A → 10.0.0.1 반환 → 해당 Resolver 사용자 전체가 10.0.0.1
  Resolver B → 10.0.0.2 반환 → 해당 Resolver 사용자 전체가 10.0.0.2
  → Resolver 단위로 분산 (사용자 단위가 아님)
  → 진짜 로드밸런싱은 L4/L7 로드밸런서가 필요
```

---

## 😱 흔한 실수

```
Before — DNS 보안을 모를 때:

실수 1: DNSSEC이 HTTPS를 대체한다고 생각
  DNSSEC: DNS 응답의 무결성 보장 (DNS 위변조 방지)
  HTTPS(TLS): 전송 데이터 암호화, 서버 인증
  → 역할이 완전히 다름, 둘 다 필요

실수 2: DoH가 완전한 프라이버시를 보장한다고 생각
  DoH: DNS 쿼리를 HTTPS로 암호화 → ISP가 DNS 쿼리 못 봄
  하지만: SNI(Server Name Indication) → TLS에서 도메인명 평문
         IP 주소 자체는 숨길 수 없음
  → 완전한 익명성은 VPN 또는 Tor가 필요

실수 3: DNS 로드밸런싱으로 정확한 가중치 분산 기대
  Route53 Weighted: Weight=50:50 설정
  실제: Resolver별로 응답이 다름 → 불균등 분산
  → 정확한 부하 분산은 L4/L7 LB 필수

실수 4: GeoDNS가 클라이언트 IP를 직접 본다고 생각
  GeoDNS는 Recursive Resolver의 IP를 기반으로 지역 판단
  클라이언트가 8.8.8.8을 사용하면 → Google 데이터센터 IP
  → Google의 데이터센터 위치 기반으로 지역 판단 (부정확)
  → EDNS Client Subnet(ECS)으로 부분 해결
```

---

## ✨ 올바른 접근

```
After — DNS 보안을 알면:

DoH 설정 (Chrome):
  설정 → 보안 → 안전한 DNS → 사용자 지정 주소
  https://dns.google/dns-query
  https://cloudflare-dns.com/dns-query

DoT 설정 (Android):
  네트워크 설정 → 개인 DNS → "one.one.one.one"

조직 내 DoH 프록시:
  # nginx DoH 프록시 (내부 resolver 보호)
  # 클라이언트 → HTTPS → DoH 프록시 → 내부 DNS
  location /dns-query {
    proxy_pass http://internal-dns:8053;
    add_header Content-Type application/dns-message;
  }

DNSSEC 검증 확인:
  dig +dnssec example.com @8.8.8.8
  # AD 플래그 (Authenticated Data) → DNSSEC 검증됨

DNS 보안 모니터링:
  # 의심스러운 DNS 쿼리 탐지
  # ELK + Suricata 또는 Zeek
  # K8s: Falco로 이상 DNS 쿼리 탐지

로드밸런싱 올바른 접근:
  DNS 라운드로빈 + L4/L7 LB 조합:
  
  DNS: 여러 LB IP 반환 (고가용성, 지역 분산)
  LB: 실제 백엔드 서버 부하 분산 (정확한 가중치)
  → DNS는 LB IP를 광고, LB가 실제 분산 담당
```

---

## 🔬 내부 동작 원리

### 1. DNS Cache Poisoning 공격

```
평문 UDP DNS의 취약점:

정상 DNS 조회:
  Client → Resolver: "example.com의 IP는?" (UDP)
  Resolver → Auth NS: "example.com의 IP는?" (UDP)
  Auth NS → Resolver: "203.0.113.10" (UDP, 포트 53)
  Resolver: 캐시에 저장

Kaminsky Attack (2008):
  공격자 목표: Resolver의 캐시에 가짜 IP 삽입

  공격 과정:
  1. 공격자: Resolver에 "random1234.example.com?" 쿼리
  2. Resolver: Auth NS에 "random1234.example.com?" 질의
  3. 공격자: Auth NS 응답 전에 수천 개의 가짜 응답 전송
     (포트 53, 트랜잭션 ID를 추측하며)
  
  가짜 응답 내용:
    random1234.example.com  A  나쁜IP
    example.com             NS  ns1.example.com
    ns1.example.com         A  공격자IP  ← Glue Record 위조!
  
  4. 트랜잭션 ID가 맞으면: Resolver가 가짜 응답 수락
  5. Resolver 캐시: example.com → NS → 공격자IP
  6. 이후 example.com 조회 → 공격자 서버로 연결

취약점 원인:
  UDP: 비연결, 응답 진위 확인 어려움
  트랜잭션 ID: 16 bits (65536 종류) → 추측 가능
  소스 포트 고정: 포트 53 → ID 맞추기 더 쉬움

방어 방법:
  소스 포트 랜덤화: BIND 9.4+ (2007)
  → ID(16 bits) + 포트(16 bits) = 32 bits → 추측 어려움
  DNSSEC: 암호학적 서명으로 근본 해결
```

### 2. DNSSEC — DNS 응답 서명

```
DNSSEC (DNS Security Extensions, RFC 4033-4035):

핵심 원리:
  Authoritative NS가 응답에 디지털 서명 추가
  클라이언트(Resolver)가 서명 검증
  위조된 응답 → 서명 불일치 → 거부

DNSSEC 레코드 종류:
  DNSKEY: 존 서명에 사용하는 공개키
  RRSIG:  레코드 세트의 디지털 서명
  DS:     상위 존의 DNSKEY 해시 (신뢰 앵커)
  NSEC/NSEC3: 없는 도메인 증명 (Authenticated Denial)

신뢰 체인 (Chain of Trust):

  Root (.)
  DNSKEY: Root의 공개키 (Trust Anchor — OS/Resolver에 하드코딩)
  RRSIG:  Root 키로 서명
    ↓ DS 레코드 (com의 DNSKEY 해시)
  
  .com TLD
  DNSKEY: com의 공개키
  RRSIG:  com 키로 서명
    ↓ DS 레코드 (example.com의 DNSKEY 해시)
  
  example.com
  DNSKEY: example.com의 공개키
  RRSIG:  example.com 키로 각 RRset 서명
  
  api.example.com  A  203.0.113.10  → RRSIG 포함

검증 과정:
  1. Resolver: api.example.com A + RRSIG 수신
  2. example.com의 DNSKEY 조회
  3. DNSKEY로 RRSIG 검증
  4. example.com의 DS가 .com에 있는지 확인
  5. .com의 DS가 Root에 있는지 확인
  6. Root DNSKEY는 Trust Anchor (신뢰)
  → 전체 체인 검증 성공 → AD 플래그 설정

ZSK vs KSK:
  ZSK (Zone Signing Key): 일반 레코드 서명 (주기적 교체, 작은 키)
  KSK (Key Signing Key): DNSKEY 자체 서명 (드물게 교체, 큰 키)
  → KSK의 해시 = DS 레코드 → 상위 NS에 등록

DNSSEC의 한계:
  복잡한 설정 및 운영
  키 롤오버 절차 복잡
  NSEC: 없는 도메인 목록이 노출 (Zone Walking 취약점)
  NSEC3: NSEC의 해시 버전 (더 안전하지만 완전하지 않음)
  주요 사이트들이 DNSSEC 미사용하는 이유
```

### 3. DoH와 DoT — DNS 쿼리 암호화

```
평문 DNS의 프라이버시 문제:
  UDP 포트 53 DNS 쿼리 → 평문 전송
  ISP가 DNS 쿼리 로그 → 방문 사이트 파악
  공격자가 Wi-Fi 스니핑 → DNS 쿼리 도청

DoT (DNS over TLS, RFC 7858):
  포트: 853 (TCP)
  DNS 쿼리를 TLS 터널로 암호화
  
  클라이언트 → TLS 연결 → DoT 서버 → DNS 응답
  
  장점: 표준 TLS → 강력한 암호화, 인증
  단점: 포트 853 → 방화벽에서 식별/차단 가능
        TLS 핸드쉐이크 오버헤드
  
  지원: Android 9+ (Private DNS), systemd-resolved

DoH (DNS over HTTPS, RFC 8484):
  포트: 443 (HTTPS)
  DNS 쿼리를 HTTP/2 또는 HTTP/3로 전송
  
  GET https://dns.google/resolve?name=example.com&type=A
  POST https://cloudflare-dns.com/dns-query (바이너리)
  
  장점: HTTPS 포트 → 방화벽 통과 쉬움
        HTTP/2 멀티플렉싱으로 효율적
        일반 HTTPS 트래픽과 구분 불가
  단점: 기업 네트워크의 DNS 필터링 우회 가능 → 보안 우려
        중앙화 우려 (대형 DoH 제공자 집중)
  
  지원: Firefox, Chrome (기본), iOS

DoQ (DNS over QUIC, RFC 9250):
  포트: 853 (UDP/QUIC)
  QUIC 기반 → 더 빠른 연결 (0-RTT)
  아직 광범위하게 지원되지 않음

비교:
┌─────────────────────────────────────────────────────────────────────┐
│  방식     │  포트     │  방화벽 탐지   │  속도   │  지원 범위                │
├─────────────────────────────────────────────────────────────────────┤
│  UDP DNS │  53 UDP  │  쉬움        │  빠름   │  전통적                  │
│  DoT     │  853 TCP │  중간        │  보통   │  Android, Linux        │
│  DoH     │  443 TCP │  어려움      │  보통   │  브라우저, iOS            │
│  DoQ     │  853 UDP │  중간        │  빠름   │  실험적                  │
└─────────────────────────────────────────────────────────────────────┘

기업 환경의 DoH 딜레마:
  직원 PC에서 DoH 사용 → 기업 DNS 필터링 우회
  → 악성 사이트 접근 차단 불가
  → Microsoft, 기업 환경에서 DoH 제한 권장
  → 해결: 기업 전용 DoH 서버 운영 + DoH 강제 적용
```

### 4. DNS 기반 로드밸런싱과 한계

```
DNS 라운드로빈:
  example.com  A  10.0.0.1
  example.com  A  10.0.0.2
  example.com  A  10.0.0.3
  
  Authoritative NS: 요청마다 다른 순서로 반환
  Resolver: 첫 번째 IP를 TTL 동안 캐시
  
  문제 1 — Resolver 단위 분산:
    Resolver A(서울)→ 10.0.0.1 캐시 → 서울 사용자 전체가 10.0.0.1
    Resolver B(부산)→ 10.0.0.2 캐시 → 부산 사용자 전체가 10.0.0.2
    → 서울 사용자 100만, 부산 사용자 10만 → 불균등!
  
  문제 2 — 헬스 체크 없음:
    10.0.0.1 서버 다운 → DNS는 계속 10.0.0.1 반환
    클라이언트: 접속 실패 → 다음 IP로 재시도 (있으면)
    → L4 LB의 헬스 체크 필수

GeoDNS 동작:

  Route53 Latency Based Routing:
    Resolver IP → AWS 지역 판단
    → 가장 가까운 지역의 리소스 IP 반환
    
    예:
    Resolver 서울(KT): example.com → ap-northeast-2 서버 IP
    Resolver 런던: example.com → eu-west-2 서버 IP
  
  EDNS Client Subnet (ECS):
    Resolver가 클라이언트 IP의 /24 또는 /56 prefix를 Authoritative NS에 전달
    → GeoDNS가 Resolver IP가 아닌 실제 클라이언트 위치 기반 응답
    
    단점:
    클라이언트 IP 일부 노출 → 프라이버시 이슈
    DoH/DoT 사용 시 ECS 비활성화 (프라이버시 모드)
    1.1.1.1(Cloudflare): ECS 기본 비활성화 (프라이버시 우선)
    8.8.8.8(Google): ECS 활성화 (정확한 GeoDNS)

DNS 기반 로드밸런싱 한계 정리:
  ① 불균등 분산 (Resolver별 캐시)
  ② 헬스 체크 없음
  ③ 연결 수 기반 분산 불가
  ④ TTL 동안 장애 서버로 계속 연결
  ⑤ 무거운 클라이언트(다수 요청)에 유리 (Resolver 공유)
  
  DNS의 역할: 지역별 L4 LB로 트래픽 안내
  L4/L7 LB의 역할: 실제 서버 부하 분산
  → 두 계층 조합이 표준
```

### 5. K8s CoreDNS — 컨테이너 서비스 디스커버리

```
CoreDNS 아키텍처:
  K8s Control Plane에서 실행
  kube-dns Service → CoreDNS Pod들
  모든 Pod의 /etc/resolv.conf → CoreDNS로 설정
  
  CoreDNS 플러그인 체인:
    kubernetes → K8s Service/Endpoint 조회
    forward → 외부 DNS 서버로 전달
    cache → 응답 캐시
    health → 헬스 체크
    log → 쿼리 로깅

K8s DNS 이름 규칙:
  Service:
    <service>.<namespace>.svc.cluster.local
  
  Headless Service의 Pod:
    <pod-name>.<service>.<namespace>.svc.cluster.local
  
  StatefulSet Pod:
    web-0.my-service.default.svc.cluster.local
    web-1.my-service.default.svc.cluster.local
  
  ExternalName Service:
    my-external.default.svc.cluster.local → CNAME → external-host.com

CoreDNS 설정 (Corefile):
  .:53 {
      errors
      health {
          lameduck 5s
      }
      ready
      kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
      }
      prometheus :9153
      forward . /etc/resolv.conf {
          max_concurrent 1000
      }
      cache 30         ← 30초 캐시
      loop
      reload           ← Corefile 변경 감지
      loadbalance      ← 복수 A 레코드 순서 랜덤화
  }

DNS 기반 서비스 디스커버리 흐름:
  1. Pod A: HTTP GET http://my-service/api/data
  2. OS: DNS 조회 my-service.default.svc.cluster.local
  3. CoreDNS: kube-apiserver에서 Service IP 조회
     → 10.96.100.1 (ClusterIP)
  4. kube-proxy (iptables/IPVS):
     10.96.100.1:80 → Pod B:8080 또는 Pod C:8080 (부하분산)
  5. Pod A가 실제 Pod B 또는 C에 연결
  
  → DNS는 ClusterIP 하나만 반환
  → 실제 부하 분산은 kube-proxy가 담당
```

---

## 💻 실전 실험

### 실험 1: DNSSEC 검증 확인

```bash
# DNSSEC 지원 도메인 확인 (+dnssec 옵션)
dig +dnssec example.com @8.8.8.8

# AD 플래그 확인 (Authenticated Data = DNSSEC 검증됨)
dig example.com @8.8.8.8 | grep "flags:"
# flags: qr rd ra ad  ← ad = DNSSEC 검증됨

# DNSSEC 미지원 도메인
dig example.com @8.8.8.8 | grep "flags:"
# flags: qr rd ra  ← ad 없음 = DNSSEC 없음

# DNSKEY 레코드 조회 (DNSSEC 키 확인)
dig DNSKEY iana.org @8.8.8.8

# DS 레코드 조회 (신뢰 앵커 체인)
dig DS iana.org @8.8.8.8

# DNSSEC 검증 실패 시뮬레이션
# dnssec-failed.org는 의도적으로 DNSSEC 검증 실패 도메인
dig dnssec-failed.org @8.8.8.8
# SERVFAIL → 검증 실패로 응답 거부

# DNSSEC 우회 (검증 없이 조회)
dig dnssec-failed.org @8.8.8.8 +cd  # +cd = DNSSEC Check Disabled
# 정상 응답 (검증 우회 확인)
```

### 실험 2: DoH 직접 테스트

```bash
# DoH (DNS over HTTPS) 직접 쿼리
# Google DoH
curl -s "https://dns.google/resolve?name=example.com&type=A" | python3 -m json.tool

# Cloudflare DoH
curl -s "https://cloudflare-dns.com/dns-query?name=example.com&type=A" \
     -H "Accept: application/dns-json" | python3 -m json.tool

# 바이너리 형식 (RFC 8484)
DNS_QUERY=$(echo -n "\x00\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00\x07example\x03com\x00\x00\x01\x00\x01")
curl -s -X POST "https://cloudflare-dns.com/dns-query" \
     -H "Content-Type: application/dns-message" \
     --data-binary "$DNS_QUERY" | xxd | head

# systemd-resolved에서 DoT 설정
# /etc/systemd/resolved.conf
# DNS=1.1.1.1
# DNSOverTLS=yes
# systemctl restart systemd-resolved
```

### 실험 3: DNS 라운드로빈 불균등 관찰

```bash
# 여러 A 레코드를 가진 도메인 조회 (순서 변화 확인)
for i in $(seq 1 5); do
  dig google.com +short | head -3
  echo "---"
done
# 각 조회마다 IP 순서가 다름 확인

# 특정 Resolver에서의 캐시 고정 확인
dig @8.8.8.8 google.com +short
# 같은 IP가 계속 반환됨 (Resolver 캐시)

dig @1.1.1.1 google.com +short
# 다른 IP가 반환될 수 있음 (다른 Resolver)

# TTL별 캐시 영향
dig @8.8.8.8 google.com | grep "google.com.*A" | awk '{print $2}'
# 낮은 TTL → 자주 재조회 → 더 균등한 분산 가능성
```

### 실험 4: K8s CoreDNS 동작 확인

```bash
# K8s 클러스터 내부에서 실행
kubectl run dns-test --image=busybox --rm -it -- sh

# 서비스 DNS 조회
nslookup kubernetes.default.svc.cluster.local
# Server: 10.96.0.10 (CoreDNS)
# Address: 10.96.1.1 (ClusterIP)

# Headless Service SRV 조회
dig SRV _http._tcp.my-headless-service.default.svc.cluster.local

# CoreDNS 메트릭 확인
kubectl exec -n kube-system deployment/coredns -- \
  wget -q -O- localhost:9153/metrics | grep coredns_cache

# CoreDNS 로그 (쿼리 로깅 활성화 시)
kubectl logs -n kube-system deployment/coredns
```

---

## 📊 성능/비용 비교

```
DNS 보안 방식 성능 비교:

┌─────────────────────────────────────────────────────────────────────┐
│  방식           │  쿼리 지연   │  암호화   │  인증  │  구현 난이도           │
├─────────────────────────────────────────────────────────────────────┤
│  UDP DNS       │  1~10ms    │  없음    │  없음  │  없음 (기본)           │
│  DNSSEC        │  +5~20ms   │  없음    │  있음  │  높음                 │
│  DoT           │  +TLS RTT  │  있음    │  있음  │  중간                 │
│  DoH           │  +TLS RTT  │  있음    │  있음  │  낮음 (브라우저)        │
│  DoH+DNSSEC    │  +TLS+sig  │  있음    │  있음  │  높음                │
└─────────────────────────────────────────────────────────────────────┘

GeoDNS vs Anycast:
  GeoDNS:
    Resolver IP 기반 → 지역 특정 IP 반환
    ECS 있으면 클라이언트 IP 기반 → 더 정확
    설정: Route53 Latency/Geolocation, Cloudflare
    한계: Resolver 위치 ≠ 클라이언트 위치
  
  Anycast:
    같은 IP를 여러 PoP에서 광고
    BGP 라우팅으로 최적 PoP으로 자동 라우팅
    클라이언트 위치 관계없이 항상 최적 서버
    Cloudflare, AWS Global Accelerator 사용
    → DNS 변경 없음, IP 하나로 전 세계 커버
    DNS + L4 LB보다 더 빠름 (네트워크 레벨 최적화)
```

---

## ⚖️ 트레이드오프

```
DNSSEC 도입 여부:

도입 장점:
  DNS Cache Poisoning 원천 차단
  MITM 공격에서 DNS 위조 방어

도입 비용:
  키 관리 (ZSK/KSK 주기적 교체)
  키 롤오버 절차 복잡 (DS 레코드 업데이트 필요)
  응답 크기 증가 (서명 데이터 추가)
  설정 오류 시 도메인 완전 접근 불가
  → 많은 대형 사이트가 DNSSEC 미사용하는 이유

DoH/DoT 도입:
  개인 환경: DoH 강력 권장 (프라이버시)
  기업 환경: DoH가 보안 필터링 우회 → 관리형 DoH 또는 DoT 사용
  ISP: DoH 확산으로 DNS 기반 광고/필터링 수익 감소

DNS 라운드로빈 vs 전용 LB:
  DNS 라운드로빈:
    설정 단순, 추가 인프라 없음
    헬스 체크 없음, 불균등 분산
    소규모 내부 서비스에 적합
  
  전용 LB (ALB, Nginx):
    헬스 체크, 정확한 가중치 분산
    추가 인프라 비용 필요
    중요 프로덕션 서비스에 필수
```

---

## 📌 핵심 정리

```
DNS 보안과 고급 핵심 요약:

DNS Cache Poisoning:
  UDP 53 평문 + 짧은 트랜잭션 ID → 가짜 응답 삽입 가능
  방어: DNSSEC (서명 검증), 소스 포트 랜덤화

DNSSEC:
  Authoritative NS가 레코드에 디지털 서명 (RRSIG)
  Root → TLD → Auth NS로 이어지는 신뢰 체인
  AD 플래그로 검증 성공 확인
  dig +dnssec, dig @8.8.8.8 → AD 플래그 확인

DoH/DoT:
  DNS 쿼리 암호화 → ISP/공격자가 쿼리 내용 못 봄
  DoT: 포트 853, TLS → 방화벽에서 식별 가능
  DoH: 포트 443, HTTPS → 일반 트래픽과 구분 어려움
  한계: SNI, IP 주소 자체는 여전히 노출

DNS 로드밸런싱:
  라운드로빈: Resolver 단위 분산 (불균등)
  GeoDNS: Resolver IP 기반 지역 판단 (ECS로 개선)
  Anycast: BGP 라우팅으로 최적 PoP 자동 선택 (더 정확)
  DNS 로드밸런싱의 역할: 지역별 L4/L7 LB로 안내

K8s CoreDNS:
  cluster.local 존 관리 → Service/Pod DNS 이름
  service.namespace.svc.cluster.local → ClusterIP
  Headless: Pod별 A/SRV 레코드
  kube-proxy가 실제 부하 분산 담당

진단:
  dig +dnssec: DNSSEC 검증 확인 (AD 플래그)
  curl https://dns.google/resolve?name=: DoH 테스트
  dig @다른resolver: 지역별 응답 비교
```

---

## 🤔 생각해볼 문제

**Q1.** DNSSEC이 적용된 도메인의 키(KSK)를 교체할 때 왜 복잡한 절차가 필요한가?

<details>
<summary>해설 보기</summary>

**KSK 롤오버가 복잡한 이유:**

KSK(Key Signing Key)의 해시는 상위 도메인의 DS 레코드에 등록됩니다. 이 DS 레코드가 신뢰 체인의 링크 역할을 합니다.

**문제:** DS 레코드를 업데이트하는 것은 상위 도메인(레지스트라)을 거쳐야 합니다. 이 작업에는 시간이 걸립니다.

**올바른 절차 (Double-DS 방식):**

```
1단계: 새 KSK 생성, 기존 KSK 유지 (둘 다 DNSKEY에 게시)
2단계: 새 KSK의 DS를 레지스트라에 등록 (상위 NS에 전파 대기)
3단계: Resolver들이 새 DS로 업데이트될 때까지 대기 (DS TTL만큼)
4단계: 구 KSK 제거, 신 KSK만 유지
5단계: 구 DS 레지스트라에서 제거
```

**왜 이렇게 복잡한가:**
- KSK를 즉시 교체하면 일부 Resolver는 여전히 구 DS로 검증 시도 → 서명 불일치 → SERVFAIL
- 사이트가 전혀 접근 불가능해짐
- DNS TTL이 길면 더 오래 기다려야 함

**실제 사례:** 2019년 ICANN의 Root Zone KSK 롤오버는 수년간의 준비와 단계적 전환이 필요했습니다.

**관리 도구:** Route53은 KSK 롤오버를 자동화해줍니다. 직접 운영 시 BIND나 PowerDNS의 자동 롤오버 기능 사용.

</details>

---

**Q2.** Cloudflare 1.1.1.1은 DoH 쿼리에서 EDNS Client Subnet을 기본으로 비활성화한다. 이로 인해 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**ECS 비활성화의 효과:**

**긍정적 측면 (프라이버시):**
- Authoritative NS가 클라이언트의 실제 IP 대역을 알 수 없음
- 사용자 위치 정보 보호
- GDPR 등 개인정보 보호 규정 준수

**부정적 측면 (성능):**
- GeoDNS가 클라이언트 위치 대신 Cloudflare 데이터센터 위치를 기반으로 응답
- 예: 한국 사용자가 1.1.1.1을 사용 → Cloudflare Singapore PoP → 도메인이 Singapore용 서버 IP 반환 → 한국 사용자가 Singapore 서버에 접속 → 지연 증가

**실제 영향:**
- Netflix, YouTube 같은 CDN 서비스: GeoDNS로 최적 서버 선택 → ECS 없으면 최적이 아닌 서버로 안내될 수 있음
- AWS CloudFront, 네이버 CDN 등도 동일

**Cloudflare의 입장:**
"우리 네트워크는 전 세계에 퍼져있어서 Cloudflare PoP 위치가 이미 클라이언트 근처" → ECS 없어도 큰 차이 없다고 주장

**Google 8.8.8.8:** ECS 활성화 → GeoDNS 정확도 높지만 클라이언트 IP 일부 노출

**선택:**
- 프라이버시 우선: 1.1.1.1 (ECS 비활성화)
- 성능 우선: 8.8.8.8 (ECS 활성화)
- 또는 ISP DNS (지역적으로 가장 정확)

</details>

---

**Q3.** K8s에서 CoreDNS가 다운되면 클러스터 전체에 어떤 일이 발생하는가? 어떻게 내결함성을 확보하는가?

<details>
<summary>해설 보기</summary>

**CoreDNS 다운 시 영향:**

1. **새 DNS 조회 실패:** 도메인 기반 서비스 연결 불가
2. **새 Pod 시작 불가:** K8s 내부 통신 (API 서버 조회 등) 실패 가능
3. **기존 연결:** 이미 IP를 알고 있는 연결은 유지됨 (TCP 레벨)
4. **OS 캐시:** 짧은 시간 동안 캐시된 DNS 응답으로 동작 가능

**내결함성 확보 방법:**

1. **CoreDNS 복수 레플리카:**
```yaml
spec:
  replicas: 3  # 최소 2개 이상
```
HPA(Horizontal Pod Autoscaler)와 연동하여 부하에 따라 스케일 아웃.

2. **PodAntiAffinity로 분산 배치:**
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          k8s-app: kube-dns
      topologyKey: kubernetes.io/hostname
```
같은 노드에 CoreDNS Pod 몰리지 않도록.

3. **Node Local DNS Cache:**
각 노드에 DNS 캐시를 실행 → CoreDNS 완전 다운되어도 캐시된 응답 제공:
```yaml
# nodelocaldns DaemonSet
# 각 노드의 DNS 캐시가 CoreDNS에 연결
# CoreDNS 다운 → 노드 캐시에서 응답 (TTL 동안)
```

4. **Critical 우선순위:**
```yaml
priorityClassName: system-cluster-critical
```
CoreDNS가 리소스 부족 시 마지막에 Evict되도록.

5. **모니터링:**
```yaml
# Prometheus 알람
- alert: CoreDNSDown
  expr: absent(up{job="coredns"})
  for: 5m
```

</details>

---

<div align="center">

**[⬅️ 이전: DNS 캐싱과 전파](./03-dns-caching-and-propagation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — 로드밸런싱과 프록시 ➡️](../load-balancing-and-proxy/01-l4-vs-l7-load-balancer.md)**

</div>
