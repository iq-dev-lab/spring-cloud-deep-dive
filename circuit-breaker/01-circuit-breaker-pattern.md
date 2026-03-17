# Circuit Breaker 패턴과 필요성 — 연쇄 장애를 차단하는 분산 시스템 방어 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 분산 환경에서 Cascading Failure(연쇄 장애)가 발생하는 구체적인 메커니즘은?
- Thread Pool 고갈이 Gateway 또는 호출 서비스에서 어떻게 전체 장애로 이어지는가?
- Circuit Breaker 패턴은 이 문제를 어떤 원리로 해결하는가?
- Resilience4j가 Netflix Hystrix와 다른 핵심 설계 철학은 무엇인가?
- Hystrix가 Maintenance 상태가 된 이유와 Resilience4j가 표준이 된 배경은?
- `@CircuitBreaker` 어노테이션이 AOP 프록시를 통해 동작하는 방식의 개요는?

---

## 🔍 왜 MSA에서 필요한가

### Cascading Failure — 하나의 장애가 전체로 번지는 원리

```
MSA 호출 체인:

  Client → Service A → Service B → Service C (DB 장애)

Service C(DB) 장애 시나리오:
  
  ① Service C: DB 연결 실패 → 응답 지연 (타임아웃 대기 중)
  
  ② Service B: Service C 응답 대기
     스레드 풀 200개 중 하나 점유 (30초 타임아웃 대기)
     요청이 계속 들어옴 → 200개 스레드 전부 C 응답 대기
     → 새 요청 큐에 쌓임
     → 큐도 꽉 참 → Service B: 503 Service Unavailable
  
  ③ Service A: Service B가 503 반환
     Service A도 B 응답 대기 → 스레드 고갈
     → Service A: 503
  
  ④ 최종 결과:
     Service C DB 하나의 장애가
     Service B, Service A, 클라이언트까지 전체 장애로 전파

  핵심 문제:
  "장애 서비스를 계속 호출하는 것" 자체가 장애를 증폭시킴
  → Circuit Breaker: 장애 서비스 호출을 차단
```

---

## 😱 잘못된 구성

### Before: Circuit Breaker 없이 타임아웃만 설정

```java
// ❌ 타임아웃만 있고 Circuit Breaker 없는 구성
@Service
public class OrderService {

    private final RestTemplate restTemplate;

    // connect-timeout: 1s, read-timeout: 3s
    public Inventory checkInventory(Long productId) {
        // 타임아웃 3초 설정 → 3초 대기 후 실패
        return restTemplate.getForObject(
            "http://inventory-service/inventory/" + productId,
            Inventory.class);
    }
}
```

```
inventory-service 다운 시:
  모든 요청이 3초씩 대기
  동시 요청 100개 → 모든 스레드 3초 블로킹
  → 3초 × 100개 = 300초 분량의 스레드 점유
  → 새 요청 처리 불가

  타임아웃을 짧게 줄이면?
  → 정상 응답이 느릴 때 false positive (오탐)
  → 타임아웃 조정으로는 근본 해결 안 됨

Circuit Breaker 있으면:
  inventory-service 5회 연속 실패
  → Circuit OPEN
  → 이후 요청: 타임아웃 대기 없이 즉시 Fallback 반환 (< 1ms)
  → 스레드 풀 고갈 없음
  → 다른 서비스 정상 운영 유지
```

### Before: Hystrix 방식 유지 (Maintenance 상태)

```java
// ❌ Hystrix (2018년 이후 Maintenance)
@HystrixCommand(fallbackMethod = "fallback")
public Order getOrder(Long id) {
    return orderClient.getOrder(id);
}
// Hystrix는 내부적으로 ThreadPool 기반 격리
// 각 Command마다 별도 스레드 풀 → 리소스 낭비
// Reactive(WebFlux) 미지원 → 현대 MSA에 부적합
```

---

## ✨ 올바른 패턴

### After: Circuit Breaker 동작 원리

