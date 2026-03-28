# HTTP 캐싱 완전 분해 — Cache-Control과 캐시 무효화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Cache-Control: no-cache`와 `no-store`의 정확한 차이는 무엇인가?
- `max-age`와 `s-maxage`는 각각 어디에 적용되는가?
- ETag와 `If-None-Match`로 조건부 요청이 어떻게 동작하는가?
- 304 Not Modified 응답은 언제 반환되고 클라이언트는 이를 어떻게 처리하는가?
- CDN 캐시와 브라우저 캐시의 역할 분리는 어떻게 설계하는가?
- 배포 시 오래된 캐시를 어떻게 무효화(Cache Busting)하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"배포했는데 사용자들이 아직 이전 버전을 보고 있어요":
  → Cache-Control 설계 문제
  → JS/CSS 파일에 파일명에 해시 미포함
  → 브라우저/CDN이 이전 버전 캐시 유지

"API 응답에 캐시가 적용되면 안 되는데 왜 캐시되나요?":
  → Cache-Control 헤더 누락 시 브라우저/Proxy가 임의 캐싱
  → 명시적으로 no-store 설정 필요

"CDN을 도입했는데 오히려 느려졌어요":
  → CDN이 캐시를 못 하고 매번 Origin으로 요청
  → Cache-Control 헤더에 s-maxage 없음
  → CDN이 캐시 가능 여부를 판단 못 함

비용 관점:
  서버 처리 비용 >> 캐시 서빙 비용
  DB 쿼리 1회:   10ms × N req/s = 10N ms
  캐시 히트:     1ms × N req/s = N ms
  CDN 서빙:      0ms (Origin 요청 없음)
  
  정적 파일(JS, CSS, 이미지)에 적절한 캐시:
    Origin 서버 부하 90%+ 감소
    응답 시간 수백ms → 수십ms
```

---

## 😱 흔한 실수

```
Before — 캐싱을 모를 때:

실수 1: no-cache가 "캐시하지 마라"인 줄 알고 사용
  Cache-Control: no-cache
  → 실제 의미: "캐시하되, 사용 전에 서버에 유효성 확인"
  → 캐시는 저장됨! (no-store와 전혀 다름)

실수 2: 정적 파일에 캐시 미설정
  main.js 배포 시 파일명 변경 없이 덮어쓰기
  Cache-Control: max-age=3600 설정
  → 배포 후 1시간 동안 이전 JS 제공
  → 해결: 파일명에 빌드 해시 포함 (main.abc123.js)

실수 3: API 응답을 캐시 가능하게 방치
  GET /api/user/profile
  Cache-Control 헤더 없음
  → Proxy/CDN이 임의로 캐시할 수 있음
  → 다른 사용자가 이전 사용자의 프로필 조회 가능!
  → 개인 정보가 포함된 API: Cache-Control: no-store, private

실수 4: CDN에서 s-maxage 없이 max-age만 설정
  Cache-Control: max-age=3600
  → 브라우저: 3600초 캐시 ✅
  → CDN: max-age를 존중할 수도 있고 아닐 수도 있음
  → Cache-Control: s-maxage=86400, max-age=3600 명시 필요
```

---

## ✨ 올바른 접근

```
After — 캐싱을 알고 나면:

리소스 유형별 올바른 Cache-Control:

정적 파일 (JS, CSS, 이미지 - 파일명에 해시 포함):
  Cache-Control: public, max-age=31536000, immutable
  → 1년 캐시 (파일명이 바뀌면 새 URL → 캐시 자동 무효화)
  → immutable: 기간 내 절대 변경 없음 (재검증 요청 없음)

HTML 파일 (최신 버전 필요):
  Cache-Control: no-cache
  → 캐시하되 항상 서버에 유효성 확인 (ETag로 빠른 응답)
  → 파일 변경 없으면 304 → 즉시 캐시 사용

API 응답 (개인 정보 없음):
  Cache-Control: public, max-age=60, s-maxage=300
  → 브라우저: 60초 캐시
  → CDN: 5분 캐시 (Origin 부하 감소)

