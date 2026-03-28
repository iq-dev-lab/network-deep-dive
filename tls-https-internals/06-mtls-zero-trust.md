# mTLS — 양방향 인증과 Zero Trust

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 일반 TLS와 mTLS의 핸드쉐이크에서 무엇이 다른가?
- mTLS가 마이크로서비스 Zero Trust에 어떻게 기여하는가?
- Kubernetes Istio는 어떻게 사이드카로 mTLS를 투명하게 적용하는가?
- openssl로 CA, 서버 인증서, 클라이언트 인증서를 직접 생성하고 mTLS를 실험하는 방법은?
- mTLS와 API Key/OAuth의 차이는?
- mTLS 클라이언트 인증서를 Spring Boot에서 어떻게 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"마이크로서비스 환경에서 서비스 간 인증을 어떻게 하나요?":
  방법 1: API Key (헤더에 포함)
    → 네트워크 상에서 탈취 가능 (TLS 없으면)
    → TLS 있어도: 클라이언트 신원 증명 없음
    → 키 유출 시 누구든 서비스 호출 가능
  
  방법 2: JWT (Bearer Token)
    → 발급된 토큰이 유효하면 통과
    → 토큰 탈취 시 만료까지 사용 가능
    → "어느 서비스가 호출했는지" 불분명
  
  방법 3: mTLS (Mutual TLS)
    → 클라이언트 인증서로 신원 증명
    → 인증서 = 조직이 발급한 암호화 서명
    → 탈취해도 개인키 없으면 사용 불가
    → "어떤 서비스가 호출했는지" 명확

Zero Trust 원칙:
  "내부 네트워크라도 신뢰하지 않는다"
  → 모든 서비스 간 통신에 인증 필요
  → mTLS = Zero Trust의 기술적 구현체

Istio(서비스 메시)와 mTLS:
  앱 코드 수정 없이 사이드카(Envoy)가 자동으로 mTLS 처리
  → 개발자: mTLS 구현 없이 Zero Trust 혜택
  → 운영자: 정책으로 서비스 간 통신 제어
```

---

## 😱 흔한 실수

```
Before — mTLS를 모를 때:

실수 1: 내부 네트워크는 안전하다고 가정
  "VPN 안에 있으니 서비스 간 인증 불필요"
  → 공격자가 내부 네트워크 침투 시 모든 서비스에 접근
  → 이것이 "Perimeter Security"의 한계
  → Zero Trust: 내부도 외부처럼 검증

실수 2: mTLS와 일반 HTTPS를 혼동
  일반 TLS: 서버만 인증 (서버 → 클라이언트 방향)
  mTLS:    서버 + 클라이언트 모두 인증 (양방향)
  → 클라이언트 인증서 없이 mTLS 연결 = TLS 연결 실패 (403 아님, 연결 거부)

실수 3: 클라이언트 인증서를 코드에 하드코딩
  클라이언트 인증서/개인키를 소스 코드에 포함
  → GitHub에 올리면 즉시 노출
  → Kubernetes Secret, Vault 등 시크릿 관리 필수

실수 4: Intermediate CA 없이 Root CA로 직접 서비스 인증서 발급
  Root CA 개인키가 Kubernetes Secret에 저장됨
  → 유출 시 모든 내부 인증서 무효화
  → Intermediate CA를 별도로 운영하고 Root CA는 오프라인 보관
```

---

## ✨ 올바른 접근

```
After — mTLS를 알고 나면:

Spring Boot mTLS 클라이언트 설정:
  @Configuration
  public class HttpClientConfig {
      @Bean
      public RestTemplate mtlsRestTemplate() throws Exception {
          // 클라이언트 인증서 로드
          KeyStore keyStore = KeyStore.getInstance("PKCS12");
          keyStore.load(
              new FileInputStream("/secrets/client.p12"),
              "keystore-password".toCharArray()
          );
          
          // 서버 CA 인증서 (신뢰 저장소)
          KeyStore trustStore = KeyStore.getInstance("JKS");
          trustStore.load(
              new FileInputStream("/secrets/truststore.jks"),
              "truststore-password".toCharArray()
          );
          
          SSLContext sslContext = SSLContexts.custom()
              .loadKeyMaterial(keyStore, "key-password".toCharArray())
              .loadTrustMaterial(trustStore, null)
              .build();
          
          CloseableHttpClient client = HttpClients.custom()
              .setSSLContext(sslContext)
              .build();
          
          return new RestTemplate(
              new HttpComponentsClientHttpRequestFactory(client));
      }
  }

