# Docker 네트워킹 — bridge, host, overlay 모드와 veth

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Docker bridge 네트워크에서 veth 페어가 어떻게 컨테이너를 격리하는가?
- iptables MASQUERADE 규칙으로 컨테이너 트래픽이 어떻게 NAT되는가?
- `docker run --network host`가 bridge와 어떻게 다르고 언제 쓰는가?
- overlay 네트워크가 VXLAN으로 멀티 호스트 컨테이너를 연결하는 원리는?
- `docker network inspect`와 `ip link show`로 무엇을 확인할 수 있는가?
- 컨테이너 간 DNS 기반 통신은 어떻게 이루어지는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"컨테이너 A에서 컨테이너 B로 ping이 안 간다":
  같은 docker-compose 파일에 있어도:
    network: 설정 없으면 → 같은 default 네트워크 O
    다른 docker-compose 파일 → 다른 네트워크 → 통신 불가
  진단: docker network inspect [network명]
        → Containers 섹션에서 어느 컨테이너가 연결됐는지 확인

"호스트 포트와 컨테이너 포트 매핑이 헷갈린다":
  docker run -p 8080:3000 my-app
  → 호스트 8080 → 컨테이너 3000
  NAT: iptables가 호스트 8080으로 들어온 패킷을 컨테이너 IP:3000으로 변환
  
  curl localhost:8080 → iptables → 172.17.0.2:3000

"Kubernetes에서 컨테이너 간 포트가 다른 호스트에 있을 때":
  Docker overlay 네트워크 (Swarm) 또는 K8s CNI:
  호스트 A의 컨테이너 → VXLAN 터널 → 호스트 B의 컨테이너
  → 물리 네트워크 위에 가상 L2 네트워크
```

---

## 😱 흔한 실수

```
Before — Docker 네트워킹을 모를 때:

실수 1: 컨테이너 내부에서 localhost로 다른 컨테이너 접근
  컨테이너 A에서: curl http://localhost:3000 → 컨테이너 A 자신
  → 다른 컨테이너의 localhost와 다름 (네트워크 네임스페이스 분리)
  → 컨테이너 이름으로 접근: curl http://db:5432

실수 2: --network host에서 포트 충돌
  docker run --network host myapp  # 포트 3000 사용
  호스트에서 이미 3000 포트 사용 중 → 충돌
  --network bridge: 컨테이너 내부에서만 포트 사용 → 충돌 없음

실수 3: docker-compose 파일 간 네트워크 격리 무시
  서비스 A (compose-a.yml): my-network
  서비스 B (compose-b.yml): my-network  ← 이름 같아도 다른 네트워크!
  → external 네트워크로 공유:
    networks:
      my-network:
        external: true

실수 4: overlay 네트워크 없이 멀티 호스트 컨테이너 통신 시도
  호스트 A의 컨테이너 → 호스트 B의 컨테이너
  bridge 네트워크: 같은 호스트 내에서만 가능
  → Docker Swarm overlay 또는 Kubernetes CNI 필요
```

---

## ✨ 올바른 접근

```
After — Docker 네트워킹을 알면:

Docker Compose 멀티 서비스 통신:
  version: '3.8'
  services:
    api:
      image: my-api
      networks:
        - backend
        - frontend
    db:
      image: postgres
      networks:
        - backend  # api만 접근 가능, frontend는 불가
    nginx:
      image: nginx
      networks:
        - frontend
      ports:
        - "80:80"
  networks:
    backend:
    frontend:

  api 컨테이너: curl http://db:5432 → 정상 (같은 backend 네트워크)
  nginx: db에 직접 접근 불가 (다른 네트워크)

네트워크 진단 워크플로:
  # 1. 컨테이너 네트워크 확인
  docker inspect [container] | jq '.[0].NetworkSettings.Networks'
  
  # 2. 네트워크 내 컨테이너 목록
  docker network inspect [network] | jq '.[0].Containers'
  
  # 3. veth 페어 확인
  ip link show type veth
  
  # 4. 컨테이너 내부에서 DNS 확인
  docker exec [container] nslookup other-service
  
  # 5. iptables NAT 규칙 확인
  sudo iptables -t nat -L -n -v | grep DOCKER