API 응답 (개인 정보 포함):
  Cache-Control: private, no-store
  → 브라우저 개인 캐시 (Shared Cache인 CDN에는 캐시 안 됨)
  → 또는 no-store로 완전 방지

Spring Boot 설정:
  @GetMapping("/api/public/data")
  public ResponseEntity<Data> getPublicData() {
      return ResponseEntity.ok()
          .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS)
              .cachePublic()
              .sMaxAge(300, TimeUnit.SECONDS))
          .body(data);
  }

  @GetMapping("/api/user/profile")
  public ResponseEntity<Profile> getProfile() {
      return ResponseEntity.ok()
          .cacheControl(CacheControl.noStore())
          .body(profile);
  }
```

---

## 🔬 내부 동작 원리

### 1. Cache-Control 지시어 완전 정리

```
요청 지시어 (Request Cache-Control):
  no-cache:   캐시된 응답 사용 전 서버 재검증 강제
  no-store:   응답을 캐시에 저장하지 않도록 요청
  no-transform: 중간 Proxy가 콘텐츠 변환하지 않도록
  only-if-cached: 캐시에 없으면 504, 네트워크 요청 금지
  max-age=N:  N초보다 오래된 캐시는 수락 안 함
  max-stale:  만료된 캐시도 수락
  min-fresh=N: N초 이상 유효 기간 남은 캐시만 수락

응답 지시어 (Response Cache-Control):
  public:     모든 캐시(브라우저, CDN 등) 저장 가능
  private:    브라우저 개인 캐시에만 저장 가능 (CDN 불가)
  no-cache:   캐시 저장은 하되, 사용 전 서버 재검증 필수
  no-store:   어디에도 저장 금지 (가장 강력한 비캐시)
  max-age=N:  브라우저/Proxy 모두에서 N초간 신선함 유지
  s-maxage=N: Shared Cache(CDN, Proxy)에서 N초간 유효
              (있으면 max-age보다 우선, 없으면 max-age 적용)
  must-revalidate: 만료 후 반드시 서버 재검증 (stale-while-revalidate와 반대)
  immutable:  유효 기간 중 절대 변경 없음 (재검증 요청 자제 요청)
  stale-while-revalidate=N: 만료 후 N초 동안은 stale 응답 제공하면서 백그라운드 재검증
  stale-if-error=N: 서버 오류 시 N초까지 stale 응답 제공

no-cache vs no-store 비교:
┌──────────────────────────────────────────────────────────────────────┐
│  지시어       │  캐시 저장    │  사용 전 검증   │  주요 용도                  │
├──────────────────────────────────────────────────────────────────────┤
│  no-cache   │  예          │  항상 필요     │  항상 최신 필요하지만          │
│             │             │              │  304로 빠른 응답 원할 때       │
├──────────────────────────────────────────────────────────────────────┤
│  no-store   │  아니오       │  해당 없음      │  개인정보, 금융 데이터        │
│             │  (완전 금지)   │              │  캐시 자체 불가              │
└──────────────────────────────────────────────────────────────────────┘
```

### 2. ETag와 조건부 요청

```
신선도 검사 (Freshness Check):

  브라우저 캐시: max-age 기간 내 → 서버 요청 없이 캐시 직접 사용
  max-age 초과 → "Stale 상태" → 서버에 재검증 필요

