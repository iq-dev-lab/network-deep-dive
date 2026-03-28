# 서킷 브레이커 — 장애 전파 방지와 Half-Open 상태

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Closed → Open → Half-Open 각 상태에서 요청이 어떻게 처리되는가?
- 실패율(Failure Rate) 기반과 슬로우 콜(Slow Call) 기반 임계값의 차이는?
- Resilience4j가 슬라이딩 윈도우로 실패율을 계산하는 내부 구조는?
- Fallback 전략(캐시 응답, 기본값 반환)을 어떻게 설계하는가?
- 서킷 브레이커와 재시도(Retry)를 함께 사용할 때 주의할 점은?
- Half-Open 상태에서 "탐침 요청"은 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"결제 서비스가 느려졌더니 주문 서비스 전체가 다운됐다":
  서킷 브레이커 없음:
  결제 서비스 응답 30초 소요
  주문 서비스: 결제 API 호출 → 30초 블로킹
  주문 서버 스레드 100개 × 30초 대기 = 3000초 동안 100 스레드 점유
  → 새 주문 요청: 스레드 없음 → 거부 → 주문 서비스 다운!
  
  서킷 브레이커 있음:
  결제 서비스 응답 느림 → 실패율 임계값 초과
  → CB OPEN: 결제 API 호출 즉시 차단 (30초 대기 없음)
  → 주문 서비스: 빠른 Fallback 응답 (캐시된 값 또는 임시 메시지)
  → 스레드 해방 → 주문 서비스 정상 동작 유지

"마이크로서비스 연쇄 장애":
  A → B → C → D (의존성 체인)
  D 장애 → C가 D 응답 기다림 → B가 C 기다림 → A 다운
  서킷 브레이커 없으면 장애가 상위로 전파
  
  서킷 브레이커:
  D 장애 → C의 CB OPEN → C 빠른 응답 (Fallback)
  → B, A는 정상 동작 (장애 격리)
```

---

## 😱 흔한 실수

```
Before — 서킷 브레이커를 모를 때:

실수 1: 서킷 브레이커를 HTTP timeout만으로 대체
  connectTimeout=5s, readTimeout=5s 설정
  → 실패한 요청마다 5초 대기
  → 100 동시 요청 × 5초 = 500초 스레드 점유
  → 서킷 브레이커는 빠른 실패(Fail Fast)로 즉시 해방

실수 2: OPEN → CLOSED 자동 전환 기대
  서킷 브레이커는 OPEN → Half-Open → (성공 시) CLOSED
  Half-Open에서 탐침 요청이 실패하면 다시 OPEN
  → 자동 복구는 되지만 Half-Open을 거쳐야 함
  → 운영자가 수동으로 CLOSED로 강제할 수도 있음

실수 3: Retry + 서킷 브레이커 순서 오류
  잘못된 순서: CB(Retry(외부 API 호출))
  Retry가 CB 안에 있음 → 각 시도가 CB의 실패 카운터 증가
  올바른 순서: Retry(CB(외부 API 호출))
  CB 안에서 재시도 → CB에는 최종 결과만 기록

실수 4: 모든 예외를 실패로 카운트
  HTTP 404: 비즈니스 오류 (존재하지 않는 리소스) → CB 실패 카운트 불필요
  HTTP 503: 서비스 장애 → CB 실패 카운트 필요
  
  recordExceptions(CallNotPermittedException.class)  # 잘못
  ignoreExceptions(BusinessNotFoundException.class)  # 올바름 (비즈니스 오류 제외)
```

---

## ✨ 올바른 접근

```
After — 서킷 브레이커를 알면:

Resilience4j 기본 설정:
  resilience4j:
    circuitbreaker:
      instances:
        paymentService:
          # 슬라이딩 윈도우 설정
          sliding-window-type: COUNT_BASED  # 또는 TIME_BASED
          sliding-window-size: 100           # 최근 100개 요청
          minimum-number-of-calls: 20        # 최소 20개 후 평가
          
          # CLOSED → OPEN 조건
          failure-rate-threshold: 50         # 실패율 50% 이상
          slow-call-rate-threshold: 50       # 슬로우 콜 비율 50% 이상
          slow-call-duration-threshold: 2000ms # 2초 이상 = 슬로우 콜
          
          # OPEN → Half-Open 전환
          wait-duration-in-open-state: 30s   # 30초 후 Half-Open
          
          # Half-Open 설정
          permitted-number-of-calls-in-half-open-state: 5  # 탐침 요청 5개
          
          # 기록할 예외 (나머지는 무시)
          record-exceptions:
            - java.io.IOException
            - java.util.concurrent.TimeoutException
          ignore-exceptions:
            - com.example.BusinessException  # 비즈니스 오류 제외

Spring Boot 통합:
  @Service
  @RequiredArgsConstructor
  public class PaymentServiceClient {
      private final RestTemplate restTemplate;
      
      @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
      @Retry(name = "paymentService")  # CB 밖에 Retry
      public PaymentResponse requestPayment(PaymentRequest request) {
          return restTemplate.postForObject(
              "http://payment-service/api/payments",
              request, PaymentResponse.class);
      }
      
      // Fallback
      public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
          log.warn("Payment service unavailable, using fallback: {}", ex.getMessage());
          // 방법 1: 캐시에서 조회
          // 방법 2: 임시 대기열에 저장 후 나중에 처리
          // 방법 3: 기본값 반환
          return PaymentResponse.pending("결제가 잠시 지연됩니다. 결과를 이메일로 알려드립니다.");
      }
  }
```

---

## 🔬 내부 동작 원리

### 1. 서킷 브레이커 상태 머신

```
세 가지 상태:

CLOSED (정상):
  ┌─────────────────────────────────────────────────┐
  │  모든 요청 통과                                    │
  │  실패율 측정 중                                    │
  │  실패율 ≥ threshold → OPEN 전환                    │
  └─────────────────────────────────────────────────┘
  
  클라이언트 → CB(CLOSED) → 외부 서비스
  CB: 요청 기록 (성공/실패/슬로우)

OPEN (차단):
  ┌─────────────────────────────────────────────────┐
  │  모든 요청 즉시 차단 (외부 서비스 호출 없음)             │
  │  CallNotPermittedException 즉시 반환              │
  │  wait-duration 경과 후 → Half-Open 전환            │
  └─────────────────────────────────────────────────┘
  
  클라이언트 → CB(OPEN) → 즉시 예외 (Fallback 실행)
  CB: 외부 서비스 불필요한 부하 차단

Half-Open (복구 탐침):
  ┌─────────────────────────────────────────────────┐
  │  제한된 수의 요청만 통과 (탐침)                        │
  │  permitted-number-of-calls-in-half-open-state   │
  │                                                 │
  │  탐침 성공 ≥ threshold → CLOSED 전환               │
  │  탐침 실패 ≥ threshold → 다시 OPEN 전환             │
  └─────────────────────────────────────────────────┘
  
  클라이언트 → CB(Half-Open) →
    - 탐침 자리 있음: 외부 서비스 호출 (실제 확인)
    - 탐침 자리 없음: 즉시 차단

상태 전환 다이어그램:
         실패율 초과
  CLOSED ──────────────────► OPEN
    ▲                           │
    │ 탐침 성공                   │ wait-duration 경과
    │                           ▼
    └────────────────── Half-Open
         탐침 실패 → OPEN (반복)
```

### 2. Sliding Window — 실패율 계산

```
COUNT_BASED 슬라이딩 윈도우:
  최근 N개 요청을 원형 배열로 유지
  각 슬롯: SUCCESS, FAILURE, SLOW_SUCCESS, SLOW_FAILURE
  
  window-size=5, 최근 요청: [S, S, F, F, F]
  실패율 = 3/5 = 60% → threshold=50%이면 OPEN
  
  새 요청 결과 S 추가: [S, F, F, F, S] (가장 오래된 S 제거)
  실패율 = 3/5 = 60% → 여전히 OPEN

TIME_BASED 슬라이딩 윈도우:
  최근 N초 동안의 요청 집계
  window-size=10s → 최근 10초의 요청
  
  11초 전 요청들은 자동으로 집계에서 제외
  → 시간 경과로 이전 실패가 사라지면 자동 회복 가능