Spring Boot mTLS 서버 설정:
  server.ssl.enabled=true
  server.ssl.client-auth=need           # mTLS 필수
  server.ssl.trust-store=classpath:truststore.jks
  server.ssl.trust-store-password=password
  server.ssl.key-store=classpath:server.p12
  server.ssl.key-store-password=password

Nginx mTLS 서버:
  server {
      ssl_client_certificate /etc/nginx/ssl/ca.crt;   # 신뢰할 클라이언트 CA
      ssl_verify_client on;                            # 클라이언트 인증서 필수
      ssl_verify_depth 2;                              # 체인 검증 깊이
      
      # 클라이언트 인증서 정보를 헤더로 전달
      proxy_set_header X-SSL-Client-Subject $ssl_client_s_dn;
      proxy_set_header X-SSL-Client-Verify $ssl_client_verify;
  }
```

---

## 🔬 내부 동작 원리

### 1. 일반 TLS vs mTLS 핸드쉐이크

```
일반 TLS (서버 인증만):

Client                                        Server
│  ① ClientHello                                │
│ ─────────────────────────────────────────────►│
│  ② ServerHello + Certificate                  │
│ ◄───────────────────────────────────────────  │
│  [클라이언트: 서버 인증서 검증]                      │
│  ③ ClientKeyExchange + ChangeCipherSpec       │
│ ─────────────────────────────────────────────►│
│  ④ ChangeCipherSpec + Finished                │
│ ◄───────────────────────────────────────────  │

클라이언트 → 서버: 서버가 신뢰할 수 있는지 확인
서버 → 클라이언트: 아무나 접속 가능 (IP만 있으면)

─────────────────────────────────────────────────────────────────────

mTLS (양방향 인증):

Client                                     Server
│  ① ClientHello                               │
│ ────────────────────────────────────────────►│
│  ② ServerHello                               │
│  ③ Certificate (서버 인증서)                    │
│  ④ CertificateRequest ← 추가!                 │
│    "클라이언트 인증서를 보내주세요"                  │
│    acceptable_certificate_types              │
│    certificate_authorities (신뢰 CA 목록)      │
│  ⑤ ServerHelloDone                           │
│ ◄─────────────────────────────────────────── │
│                                              │
│  [클라이언트: 서버 인증서 검증]                     │
│  [클라이언트: 자신의 인증서 준비]                   │
│                                              │
│  ⑥ Certificate (클라이언트 인증서) ← 추가!        │
│  ⑦ ClientKeyExchange                        │
│  ⑧ CertificateVerify ← 추가!                 │
│    [클라이언트 개인키로 핸드쉐이크 서명]             │
│    = "이 인증서의 개인키 소유자임을 증명"            │
│  ⑨ ChangeCipherSpec + Finished              │
│ ────────────────────────────────────────────►│
│                                              │
│  [서버: 클라이언트 인증서 검증]                     │
│  [서버: CertificateVerify 서명 검증]             │
│                                              │
│  ⑩ ChangeCipherSpec + Finished               │
│ ◄─────────────────────────────────────────── │

추가 단계 3개:
  ④ CertificateRequest: 서버가 클라이언트 인증서 요청
  ⑥ Certificate (클라이언트): 클라이언트 인증서 전송
  ⑧ CertificateVerify: 클라이언트가 개인키 소유 증명
```

### 2. CertificateVerify — 개인키 소유 증명

```
왜 CertificateVerify가 필요한가:
  인증서 자체는 공개된 정보 (누구나 볼 수 있음)
  공격자가 클라이언트 인증서를 복사해서 제출할 수도 있음
  → 인증서만으로는 개인키 소유 증명 불가