```
Circuit Breaker 상태 머신:

  ┌─────────────────────────────────────────────┐
  │                   CLOSED                    │
  │  (정상 운영, 모든 요청 통과)                      │
  │  실패율 >= threshold → OPEN으로 전환            │
  └─────────────────────┬───────────────────────┘
                        │ 실패율 초과
                        ▼
  ┌─────────────────────────────────────────────┐
  │                    OPEN                     │
  │  (차단 상태, 모든 요청 즉시 Fallback)             │
  │  waitDurationInOpenState 경과 → HALF_OPEN    │
  └─────────────────────┬───────────────────────┘
                        │ 대기 시간 경과
                        ▼
  ┌─────────────────────────────────────────────┐
  │                 HALF_OPEN                   │
  │  (탐색 상태, 제한된 요청만 통과)                   │
  │  성공 → CLOSED                               │
  │  실패 → OPEN                                 │
  └─────────────────────────────────────────────┘

전기 회로 브레이커와 동일한 원리:
  과부하 시 회로 차단(OPEN) → 장비 보호
  이후 회로 연결(CLOSED) → 정상 동작 재개
```

---

## 🔬 내부 동작 원리

### 1. Resilience4j vs Hystrix 설계 철학

```
Hystrix (Netflix, 2012~2018 Maintenance):

  ① ThreadPool 기반 격리
     각 Command마다 별도 ThreadPool (기본 10개 스레드)
     → 외부 서비스 A: 10개 스레드
     → 외부 서비스 B: 10개 스레드
     → 10개 서비스 × 10개 = 100개 추가 스레드 항상 대기
     → 리소스 낭비 (서비스 수에 비례)

  ② Blocking 전용
     HystrixCommand가 동기 반환
     Reactive(WebFlux) 지원 제한적

  ③ 복잡한 설정 시스템
     HystrixCommandProperties, HystrixThreadPoolProperties 등
     프로퍼티 이름이 장황함

  ④ 2018년 Netflix Maintenance 선언
     Netflix 내부에서 Envoy Proxy로 전환
     Spring Cloud는 대안 필요

Resilience4j (2019~ 표준):

  ① 함수형 데코레이터 패턴
     CheckedSupplier, CheckedFunction 등 함수형 인터페이스 래핑
     람다 기반 → 직관적이고 간결

  ② Semaphore 기반 격리 (기본)
     별도 스레드 풀 없음 → 리소스 효율적
     (ThreadPool 방식도 선택 가능: Bulkhead)

  ③ 완전한 Reactive 지원
     Resilience4j-reactor 모듈
     Mono/Flux 체인에 자연스럽게 통합

  ④ 모듈화 설계
     CircuitBreaker, Bulkhead, RateLimiter, Retry, TimeLimiter
     필요한 모듈만 선택적으로 사용

  ⑤ Spring Boot Actuator 통합
     /actuator/circuitbreakers, /actuator/metrics
     상태 실시간 모니터링
```

### 2. Resilience4j CircuitBreaker 핵심 클래스 구조

```java
// CircuitBreaker 인터페이스 — 핵심 API
public interface CircuitBreaker {

    // ① 함수형 래핑: Supplier를 Circuit Breaker로 감싸기
    <T> CheckedSupplier<T> decorateCheckedSupplier(CheckedSupplier<T> supplier);
    <T> Supplier<T> decorateSupplier(Supplier<T> supplier);
    Runnable decorateRunnable(Runnable runnable);

    // ② 직접 실행
    <T> T executeSupplier(Supplier<T> supplier);
    <T> T executeCheckedSupplier(CheckedSupplier<T> supplier) throws Throwable;

    // ③ 상태 조회
    State getState();
    CircuitBreakerConfig getCircuitBreakerConfig();
    Metrics getMetrics();

    // ④ 수동 상태 전이 (테스트 및 운영)
    void transitionToOpenState();
    void transitionToClosedState();
    void transitionToHalfOpenState();

    enum State {
        CLOSED, OPEN, HALF_OPEN,
        DISABLED,     // 항상 CLOSED (비활성화)
        FORCED_OPEN,  // 항상 OPEN (강제 차단)
        METRICS_ONLY  // 실패 기록만, 차단 없음
    }

    // ⑤ 이벤트 발행 (Pub-Sub)
    CircuitBreakerEventPublisher getEventPublisher();
    // 상태 변경, 성공, 실패, 무시된 예외 등 이벤트 구독 가능
}
```

