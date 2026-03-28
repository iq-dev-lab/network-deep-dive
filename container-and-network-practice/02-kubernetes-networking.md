# Kubernetes 네트워킹 — Pod IP, kube-proxy, Service 타입

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- K8s 네트워킹 모델에서 "NAT 없이 Pod 간 통신"이 가능한 이유는?
- kube-proxy가 iptables/IPVS 규칙으로 ClusterIP를 어떻게 구현하는가?
- ClusterIP, NodePort, LoadBalancer, Ingress의 트래픽 흐름은 어떻게 다른가?
- CNI 플러그인이 없으면 K8s 네트워킹이 동작하지 않는 이유는?
- Calico, Flannel, Cilium의 구현 방식 차이는?
- Pod가 재시작되면 IP가 바뀌는데 어떻게 안정적으로 통신하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"K8s에서 서비스가 통신이 안 돼요":
  진단 체크리스트:
  1. kubectl get svc → ClusterIP 확인
  2. kubectl get ep → Endpoint(Pod IP) 확인
     Endpoints가 <none>이면 → label selector 불일치
  3. kubectl exec pod -- curl http://service-name
  4. DNS 확인: kubectl exec pod -- nslookup service-name
  5. kube-proxy 로그: kubectl logs -n kube-system -l k8s-app=kube-proxy

"LoadBalancer 서비스를 만들었는데 EXTERNAL-IP가 <pending>":
  온프레미스 K8s: Cloud LB 없음 → <pending>
  해결: MetalLB 설치 (온프레미스 LoadBalancer)
  또는 NodePort + 외부 LB 사용
  또는 Ingress Controller (Nginx Ingress)

"Pod가 많아지니 kube-proxy iptables 성능이 저하된다":
  iptables: 규칙 선형 탐색 O(n)
  100개 Service → 수천 개 iptables 규칙
  IPVS: 해시 테이블 O(1) → 대규모 클러스터 권장
  kube-proxy --proxy-mode=ipvs
```

---

## 😱 흔한 실수

```
Before — K8s 네트워킹을 모를 때:

실수 1: Pod IP로 직접 통신
  kubectl get pod -o wide → Pod IP 확인
  curl http://10.244.1.5:8080 → 동작은 하지만...
  Pod 재시작 → IP 변경 → 통신 끊김!
  → 항상 Service (DNS 이름)로 통신

실수 2: Service selector 레이블 불일치
  Service selector: app=my-api
  Pod 레이블: app=myapi (오타!)
  → kubectl get ep: Endpoints가 <none>
  → Service 있어도 어디로 보낼지 모름

실수 3: 같은 네임스페이스가 아닌 서비스에 단순 이름으로 접근
  ns=frontend에서: curl http://my-api (단순 이름)
  → CoreDNS: my-api.frontend.svc.cluster.local 탐색
  → my-api가 ns=backend에 있음 → 찾지 못함!
  → 크로스 네임스페이스: curl http://my-api.backend.svc.cluster.local

실수 4: NodePort 범위 오해
  NodePort 기본 범위: 30000~32767
  80, 443으로 NodePort 설정 시도 → 오류
  → Ingress로 80/443 처리 (NodePort 앞에 배치)
```

---

## ✨ 올바른 접근

```
After — K8s 네트워킹을 알면:

Service 유형별 올바른 선택:
  ClusterIP (기본):
    클러스터 내부 통신만
    내부 마이크로서비스 간 통신
    DB, Redis 등 내부 서비스
  
  NodePort:
    외부 접근 필요 (개발 환경)
    특정 노드 IP:NodePort로 접근
    LoadBalancer 없는 환경에서 임시 외부 노출
  
  LoadBalancer:
    프로덕션 외부 트래픽
    클라우드 환경 (AWS ELB, GCP LB)
    온프레미스: MetalLB 필요
  
  ExternalName:
    외부 서비스를 K8s DNS로 추상화
    my-db.internal → external-db.example.com

Ingress 전략:
  # 단일 Ingress로 여러 서비스 라우팅
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - host: api.example.com
      http:
        paths:
        - path: /v1/users
          pathType: Prefix
          backend:
            service:
              name: user-service
              port: number: 80
        - path: /v1/orders
          pathType: Prefix
          backend:
            service:
              name: order-service
              port: number: 80
    tls:
    - hosts: [api.example.com]
      secretName: tls-secret