ETag 기반 조건부 요청:

  첫 요청:
  ─────────────────────────────────────────────────────────
  GET /api/config HTTP/1.1
  Host: api.example.com
  
  ← HTTP/1.1 200 OK
     Cache-Control: no-cache
     ETag: "v2.3.1-abc123"          ← 리소스 버전 식별자
     Content-Type: application/json
     [body: 설정 데이터]
  
  브라우저: 응답 + ETag 저장

  재검증 요청 (max-age 초과 또는 no-cache):
  ─────────────────────────────────────────────────────────
  GET /api/config HTTP/1.1
  Host: api.example.com
  If-None-Match: "v2.3.1-abc123"    ← 내가 가진 ETag
  
  서버 판단:
    현재 ETag == "v2.3.1-abc123"? → 304 Not Modified (body 없음)
    현재 ETag != "v2.3.1-abc123"? → 200 OK + 새 body + 새 ETag
  
  ← HTTP/1.1 304 Not Modified
     ETag: "v2.3.1-abc123"          ← 동일 ETag
     Cache-Control: no-cache
     [body 없음!]
  
  브라우저: 캐시된 body 그대로 사용 (body 전송 비용 절약)

Last-Modified 기반 조건부 요청:
  서버 응답: Last-Modified: Sat, 28 Mar 2026 10:00:00 GMT
  재검증:    If-Modified-Since: Sat, 28 Mar 2026 10:00:00 GMT
  → 이후 변경됐으면 200 + 새 body, 없으면 304

ETag vs Last-Modified 비교:
  ETag:          파일 내용의 해시 → 더 정확
  Last-Modified: 수정 시간 → 1초 단위 정밀도 한계
  → ETag가 우선 (둘 다 있으면 ETag 사용)
  → 파일을 저장했지만 내용이 같으면: ETag 불변, Last-Modified 변경

Strong ETag vs Weak ETag:
  Strong: "abc123"    → 바이트 단위 동일성 (콘텐츠 완전 동일)
  Weak:   W/"abc123"  → 의미적 동일성 (일부 차이 허용, gzip 여부 등)
  
  Range 요청에는 Strong ETag만 사용 가능
```

### 3. 캐시 계층 구조

```
HTTP 캐시 계층:

  [클라이언트]           [인프라]              [서버]
  브라우저 캐시    →   CDN 캐시    →    Origin 서버
  (Private Cache)    (Shared Cache)    (Source of Truth)

브라우저 캐시 (Private):
  사용자 개인 캐시 (다른 사용자와 공유 안 됨)
  Cache-Control: private → 여기까지만 캐시
  Cache-Control: public  → CDN도 캐시 가능

CDN (Shared Cache):
  Cache-Control: public + max-age or s-maxage → 캐시
  Cache-Control: private → 캐시 안 함
  Cache-Control: no-store → 캐시 안 함
  
  CDN별 s-maxage 해석:
    대부분의 CDN: s-maxage를 CDN TTL로 사용
    s-maxage 없으면: max-age 또는 자체 기본값 사용
    Surrogate-Control: max-age=86400 (Fastly, Varnish 전용)

캐시 계층 설계 예:
  GET /api/products (공개 상품 목록):
    Cache-Control: public, max-age=60, s-maxage=600
    → 브라우저: 60초 (자주 업데이트 반영)
    → CDN: 10분 (Origin 부하 감소)
  
  GET /users/{id}/profile (개인 정보):
    Cache-Control: private, max-age=300
    → 브라우저 개인 캐시: 5분 (재요청 감소)
    → CDN: 캐시 안 함 (private)
  
  GET /static/main.abc123.js (정적 파일):
    Cache-Control: public, max-age=31536000, immutable
    → 브라우저: 1년
    → CDN: 1년
    → 변경 시 파일명 해시 변경 → 새 URL → 자동 무효화

Vary 헤더:
  Vary: Accept-Encoding
  → Accept-Encoding이 다른 요청은 별도 캐시 항목
  → gzip 응답과 plain 응답을 각각 캐시
  
  Vary: Cookie 또는 Authorization 사용 금지!
  → 사용자마다 다른 캐시 항목 → CDN 캐시 폭발
  → 개인화 데이터는 private + CDN 캐시 안 함
```

### 4. Cache Busting 전략

```
배포 시 캐시 무효화 방법:

전략 1: URL 해시 (권장 — 정적 파일)
  배포 전: /static/main.js        (max-age=31536000)
  배포 후: /static/main.abc123.js (새 URL → 새 캐시)
  
  Webpack/Vite 설정:
    output: { filename: '[name].[contenthash].js' }
  → 파일 내용이 같으면 해시 동일 (불필요한 캐시 무효화 방지)
  → 파일 내용이 다르면 해시 변경 → 자동 무효화

전략 2: Query String (간단하지만 CDN에 따라 무시될 수 있음)
  /static/main.js?v=20260328
  → 일부 CDN은 Query String을 캐시 키에 미포함
  → 신뢰성 낮음

전략 3: CDN 강제 무효화 (Purge)
  배포 완료 후 CDN API로 캐시 삭제
  
  CloudFront:
    aws cloudfront create-invalidation \
      --distribution-id E1EXAMPLE \
      --paths "/api/*"
  
  Cloudflare:
    curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
         -H "Authorization: Bearer {token}" \
         -d '{"purge_everything":true}'
  
  단점: API 호출 비용, 전파 지연 (수 초~수십 초)

전략 4: Versioned Path
  /v2/api/users 대신 /api/users (URL 변경 없이 버전 관리)
  → API 응답 변경 시 Cache-Control max-age 단축
  → ETag로 변경 감지

Spring Boot 배포 전략:
  정적 리소스: WebMvcConfigurer + ResourceChain
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addResolver(new VersionResourceResolver()
                .addContentVersionStrategy("/**"));
        // → /static/main.js → /static/main-abc123.js 자동 변환
    }
```

### 5. stale-while-revalidate 패턴

```
stale-while-revalidate: 성능과 신선도의 균형

문제:
  max-age 초과 → 동기 재검증 → 사용자 대기
  
  [브라우저] → [서버] (재검증)
                      ↕ 100ms 대기
  [사용자 대기 중...]
  [브라우저] ← [서버] (200 또는 304)

stale-while-revalidate 해결:
  Cache-Control: max-age=60, stale-while-revalidate=120

  시나리오:
    0초:   캐시 저장
    60초:  max-age 초과 → stale 상태
    60~180초: stale 응답 즉시 반환 + 백그라운드 재검증
    180초: stale-while-revalidate 초과 → 동기 재검증 필요

  효과:
    사용자: stale 응답 즉시 받음 (0ms 대기)
    백그라운드: 서버에서 새 응답 가져와서 캐시 갱신
    다음 요청: 갱신된 캐시 사용

  적합한 사용 사례:
    자주 변경되지만 즉각성이 중요한 데이터
    예: 뉴스 피드, 상품 가격, 사용자 통계

  부적합한 사용 사례:
    보안 토큰, 결제 정보, 재고 등 실시간 정확성 필요
    → must-revalidate 또는 no-cache 사용
```

---

## 💻 실전 실험

### 실험 1: Cache-Control 헤더 직접 확인

```bash
# 여러 사이트의 캐시 전략 비교
for url in \
  "https://www.google.com" \
  "https://www.github.com/favicon.ico" \
  "https://cdnjs.cloudflare.com/ajax/libs/jquery/3.6.0/jquery.min.js"; do
  echo "=== $url ==="
  curl -s -I "$url" | grep -iE "cache-control|etag|last-modified|expires"
  echo ""
done
```

### 실험 2: ETag와 304 Not Modified 직접 확인

```bash
# 첫 요청 - ETag 확인
curl -v https://httpbin.org/etag/test 2>&1 | grep -iE "etag|status"
# ETag: "test"

# ETag로 조건부 재요청
curl -v https://httpbin.org/etag/test \
     -H 'If-None-Match: "test"' 2>&1 | grep -iE "status|304"
# HTTP/2 304 (body 없음)

# 다른 ETag로 재요청 (변경된 척)
curl -v https://httpbin.org/etag/test \
     -H 'If-None-Match: "wrong-etag"' 2>&1 | grep -iE "status|200"
# HTTP/2 200 (새 body 반환)