CertificateVerify:
  content = Hash(모든 이전 핸드쉐이크 메시지)
  signature = Sign(클라이언트 개인키, content)
  → CertificateVerify 메시지로 서버에 전송

서버 검증:
  클라이언트 공개키(인증서에 포함) = 클라이언트 Certificate의 공개키
  Verify(공개키, content, signature) → 검증 성공
  → 이 클라이언트는 인증서의 개인키를 실제로 소유함

TLS 1.3에서:
  CertificateVerify의 서명 대상:
  content = "TLS 1.3, client CertificateVerify" + Hash(핸드쉐이크 메시지)
  → 컨텍스트 구분으로 다른 프로토콜과의 서명 혼용 방지
```

### 3. Zero Trust와 mTLS

```
전통적 Perimeter Security:
  ┌─────────────────────────────────────────┐
  │  회사 내부 네트워크 (신뢰 구역)                │
  │  ┌────────┐   ┌────────┐   ┌────────┐   │
  │  │ 서비스 A │←→│ 서비스 B  │←→│ 서비스 C │   │
  │  └────────┘   └────────┘   └────────┘   │
  │  내부는 모두 신뢰 → 인증 없이 통신             │
  └─────────────────────────────────────────┘
  외부 ───[방화벽]──→ 내부

  문제:
  공격자가 서비스 B 침투 → 서비스 A, C에 자유롭게 접근
  내부 직원의 악의적 행동 방지 불가

Zero Trust 모델:
  "절대 신뢰하지 말고, 항상 검증하라 (Never Trust, Always Verify)"
  
  네트워크 위치와 무관하게 모든 연결을 검증
  최소 권한 원칙: 필요한 서비스에만 접근
  지속적 검증: 한번 인증으로 영구 신뢰 없음

mTLS가 Zero Trust를 구현하는 방법:
  서비스 A → 서비스 B 요청 시:
  
  1. 서비스 B 검증 (일반 TLS): "진짜 서비스 B인가?"
  2. 서비스 A 검증 (mTLS 추가): "진짜 서비스 A인가?"
  3. 정책 확인: "서비스 A가 서비스 B에 접근할 권한이 있는가?"
  
  → 인증서 = 신원 (어떤 서비스인지)
  → 정책 = 권한 (어떤 서비스에 접근 가능한지)
  
  공격자가 내부 네트워크 접근해도:
  → 유효한 클라이언트 인증서 없음
  → mTLS 연결 거부 → 서비스 접근 불가

SPIFFE (Secure Production Identity Framework For Everyone):
  마이크로서비스 신원 표준
  SPIFFE ID: spiffe://trust-domain/path
  예: spiffe://cluster.local/ns/default/sa/order-service
  → 클라이언트 인증서의 SAN에 SPIFFE ID 포함
  → 서비스 신원 표준화
```

### 4. Istio와 사이드카 mTLS

```
Istio 아키텍처:

  ┌────────────────────────────────────────────────────────┐
  │  Pod A                        Pod B                    │
  │  ┌──────────────────┐  ┌──────────────────┐            │
  │  │ App (주문 서비스)   │  │ App (결제 서비스)   │            │
  │  └────────┬─────────┘  └─────────┬────────┘            │
  │  Envoy    │              Envoy   │                     │
  │  Sidecar ←┘              Sidecar←┘                     │
  │  (mTLS 처리)             (mTLS 처리)                     │
  └─────────────────────┬──────────────────────────────────┘
                        │
                   Istiod (Control Plane)
                   인증서 발급/교체/정책 배포

Istio mTLS 흐름:
  1. Istiod가 각 서비스의 SPIFFE 인증서 자동 발급 (SVID)
  2. 인증서를 각 Envoy Sidecar에 배포
  
  3. 주문 서비스 → 결제 서비스 호출:
     주문 앱: HTTP (평문, 로컬 Envoy로)
     주문 Envoy → 결제 Envoy: mTLS (자동!)
     결제 Envoy → 결제 앱: HTTP (평문, 로컬로)
  
  앱 코드 변경 없음! Envoy가 투명하게 처리