```

---

## 🔬 내부 동작 원리

### 1. Linux 네트워크 네임스페이스 — 격리의 기반

```
Linux 네트워크 네임스페이스:
  각 네임스페이스: 독립된 네트워크 스택
    - 네트워크 인터페이스 (eth0, lo)
    - 라우팅 테이블
    - iptables 규칙
    - 소켓

Docker 컨테이너 = 독립된 네트워크 네임스페이스:
  호스트 네임스페이스: eth0 (192.168.1.10), docker0 (172.17.0.1)
  컨테이너 1 네임스페이스: eth0 (172.17.0.2), lo
  컨테이너 2 네임스페이스: eth0 (172.17.0.3), lo
  
  컨테이너가 "localhost"라고 하면:
  → 컨테이너 자신의 네트워크 네임스페이스 내 lo 인터페이스
  → 호스트나 다른 컨테이너의 localhost와 완전히 다름

네임스페이스 직접 확인:
  컨테이너 PID 찾기: docker inspect [id] | grep Pid
  그 PID의 네임스페이스: /proc/[PID]/ns/net
  nsenter로 진입: nsenter -t [PID] -n ip addr
```

### 2. veth 페어 — 컨테이너와 bridge 연결

```
veth (Virtual Ethernet) 페어:
  항상 쌍으로 존재하는 가상 인터페이스
  한쪽에 들어온 패킷이 다른 쪽으로 나옴
  (마치 파이프처럼)

Docker bridge 네트워크 구성:

호스트:
┌──────────────────────────────────────────────────────────┐
│  docker0 (bridge, 172.17.0.1)                            │
│    │                                                     │
│  veth0a1b2c ←───────────┐  veth0d3e4f ←───────┐          │
└─────────────────────────┼─────────────────────┼──────────┘
                          │                     │
컨테이너 1 네임스페이스:        │     컨테이너 2:       │
  eth0 ───────────────────┘       eth0 ─────────┘
  172.17.0.2                     172.17.0.3

동작 방식:
  컨테이너 1이 컨테이너 2로 패킷 전송:
  컨테이너1 eth0 → veth0a1b2c → docker0(bridge) → veth0d3e4f → 컨테이너2 eth0

컨테이너 생성 시 Docker가 하는 일:
  1. 새 네트워크 네임스페이스 생성
  2. veth 페어 생성 (vethXXX, vethYYY)
  3. vethXXX → docker0 브리지에 연결
  4. vethYYY → 새 네임스페이스로 이동, eth0으로 rename
  5. eth0에 IP 할당 (DHCP 또는 고정)
  6. 기본 라우트 설정 (172.17.0.1이 게이트웨이)

확인:
  호스트에서: ip link show type veth
  # vethXXX@if7: <BROADCAST,MULTICAST,UP,LOWER_UP>
  # 컨테이너 내부 인터페이스 번호(@if7)로 매핑 가능
  
  컨테이너 내부: ip link show eth0
  # eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP>
  # @if15 = 호스트의 vethXXX 번호
```

### 3. iptables NAT — 컨테이너 외부 통신

```
컨테이너 → 인터넷 (SNAT/MASQUERADE):

컨테이너 (172.17.0.2)가 외부 서버(1.2.3.4)에 요청:
  패킷: src=172.17.0.2, dst=1.2.3.4

호스트 iptables POSTROUTING:
  -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
  → 출발지가 172.17.0.0/16이고 docker0이 아닌 인터페이스로 나갈 때
  → 출발지 IP를 호스트 eth0 IP로 변환 (MASQUERADE)
  변환: src=172.17.0.2 → src=192.168.1.10 (호스트 IP)

응답 패킷: src=1.2.3.4, dst=192.168.1.10
  → iptables conntrack이 원래 연결 추적
  → dst=192.168.1.10 → dst=172.17.0.2로 역변환
  → 컨테이너로 전달

포트 매핑 (-p 8080:3000) 동작 (DNAT):
  호스트 iptables:
  -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:3000
  
  외부 → 호스트:8080 패킷:
  src=외부IP, dst=호스트IP:8080
  → DNAT: dst=172.17.0.2:3000으로 변환
  → 컨테이너로 전달