# Last-Modified 기반 조건부 요청
curl -v https://httpbin.org/cache/60 2>&1 | grep -i "last-modified"
LAST_MOD=$(curl -s -I https://httpbin.org/cache/60 | grep -i "last-modified" | cut -d' ' -f2-)
curl -v https://httpbin.org/cache/60 \
     -H "If-Modified-Since: $LAST_MOD" 2>&1 | grep "304\|200"
```

### 실험 3: 캐시 계층 동작 관찰

```bash
# 응답 시간으로 캐시 히트/미스 구분
# X-Cache 헤더로 CDN 캐시 상태 확인

# 첫 요청 (CDN 미스)
curl -v https://cdn.example.com/large-file.js 2>&1 | \
  grep -iE "x-cache|age|cache-control|time_total"
# X-Cache: MISS (CDN 미스 → Origin 요청)
# Age: 0 (방금 캐시됨)

# 두 번째 요청 (CDN 히트)
curl -v https://cdn.example.com/large-file.js 2>&1 | \
  grep -iE "x-cache|age"
# X-Cache: HIT (CDN에서 서빙)
# Age: 5 (캐시된 지 5초)

# 응답 시간 비교
curl -w "Total: %{time_total}s\n" -o /dev/null -s \
  https://cdn.example.com/large-file.js
# MISS: 200ms (Origin까지 왕복)
# HIT:  10ms  (CDN에서 즉시)
```

### 실험 4: Spring Boot 캐시 헤더 설정

```bash
# Spring Boot 앱 실행 후 캐시 헤더 확인
curl -v http://localhost:8080/api/public/products 2>&1 | \
  grep -i "cache-control\|etag\|vary"

# no-store 확인 (개인 정보 API)
curl -v http://localhost:8080/api/user/profile 2>&1 | \
  grep -i "cache-control"
# Cache-Control: no-store, must-revalidate

# ETag 지원 확인 (Spring의 ShallowEtagHeaderFilter)
# 첫 요청 → ETag 수신
ETAG=$(curl -s -I http://localhost:8080/api/data | grep -i etag | awk '{print $2}')
# 재검증 → 304 확인
curl -v http://localhost:8080/api/data \
     -H "If-None-Match: $ETAG" 2>&1 | grep "304\|200"
```

---

## 📊 성능/비용 비교

```
캐시 전략별 성능 차이:

시나리오: 초당 1000 요청, Origin 처리 시간 100ms, CDN RTT 10ms

CDN 캐시 없음 (no-cache + 검증 없음):
  모든 요청 → Origin
  처리량: 1000 × 100ms = 100초 처리 필요
  필요 서버: 100+ 인스턴스
  응답시간: 100ms

CDN 캐시 있음 (max-age=60, 캐시 히트율 95%):
  950 req → CDN (10ms)
  50 req  → Origin (100ms)
  평균 응답시간: 0.95×10 + 0.05×100 = 14.5ms
  필요 서버: 5+ 인스턴스 (95% 감소!)

ETag 재검증 효과:
  변경 없는 경우:
    no-cache + ETag: 요청 1개 (검증) → 304 → 즉시 (0 body 전송)
    max-age 캐시:    요청 0개 → 즉시 (0 RTT, 0 서버 처리)
  
  변경 있는 경우:
    no-cache + ETag: 요청 1개 → 200 + 새 body
    max-age 캐시:    max-age 기간 동안 구버전 제공

Cache-Control: immutable 효과:
  max-age=31536000 만으로: 1년 내에도 브라우저가 재검증 요청 가능 (새로고침 시)
  immutable 추가:          재검증 요청 자체를 하지 않음 → 네트워크 요청 0
  정적 파일(해시 포함)에 immutable → 불필요한 서버 요청 완전 제거
```

---

## ⚖️ 트레이드오프

```
캐시 기간 설정:

짧은 max-age (< 1분):
  장점: 항상 최신 데이터
  단점: Origin 부하 높음, 응답 지연 가능

긴 max-age (> 1시간):
  장점: Origin 부하 최소, 빠른 응답
  단점: 배포 후 변경사항 반영 지연

신선도 vs 가용성:
  must-revalidate:
    서버 다운 시 → 504 (만료된 캐시 사용 불가)
    항상 최신 보장 but 서버 장애 시 가용성 저하
  
  stale-if-error=86400:
    서버 다운 시 → 1일 동안 만료된 캐시 제공
    최신성은 희생하지만 서비스 가용성 유지

ETag vs max-age:
  ETag만:    항상 서버 요청, 304면 빠르지만 요청 자체는 발생
  max-age:   기간 내 서버 요청 없음 (네트워크 0)
  둘 다:     max-age 기간 후 ETag로 재검증 (균형)

CDN 캐시 무효화 비용:
  API 호출 + 전파 지연
  잦은 무효화 → CDN 비용 증가
  → 캐시 전략을 잘 설계해서 무효화 최소화
  → 정적 파일: URL 해시로 무효화 API 불필요
```

---

## 📌 핵심 정리

```
HTTP 캐싱 핵심 요약:

Cache-Control 지시어:
  no-cache:  캐시O, 사용 전 서버 검증 필수
  no-store:  캐시X, 저장 자체 금지
  max-age=N: 브라우저+CDN에서 N초 신선함 유지
  s-maxage=N: CDN(Shared Cache)에서 N초 (max-age 대체)
  private:   브라우저만 캐시 (CDN 불가)
  public:    모든 캐시 허용
  immutable: 기간 내 변경 없음 (재검증 요청 자제)
  stale-while-revalidate=N: N초 동안 stale 제공 + 백그라운드 갱신

조건부 요청:
  ETag: 리소스 버전 식별자 (서버 생성)
  If-None-Match: 캐시의 ETag로 재검증
  304: 변경 없음 (body 없음, 캐시 사용)
  200: 변경 있음 (새 body + 새 ETag)

캐시 계층:
  브라우저(Private) → CDN(Shared) → Origin
  private 응답: 브라우저만 캐시
  public 응답: CDN도 캐시

Cache Busting:
  정적 파일: 파일명에 컨텐츠 해시 포함
  API: CDN Purge API 또는 짧은 max-age
  HTML: no-cache + ETag (항상 최신 but 빠른 304)

Spring Boot:
  CacheControl.maxAge(60, TimeUnit.SECONDS).cachePublic()
  CacheControl.noStore()
  ShallowEtagHeaderFilter: 자동 ETag 생성/검증
```

---

## 🤔 생각해볼 문제

**Q1.** 상품 목록 페이지의 HTML은 `Cache-Control: no-cache`로, 상품 이미지는 `Cache-Control: max-age=86400`로 설정했다. 새 상품이 추가됐을 때 각각 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

**HTML (no-cache):**
- 브라우저가 캐시를 가지고 있어도 반드시 서버에 재검증 요청을 보냄
- 서버가 새 상품 목록으로 HTML을 변경했다면 → 200 + 새 HTML
- HTML이 변경되지 않았다면 → 304 Not Modified (빠른 응답)

**이미지 (max-age=86400, 24시간):**
- 24시간 내에는 브라우저가 서버 요청 없이 캐시에서 직접 사용
- 새로 추가된 상품의 이미지 URL이 새로운 URL이면: 새 URL → 새 캐시 항목 → 정상 표시
- 기존 이미지 URL이 업데이트된 경우: 24시간 동안 이전 이미지 표시

**문제 상황:**
같은 URL의 이미지가 교체된 경우 (예: `/products/1/main.jpg` 내용만 변경):
- 24시간 동안 이전 이미지가 계속 표시됨
- 해결: URL에 버전 정보 포함 (`/products/1/main.jpg?v=20260328`)

**올바른 설계:**
- 이미지 URL에 내용 해시 포함: `/products/1/main_abc123.jpg`
- CDN Purge API로 강제 무효화
- 또는 `max-age` 단축 (1시간 등) + `stale-while-revalidate`

</details>

---

**Q2.** Vary: Accept-Encoding을 CDN에 설정했을 때 어떤 일이 일어나는가? Vary: Authorization은 왜 문제가 되는가?

<details>
<summary>해설 보기</summary>

**Vary: Accept-Encoding:**

CDN은 같은 URL이라도 `Accept-Encoding` 헤더 값에 따라 별도 캐시 항목을 유지합니다.

```
GET /data.json (Accept-Encoding: gzip)    → gzip 응답 캐시
GET /data.json (Accept-Encoding: br)      → brotli 응답 캐시
GET /data.json (Accept-Encoding: identity)→ 원본 응답 캐시
```

→ 적절한 사용: 압축 방식에 따라 다른 응답을 캐시 (올바른 동작)

**Vary: Authorization의 문제:**

`Authorization` 헤더는 사용자마다 다른 값을 가집니다.

```
GET /api/data (Authorization: Bearer token-user-A) → A의 응답 캐시
GET /api/data (Authorization: Bearer token-user-B) → B의 응답 캐시
GET /api/data (Authorization: Bearer token-user-C) → C의 응답 캐시
...
```

→ 사용자 수만큼 캐시 항목 생성 → CDN 메모리 폭발
→ 캐시 히트율 0% (각 사용자가 자신의 캐시만 사용)
→ CDN이 개인 데이터를 저장하는 보안 위험

**올바른 접근:**
- Authorization이 필요한 API: `Cache-Control: private, no-cache` 또는 `no-store`
- CDN은 인증 없는 공개 리소스만 캐시
- 사용자별 개인화 데이터는 브라우저 캐시 또는 애플리케이션 레벨 캐시 사용

</details>

---

**Q3.** 동시에 수천 명이 같은 캐시 만료 시각에 Origin 서버로 요청이 몰리는 "Thundering Herd" 문제를 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**Thundering Herd (Cache Stampede):**

모든 CDN 노드 또는 사용자의 캐시가 같은 시각에 만료되면, 그 순간 수천 개의 요청이 동시에 Origin으로 몰립니다.

**해결 방법 1: Stale-while-revalidate**
```
Cache-Control: max-age=300, stale-while-revalidate=60
```
- 만료 후 60초 동안 이전 캐시 제공하면서 백그라운드에서 하나만 Origin 요청
- 동시 Origin 요청을 1개로 줄임

**해결 방법 2: CDN Request Collapsing (Request Coalescing)**
- 대부분의 CDN이 기본 지원
- 같은 URL에 대한 동시 요청을 1개의 Origin 요청으로 합침
- Origin 응답을 기다리는 동안 다른 요청은 대기

**해결 방법 3: 만료 시각 분산 (Jitter)**
```
max-age = 기본시간 + 랜덤(0, 기본시간×0.1)
```
- 서버 측에서 응답마다 약간 다른 max-age 설정
- 모든 캐시가 동시에 만료되는 것 방지

**해결 방법 4: Cache Lock (Mutex)**
- 첫 번째 재검증 요청이 락을 획득
- 다른 요청은 락이 해제될 때까지 대기 또는 stale 응답 수신
- Redis나 메모리 기반 락 구현 필요

**Spring Boot + Redis 예시:**
```java
public String getCachedData(String key) {
    String cached = redis.get(key);
    if (cached != null) return cached;
    
    // 락 획득 시도
    boolean locked = redis.setnx("lock:" + key, "1", 5s);
    if (locked) {
        try {
            String fresh = fetchFromDB();
            redis.set(key, fresh, Duration.ofMinutes(5));
            return fresh;
        } finally {
            redis.del("lock:" + key);
        }
    } else {
        // 락 못 얻으면 잠시 대기 후 재시도 또는 stale 반환
        Thread.sleep(100);
        return redis.get(key);  // 다른 스레드가 갱신했을 것
    }
}
```

</details>

---

<div align="center">

**[⬅️ 이전: HTTP/3](./03-http3-quic.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP 메서드와 상태 코드 ➡️](./05-http-methods-status-codes.md)**

</div>