Istio PeerAuthentication 정책:
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: production
  spec:
    mtls:
      mode: STRICT   # 모든 트래픽에 mTLS 필수
                     # PERMISSIVE: mTLS + 평문 둘 다 허용

Istio AuthorizationPolicy (접근 제어):
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: payment-allow
    namespace: production
  spec:
    selector:
      matchLabels:
        app: payment-service
    action: ALLOW
    rules:
    - from:
      - source:
          principals:  # SPIFFE ID로 접근 제어
            - "cluster.local/ns/production/sa/order-service"
    # order-service 외 다른 서비스는 payment-service 접근 불가

인증서 자동 교체:
  Istiod: 서비스 인증서를 24시간마다 자동 교체
  → 인증서 만료 걱정 없음
  → 개인키 유출 피해 기간 최소화
```

---

## 💻 실전 실험

### 실험 1: openssl로 mTLS 환경 구축

```bash
# 1. Root CA 생성
mkdir -p /tmp/mtls/{ca,server,client}
cd /tmp/mtls

openssl genrsa -out ca/ca.key 4096
openssl req -new -x509 -key ca/ca.key -out ca/ca.crt -days 3650 \
  -subj "/C=KR/O=MyOrg/CN=MyOrg Root CA"

# 2. 서버 인증서 생성
openssl genrsa -out server/server.key 2048
openssl req -new -key server/server.key -out server/server.csr \
  -subj "/C=KR/O=MyOrg/CN=api.myorg.internal"

openssl x509 -req -in server/server.csr -CA ca/ca.crt -CAkey ca/ca.key \
  -CAcreateserial -out server/server.crt -days 365 \
  -extfile <(cat <<EOF
[ext]
subjectAltName=DNS:api.myorg.internal,IP:127.0.0.1
keyUsage=digitalSignature
extendedKeyUsage=serverAuth
EOF
) -extensions ext

# 3. 클라이언트 인증서 생성 (order-service용)
openssl genrsa -out client/client.key 2048
openssl req -new -key client/client.key -out client/client.csr \
  -subj "/C=KR/O=MyOrg/CN=order-service"

openssl x509 -req -in client/client.csr -CA ca/ca.crt -CAkey ca/ca.key \
  -CAcreateserial -out client/client.crt -days 365 \
  -extfile <(echo "extendedKeyUsage=clientAuth")

echo "인증서 생성 완료!"
ls -la ca/ server/ client/
```

### 실험 2: mTLS 서버-클라이언트 연결 테스트

```bash
# 서버 실행 (Python 간단 HTTPS 서버)
python3 -c "
import ssl, http.server
class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        client_cert = self.connection.getpeercert()
        self.send_response(200)
        self.end_headers()
        self.wfile.write(f'Hello {client_cert.get(\"subject\", {})}!'.encode())
    def log_message(self, *args): pass

ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain('server/server.crt', 'server/server.key')
ctx.load_verify_locations('ca/ca.crt')
ctx.verify_mode = ssl.CERT_REQUIRED  # 클라이언트 인증서 필수!

server = http.server.HTTPServer(('127.0.0.1', 8443), Handler)
server.socket = ctx.wrap_socket(server.socket, server_side=True)
print('mTLS 서버 시작: https://127.0.0.1:8443')
server.serve_forever()
" &

sleep 1

# 클라이언트 인증서로 연결 (성공)
curl -v --cacert ca/ca.crt \
     --cert client/client.crt \
     --key client/client.key \
     https://127.0.0.1:8443
# Hello order-service!

# 인증서 없이 연결 시도 (실패)
curl -v --cacert ca/ca.crt https://127.0.0.1:8443
# SSL alert: handshake failure (클라이언트 인증서 없음)
```

### 실험 3: Nginx mTLS 설정

```bash
# Nginx mTLS 설정
cat > /tmp/nginx-mtls.conf << 'EOF'
events {}
http {
    server {
        listen 443 ssl;
        server_name localhost;
        
        ssl_certificate /tmp/mtls/server/server.crt;
        ssl_certificate_key /tmp/mtls/server/server.key;
        
        # mTLS: 클라이언트 인증서 필수
        ssl_client_certificate /tmp/mtls/ca/ca.crt;
        ssl_verify_client on;
        ssl_verify_depth 2;
        
        location / {
            # 클라이언트 인증서 정보 헤더로 전달
            add_header X-Client-CN $ssl_client_s_dn_cn always;
            add_header X-Client-Verify $ssl_client_verify always;
            return 200 "Authenticated: $ssl_client_s_dn_cn\n";
        }
    }
}
EOF

