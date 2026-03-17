# Fallback 메서드 작성 — AOP 프록시가 예외를 가로채는 시점과 Fallback 규칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@CircuitBreaker(fallbackMethod=...)` AOP Aspect가 Fallback을 호출하는 정확한 코드 경로는?
- Fallback 메서드가 가져야 하는 파라미터 시그니처 규칙은 무엇인가?
- `CallNotPermittedException`(OPEN 상태)과 실제 업스트림 예외를 Fallback에서 구분하는 방법은?
- Fallback 체이닝 — Fallback의 Fallback은 어떻게 구현하는가?
- Fallback 메서드가 예외를 던지면 어떻게 처리되는가?
- Reactive(`Mono`/`Flux`) 반환 타입에서의 Fallback 구현 방식은?

---

## 🔍 왜 MSA에서 필요한가

### Fallback이 없는 Circuit Breaker는 절반짜리

```
Circuit Breaker OPEN 시:
  CallNotPermittedException 발생
  → 처리하지 않으면 클라이언트에게 500 오류 반환

Fallback의 역할:
  "서비스가 현재 이용 불가능합니다" 대신
  의미 있는 대체 응답 제공

실제 Fallback 전략들:

1. 캐시에서 마지막 성공 응답 반환
   inventory-service OPEN → 마지막으로 캐시된 재고 수량 반환
   "현재 재고 정보가 정확하지 않을 수 있습니다" 알림

2. 기본값 반환
   recommendation-service OPEN → 기본 상품 목록 반환

3. 다른 데이터 소스 사용
   primary-db OPEN → read-replica에서 조회

4. 우아한 거부
   결제 서비스 OPEN → "일시적으로 서비스가 중단됐습니다. 나중에 다시 시도하세요"
   503 상태 코드와 함께 반환

Fallback 없음 = 장애 노출
Fallback 있음 = 장애 격리 + 서비스 품질 유지
```

---

## 😱 잘못된 구성

### Before: Fallback 메서드 시그니처 규칙 위반

```java
// ❌ 원본 메서드와 파라미터가 다른 Fallback
@CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallback")
public Inventory getInventory(Long productId, boolean fresh) {
    return inventoryClient.get(productId, fresh);
}

// ❌ 잘못된 Fallback 시그니처들
private Inventory inventoryFallback() {
    // 파라미터 없음 → 매칭 실패
    return Inventory.empty();
}

private Inventory inventoryFallback(String error) {
    // 파라미터 타입 불일치 → 매칭 실패
    return Inventory.empty();
}

private String inventoryFallback(Long productId, boolean fresh, Exception e) {
    // 반환 타입 불일치 → 매칭 실패
    return "fallback";
}
```

### Before: Fallback에서 동일한 장애 서비스 재호출

```java
// ❌ Fallback이 같은 실패 지점을 재호출
@CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallback")
public Inventory getInventory(Long productId) {
    return inventoryClient.get(productId);
}

private Inventory inventoryFallback(Long productId, Exception e) {
    // ❌ CB가 OPEN인데 같은 inventoryClient를 또 호출!
    // → 즉시 CallNotPermittedException 발생
    // → 또 Fallback 호출 → 무한 루프 아님 (예외 전파)
    return inventoryClient.getFromCache(productId);
    // inventoryClient가 내부적으로 같은 CB를 사용한다면 문제
}
```

---

## ✨ 올바른 패턴

### After: 올바른 Fallback 메서드 규칙

```java
@Service
public class InventoryService {

    // 규칙 1: 원본 메서드의 모든 파라미터 포함
    // 규칙 2: 예외 타입 파라미터 추가 (마지막에)
    // 규칙 3: 반환 타입 동일
    // 규칙 4: 접근 제어자 무관 (private OK)

    @CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallback")
    public Inventory getInventory(Long productId) {
        return inventoryClient.get(productId);
    }

    // ✅ 올바른 Fallback 시그니처
    private Inventory inventoryFallback(Long productId, Exception e) {
        log.warn("Inventory CB fallback for product {}: {}",
            productId, e.getMessage());

        // CallNotPermittedException: CB OPEN 상태 (장애 진행 중)
        if (e instanceof CallNotPermittedException) {
            return inventoryCache.getLastKnown(productId)  // 캐시에서 반환
                .orElse(Inventory.unavailable());
        }

        // 실제 업스트림 예외: 일시적 오류
        return Inventory.unavailable();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. CircuitBreakerAspect — Fallback 호출 메커니즘

```java
// CircuitBreakerAspect.java (Resilience4j Spring Boot Starter)