네트워크 진단:
  kubectl describe svc my-service     # Endpoints 확인
  kubectl get ep my-service           # Pod IP 목록
  kubectl exec -it pod -- nslookup svc-name  # DNS 확인
  kubectl exec -it pod -- curl http://svc-name:port  # 연결 확인
```

---

## 🔬 내부 동작 원리

### 1. K8s 네트워킹 기본 요구사항

```
K8s 네트워킹 3가지 요구사항 (RFC):

1. 모든 Pod는 NAT 없이 다른 모든 Pod와 통신 가능
2. 모든 노드는 NAT 없이 모든 Pod와 통신 가능
3. Pod가 자신의 IP를 인식하는 IP == 다른 Pod가 보는 IP

이것이 Docker bridge와의 가장 큰 차이:
  Docker: 컨테이너 172.17.x.x → NAT → 호스트 IP로 변환 후 통신
  K8s: Pod 10.244.x.x → 그대로 다른 노드의 Pod와 통신

K8s Pod 네트워크 (플랫 네트워크):
  Node 1 Pod CIDR: 10.244.1.0/24
  Node 2 Pod CIDR: 10.244.2.0/24
  
  Node1의 Pod (10.244.1.5) → Node2의 Pod (10.244.2.7):
  NAT 없이 직접 통신 가능
  → CNI 플러그인이 이 라우팅을 구성

이 모델이 가능한 이유:
  각 노드에 Pod CIDR 할당 (겹치지 않음)
  CNI가 호스트 라우팅 테이블에 경로 추가
  또는 VXLAN/터널로 가상 플랫 네트워크 구성
```

### 2. kube-proxy — Service 구현체

```
Service ClusterIP의 실체:
  ClusterIP는 어떤 인터페이스에도 바인딩되지 않는 가상 IP
  kube-proxy가 이 VIP를 실제 Pod IP로 변환

kube-proxy iptables 모드:

  Service: my-svc (ClusterIP: 10.96.100.1:80)
  Endpoints: [10.244.1.5:8080, 10.244.2.7:8080]
  
  kube-proxy가 생성하는 iptables 규칙:
  
  PREROUTING (→ KUBE-SERVICES):
    -A KUBE-SERVICES -d 10.96.100.1 -p tcp --dport 80
    -j KUBE-SVC-XXXX
  
  KUBE-SVC-XXXX (로드밸런싱):
    -A KUBE-SVC-XXXX -m statistic --mode random --probability 0.50000
    -j KUBE-SEP-AAAA  (50% → Pod 1)
    -A KUBE-SVC-XXXX -j KUBE-SEP-BBBB  (나머지 → Pod 2)
  
  KUBE-SEP-AAAA (DNAT):
    -A KUBE-SEP-AAAA -j DNAT --to-destination 10.244.1.5:8080
  
  KUBE-SEP-BBBB:
    -A KUBE-SEP-BBBB -j DNAT --to-destination 10.244.2.7:8080
  
  결과:
  요청 → 10.96.100.1:80 → iptables → 10.244.1.5:8080 또는 10.244.2.7:8080

kube-proxy IPVS 모드:
  iptables의 O(n) 탐색 대신 해시 테이블 O(1)
  
  ipvsadm -ln:
  TCP  10.96.100.1:80 rr  (round-robin)
    -> 10.244.1.5:8080   Weight: 1
    -> 10.244.2.7:8080   Weight: 1
  
  대규모 클러스터 (Service 수천 개):
  iptables: 규칙 수만 개 → 새 연결마다 선형 탐색 → CPU 폭증
  IPVS: 해시 기반 → 규모 무관 빠른 조회

iptables vs IPVS 선택:
  Service < 1000개: iptables (기본)
  Service >= 1000개: IPVS 권장
  kube-proxy --proxy-mode=ipvs
```

### 3. Service 타입 트래픽 흐름

```
ClusterIP (내부 전용):
  
  Pod A → my-svc:80
  ↓ DNS: my-svc.default.svc.cluster.local → 10.96.100.1
  ↓ iptables DNAT: 10.96.100.1:80 → 10.244.1.5:8080
  Pod B (10.244.1.5:8080)