### 3. @CircuitBreaker AOP 프록시 동작 방식

```java
// @CircuitBreaker: Spring AOP를 통해 동작
// Resilience4j Spring Boot Starter가 자동 설정

// 사용자 코드:
@Service
public class OrderService {

    @CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallback")
    public Inventory checkInventory(Long productId) {
        // 이 메서드가 Circuit Breaker로 감싸짐
        return inventoryClient.getInventory(productId);
    }

    // Fallback 메서드 (파라미터에 예외 타입 추가)
    private Inventory inventoryFallback(Long productId, Exception e) {
        log.warn("Circuit breaker activated for product {}: {}", productId, e.getMessage());
        return Inventory.unavailable();
    }
}

// 내부 동작 (AOP 프록시):
// ① Spring이 OrderService를 CGLIB 프록시로 감쌈
// ② checkInventory() 호출 시 프록시가 가로챔
// ③ CircuitBreakerAspect.around() 실행:

@Around("@annotation(circuitBreaker)")
public Object circuitBreakerAspect(ProceedingJoinPoint joinPoint,
        io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker circuitBreaker)
        throws Throwable {

    String backend = circuitBreaker.name();
    // ④ CircuitBreakerRegistry에서 이름으로 CB 조회
    io.github.resilience4j.circuitbreaker.CircuitBreaker cb =
        circuitBreakerRegistry.circuitBreaker(backend);

    // ⑤ CB 상태 확인: OPEN이면 CallNotPermittedException 발생
    cb.acquirePermission();

    long start = System.nanoTime();
    try {
        // ⑥ 실제 메서드 호출
        Object result = joinPoint.proceed();
        // ⑦ 성공 기록
        cb.onSuccess(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        return result;
    } catch (Exception e) {
        // ⑧ 실패 기록 (설정된 예외만)
        cb.onError(System.nanoTime() - start, TimeUnit.NANOSECONDS, e);
        // ⑨ Fallback 메서드 호출 (있는 경우)
        return callFallback(joinPoint, circuitBreaker.fallbackMethod(), e);
    }
}
```

### 4. Resilience4j 함수형 패턴

```java
// Spring 어노테이션 없이도 프로그래매틱하게 사용 가능

// ① Registry에서 CB 생성
CircuitBreakerRegistry registry = CircuitBreakerRegistry.ofDefaults();
CircuitBreaker cb = registry.circuitBreaker("inventory");

// ② 함수형 래핑
Supplier<Inventory> supplier = CircuitBreaker.decorateSupplier(cb,
    () -> inventoryClient.getInventory(productId));

// ③ Fallback과 함께 실행
Inventory result = Try.ofSupplier(supplier)
    .recover(throwable -> Inventory.unavailable())
    .get();

// Reactive 방식:
Mono<Inventory> reactiveResult = CircuitBreakerOperator.of(cb)
    .apply(Mono.fromSupplier(() -> inventoryClient.getInventory(productId)));
```

---

## 💻 실전 구성

### 의존성 및 기본 설정

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

<!-- Actuator: 상태 모니터링 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      inventory:                        # CB 이름 (@CircuitBreaker name과 일치)
        registerHealthIndicator: true   # Actuator /health에 CB 상태 포함
        slidingWindowSize: 10           # 슬라이딩 윈도우 크기
        minimumNumberOfCalls: 5         # 최소 호출 수 (이하면 CB 미발동)
        failureRateThreshold: 50        # 실패율 50% 이상이면 OPEN
        waitDurationInOpenState: 5s     # OPEN 유지 시간 (이후 HALF_OPEN)
        permittedNumberOfCallsInHalfOpenState: 3  # HALF_OPEN에서 허용 호출 수