@Aspect
@Component
public class CircuitBreakerAspect {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final FallbackDecorators fallbackDecorators;

    @Around(value = "matchesCircuitBreakerAnnotation(circuitBreakerAnnotation)",
            argNames = "joinPoint,circuitBreakerAnnotation")
    public Object circuitBreakerAroundAdvice(
            ProceedingJoinPoint joinPoint,
            CircuitBreaker circuitBreakerAnnotation) throws Throwable {

        // ① CB 이름 가져오기 (name 또는 fallbackMethod에서)
        String backend = circuitBreakerAnnotation.name();
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();

        // ② CircuitBreakerRegistry에서 CB 인스턴스 조회/생성
        io.github.resilience4j.circuitbreaker.CircuitBreaker circuitBreaker =
            circuitBreakerRegistry.circuitBreaker(backend,
                () -> circuitBreakerConfig);

        // ③ Fallback 메서드 탐색 (컴파일 타임 아님, 런타임 리플렉션)
        FallbackMethod fallbackMethod = null;
        if (StringUtils.hasText(circuitBreakerAnnotation.fallbackMethod())) {
            fallbackMethod = FallbackMethod.create(
                circuitBreakerAnnotation.fallbackMethod(),
                method,
                joinPoint.getArgs(),
                joinPoint.getTarget());
        }

        // ④ Supplier로 래핑 후 CB 실행
        CircuitBreakerEntry entry = CircuitBreakerEntry.of(
            circuitBreaker,
            joinPoint::proceed,   // 실제 메서드 호출
            fallbackMethod);

        return execute(entry);
    }

    private Object execute(CircuitBreakerEntry entry) throws Throwable {
        try {
            // ⑤ Permission 획득 (OPEN이면 CallNotPermittedException)
            entry.circuitBreaker.acquirePermission();

            long startTime = System.nanoTime();
            try {
                // ⑥ 실제 메서드 실행
                Object result = entry.supplier.get();
                // ⑦ 성공 기록
                long duration = System.nanoTime() - startTime;
                entry.circuitBreaker.onSuccess(duration, TimeUnit.NANOSECONDS);
                return result;

            } catch (Throwable throwable) {
                // ⑧ 실패 기록
                long duration = System.nanoTime() - startTime;
                entry.circuitBreaker.onError(duration, TimeUnit.NANOSECONDS, throwable);
                // ⑨ Fallback 호출
                return handleFallback(entry, throwable);
            }
        } catch (CallNotPermittedException e) {
            // ⑩ OPEN 상태 → Fallback 호출
            return handleFallback(entry, e);
        }
    }

    private Object handleFallback(CircuitBreakerEntry entry, Throwable throwable)
            throws Throwable {
        if (entry.fallbackMethod == null) {
            throw throwable;  // Fallback 없으면 예외 전파
        }
        // ⑪ Fallback 메서드 리플렉션 호출
        return entry.fallbackMethod.fallback(throwable);
    }
}
```

### 2. FallbackMethod — 시그니처 매칭 알고리즘

```java
// FallbackMethod.java — 런타임 Fallback 메서드 탐색

class FallbackMethod {

    static FallbackMethod create(String fallbackMethodName,
            Method originalMethod,
            Object[] args,
            Object target) {

        // ① 대상 클래스의 모든 메서드에서 이름 일치 탐색
        Class<?> targetClass = target.getClass();

        // ② 원본 파라미터 타입 + 예외 타입 조합으로 매칭
        // 매칭 우선순위:
        // 1. 원본 파라미터 + 정확한 예외 타입 (CallNotPermittedException 등)
        // 2. 원본 파라미터 + 상위 예외 타입 (RuntimeException)
        // 3. 원본 파라미터 + Exception (가장 넓음)

        for (Method method : targetClass.getDeclaredMethods()) {
            if (method.getName().equals(fallbackMethodName)
                    && method.getReturnType()
                        .isAssignableFrom(originalMethod.getReturnType())) {

                Class<?>[] params = method.getParameterTypes();
                Class<?>[] originalParams = originalMethod.getParameterTypes();

                // 파라미터 수: 원본 파라미터 수 + 1 (예외)
                if (params.length == originalParams.length + 1) {
                    // 마지막 파라미터가 Throwable의 서브타입인지 확인
                    if (Throwable.class.isAssignableFrom(
                            params[params.length - 1])) {
                        // 앞 파라미터들이 원본과 일치하는지 확인
                        if (parametersMatch(params, originalParams)) {
                            method.setAccessible(true);
                            return new FallbackMethod(method, args, target);
                        }
                    }
                }
            }
        }
        throw new FallbackMethodNotFoundException(fallbackMethodName);
    }