Resilience4j 내부 구조:
  @ThreadSafe
  class SlidingWindowMetrics {
      // 원형 배열로 구현
      Measurement[] measurements;
      int headIndex;  // 현재 위치
      
      long totalFailures;
      long totalSlowCalls;
      long totalCalls;
      
      synchronized void record(long duration, Outcome outcome) {
          // headIndex 위치에 새 측정 저장
          Measurement old = measurements[headIndex];
          measurements[headIndex] = new Measurement(duration, outcome);
          headIndex = (headIndex + 1) % windowSize;
          
          // 이전 측정 제거, 새 측정 추가
          updateCounters(old, outcome);
      }
      
      float getFailureRate() {
          return (float) totalFailures / totalCalls * 100;
      }
  }

실패 카운트 기준:
  실패: IOException, TimeoutException 등 recordExceptions에 해당
  슬로우 콜: slow-call-duration-threshold 초과 (성공이라도!)
  성공: 위에 해당 안 하는 모든 응답

주의: 슬로우 콜도 CB에 영향!
  응답은 왔지만 3초가 걸렸다 → slow-call-rate 증가
  → 느린 외부 API → slow-call-threshold 초과 → OPEN
  → 빠른 Fallback으로 전환
```

### 3. Half-Open — 탐침 전략

```
Half-Open 진입:
  OPEN 상태에서 wait-duration(30초) 경과
  → Half-Open으로 자동 전환
  → permitted-number-of-calls-in-half-open-state개 탐침 허용

탐침 요청 처리:
  permitted=5, 5개 탐침 슬롯 있음
  
  요청 1: 탐침 자리 있음 → 외부 서비스 호출 → 결과 기록
  요청 2: 탐침 자리 있음 → 외부 서비스 호출 → 결과 기록
  ...
  요청 6: 탐침 자리 없음 (5개 이미 진행 중) → 즉시 차단

탐침 완료 후 판단:
  5개 탐침 결과: [S, S, F, S, S] → 실패율 20%
  threshold=50%: 20% < 50% → CLOSED (복구!)
  
  5개 탐침 결과: [F, F, S, F, F] → 실패율 80%
  threshold=50%: 80% ≥ 50% → OPEN (다시 차단)

Half-Open에서의 Fallback:
  탐침 자리 없는 요청: CallNotPermittedException
  → Fallback 실행 (OPEN과 동일하게)

정교한 Half-Open 전략:
  단일 탐침 (permitted=1):
    실패 위험 분산 없음, 하나의 실패로 OPEN 복귀
    → 서비스가 완전히 안정된 후 CLOSED 전환
  
  다중 탐침 (permitted=10):
    더 많은 데이터로 판단 → 더 신뢰할 수 있는 결정
    → 진짜 복구인지 일시적 성공인지 구분
  
  권장: permitted=5~10 (일반적)
```

### 4. Fallback 전략 설계

```
Fallback의 목적:
  외부 서비스 장애 시 최소한의 기능 유지
  사용자에게 적절한 대응 (무한 대기 대신 빠른 응답)

Fallback 전략 유형:

1. 캐시된 응답 반환:
  @CircuitBreaker(name = "productService", fallbackMethod = "cachedProducts")
  public List<Product> getProducts() {
      return productServiceClient.getProducts();
  }
  
  public List<Product> cachedProducts(Exception ex) {
      // Redis 캐시에서 마지막 성공 응답 반환
      return cacheService.getProducts().orElse(Collections.emptyList());
  }
  
  → 최신 데이터는 아니지만 서비스 유지 가능
  → TTL로 캐시 신선도 관리

2. 기본값 반환:
  public RecommendationResponse cachedRecommendations(Exception ex) {
      // 개인화 추천 실패 시 베스트셀러 반환
      return RecommendationResponse.of(defaultBestsellers());
  }
  
  → 품질은 낮아지지만 서비스 계속 동작

3. 비동기 처리 (저장 후 나중에):
  public PaymentResponse paymentFallback(PaymentRequest req, Exception ex) {
      // 결제 요청을 큐에 저장
      paymentQueue.enqueue(req);
      return PaymentResponse.pending("결제 처리 중. 이메일로 결과를 알려드립니다.");
  }
  
  → 완전한 기능 손실 없이 나중에 처리