iptables 규칙 확인:
  sudo iptables -t nat -L -n -v --line-numbers
  
  Chain DOCKER:
  num   pkts  bytes  target   prot  opt  source     destination
  1     1234  567k   DNAT     tcp   --   0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:3000
  
  Chain POSTROUTING:
  MASQUERADE  all  --  172.17.0.0/16  !172.17.0.0/16
```

### 4. host 네트워크 모드

```
docker run --network host myapp:

  컨테이너가 독립된 네트워크 네임스페이스를 갖지 않음
  → 호스트의 eth0, lo, docker0 등 모두 공유
  → 컨테이너의 포트 = 호스트의 포트

장점:
  NAT 오버헤드 없음 (포트 매핑 없음)
  최고 성능 (네트워크 격리 없음)
  호스트 네트워크 직접 접근 가능

단점:
  포트 충돌: 컨테이너가 쓰는 포트가 호스트에 직접 바인딩
  보안 약화: 네트워크 격리 없음
  여러 동일 컨테이너 실행 어려움 (포트 중복)

사용 사례:
  네트워크 모니터링 도구 (tcpdump, ntopng)
  성능 극한 요구 (고성능 네트워크 프록시)
  호스트 네트워크 설정 컨테이너 (Cilium agent 등)

bridge vs host 비교:
  bridge: 격리 O, NAT 있음, 포트 매핑 필요
  host:   격리 X, NAT 없음, 포트 매핑 불필요

none 네트워크:
  네트워크 인터페이스 없음 (lo만 있음)
  외부와 완전 격리
  사용: 보안이 중요한 단독 처리 작업
```

### 5. overlay 네트워크 — 멀티 호스트

```
overlay 네트워크 필요성:
  Docker bridge: 같은 호스트 내 컨테이너만 연결
  멀티 호스트(Swarm, K8s):
  호스트 A의 컨테이너 ↔ 호스트 B의 컨테이너 통신 필요

VXLAN (Virtual Extensible LAN):
  L2 프레임을 UDP 패킷으로 캡슐화
  포트: 4789 (IANA 표준)
  
  컨테이너A → 컨테이너B (다른 호스트) 전송:
  
  원본 L2 프레임:
  [Ethernet][IP: src=10.0.0.2, dst=10.0.0.3]
  
  VXLAN 캡슐화 후:
  [외부 Ethernet][외부 IP][UDP:4789][VXLAN Header][원본 L2 프레임]
  외부 IP: 호스트 A → 호스트 B의 물리 IP
  
  → 물리 네트워크(인터넷)를 통과 가능
  → 수신 호스트가 캡슐화 해제 → 원본 L2 프레임 복원

Docker Swarm overlay:
  docker network create -d overlay my-overlay
  
  VTEP(VXLAN Tunnel Endpoint):
  각 호스트에 VTEP 생성
  컨테이너 패킷 → VTEP → VXLAN 캡슐화 → 물리 네트워크
  → 목적지 호스트 VTEP → 캡슐화 해제 → 컨테이너 전달

네트워크 인터페이스 구조 (overlay):
  호스트: eth0 (물리), docker_gwbridge (게이트웨이), veth*
  컨테이너: eth0 (overlay IP), eth1 (docker_gwbridge, 외부 통신용)
  
  docker network inspect overlay-network:
  {
    "Driver": "overlay",
    "IPAM": {"Subnet": "10.0.0.0/24"},
    "Options": {"encrypted": "true"}  # 선택적 암호화
  }
```

### 6. 컨테이너 DNS

```
Docker 내부 DNS:
  컨테이너 /etc/resolv.conf:
    nameserver 127.0.0.11  # Docker 내장 DNS (embedded DNS)
    search myapp_default   # 기본 검색 도메인
  
  동작:
  컨테이너에서 "db" 조회
  → 127.0.0.11 (Docker DNS)
  → "db.myapp_default" 또는 "db"로 서비스 찾기
  → Docker가 관리하는 컨테이너 IP 반환

서비스 이름 해석:
  docker run --name my-db postgres
  docker run --link my-db:db my-app  # 레거시
  
  더 나은 방법 (user-defined network):
  docker network create mynet
  docker run --network mynet --name db postgres
  docker run --network mynet myapp  # → "db"로 접근 가능