nginx -c /tmp/nginx-mtls.conf

# 테스트
curl --cacert /tmp/mtls/ca/ca.crt \
     --cert /tmp/mtls/client/client.crt \
     --key /tmp/mtls/client/client.key \
     https://localhost/
# Authenticated: order-service
```

### 실험 4: Spring Boot mTLS 서버 설정

```bash
# PKCS12 키스토어 생성
openssl pkcs12 -export -in server/server.crt -inkey server/server.key \
  -out server/server.p12 -name "server" -passout pass:serverpass

# 트러스트스토어 생성 (클라이언트 CA 포함)
keytool -importcert -file ca/ca.crt -alias "myorg-ca" \
  -keystore truststore.jks -storepass trustpass -noprompt

# application.properties
cat << 'EOF'
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:server.p12
server.ssl.key-store-password=serverpass
server.ssl.key-store-type=PKCS12
server.ssl.client-auth=need
server.ssl.trust-store=classpath:truststore.jks
server.ssl.trust-store-password=trustpass
EOF

# 클라이언트 인증서 정보 추출 (Spring MVC)
@RestController
public class AuthController {
    @GetMapping("/authenticated")
    public String authenticated(HttpServletRequest request) {
        X509Certificate[] certs = (X509Certificate[]) 
            request.getAttribute("javax.servlet.request.X509Certificate");
        if (certs != null && certs.length > 0) {
            return "Client: " + certs[0].getSubjectDN().getName();
        }
        return "No client cert";
    }
}
```

---

## 📊 성능/비용 비교

```
인증 방식 비교:

┌──────────────────────────────────────────────────────────────────────┐
│  방식            │  보안 수준   │  구현 복잡도   │  확장성  │  인증 위치       │
├──────────────────────────────────────────────────────────────────────┤
│  API Key        │  낮음       │  낮음        │  높음    │  헤더           │
│  JWT (Bearer)   │  중간       │  중간        │  높음    │  헤더           │
│  mTLS           │  높음       │  높음        │  중간    │  TLS 레이어     │
│  mTLS + JWT     │  매우 높음   │  높음        │  중간    │  양쪽           │
└──────────────────────────────────────────────────────────────────────┘

mTLS 핸드쉐이크 추가 비용:
  일반 TLS: ECDSA 서명 1회 (서버)
  mTLS:     ECDSA 서명 2회 (서버 + 클라이언트)
            + ECDSA 검증 2회
  추가 CPU: ~0.2ms (무시 가능)
  추가 RTT: 0 (같은 핸드쉐이크 내에서 처리)

인증서 관리 비용:
  서비스 100개 × 인증서 갱신 주기 30일
  = 월 100번 갱신 작업 (cert-manager로 자동화)
  Istio: 24시간 주기 자동 교체 → 관리 부담 없음
```

---

## ⚖️ 트레이드오프

```
mTLS 도입 트레이드오프:

장점:
  ① 양방향 인증: 클라이언트 신원을 암호학적으로 증명
  ② Zero Trust: 내부 네트워크도 신뢰하지 않음
  ③ 탈취 저항: 인증서 + 개인키 모두 필요 (API Key보다 강력)
  ④ 세분화된 접근 제어: 서비스별 인증서 → 서비스별 권한