4. Degraded Mode:
  public OrderResponse orderFallback(OrderRequest req, Exception ex) {
      // 재고 확인 없이 주문 임시 수락 (재고 나중에 확인)
      return orderService.createPendingOrder(req);
  }
  
  → 정확도 희생, 가용성 유지

Fallback에서 주의할 점:
  Fallback 자체도 실패할 수 있음!
  → Fallback에도 간단한 예외 처리 필요
  
  Fallback이 너무 복잡하면:
  → Fallback에서 다른 외부 서비스 호출 → 또 다른 장애 가능성
  → Fallback은 단순하고 빠르게 (로컬 데이터, 기본값)
```

### 5. Retry와 서킷 브레이커 조합

```
잘못된 조합 (CB 안에 Retry):
  CB(Retry(외부 호출))
  
  외부 서비스 실패:
  Retry: 3번 재시도 → 3번 모두 실패 → 예외 반환
  CB: 1번 호출 = 3번 실패 카운트 (Retry 각각 카운트)
  → CB가 너무 빨리 OPEN

올바른 조합 (Retry 밖에 CB):
  Retry(CB(외부 호출))
  
  외부 서비스 실패:
  CB(외부 호출): 1번 실패 → 예외
  Retry: 재시도 → CB(외부 호출): 1번 실패
  Retry: 재시도 → CB(외부 호출): 1번 실패 → 모두 실패
  CB: 3번 실패 카운트 (각 재시도가 독립적으로 CB에 기록)
  → Retry 최대 횟수 후 CB가 OPEN 가능 (더 자연스러움)

@Retry + @CircuitBreaker 순서:
  Resilience4j 어노테이션 순서 주의
  @Retry(name = "service")
  @CircuitBreaker(name = "service", fallbackMethod = "fallback")
  public Response call() { ... }
  
  → 실제 실행 순서: CB(Retry(call))
  → 어노테이션은 반대로 적용되므로 Retry 먼저 선언 = Retry 바깥

권장 구성:
  @TimeLimiter(name = "service")   # 타임아웃 제한 (가장 안쪽)
  @CircuitBreaker(name = "service") # CB (가운데)
  @Retry(name = "service")          # 재시도 (가장 바깥)
  public Response call() { ... }
  
  실행 순서: Retry(CB(TimeLimiter(call())))
  → 타임아웃 → CB 판단 → Retry
```

---

## 💻 실전 실험

### 실험 1: Resilience4j 상태 전환 관찰

```java
// 테스트 설정
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .slidingWindowSize(10)
    .minimumNumberOfCalls(5)
    .waitDurationInOpenState(Duration.ofSeconds(5))
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

CircuitBreaker cb = CircuitBreakerRegistry.of(config)
    .circuitBreaker("test");

// 상태 변화 모니터링
cb.getEventPublisher()
    .onStateTransition(e -> System.out.println(
        "State: " + e.getStateTransition()));

// 실패 주입
for (int i = 0; i < 6; i++) {
    try {
        cb.executeSupplier(() -> { throw new RuntimeException("fail"); });
    } catch (Exception e) {
        System.out.println("Call " + i + ": " + cb.getState());
    }
}
// CLOSED → CLOSED → CLOSED → CLOSED → CLOSED → OPEN

// OPEN 상태에서 즉시 거부 확인
try {
    cb.executeSupplier(() -> "success");
} catch (CallNotPermittedException e) {
    System.out.println("OPEN: 즉시 거부됨 " + cb.getState());
}
```

### 실험 2: Fallback 동작 확인

```bash
# 외부 서비스 다운 시뮬레이션
docker stop payment-service

# 처음 몇 요청: CLOSED, 실패 기록
for i in $(seq 1 5); do
  curl -s http://localhost:8080/api/orders -d '{"items": ["item1"]}' \
    -H "Content-Type: application/json" | jq .status
done
# "error" ... "error" ... (실패 기록)