NodePort (외부 → 모든 노드):
  
  외부 클라이언트 → Node1:30080 (또는 Node2:30080)
  ↓ Node1의 iptables:
    KUBE-NODEPORTS: dport=30080 → KUBE-SVC-XXXX
    → DNAT → Pod IP:8080
  
  특징:
  어느 노드로 가도 동작 (kube-proxy가 모든 노드에 규칙 설정)
  Pod가 없는 노드로 요청이 가면 → 다른 노드의 Pod로 포워딩 (externalTrafficPolicy: Cluster)
  
  externalTrafficPolicy: Local:
    Pod가 있는 노드로만 라우팅 → 원본 클라이언트 IP 보존
    Pod 없는 노드로 요청 → 503 (트래픽을 다른 노드로 보내지 않음)

LoadBalancer (클라우드):
  
  외부 클라이언트 → AWS ELB/GCP LB
  → 노드 NodePort로 분산 (ELB 자체 분산)
  → iptables DNAT → Pod IP
  
  AWS ALB → Target Group (NodePort or Pod IP, EKS AWS VPC CNI):
  EKS: Pod IP를 직접 Target으로 (더 효율적, NAT 감소)

Ingress (L7 라우팅):
  
  외부 클라이언트 → Nginx Ingress Pod
  → Host/Path 기반 라우팅
  → 해당 Service ClusterIP → Pod IP
  
  Ingress Controller: Nginx, Traefik, AWS ALB Ingress, Kong
  각각 LoadBalancer Service로 외부 노출됨
```

### 4. CNI (Container Network Interface)

```
CNI의 역할:
  Pod 생성 시 kubelet이 CNI 플러그인 호출
  CNI가 Pod 네트워크 설정:
  - 네트워크 네임스페이스에 인터페이스 추가
  - IP 할당
  - 라우팅 규칙 설정
  - 노드 간 통신 구성

주요 CNI 플러그인:

Flannel (가장 단순):
  VXLAN 또는 host-gw 모드
  host-gw: 노드 간 직접 라우팅 (동일 L2 네트워크 필요)
  VXLAN: L3 네트워크에서도 동작 (오버레이)
  단점: 네트워크 정책(NetworkPolicy) 미지원

Calico (프로덕션 표준):
  BGP 기반 라우팅 (오버레이 없음 → 낮은 오버헤드)
  NetworkPolicy 완전 지원
  WireGuard 암호화 지원
  대규모 클러스터에서 안정적
  eBPF 모드 가능

Cilium (최신, eBPF 기반):
  iptables 대신 eBPF로 구현
  kube-proxy 완전 대체 가능 (kube-proxy-free)
  높은 성능 (eBPF: 커널 레벨, iptables보다 빠름)
  L7 정책 (HTTP 메서드/경로 기반 제어)
  Hubble: 관찰 가능성 (트래픽 시각화)
  대규모/고성능 클러스터 권장

AWS VPC CNI (EKS 기본):
  Pod에 실제 VPC IP 할당 (VXLAN 없음)
  Pod → VPC 네이티브 라우팅
  AWS 보안 그룹을 Pod 레벨에서 적용 가능
  단점: 노드당 Pod 수 제한 (ENI × IP 수)

CNI 선택 기준:
  단순/소규모: Flannel
  프로덕션 표준: Calico
  고성능/관찰가능성: Cilium
  AWS EKS: AWS VPC CNI (기본)
```

### 5. NetworkPolicy — Pod 레벨 방화벽

```
NetworkPolicy:
  K8s의 Pod 레벨 네트워크 접근 제어
  "이 Pod는 어느 Pod로부터/에게 트래픽을 허용하는가"

기본: 모든 Pod 간 통신 허용 (NetworkPolicy 없으면)

NetworkPolicy 예시:
  # db Pod는 api Pod에서만 접근 가능
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: db-allow-api
    namespace: default
  spec:
    podSelector:
      matchLabels:
        app: db          # 이 규칙이 적용될 Pod
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: api     # api Pod에서만 허용
      ports:
      - protocol: TCP
        port: 5432
    egress:
    - {}  # 모든 Egress 허용

주의:
  NetworkPolicy는 CNI 플러그인이 구현
  Flannel: NetworkPolicy 미지원!
  Calico, Cilium: NetworkPolicy 지원

Zero Trust in K8s:
  기본 Deny-All 정책 설정:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny-all
  spec:
    podSelector: {}  # 모든 Pod에 적용
    policyTypes:
    - Ingress
    - Egress
  # 아무것도 없음 = 모두 차단
  
  이후 필요한 통신만 허용하는 NetworkPolicy 추가
  → Istio mTLS와 조합하면 완전한 Zero Trust