단점:
  ① 인증서 관리 복잡성:
     서비스 수 × 인증서 = 관리 대상 증가
     자동화 필수 (cert-manager, Vault)
  
  ② 레거시 클라이언트:
     모든 HTTP 클라이언트가 클라이언트 인증서 지원
     → 구형 시스템, 파트너 API에서 미지원 가능
  
  ③ 디버깅 어려움:
     TLS 연결 실패 원인 파악이 어려움 (인증서 문제 vs 네트워크 문제)
     tcpdump + openssl s_client로 진단 필요
  
  ④ 수평 확장:
     클라이언트 인증서 배포/갱신이 수평 확장 시 복잡
     → Istio/서비스 메시로 자동화

mTLS vs OAuth 2.0 (서비스 계정):
  mTLS:
    인증: TLS 레이어 (암호학적)
    설정: 서버/클라이언트 모두 TLS 구성 변경 필요
    적합: 내부 마이크로서비스 통신
  
  OAuth Client Credentials:
    인증: HTTP 레이어 (토큰 기반)
    설정: HTTP 헤더만 추가
    적합: 외부 API, 파트너 통합
  
  둘 다 사용 (최선):
    내부: mTLS (인증서 기반 신원)
    외부: OAuth (토큰 기반 인증) + TLS (전송 암호화)
```

---

## 📌 핵심 정리

```
mTLS 핵심 요약:

일반 TLS vs mTLS:
  일반 TLS: 서버만 인증 (클라이언트 → 서버 방향)
  mTLS:     서버 + 클라이언트 모두 인증 (양방향)
  추가 단계: CertificateRequest, 클라이언트 Certificate, CertificateVerify

Zero Trust:
  "내부도 신뢰하지 않는다"
  모든 서비스 간 통신에 인증 필요
  mTLS = Zero Trust의 기술적 구현체
  SPIFFE ID로 서비스 신원 표준화

Istio 사이드카 mTLS:
  Envoy Sidecar가 앱 대신 mTLS 처리
  앱 코드 변경 없음
  PeerAuthentication: mTLS 필수 정책
  AuthorizationPolicy: SPIFFE ID 기반 접근 제어
  인증서 24시간마다 자동 교체

인증서 계층:
  Root CA (오프라인) → Intermediate CA → 서버/클라이언트 인증서
  Root CA 개인키는 절대 온라인에 두지 않음

Spring Boot:
  server.ssl.client-auth=need: 클라이언트 인증서 필수
  X509Certificate를 request에서 추출 가능

진단:
  openssl s_client -cert client.crt -key client.key -connect host:443
  연결 실패 시: alert 코드 확인 (access_denied vs unknown_ca)
```

---

## 🤔 생각해볼 문제

**Q1.** mTLS 환경에서 클라이언트 인증서가 만료됐다. 어떤 오류가 발생하고 어떻게 갱신 프로세스를 설계해야 하는가?

<details>
<summary>해설 보기</summary>

**발생하는 오류:**
```
TLS Alert: certificate_expired (42)
```
서버가 클라이언트 인증서의 Validity.NotAfter를 확인하고 만료됐으면 TLS 핸드쉐이크를 거부합니다. 클라이언트 측에서는 연결 자체가 실패합니다 (HTTP 에러 코드도 받지 못함).

**갱신 프로세스 설계:**

1. **Zero-downtime 갱신 (권장):**
```
Day 0:    클라이언트 인증서 발급 (30일 유효)
Day 20:   새 인증서 발급 시작 (10일 전)
Day 20-25: 서버가 기존 + 새 인증서 모두 신뢰 (transition period)
Day 25:   클라이언트가 새 인증서로 전환
Day 30:   기존 인증서 만료, 서버에서 기존 인증서 신뢰 제거
```

2. **Kubernetes cert-manager:**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: order-service-cert
spec:
  secretName: order-service-tls
  duration: 720h    # 30일
  renewBefore: 168h # 7일 전에 자동 갱신
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
  subject:
    organizations: [myorg]
  commonName: order-service
  uris:
    - spiffe://cluster.local/ns/default/sa/order-service
```
cert-manager가 자동으로 7일 전에 갱신하고 Secret을 교체. Pod가 새 Secret을 읽도록 재시작.

3. **Istio의 경우:**
Istiod가 24시간마다 자동 갱신. 앱 재시작 없음.