# 임계값 초과 후: OPEN, Fallback 즉시 반환
for i in $(seq 1 5); do
  curl -s http://localhost:8080/api/orders -d '{"items": ["item1"]}' \
    -H "Content-Type: application/json" | jq .
done
# "pending": "결제 처리 중. 이메일로 알려드립니다." (Fallback)

# 서비스 복구
docker start payment-service

# Half-Open 탐침 대기 (wait-duration 경과)
sleep 30

# 복구 확인
curl -s http://localhost:8080/api/orders -d '{"items": ["item1"]}' \
  -H "Content-Type: application/json" | jq .status
# "success" (CLOSED 복구)
```

### 실험 3: Actuator로 CB 상태 모니터링

```bash
# 서킷 브레이커 상태 확인 (Spring Actuator)
curl http://localhost:8080/actuator/circuitbreakers | python3 -m json.tool
# {
#   "circuitBreakers": {
#     "paymentService": {
#       "state": "CLOSED",
#       "failureRate": "20.00%",
#       "slowCallRate": "0.00%",
#       "numberOfBufferedCalls": 10,
#       "numberOfFailedCalls": 2
#     }
#   }
# }

# 상태 강제 변경 (테스트용)
curl -X POST http://localhost:8080/actuator/circuitbreakers/paymentService/OPEN
curl -X POST http://localhost:8080/actuator/circuitbreakers/paymentService/CLOSED

# 상태 변화 이벤트 스트림
curl http://localhost:8080/actuator/circuitbreakerevents/paymentService
```

### 실험 4: Prometheus + Grafana 모니터링

```yaml
# application.yml
management:
  metrics:
    export:
      prometheus:
        enabled: true
  endpoints:
    web:
      exposure:
        include: prometheus, circuitbreakers

# PromQL 쿼리
# 서킷 브레이커 상태 (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
resilience4j_circuitbreaker_state{name="paymentService"}

# 실패율
resilience4j_circuitbreaker_failure_rate{name="paymentService"}

# 슬로우 콜 비율
resilience4j_circuitbreaker_slow_call_rate{name="paymentService"}

# 알람 설정
# OPEN 상태가 5분 이상 지속 시 알람
```

---

## 📊 성능/비용 비교

```
서킷 브레이커 유무 비교:

외부 서비스 장애 시나리오:
  응답 시간: 30초 (타임아웃)
  동시 요청: 100개
  
  CB 없음:
    100 요청 × 30초 = 3000초 스레드 점유
    서버 스레드 고갈 → 새 요청 거부
    서버 전체 응답 불가 → 연쇄 장애
    복구 시간: 외부 서비스 복구 + 스레드 해방까지 (수 분)
  
  CB 있음 (OPEN 상태):
    요청 즉시 차단 → 스레드 해방
    Fallback 처리: ~1ms
    서버 다른 기능 정상 동작
    외부 서비스 복구 후 Half-Open → CLOSED → 자동 복구

Resilience4j 오버헤드:
  CLOSED 상태 (정상): ~0.1ms (슬라이딩 윈도우 업데이트)
  OPEN 상태 (차단): ~0.01ms (즉시 예외, 거의 없음)
  → 성능 영향 미미
```

---

## ⚖️ 트레이드오프

```
임계값 설정:

실패율 threshold 낮음 (20%):
  작은 문제에도 OPEN → 과도한 차단
  일시적 오류에도 서비스 중단
  → 안정적이지 않은 서비스 보호에 유리

실패율 threshold 높음 (80%):
  심각한 장애만 차단
  경미한 장애 시 스레드 낭비 지속
  → 매우 안정적인 서비스에 보수적 적용

권장: 50% (표준 시작점), 실제 트래픽 패턴에 맞게 조정

wait-duration 설정:
  짧음 (5초):
    빠른 복구 탐침 → 외부 서비스 부하 증가 가능
    Half-Open → OPEN 반복 (외부 서비스 부분 복구 시)
  
  길음 (300초):
    외부 서비스 충분한 복구 시간
    하지만 긴 서비스 중단 (Fallback 계속)
  
  권장: 30~60초 (외부 서비스 재시작 시간 고려)