```

---

## 💻 실전 실험

### 실험 1: Service → Endpoint → Pod 흐름 추적

```bash
# 테스트 Deployment + Service 생성
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl expose deployment nginx-test --port=80 --target-port=80

# Service 확인
kubectl get svc nginx-test
# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# nginx-test   ClusterIP   10.96.123.45    <none>        80/TCP

# Endpoint 확인 (Pod IP 목록)
kubectl get endpoints nginx-test
# NAME         ENDPOINTS                      AGE
# nginx-test   10.244.1.3:80,10.244.2.4:80

# DNS 조회
kubectl run dns-test --image=busybox --rm -it -- nslookup nginx-test.default.svc.cluster.local
# Server: 10.96.0.10 (CoreDNS)
# Name: nginx-test.default.svc.cluster.local
# Address: 10.96.123.45

# ClusterIP로 요청 → Pod로 분산 확인
kubectl run curl-test --image=curlimages/curl --rm -it -- sh
# curl http://nginx-test  # 여러 번 실행
# → 각 Pod가 응답 (kube-proxy가 라운드로빈)
```

### 실험 2: kube-proxy iptables 규칙 확인

```bash
# Service 관련 iptables 규칙 확인
sudo iptables -t nat -L KUBE-SERVICES -n -v | grep nginx-test

# 특정 Service의 DNAT 체인 확인
CHAIN=$(sudo iptables -t nat -L KUBE-SERVICES -n | grep "10.96.123.45" | awk '{print $2}')
sudo iptables -t nat -L $CHAIN -n -v
# 각 Pod로의 DNAT 규칙과 확률값 확인

# IPVS 모드 사용 시 (설정된 경우)
ipvsadm -ln | grep -A5 "10.96.123.45"

# Pod 삭제 → Endpoint 자동 제거 확인
kubectl delete pod [pod-name]
kubectl get endpoints nginx-test  # 해당 Pod IP 제거됨 확인
sudo iptables -t nat -L $CHAIN -n -v  # iptables도 자동 업데이트
```

### 실험 3: NodePort 트래픽 흐름

```bash
# NodePort Service 생성
kubectl expose deployment nginx-test --type=NodePort --port=80 --name=nginx-np

# NodePort 확인
kubectl get svc nginx-np
# PORT(S): 80:31234/TCP → 31234이 NodePort

# 노드 IP 확인
kubectl get nodes -o wide

# NodePort로 접근 (클러스터 외부에서)
curl http://[노드IP]:31234

# iptables NodePort 규칙 확인
sudo iptables -t nat -L KUBE-NODEPORTS -n -v | grep 31234

# externalTrafficPolicy 비교
kubectl patch svc nginx-np -p '{"spec":{"externalTrafficPolicy":"Local"}}'
# Local: 해당 노드의 Pod만 서비스 (클라이언트 IP 보존)
kubectl patch svc nginx-np -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
# Cluster: 모든 노드의 Pod로 라우팅 (기본)
```

### 실험 4: NetworkPolicy 적용 확인

```bash
# 기본 상태: 모든 Pod 간 통신 허용
kubectl run server --image=nginx
kubectl run client --image=curlimages/curl -- sleep 3600
kubectl exec client -- curl http://server  # 성공

# Default Deny 적용
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

kubectl exec client -- curl --max-time 3 http://server  # 타임아웃!

# server Pod만 client에서 허용
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-server
spec:
  podSelector:
    matchLabels:
      run: server
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: client
EOF

kubectl exec client -- curl http://server  # 성공!

# 정리
kubectl delete networkpolicy deny-all allow-client-to-server
kubectl delete pod server client
```

---

## 📊 성능/비용 비교

```
kube-proxy 모드별 성능:

iptables:
  규칙 수: Service당 약 5~20개 iptables 규칙
  1000 Service → 수만 개 규칙
  새 연결마다 규칙 선형 탐색 O(n)
  1000 Service 환경: 수십ms 오버헤드 가능
  CPU: 규칙 업데이트(Pod 추가/삭제) 시 비용 높음

IPVS:
  해시 테이블 기반 O(1) 조회
  규모와 무관하게 빠른 응답
  CPU: 규칙 업데이트 효율적
  추가 커널 모듈 필요 (ipvs, nf_conntrack)