    Object fallback(Throwable throwable) throws Exception {
        // 원본 args + throwable로 Fallback 메서드 호출
        Object[] fallbackArgs = Arrays.copyOf(args, args.length + 1);
        fallbackArgs[fallbackArgs.length - 1] = throwable;
        return fallbackMethod.invoke(target, fallbackArgs);
    }
}
```

### 3. Fallback 체이닝 — Fallback의 Fallback

```java
@Service
public class OrderService {

    // 1차 Fallback: 캐시에서 시도
    @CircuitBreaker(name = "order", fallbackMethod = "orderFallback")
    public Order getOrder(Long orderId) {
        return orderClient.getOrder(orderId);
    }

    // 2차 Fallback: 캐시 조회 (1차 Fallback에 @CircuitBreaker 추가)
    @CircuitBreaker(name = "order-cache", fallbackMethod = "orderCacheFallback")
    private Order orderFallback(Long orderId, Exception e) {
        log.info("Primary fallback triggered, trying cache");
        return orderCache.get(orderId)
            .orElseThrow(() -> new RuntimeException("Cache miss"));
    }

    // 3차 Fallback: 최소 정보만 반환 (최후 수단)
    private Order orderCacheFallback(Long orderId, Exception e) {
        log.warn("All fallbacks failed for order {}", orderId);
        // 최소한의 응답: 주문 번호만 포함한 불완전 객체
        return Order.minimal(orderId);
    }
}
```

### 4. Reactive Fallback (Mono/Flux)

```java
@Service
public class ReactiveInventoryService {

    // Reactive 반환 타입에서의 CB + Fallback
    @CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallbackReactive")
    public Mono<Inventory> getInventory(Long productId) {
        return inventoryWebClient.get(productId);
        // Mono 반환 → WebFlux 환경
    }

    // Reactive Fallback: Mono 반환 타입 일치 필수
    private Mono<Inventory> inventoryFallbackReactive(Long productId, Exception e) {
        if (e instanceof CallNotPermittedException) {
            // CB OPEN: 캐시에서 Mono로 래핑
            return Mono.justOrEmpty(inventoryCache.getLastKnown(productId))
                .defaultIfEmpty(Inventory.unavailable());
        }
        // 일반 오류: 빈 Inventory Mono
        return Mono.just(Inventory.unavailable());
    }

    // Flux 반환 타입
    @CircuitBreaker(name = "products", fallbackMethod = "productsFallback")
    public Flux<Product> getProducts(String category) {
        return productWebClient.getByCategory(category);
    }

    private Flux<Product> productsFallback(String category, Exception e) {
        // 캐시된 상품 목록으로 Flux 생성
        List<Product> cached = productCache.getByCategory(category);
        return Flux.fromIterable(cached.isEmpty()
            ? List.of(Product.placeholder())
            : cached);
    }
}
```

### 5. CallNotPermittedException vs 실제 예외 구분

```java
// Fallback 메서드에서 예외 타입 기반 분기 처리

private Inventory inventoryFallback(Long productId, Exception e) {

    if (e instanceof CallNotPermittedException cnpe) {
        // CB OPEN 상태: 서비스가 이미 불안정한 것으로 판단
        // waitDurationInOpenState 동안 차단 중
        log.warn("[CB-OPEN] Inventory CB open, returning cached data. " +
            "State: {}", cnpe.getCausingCircuitBreakerName());
        return inventoryCache.getLastKnown(productId)
            .orElse(Inventory.unavailable());
    }

    if (e instanceof TimeoutException) {
        // TimeLimiter 타임아웃: 서비스 응답 느림
        log.warn("[TIMEOUT] Inventory service timeout for product {}", productId);
        return Inventory.withWarning("재고 정보 로딩 중. 잠시 후 다시 시도하세요.");
    }

    if (e instanceof ConnectException) {
        // 네트워크 연결 실패: 서비스 다운
        log.error("[CONNECT] Inventory service unreachable for product {}", productId);
        return Inventory.unavailable();
    }

    // 기타 예외: 예상치 못한 오류
    log.error("[UNKNOWN] Unexpected error for product {}: {}", productId, e.getMessage());
    return Inventory.error();
}
```

---

## 💻 실전 구성

### 응답 품질 레벨별 Fallback 전략

```java
// 응답 품질을 표시하는 도우미 클래스
@Value
public class FallbackQuality {
    public static final String FULL = "FULL";       // 완전한 데이터
    public static final String CACHED = "CACHED";   // 캐시 데이터 (최신 아닐 수 있음)
    public static final String PARTIAL = "PARTIAL"; // 부분 데이터
    public static final String UNAVAILABLE = "UNAVAILABLE"; // 불가
}