minimum-number-of-calls:
  너무 작음 (5): 통계적 의미 없음, 오탐 가능
  너무 큼 (100): 장애 감지 늦음 (100개 이후에야 평가)
  
  권장: 20~50 (트래픽에 따라)

슬라이딩 윈도우 타입:
  COUNT_BASED: 요청 수 기준 (트래픽 일정할 때)
  TIME_BASED: 시간 기준 (트래픽 변동 클 때)
  → 야간 저트래픽에서 COUNT_BASED 사용 시: 100개 채우는 데 오래 걸림
     → TIME_BASED 권장 (10초~1분 윈도우)
```

---

## 📌 핵심 정리

```
서킷 브레이커 핵심 요약:

상태 머신:
  CLOSED: 정상, 실패율 측정 → threshold 초과 시 OPEN
  OPEN:   즉시 차단, Fallback 실행 → wait-duration 후 Half-Open
  Half-Open: 제한된 탐침 → 성공 시 CLOSED, 실패 시 OPEN

실패 조건:
  recordExceptions에 해당하는 예외
  슬로우 콜 (slow-call-duration-threshold 초과)
  두 가지 각각 threshold 설정 가능

Sliding Window:
  COUNT_BASED: 최근 N개 요청의 실패율
  TIME_BASED:  최근 N초의 실패율
  원형 배열로 효율적 구현

Fallback 전략:
  캐시 응답: 최신성 약간 희생, 가용성 유지
  기본값: 개인화 포기, 기능 유지
  비동기 큐: 나중에 처리, 완전한 기능 유지
  Degraded Mode: 정확도 희생, 가용성 극대화

Retry 조합:
  올바른 순서: Retry > CB > TimeLimiter (바깥부터)
  어노테이션: 실제 실행은 역순

모니터링:
  Spring Actuator: /actuator/circuitbreakers
  Prometheus: resilience4j_circuitbreaker_* 메트릭
  OPEN 5분 이상 → 알람

실무 권장 설정:
  failure-rate-threshold: 50%
  minimum-number-of-calls: 20
  sliding-window-size: 100
  wait-duration-in-open-state: 30s
  permitted-number-of-calls-in-half-open-state: 5
```

---

## 🤔 생각해볼 문제

**Q1.** 서킷 브레이커가 OPEN 상태인데 외부 서비스가 복구됐다. 사용자가 Fallback 응답을 최소화하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**기본 메커니즘:** OPEN → wait-duration(30초) → Half-Open → 탐침 성공 → CLOSED

**대기 최소화 방법:**

1. **wait-duration 단축:**
```yaml
wait-duration-in-open-state: 10s  # 30초 → 10초로 단축
```
단점: 외부 서비스가 완전히 복구 안 된 상태에서 탐침 → 다시 OPEN 반복

2. **Exponential Backoff (점진적 증가):**
```yaml
# Resilience4j 2.x
max-wait-duration-in-half-open-state: 0  # Half-Open에서 무한 대기 없음
```
wait-duration을 재시도마다 증가: 10s → 30s → 60s → 120s
외부 서비스 복구 시간에 맞춤

3. **수동 전환 (운영자 개입):**
```bash
# 외부 서비스 복구 확인 후 수동 CLOSED 전환
curl -X POST http://localhost:8080/actuator/circuitbreakers/paymentService/CLOSED
```
가장 빠르지만 사람이 필요함

4. **Health Check 기반 자동 복구:**
```java
// 외부 서비스 헬스 체크 주기적으로 실행
@Scheduled(fixedDelay = 5000)
public void checkAndRecoverCircuitBreaker() {
    if (cb.getState() == OPEN) {
        try {
            healthCheck.check();  // 외부 서비스 헬스 체크
            cb.transitionToHalfOpenState();  // 즉시 Half-Open
        } catch (Exception e) {
            // 아직 복구 안 됨
        }
    }
}
```

**권장:** 자동화가 최선. wait-duration 10~30초로 줄이고, 외부 서비스의 평균 복구 시간을 측정해서 맞춤 설정.

</details>

---

**Q2.** 서킷 브레이커의 `minimum-number-of-calls`가 너무 작으면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**`minimum-number-of-calls`의 역할:**
임계값 평가를 시작하는 최소 요청 수. 이 수에 도달하기 전에는 실패율이 아무리 높아도 OPEN으로 전환하지 않습니다.

**너무 작은 경우 (예: minimum=2, threshold=50%):**

- 2개 요청 중 1개 실패 → 실패율 50% → 즉시 OPEN
- 일시적 네트워크 오류 1개, 정상 1개 → OPEN
- 저트래픽 시간대(새벽)에 2개 요청이면 바로 OPEN 가능

**결과:**
- 오탐(False Positive): 실제 장애 없이 CB OPEN
- Fallback 응답이 사용자에게 불필요하게 노출
- 서비스 복구 후에도 half-open 과정 거쳐야 함

**너무 큰 경우 (예: minimum=1000, threshold=50%):**
- 500개 실패 후에야 OPEN → 그 동안 500번의 30초 타임아웃
- 15,000초(4시간)의 스레드 낭비

**권장 설정:**
```
트래픽에 따른 minimum-number-of-calls:
- 낮은 트래픽 (< 10 rps): minimum=10~20
- 중간 트래픽 (10~100 rps): minimum=20~50
- 높은 트래픽 (> 100 rps): minimum=50~100