Cilium (eBPF):
  kube-proxy 완전 대체
  iptables/IPVS 모두 우회
  eBPF: 커널 레벨 패킷 처리 → 가장 빠름
  Service 조회: 해시 맵 (IPVS와 유사)
  L7 정책: HTTP 레이어 필터링 가능

비교 측정 (500 Service 환경):
  iptables: 새 연결 레이턴시 +1~5ms
  IPVS:     새 연결 레이턴시 +0.1ms
  Cilium:   새 연결 레이턴시 +0.05ms

CNI 성능:
  Calico host-gw: 오버헤드 거의 없음 (직접 라우팅)
  Flannel VXLAN:  VXLAN 캡슐화 오버헤드 (~5%)
  AWS VPC CNI:    네이티브 VPC 라우팅 → 최고 성능
  Cilium eBPF:    iptables 우회 → 최고 효율
```

---

## ⚖️ 트레이드오프

```
Service 타입 선택:

ClusterIP:
  내부 통신만 필요 → 무조건 ClusterIP
  추가 설정 불필요, 보안 기본 보장

NodePort:
  빠른 외부 노출 (개발 환경)
  포트 범위 제한 (30000~32767)
  노드 IP가 바뀌면 클라이언트도 변경 필요
  → 프로덕션에서는 Ingress 사용

LoadBalancer:
  클라우드 환경의 표준
  각 Service마다 LB 생성 → 비용 증가
  많은 Service → Ingress로 통합 (하나의 LB)

Ingress:
  HTTP/HTTPS 라우팅에 최적
  하나의 LB로 다수 Service 라우팅
  TLS 종료, 인증, Rate Limiting 가능
  단점: gRPC 설정 복잡, TCP/UDP 불가 (별도 설정)

CNI 선택:
  단순함 원하면: Flannel
  NetworkPolicy 필요: Calico
  고성능 + 관찰가능성: Cilium
  AWS EKS: AWS VPC CNI (기본)
  
  CNI는 나중에 바꾸기 어려움 → 처음 선택이 중요

externalTrafficPolicy:
  Cluster (기본): 어느 노드로도 요청 가능, 클라이언트 IP 손실
  Local: 해당 노드 Pod만, 클라이언트 IP 보존, 불균등 분산 가능
  → Rate Limiting 등 IP 기반 기능 필요 시 Local 선택
```

---

## 📌 핵심 정리

```
K8s 네트워킹 핵심 요약:

K8s 네트워킹 모델:
  모든 Pod가 NAT 없이 통신 가능
  Pod CIDR: 노드별 분리, 겹치지 않음
  CNI 플러그인이 이 모델 구현

kube-proxy:
  Service ClusterIP → Pod IP 변환 담당
  iptables: 선형 탐색, 소규모 적합
  IPVS: 해시 기반, 대규모 권장
  Cilium: eBPF로 kube-proxy 대체 가능

Service 타입:
  ClusterIP: 내부 전용 (기본)
  NodePort: 노드IP:포트 외부 노출
  LoadBalancer: 클라우드 LB 연동
  Ingress: L7 라우팅 (단일 LB, 다수 Service)

CNI:
  Pod 생성 시 네트워크 구성
  Flannel (단순), Calico (표준), Cilium (고성능)
  NetworkPolicy 지원 여부 CNI마다 다름

DNS:
  Service: svc.namespace.svc.cluster.local
  Pod: pod-ip.namespace.pod.cluster.local
  CoreDNS가 처리

NetworkPolicy:
  Pod 레벨 방화벽
  기본: 모두 허용 → 명시적 Deny-All 후 필요한 것만 허용
  CNI 플러그인이 구현 (Flannel 미지원)

진단:
  kubectl get ep: Endpoint(Pod IP) 확인
  kubectl exec pod -- nslookup svc: DNS 확인
  kubectl exec pod -- curl http://svc: 연결 확인
  iptables -t nat -L KUBE-SERVICES: 규칙 확인
```

---

## 🤔 생각해볼 문제

**Q1.** K8s에서 `kubectl exec`로 같은 Pod의 다른 컨테이너에 접근할 때 `localhost`가 동작하는 이유는?

<details>
<summary>해설 보기</summary>

**Pod의 네트워크 모델:**
K8s Pod 내의 모든 컨테이너는 같은 네트워크 네임스페이스를 공유합니다. 이것이 Docker 컨테이너와 K8s Pod의 가장 큰 차이입니다.

```
Pod:
  컨테이너 A (app) ─┐
                    ├─ 같은 네트워크 네임스페이스 (같은 IP, 같은 lo)
  컨테이너 B (sidecar) ─┘