@RestController
public class InventoryController {

    @GetMapping("/inventory/{id}")
    public ResponseEntity<InventoryResponse> getInventory(@PathVariable Long id) {
        Inventory inventory = inventoryService.getInventory(id);

        // Fallback 응답 품질에 따른 HTTP 응답 코드 결정
        return switch (inventory.getQuality()) {
            case FallbackQuality.FULL -> ResponseEntity.ok(
                new InventoryResponse(inventory));
            case FallbackQuality.CACHED -> ResponseEntity.ok()
                .header("X-Data-Quality", "cached")
                .header("X-Cache-Time", inventory.getCacheTimestamp().toString())
                .body(new InventoryResponse(inventory));
            case FallbackQuality.UNAVAILABLE -> ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(new InventoryResponse(inventory));
            default -> ResponseEntity.internalServerError().build();
        };
    }
}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 계층적 Fallback으로 서비스 가용성 최대화

```
Primary 경로: order-service → inventory-service (REST)
  → CB: inventory-cb

Fallback 1: Redis 캐시에서 재고 조회
  → 재고 데이터를 주기적으로 Redis에 동기화
  → inventory-service OPEN → Redis에서 읽기
  → 응답: 캐시된 재고 (최대 5분 지연)

Fallback 2: 낙관적 재고 정책
  → Redis도 다운 → 낙관적으로 "재고 있음" 가정
  → 실제 결제 시 재고 검증 (보상 처리)
  → 응답: 재고 있음 (정확하지 않을 수 있음)

Fallback 3: 서비스 불가 응답
  → 모든 Fallback 실패 → 명확한 503 반환
  → 클라이언트에게 재시도 안내

가용성:
  inventory-service DOWN → Redis 캐시로 서비스 지속
  Redis도 DOWN → 낙관적 정책으로 서비스 지속 (품질 저하 허용)
  최후 → 명확한 503 (혼란 없는 거부)
```

---

## ⚖️ 트레이드오프

| Fallback 전략 | 정확성 | 가용성 | 복잡도 |
|--------------|--------|--------|--------|
| **캐시 반환** | 낮음 (구 데이터) | 높음 | 중간 |
| **기본값 반환** | 없음 | 매우 높음 | 낮음 |
| **부분 응답** | 중간 | 높음 | 중간 |
| **503 반환** | N/A | 낮음 | 낮음 |
| **보조 서비스 호출** | 중간 | 높음 | 높음 |

```
Fallback 설계 원칙:
  1. Fallback은 원본 서비스보다 빠르고 안정적이어야 함
     → Fallback 자체가 실패하면 의미 없음
  
  2. Fallback 응답은 클라이언트가 구분할 수 있어야 함
     → 헤더(X-Data-Quality), 응답 필드(quality, isFallback)로 표시
  
  3. Fallback에서 외부 서비스를 다시 호출하지 않는 것이 원칙
     → Fallback → 외부 서비스 → 실패 → Fallback의 Fallback → ...
     → 단, 명확히 다른 엔드포인트(read-replica 등)는 허용

  4. Fallback 성능 모니터링 필요
     → Fallback이 얼마나 자주 호출되는가?
     → metrics: resilience4j.circuitbreaker.calls{kind=fallback}
```

---

## 📌 핵심 정리