4. **모니터링:**
```bash
# 만료 날짜 Prometheus 메트릭
ssl_certificate_expiry_seconds{host="order-service"}
# 7일 미만 시 알람
```

</details>

---

**Q2.** mTLS에서 인증서 폐기(Revocation)는 어떻게 처리하는가? 직원이 퇴사했을 때 즉시 접근을 차단하려면?

<details>
<summary>해설 보기</summary>

**인증서 폐기의 어려움:**

mTLS에서 클라이언트 인증서 폐기는 공개 웹보다 더 복잡합니다. 내부 CA의 OCSP 서버를 운영하거나 CRL을 유지해야 합니다.

**방법 1: OCSP (권장)**
```nginx
ssl_verify_client on;
# 클라이언트 인증서 OCSP 검증
ssl_ocsp on;
ssl_ocsp_responder http://internal-ocsp.myorg.internal;
```
폐기 시: OCSP 서버에서 "Revoked" 반환 → 즉시 차단

**방법 2: CRL (Certificate Revocation List)**
```nginx
ssl_crl /etc/nginx/ssl/revoked.crl;
```
폐기 시: CRL 파일 업데이트 후 Nginx 재로드 → 차단
단점: CRL 업데이트와 Nginx 재로드 사이 시간 간격

**방법 3: 단수명 인증서 (Short-lived Certificate)**
Istio처럼 24시간마다 인증서 자동 발급/교체:
- 직원 퇴사 → CA에서 더 이상 인증서 발급 안 함
- 기존 인증서: 최대 24시간 후 만료 → 자동 차단
- 폐기 인프라 없이도 빠른 차단

**방법 4: 소프트웨어 레이어 제어 (가장 빠름)**
```yaml
# Istio AuthorizationPolicy
spec:
  action: DENY
  rules:
  - from:
    - source:
        principals:
          - "cluster.local/ns/default/sa/ex-employee-service"
```
SPIFFE ID 기반으로 즉시 차단 (인증서 폐기 없이).

**결론:** 내부 mTLS는 단수명 인증서 + 서비스 메시 정책 조합이 가장 실용적입니다.

</details>

---

**Q3.** 외부 파트너사가 우리 API를 호출해야 할 때 mTLS를 사용해야 하는가? 대안은?

<details>
<summary>해설 보기</summary>

**mTLS가 적합한 경우:**
- 파트너사와 강한 보안 요구 (금융, 의료)
- 파트너사가 기술 역량이 높고 인증서 관리 가능
- B2B 통합 (기계 간 통신, 사람 없음)

**mTLS의 현실적 어려움:**
- 파트너사가 클라이언트 인증서 설정 어려움 (비개발자, 레거시 시스템)
- 인증서 배포/갱신 조율 필요
- 방화벽, 프록시가 클라이언트 인증서를 제거할 수 있음

**대안:**

1. **OAuth 2.0 Client Credentials (가장 일반적):**
```
파트너사: client_id + client_secret → access_token
파트너사: Authorization: Bearer {token} 헤더로 API 호출
```
- 표준화됨, 광범위한 지원
- token_expiry로 유효 기간 제어

2. **API Key + IP 화이트리스트:**
```
파트너사: X-API-Key: {key} 헤더
서버: IP 범위 + API Key 검증
```
- 간단, 하지만 API Key 탈취 위험

3. **mTLS + OAuth 조합 (최선):**
```
파트너사 ──mTLS──→ API Gateway ──OAuth 검증──→ 내부 서비스
```
- API Gateway에서 mTLS 종단
- 내부는 OAuth 토큰으로 인가
- 파트너: 클라이언트 인증서만 관리
- 내부: OAuth 토큰으로 세분화된 권한 제어

**실무 권장:**
- 파트너 수가 적고 보안 요구 높음: mTLS
- 파트너 수가 많고 다양함: OAuth 2.0
- 금융/의료 규정 준수: mTLS 또는 mTLS + OAuth

</details>

---

<div align="center">

**[⬅️ 이전: HTTPS 성능](./05-https-performance.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — DNS ➡️](../dns-internals/01-dns-resolution-flow.md)**

</div>