```

- 컨테이너 A가 3000 포트 바인딩 → `localhost:3000`은 컨테이너 B에서도 접근 가능
- Istio 사이드카 프록시(Envoy)가 이 원리로 작동: 같은 Pod, 같은 네트워크 네임스페이스 → Envoy가 모든 트래픽 가로채기 가능

**실제 확인:**
```bash
# sidecar 컨테이너가 있는 Pod
kubectl exec -it my-pod -c sidecar -- curl http://localhost:8080
# → 같은 Pod의 app 컨테이너에 접근 가능
```

**주의:** 파일시스템, 프로세스, IPC 등은 컨테이너별로 격리됩니다. 네트워크만 공유.

</details>

---

**Q2.** Pod A에서 Pod B로 요청할 때 실제 패킷이 어떤 경로로 가는가? (같은 노드 vs 다른 노드)

<details>
<summary>해설 보기</summary>

**같은 노드 (Calico host-gw 모드):**
```
Pod A (10.244.1.5) → veth0 → 호스트 bridge/라우팅 → veth1 → Pod B (10.244.1.6)
```
- 직접 라우팅, 오버헤드 최소
- 약 50~100µs (마이크로초) 지연

**다른 노드 (Calico BGP/host-gw):**
```
Pod A → veth → Node1 라우팅 테이블 → Node1 eth0 → 물리 스위치 → Node2 eth0 → 라우팅 → veth → Pod B
```
- BGP: 각 노드가 자신의 Pod CIDR 라우트를 다른 노드에 광고
- 오버레이 없음 → NAT 없음 → 물리 스위치를 통한 직접 라우팅

**다른 노드 (Flannel VXLAN):**
```
Pod A → veth → VXLAN 캡슐화 (flannel0) → eth0 → 물리 네트워크 → eth0 → VXLAN 해제 → veth → Pod B
```
- VXLAN: UDP 4789 포트로 L2 프레임 캡슐화
- 추가 오버헤드: 캡슐화/해제 CPU + 50 bytes 헤더

**Service를 통한 경우:**
Pod A → veth → iptables DNAT (ClusterIP → Pod B IP) → 위의 경로 중 하나

**실제 패킷 추적:**
```bash
# 컨테이너에서 다른 Pod로의 패킷 경로
kubectl exec pod-a -- traceroute pod-b-ip
```

</details>

---

**Q3.** Ingress Controller가 없는 K8s 클러스터에서 HTTP/HTTPS 서비스를 외부에 노출하는 방법은?

<details>
<summary>해설 보기</summary>

**Ingress Controller 없이 외부 노출하는 방법:**

1. **LoadBalancer Service (클라우드):**
   ```yaml
   spec:
     type: LoadBalancer
   ```
   → AWS ALB/NLB, GCP LB 자동 생성. 각 Service마다 별도 LB → 비용 증가.

2. **NodePort + 외부 로드밸런서:**
   ```yaml
   spec:
     type: NodePort
     ports:
     - port: 80
       nodePort: 30080
   ```
   → 외부 Nginx/HAProxy/AWS ELB가 30080 NodePort를 가리키도록 설정.

3. **hostPort:**
   ```yaml
   containers:
   - ports:
     - containerPort: 80
       hostPort: 80
   ```
   → Pod가 실행되는 노드의 80 포트 직접 바인딩. DaemonSet으로 모든 노드에 배포 가능.

4. **MetalLB (온프레미스 LoadBalancer):**
   ```bash
   kubectl apply -f metallb-native.yaml
   # LoadBalancer Service에 실제 외부 IP 할당
   ```

**Ingress Controller 사용 시 장점:**
- 하나의 LB/IP로 다수 서비스 라우팅
- TLS 종료 중앙화
- 추가 기능: Rate Limiting, 인증, 리다이렉트
- 비용 절감 (단일 LB)

**결론:** 소규모/개발: NodePort, 온프레미스: MetalLB + Ingress, 클라우드: Ingress Controller (Nginx, ALB Ingress) 사용이 표준.

</details>

---

<div align="center">

**[⬅️ 이전: Docker 네트워킹](./01-docker-networking.md)** | **[홈으로 🏠](../README.md)** | **[다음: 네트워크 문제 패턴 ➡️](./03-network-failure-patterns.md)**

</div>