docker-compose에서 DNS:
  services:
    api:
      image: my-api
    db:
      image: postgres
  # api 컨테이너에서 "db"로 postgres에 접근 가능
  # Docker compose가 자동으로 같은 네트워크에 배치

alias 사용:
  services:
    api:
      networks:
        backend:
          aliases:
            - api.internal  # "api.internal"로도 접근 가능
```

---

## 💻 실전 실험

### 실험 1: veth 페어 구조 확인

```bash
# 컨테이너 실행
docker run -d --name test-nginx nginx

# 호스트에서 veth 목록 확인
ip link show type veth
# vethXXXXX@if5: <BROADCAST,MULTICAST,UP>

# 컨테이너의 인터페이스 번호 확인
docker exec test-nginx ip link show eth0
# eth0@if15: ← @if15가 호스트 veth의 인덱스

# 컨테이너 PID 찾기
PID=$(docker inspect test-nginx --format '{{.State.Pid}}')
echo "Container PID: $PID"

# 컨테이너 네임스페이스로 진입
sudo nsenter -t $PID -n ip addr
# lo, eth0 확인 (호스트의 다른 인터페이스 없음)

# docker0 브리지에 연결된 인터페이스 확인
bridge link show
# vethXXXXX dev docker0
```

### 실험 2: iptables NAT 규칙 확인

```bash
# 포트 매핑 컨테이너 실행
docker run -d -p 8080:80 --name test-nginx nginx

# DNAT 규칙 확인 (호스트 8080 → 컨테이너 80)
sudo iptables -t nat -L DOCKER -n -v
# DNAT tcp -- anywhere anywhere tcp dpt:8080 to:172.17.0.2:80

# MASQUERADE 규칙 확인 (컨테이너 → 외부)
sudo iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE
# MASQUERADE all -- 172.17.0.0/16 !172.17.0.0/16

# 패킷 흐름 추적 (외부 → 컨테이너)
sudo conntrack -L | grep 8080
# tcp 120 ESTABLISHED src=클라이언트IP dst=호스트IP sport=X dport=8080
#         src=172.17.0.2 dst=클라이언트IP sport=80 dport=X [ASSURED]
# → DNAT 매핑 확인

# 컨테이너 → 외부 NAT 확인
docker exec test-nginx curl -s https://httpbin.org/ip | grep origin
# "origin": "호스트IP"  ← 컨테이너 IP가 아닌 호스트 IP로 나감
```

### 실험 3: 사용자 정의 네트워크 격리 확인

```bash
# 두 개의 분리된 네트워크
docker network create net-a
docker network create net-b

# 각 네트워크에 컨테이너 배치
docker run -d --network net-a --name svc-a alpine sleep 3600
docker run -d --network net-b --name svc-b alpine sleep 3600
docker run -d --network net-a --network net-b --name svc-ab alpine sleep 3600

# net-a ↔ net-b 격리 확인
docker exec svc-a ping -c 1 svc-b 2>&1
# ping: svc-b: Name or service not known (격리됨!)

docker exec svc-a ping -c 1 svc-ab 2>&1
# PING svc-ab: ... 1 packets (같은 net-a → 통신 O)

# svc-ab는 두 네트워크 브리지 역할
docker exec svc-ab ping -c 1 svc-b
# PING svc-b: ... (net-b에도 연결 → 통신 O)
```

### 실험 4: overlay 네트워크 VXLAN 패킷 확인

```bash
# Docker Swarm 초기화 (단일 노드 테스트)
docker swarm init

# overlay 네트워크 생성
docker network create -d overlay --attachable my-overlay

# 두 컨테이너 실행 (같은 overlay 네트워크)
docker run -d --network my-overlay --name svc1 alpine sleep 3600
docker run -d --network my-overlay --name svc2 alpine sleep 3600

# overlay 네트워크 정보 확인
docker network inspect my-overlay | python3 -m json.tool

# VXLAN 인터페이스 확인
ip link show type vxlan
# vxlan0 또는 vx-XXXX

# UDP 4789 포트 확인 (VXLAN 포트)
ss -ulnp | grep 4789