management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,metrics
  health:
    circuitbreakers:
      enabled: true
```

### Actuator로 상태 확인

```bash
# Circuit Breaker 상태 전체 조회
curl http://localhost:8080/actuator/circuitbreakers | jq '.'
# {
#   "circuitBreakers": {
#     "inventory": {
#       "failureRate": "30.0%",
#       "slowCallRate": "0.0%",
#       "failureRateThreshold": "50.0%",
#       "slowCallRateThreshold": "100.0%",
#       "bufferedCalls": 10,
#       "failedCalls": 3,
#       "slowCalls": 0,
#       "slowSuccessfulCalls": 0,
#       "notPermittedCalls": 0,
#       "state": "CLOSED"
#     }
#   }
# }

# Health Indicator로 CB 상태 확인
curl http://localhost:8080/actuator/health | jq '.components.circuitBreakers'
# {
#   "status": "UP",
#   "details": {
#     "inventory": { "status": "CIRCUIT_CLOSED" }
#   }
# }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Circuit Breaker가 Cascading Failure를 막는 과정

```
t=0s:   inventory-service DB 장애 발생

t=0~3s: 처음 5번의 요청:
         모두 타임아웃(3초) → 실패
         CB 슬라이딩 윈도우: [실패, 실패, 실패, 실패, 실패]
         실패율: 5/5 = 100% > 50% (threshold)
         minimumNumberOfCalls(5) 충족

t=3s:   CB 상태: CLOSED → OPEN
         이후 요청: 즉시 CallNotPermittedException → Fallback 반환
         응답 시간: ~1ms (타임아웃 3초 대기 없음)
         Service A 스레드 풀: 안전

t=3~8s: CB OPEN 유지 (waitDurationInOpenState: 5s)
         모든 요청 즉시 Fallback

t=8s:   CB → HALF_OPEN
         다음 3개 요청만 inventory-service로 전달

t=8~9s: inventory-service DB 복구됨
         3개 요청 모두 성공
         → CB: HALF_OPEN → CLOSED
         정상 운영 재개

Without CB:
  t=0~120s: 모든 요청 3초 대기 → 서비스 응답 불가

With CB:
  t=0~3s: 초기 실패 (5회)
  t=3~8s: 즉시 Fallback (정상 운영)
  t=8s~: 완전 복구
```

---

## ⚖️ 트레이드오프

| 방식 | 장애 격리 | 리소스 | Reactive | 적합한 환경 |
|------|----------|--------|---------|------------|
| **Hystrix (ThreadPool)** | 강력 | 비효율 (스레드 풀 수 × N) | 미지원 | 레거시 |
| **Resilience4j (Semaphore)** | 충분 | 효율적 | 완전 지원 | 현대 MSA |
| **타임아웃만** | 없음 | 효율적 | - | 단순 환경 |

```
Circuit Breaker를 쓰지 않아도 될 때:
  ✅ 단일 서비스 (장애 전파 없음)
  ✅ 의존 서비스가 매우 안정적 (SLA 99.99%+)
  ✅ 동기 호출이 없는 순수 이벤트 기반 아키텍처

Circuit Breaker가 반드시 필요할 때:
  ✅ 동기 HTTP 호출이 있는 MSA
  ✅ 의존 서비스 장애 시 호출 서비스도 영향받는 구조
  ✅ 스레드 풀 고갈이 우려되는 고트래픽 환경
  ✅ SLA를 보장해야 하는 서비스
```

---

## 📌 핵심 정리