원칙: 1~2분치 예상 요청 수의 20~30%
```

**시간 기반 윈도우가 더 적합한 경우:**
새벽 저트래픽 시 COUNT_BASED로 minimum-number-of-calls에 도달 못함 → CB 작동 안 함. TIME_BASED(30초 윈도우)로 전환하면 트래픽에 무관하게 동작.

</details>

---

**Q3.** 서킷 브레이커 Fallback에서 다른 외부 서비스를 호출하는 것이 왜 위험한가?

<details>
<summary>해설 보기</summary>

**문제: Fallback의 복잡성 증가 → 새로운 장애 지점**

```java
// 위험한 Fallback
public List<Product> getProductsFallback(Exception ex) {
    // Fallback에서 다른 외부 서비스 호출!
    return backupProductService.getProducts();  // 이것도 장애 가능
}
```

**시나리오:**
1. 주 서비스(ProductService) 장애 → CB OPEN
2. Fallback 실행 → 백업 서비스 호출
3. 백업 서비스도 장애 → Fallback도 실패
4. 결과: 아무 응답도 없음 (주 서비스 + 백업 서비스 모두 실패)

**더 나쁜 케이스:**
```java
public List<Product> getProductsFallback(Exception ex) {
    // Fallback에서 또 다른 외부 서비스
    List<Product> backup = backupService.getProducts();
    // 추가로 캐시 서비스 호출
    cacheService.warmup(backup);  // 이것도 실패 가능
    return backup;
}
```
연쇄 호출 → 연쇄 장애 가능

**올바른 Fallback 설계:**
```java
public List<Product> getProductsFallback(Exception ex) {
    // 선택 1: 로컬 메모리 캐시 (외부 호출 없음)
    return localCache.get("products").orElse(DEFAULT_PRODUCTS);
    
    // 선택 2: 정적 기본값
    return List.of(
        new Product("default1", "기본 상품"),
        new Product("default2", "기본 상품")
    );
    
    // 선택 3: 예외를 명확히 전달
    throw new ServiceUnavailableException("상품 서비스를 일시적으로 이용할 수 없습니다.");
}
```

**원칙:**
- Fallback은 **로컬 데이터나 정적 기본값** 사용
- 외부 네트워크 호출 최소화
- Fallback 자체의 CB가 필요하면 서비스 아키텍처 재검토

**예외적 허용:**
```java
// 백업 서비스가 별도 CB로 보호된 경우
@CircuitBreaker(name = "backupService")
public List<Product> getProductsFallback(Exception ex) {
    return backupService.getProducts();  // 별도 CB로 보호
}
```
이 경우도 두 서비스 모두 장애 시 최종 Fallback(정적 기본값) 필요.

</details>

---

<div align="center">

**[⬅️ 이전: Rate Limiting](./04-rate-limiting-algorithms.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — 컨테이너와 네트워크 실습 ➡️](../container-and-network-practice/01-docker-networking.md)**

</div>