# 컨테이너 간 통신 (overlay 통해)
docker exec svc1 ping -c 3 svc2
# PING svc2 (10.0.0.X): 64 bytes from svc2 (overlay IP)

docker swarm leave --force
```

---

## 📊 성능/비용 비교

```
Docker 네트워크 모드별 성능:

┌────────────────────────────────────────────────────────────────────┐
│  모드       │  지연   │  처리량   │  격리  │  사용 사례                   │
├────────────────────────────────────────────────────────────────────┤
│  host      │  최저   │  최고    │  없음  │  네트워크 도구, 고성능          │
│  bridge    │  낮음   │  높음    │  있음  │  일반 컨테이너 (기본)           │
│  overlay   │  보통   │  보통    │  있음  │  멀티 호스트                  │
│  none      │  해당없음│ 해당없음  │  완전  │  완전 격리 작업                │
└────────────────────────────────────────────────────────────────────┘

bridge vs host 지연 차이:
  bridge (NAT 포함): +0.1~0.5ms (iptables NAT 처리)
  host:              +0ms (직접 네트워크 스택)
  
  고빈도 요청(초당 10만+)에서 차이 체감
  일반 웹 서비스: bridge로 충분

overlay VXLAN 오버헤드:
  VXLAN 헤더: 50 bytes 추가
  MTU 고려: 기본 1500 → 1450 (VXLAN 오버헤드)
  UDP 캡슐화: CPU 추가 소비
  → 동일 호스트 bridge 대비 약 10~20% 성능 감소
  → 멀티 호스트 통신 필수 기능이므로 트레이드오프 감수
```

---

## ⚖️ 트레이드오프

```
네트워크 모드 선택:

bridge (기본):
  장점: 격리, 포트 관리 편리, DNS 자동 해석
  단점: NAT 오버헤드, iptables 규칙 복잡도
  언제: 대부분의 웹 서비스 컨테이너

host:
  장점: 최고 성능, 설정 단순
  단점: 격리 없음, 포트 충돌 위험
  언제: 네트워크 집약적 도구, 성능 극한 필요

overlay:
  장점: 멀티 호스트 통신, 암호화 지원
  단점: VXLAN 오버헤드, 복잡도
  언제: Docker Swarm, 멀티 노드 서비스

사용자 정의 bridge vs 기본 bridge:
  기본 docker0 bridge:
    모든 컨테이너가 연결됨 → 격리 없음
    --link로만 DNS 사용 가능 (레거시)
  
  사용자 정의 bridge (docker network create):
    자동 DNS (컨테이너 이름으로 접근)
    격리: 다른 네트워크와 분리
    → 프로덕션에서는 사용자 정의 네트워크 권장

K8s와의 비교:
  Docker bridge: 단일 호스트 내
  K8s CNI: 전체 클러스터 플랫 네트워크
  K8s Pod: 별도 NAT 없이 직접 통신 (다른 호스트도)
  → Docker overlay와 유사하지만 K8s가 더 정교
```

---

## 📌 핵심 정리

```
Docker 네트워킹 핵심 요약:

네트워크 격리 원리:
  Linux 네트워크 네임스페이스로 격리
  컨테이너마다 독립된 네트워크 스택
  "localhost"는 컨테이너마다 다름

veth 페어:
  가상 이더넷 케이블 양 끝
  한쪽(컨테이너 eth0) ↔ 다른 쪽(호스트 docker0)
  bridge를 통해 컨테이너 간 L2 통신

iptables NAT:
  컨테이너 → 외부: MASQUERADE (SNAT)
  외부 → 컨테이너: DNAT (-p 포트 매핑)
  conntrack으로 연결 추적

네트워크 모드:
  bridge: 기본, 격리 + NAT (대부분의 경우)
  host:   격리 없음, 최고 성능
  overlay: 멀티 호스트, VXLAN 캡슐화
  none:   완전 격리

DNS:
  Docker 내장 DNS (127.0.0.11)
  사용자 정의 네트워크: 컨테이너 이름으로 자동 해석
  docker-compose: 서비스 이름으로 자동 접근