```
Circuit Breaker 패턴 핵심:

문제: Cascading Failure
  장애 서비스 계속 호출 → 타임아웃 대기 → 스레드 고갈 → 전체 장애

해결: 상태 머신으로 호출 차단
  CLOSED: 정상 (모든 요청 통과)
  OPEN: 차단 (즉시 Fallback, 타임아웃 없음)
  HALF_OPEN: 탐색 (제한 요청으로 복구 확인)

Hystrix vs Resilience4j:
  Hystrix: ThreadPool 격리, Blocking, Maintenance
  Resilience4j: Semaphore, Reactive 지원, 함수형 데코레이터

@CircuitBreaker 동작:
  Spring AOP → CircuitBreakerAspect → acquirePermission()
  성공: onSuccess() 기록
  실패: onError() 기록 → Fallback 메서드 호출

실패율 계산: Ch5-02, 03에서 상세
상태 전이: Ch5-02에서 상세
```

---

## 🤔 생각해볼 문제

**Q1.** Circuit Breaker가 CLOSED → OPEN으로 전환될 때 이미 진행 중인 요청(in-flight)들은 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

이미 업스트림으로 전송된 진행 중인 요청들은 중단되지 않고 완료까지 처리됩니다. Circuit Breaker의 OPEN 전환은 새 요청에만 적용됩니다. `acquirePermission()`은 요청 시작 시점에 호출되므로, 이미 permission을 획득하고 실행 중인 요청은 영향받지 않습니다. 이후 `onSuccess()` 또는 `onError()`로 결과가 슬라이딩 윈도우에 기록됩니다. OPEN 전환 직후에는 진행 중 요청들의 응답이 윈도우에 기록되지만, 새 요청 차단은 즉시 시작됩니다.

</details>

---

**Q2.** `@CircuitBreaker`가 AOP 프록시로 동작하므로, 같은 클래스 내에서 `checkInventory()`를 `this.checkInventory()`로 호출하면 Circuit Breaker가 동작하는가?

<details>
<summary>해설 보기</summary>

동작하지 않습니다. Spring AOP는 CGLIB 프록시 기반이므로, 외부에서 Bean을 통해 메서드를 호출할 때만 프록시가 개입합니다. 같은 클래스 내에서 `this.checkInventory()`를 호출하면 프록시를 거치지 않고 직접 메서드가 실행됩니다. 해결 방법: 1) 자신의 Bean을 주입받아 호출 (`@Autowired private OrderService self`), 2) `@CircuitBreaker` 메서드를 별도 Bean으로 분리, 3) `AopContext.currentProxy()`를 활용(권장하지 않음). 이는 Spring의 모든 AOP 기반 어노테이션(`@Transactional`, `@Cacheable` 등)에서 공통적으로 발생하는 제약입니다.

</details>

---

**Q3.** Resilience4j의 `DISABLED`와 `METRICS_ONLY` 상태는 언제 유용한가?

<details>
<summary>해설 보기</summary>

**DISABLED**: Circuit Breaker 로직을 완전히 비활성화하지만 코드는 유지합니다. 성능 테스트나 CB 없이 테스트할 때 유용합니다. `transitionToDisabledState()`로 전환합니다.

**METRICS_ONLY**: 실패/성공 메트릭을 수집하지만 실제로 호출을 차단하지 않습니다. 새 서비스 도입 시 먼저 메트릭을 수집해 적절한 threshold를 결정하는 데 유용합니다. "관찰 모드"라고도 부릅니다. 실제 트래픽 패턴을 보고 나서 올바른 `failureRateThreshold`, `slidingWindowSize`, `waitDurationInOpenState` 값을 설정한 뒤 CLOSED 상태로 전환합니다.

실무에서 새 서비스에 Circuit Breaker를 적용할 때 `METRICS_ONLY`로 2주 정도 운영하면서 메트릭을 수집하고, 이를 기반으로 설정값을 최적화한 후 정식 활성화하는 것이 좋은 관행입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: CLOSED → OPEN → HALF_OPEN 상태 전이 ➡️](./02-state-machine-transitions.md)**

</div>