```
Fallback 메서드 핵심:

시그니처 규칙:
  메서드 이름: fallbackMethod 파라미터와 일치
  파라미터: 원본 파라미터 전부 + Throwable 서브타입 (마지막)
  반환 타입: 원본과 동일 (Mono/Flux도 동일하게)
  접근 제어자: private 가능

AOP 호출 경로:
  CircuitBreakerAspect.around()
  → acquirePermission() (OPEN → CallNotPermittedException)
  → joinPoint.proceed() (실제 메서드)
  → 예외 발생 시: onError() 기록 + fallback(throwable) 호출

예외 구분:
  CallNotPermittedException: CB OPEN 상태
  TimeoutException: TimeLimiter 타임아웃
  ConnectException: 네트워크 연결 실패
  → Fallback에서 instanceof로 분기 처리

Fallback 체이닝:
  1차 Fallback에도 @CircuitBreaker 추가
  → 1차 실패 시 2차 Fallback 호출
  → 최종 Fallback: 항상 성공하는 최소 응답

Reactive Fallback:
  원본이 Mono<T> → Fallback도 Mono<T>
  원본이 Flux<T> → Fallback도 Flux<T>
  Mono.just(), Flux.fromIterable()로 즉시 반환
```

---

## 🤔 생각해볼 문제

**Q1.** Fallback 메서드에서 예외를 던지면 어떻게 처리되는가? 최종적으로 클라이언트가 받는 응답은?

<details>
<summary>해설 보기</summary>

Fallback 메서드에서 예외가 발생하면 `FallbackMethod.fallback()`이 해당 예외를 `CircuitBreakerAspect`로 전파합니다. `CircuitBreakerAspect`는 이를 처리하지 않고 다시 호출자에게 전파합니다. 즉, Fallback 예외가 그대로 클라이언트에게 전달됩니다. Spring MVC에서는 `@ExceptionHandler`나 `@ControllerAdvice`가 없으면 500 Internal Server Error가 반환됩니다.

따라서 Fallback 메서드는 반드시 예외 없이 완료되어야 합니다. try-catch를 추가하거나, 최후 수단으로 항상 성공하는 응답(예: `return Inventory.empty()`)을 반환하도록 구현해야 합니다. Fallback 자체가 실패하는 상황은 "이중 장애"로 가장 피해야 할 케이스입니다.

</details>

---

**Q2.** `@CircuitBreaker(fallbackMethod = "fallback")`이 적용된 메서드와 동일한 이름의 Fallback 메서드가 여러 개 오버로딩되어 있다면 어떤 Fallback이 선택되는가?

<details>
<summary>해설 보기</summary>

`FallbackMethod.create()`가 런타임에 리플렉션으로 매칭 메서드를 탐색합니다. 매칭 우선순위:
1. 원본 파라미터 + 더 구체적인 예외 타입 (예: `CallNotPermittedException`)
2. 원본 파라미터 + 덜 구체적인 예외 타입 (예: `RuntimeException`)
3. 원본 파라미터 + `Exception`
4. 원본 파라미터 + `Throwable`

예를 들어 실제 예외가 `CallNotPermittedException`이면 `fallback(Long id, CallNotPermittedException e)`가 우선 선택됩니다. 없으면 `fallback(Long id, Exception e)`로 폴백됩니다. 이를 활용해 예외별로 다른 Fallback 로직을 오버로딩으로 분리할 수 있습니다.

</details>

---

**Q3.** Fallback 메서드 내에서 성능 메트릭(Fallback 호출 횟수, 응답 시간)을 수집하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

Resilience4j Actuator는 `resilience4j.circuitbreaker.calls.total{kind=not_permitted}` 메트릭으로 OPEN 중 차단된 호출 수를 자동으로 기록합니다. 하지만 Fallback 내부 로직 성능(캐시 히트율, 응답 시간 등)은 별도로 수집해야 합니다.

방법 1: Micrometer 직접 사용
```java
private Inventory inventoryFallback(Long productId, Exception e) {
    Timer.Sample sample = Timer.start(meterRegistry);
    try {
        Inventory result = inventoryCache.getLastKnown(productId).orElse(Inventory.empty());
        meterRegistry.counter("fallback.cache.hit",
            "service", "inventory",
            "hit", String.valueOf(result != Inventory.empty())).increment();
        return result;
    } finally {
        sample.stop(meterRegistry.timer("fallback.duration", "service", "inventory"));
    }
}
```

방법 2: `@Timed` 어노테이션 (Micrometer 지원)
```java
@Timed(value = "fallback.duration", extraTags = {"service", "inventory"})
private Inventory inventoryFallback(Long productId, Exception e) { ... }
```

</details>

---

<div align="center">

**[⬅️ 이전: Slow Call 탐지](./04-slow-call-detection.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bulkhead 패턴 ➡️](./06-bulkhead-pattern.md)**

</div>