진단:
  docker network inspect: 네트워크 구성 확인
  ip link show type veth: veth 페어 확인
  iptables -t nat -L: NAT 규칙 확인
  nsenter: 컨테이너 네임스페이스 진입
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 호스트에서 두 개의 docker-compose 파일을 실행하면 서비스 이름이 같아도 통신이 안 된다. 왜 그런가?

<details>
<summary>해설 보기</summary>

**원인:** docker-compose는 각 프로젝트(파일)마다 독립된 사용자 정의 네트워크를 생성합니다. 네트워크 이름에 프로젝트 이름이 prefix로 붙습니다.

```
compose-a.yml → 네트워크: compose-a_default
compose-b.yml → 네트워크: compose-b_default
```

두 네트워크는 완전히 분리되어 있으므로 서비스 이름이 같아도 통신 불가.

**해결 방법:**

1. **외부 네트워크 공유:**
```bash
docker network create shared-network

# compose-a.yml
networks:
  shared-network:
    external: true

# compose-b.yml
networks:
  shared-network:
    external: true
```

2. **같은 프로젝트 이름 사용:**
```bash
docker-compose -p myproject -f compose-a.yml up -d
docker-compose -p myproject -f compose-b.yml up -d
# → myproject_default 네트워크 공유
```

3. **하나의 compose 파일로 통합 (권장):**
서비스가 관련 있다면 하나의 파일로 관리.

</details>

---

**Q2.** `docker run -p 0.0.0.0:8080:3000`과 `docker run -p 127.0.0.1:8080:3000`의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

**0.0.0.0:8080 (기본):**
- 모든 인터페이스에서 8080 포트 바인딩
- 외부에서 호스트IP:8080으로 접근 가능
- iptables 규칙: 모든 출발지에서 8080으로 들어오는 패킷을 컨테이너로 DNAT

**127.0.0.1:8080:**
- 루프백(lo) 인터페이스에만 바인딩
- 외부에서 접근 불가, 호스트 내부에서만 접근 가능
- 보안 강화: 외부 노출 없이 호스트의 다른 프로세스만 접근

```bash
# iptables 규칙 차이 확인
# 0.0.0.0: 모든 출발지 허용
# 127.0.0.1: 로컬 요청만

sudo iptables -t nat -L DOCKER -n | grep 8080
```

**실무 사용:**
```bash
# Prometheus, Grafana 등 내부 모니터링 도구 (외부 노출 불필요)
docker run -p 127.0.0.1:9090:9090 prometheus

# 웹 서버 (외부 접근 필요)
docker run -p 0.0.0.0:80:80 nginx
```

</details>

---

**Q3.** Docker 컨테이너가 MTU(Maximum Transmission Unit) 문제를 일으키는 경우는 언제이고 어떻게 진단하는가?

<details>
<summary>해설 보기</summary>

**MTU 불일치 문제 발생 상황:**

1. **overlay 네트워크 (VXLAN):** 물리 네트워크 MTU 1500에서 VXLAN 헤더 50 bytes 추가 → 실제 페이로드 MTU=1450. 컨테이너가 1500 bytes 패킷 전송 → 분할(Fragmentation) 필요 → 성능 저하 또는 패킷 드롭.

2. **VPN + Docker:** VPN 터널이 MTU를 낮춤 → Docker 컨테이너가 큰 패킷 전송 → 블랙홀 (ICMP Fragmentation Needed 차단 시).

3. **클라우드 환경:** AWS, GCP의 VPC MTU가 기본보다 낮은 경우.

**진단:**
```bash
# 컨테이너 MTU 확인
docker exec my-container ip link show eth0 | grep mtu

# 큰 패킷 ping으로 MTU 확인
docker exec my-container ping -M do -s 1400 external-host
# -M do: 분할 금지, -s: 패킷 크기

# MTU 경로 탐색 (PMTUD)
docker exec my-container tracepath external-host | grep -i mtu
```

**해결:**
```bash
# Docker 데몬 MTU 설정 (/etc/docker/daemon.json)
{
    "mtu": 1450
}

# 또는 네트워크 생성 시
docker network create --opt "com.docker.network.driver.mtu"="1450" my-net
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Kubernetes 네트워킹 ➡️](./02-kubernetes-networking.md)**

</div>
