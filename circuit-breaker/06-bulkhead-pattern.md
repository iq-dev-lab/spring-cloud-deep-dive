# Bulkhead 패턴 (격리) — 동시 호출 제한으로 장애 파급 범위 제한

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Semaphore Bulkhead와 ThreadPool Bulkhead의 내부 구현과 사용 시나리오 차이는?
- `BulkheadFullException`이 발생하는 정확한 조건과 대기 시간 설정의 의미는?
- Bulkhead가 Circuit Breaker와 다른 격리 원리는 무엇인가?
- ThreadPool Bulkhead에서 큐 크기(`queueCapacity`)와 최대 스레드 수(`maxThreadPoolSize`)의 관계는?
- Bulkhead와 Circuit Breaker를 함께 사용할 때 데코레이터 실행 순서는?
- `@Bulkhead(type = SEMAPHORE)` vs `@Bulkhead(type = THREADPOOL)`의 선택 기준은?

---

## 🔍 왜 MSA에서 필요한가

### 선박의 격벽(Bulkhead)에서 온 패턴

```
선박 격벽의 원리:
  선박을 여러 격실로 나누어 한 격실이 침수돼도
  나머지 격실이 영향받지 않도록 격리

MSA에서의 Bulkhead:
  
  Without Bulkhead:
  service-a → [external-api-1] (느려짐)
            → [external-api-2] (정상)
            → [db-query] (정상)
  
  external-api-1이 느려지면:
  100개 스레드 모두 external-api-1 대기
  → external-api-2, db-query도 스레드 없어서 처리 불가
  → service-a 전체 장애

  With Bulkhead:
  service-a → [Bulkhead-A: 10 semaphores] → external-api-1
            → [Bulkhead-B: 20 semaphores] → external-api-2
            → [Bulkhead-C: 30 semaphores] → db-query
  
  external-api-1 느려짐:
  Bulkhead-A 10개 슬롯 모두 점유 → 추가 요청 BulkheadFullException
  Bulkhead-B, C는 독립 → external-api-2, db-query 정상 동작
  → service-a의 다른 기능 정상 유지

핵심: Circuit Breaker는 장애 감지 후 차단
       Bulkhead는 동시 실행 수를 사전에 제한
```

---

## 😱 잘못된 구성

### Before: 모든 외부 호출이 공유 스레드 풀 사용

```java
// ❌ 공유 RestTemplate, 별도 Bulkhead 없음
@Service
public class DataAggregator {
    private final RestTemplate restTemplate;  // 공유 스레드 풀

    public AggregatedData aggregate(Long id) {
        ExternalA a = restTemplate.getForObject("http://service-a/...", ExternalA.class);
        ExternalB b = restTemplate.getForObject("http://service-b/...", ExternalB.class);
        ExternalC c = restTemplate.getForObject("http://service-c/...", ExternalC.class);
        // service-b가 느리면 모든 스레드가 b 대기 → a, c도 처리 불가
        return new AggregatedData(a, b, c);
    }
}
```

### Before: ThreadPool Bulkhead와 동기 코드 혼용

```java
// ❌ ThreadPool Bulkhead는 CompletableFuture 반환 필요
@Bulkhead(name = "external-api", type = Bulkhead.Type.THREADPOOL)
public ExternalData getData(Long id) {
    // ❌ ThreadPool Bulkhead는 CompletableFuture<T>를 반환해야 함
    // Supplier 반환 타입 → 런타임 오류 발생 가능
    return externalClient.getData(id);
}

// ✅ 올바른 ThreadPool Bulkhead 사용
@Bulkhead(name = "external-api", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<ExternalData> getData(Long id) {
    return CompletableFuture.supplyAsync(
        () -> externalClient.getData(id));
}
```

---

## ✨ 올바른 패턴

### After: Semaphore Bulkhead vs ThreadPool Bulkhead 선택

```
Semaphore Bulkhead:
  호출 스레드가 직접 업스트림 호출 (별도 스레드 없음)
  semaphore.acquire() → 슬롯 없으면 대기 또는 BulkheadFullException
  
  장점:
    오버헤드 낮음 (스레드 생성 없음)
    WebFlux/Reactive 환경에서 안전
  
  적합:
    Non-Blocking 호출
    Reactive 환경 (WebFlux)
    스레드 오버헤드를 최소화하고 싶을 때

ThreadPool Bulkhead:
  별도 스레드 풀에서 업스트림 호출 실행
  CompletableFuture로 결과 반환
  
  장점:
    호출 스레드와 업스트림 실행 스레드 완전 분리
    타임아웃 처리 용이
  
  적합:
    Blocking 호출 (JDBC 등)
    Spring MVC (서블릿 환경)
    호출자 스레드를 보호하고 싶을 때
```

---

## 🔬 내부 동작 원리

### 1. Semaphore Bulkhead 구현

```java
// SemaphoreBulkhead.java
public final class SemaphoreBulkhead implements Bulkhead {

    private final Semaphore semaphore;
    private final BulkheadConfig config;
    private final String name;

    SemaphoreBulkhead(String name, BulkheadConfig config) {
        this.config = config;
        // ① Semaphore 초기화: maxConcurrentCalls 개의 슬롯
        this.semaphore = new Semaphore(config.getMaxConcurrentCalls(), true);
        // fair=true: FIFO 순서로 슬롯 획득 (공정 스케줄링)
    }

    @Override
    public boolean tryAcquirePermission() {
        // ② 슬롯 획득 시도
        boolean acquired;
        if (config.getMaxWaitDuration().isZero()) {
            // 즉시 시도 (대기 없음)
            acquired = semaphore.tryAcquire();
        } else {
            // 설정된 시간만큼 대기
            acquired = semaphore.tryAcquire(
                config.getMaxWaitDuration().toNanos(),
                TimeUnit.NANOSECONDS);
        }

        if (acquired) {
            publishBulkheadEvent(BulkheadEvent.Type.CALL_PERMITTED);
            return true;
        } else {
            // ③ 슬롯 없음 → BulkheadFullException
            publishBulkheadEvent(BulkheadEvent.Type.CALL_REJECTED);
            return false;
        }
    }

    @Override
    public void acquirePermission() {
        if (!tryAcquirePermission()) {
            throw BulkheadFullException.createBulkheadFullException(this);
        }
    }

    @Override
    public void onComplete() {
        // ④ 호출 완료 시 슬롯 반납
        semaphore.release();
        publishBulkheadEvent(BulkheadEvent.Type.CALL_FINISHED);
    }

    // 현재 사용 가능한 슬롯 수 조회
    @Override
    public Metrics getMetrics() {
        return new BulkheadMetrics(
            config.getMaxConcurrentCalls(),
            semaphore.availablePermits()    // 현재 사용 가능한 슬롯 수
        );
    }
}
```

### 2. ThreadPool Bulkhead 구현

```java
// FixedThreadPoolBulkhead.java
public class FixedThreadPoolBulkhead implements ThreadPoolBulkhead {

    private final ExecutorService executorService;
    private final ThreadPoolBulkheadConfig config;

    FixedThreadPoolBulkhead(String name, ThreadPoolBulkheadConfig config) {
        this.config = config;
        // ① 전용 스레드 풀 생성
        this.executorService = new ThreadPoolExecutor(
            config.getCoreThreadPoolSize(),    // 기본 스레드 수
            config.getMaxThreadPoolSize(),     // 최대 스레드 수
            config.getKeepAliveDuration().toMillis(), TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(config.getQueueCapacity()),  // ② 작업 큐
            new NamingThreadFactory(name),     // 스레드 이름 지정
            new ThreadPoolExecutor.AbortPolicy()  // 큐도 꽉 차면 RejectedExecutionException
        );
    }

    @Override
    public <T> CompletableFuture<T> executeCallable(Callable<T> callable) {
        // ③ 스레드 풀에 작업 제출
        CompletableFuture<T> promise = new CompletableFuture<>();

        try {
            executorService.submit(() -> {
                try {
                    T result = callable.call();
                    promise.complete(result);
                } catch (Exception e) {
                    promise.completeExceptionally(e);
                }
            });
        } catch (RejectedExecutionException e) {
            // ④ 스레드 풀 + 큐 모두 꽉 참 → BulkheadFullException
            BulkheadFullException bulkheadFullException =
                BulkheadFullException.createBulkheadFullException(this);
            promise.completeExceptionally(bulkheadFullException);
        }

        return promise;
    }
}
```

### 3. Circuit Breaker + Bulkhead 데코레이터 순서

```java
// Resilience4j 데코레이터 패턴으로 직접 조합

CircuitBreaker cb = CircuitBreaker.of("inventory", cbConfig);
Bulkhead bulkhead = Bulkhead.of("inventory", bhConfig);

// ✅ 올바른 순서: Bulkhead(바깥) → CircuitBreaker(안쪽)
// Bulkhead가 먼저 동시 호출 수 제한
// 그 안에서 CircuitBreaker가 장애 감지

Supplier<Inventory> decoratedSupplier =
    Decorators.ofSupplier(() -> inventoryClient.get(productId))
        .withCircuitBreaker(cb)   // 안쪽 먼저 등록
        .withBulkhead(bulkhead)   // 바깥쪽 나중 등록
        .decorate();
// 실행 순서: Bulkhead → CircuitBreaker → 실제 호출

// @어노테이션 방식의 순서는 Aspect Order로 결정:
// BulkheadAspect Order = Integer.MAX_VALUE - 3 (먼저 실행)
// CircuitBreakerAspect Order = Integer.MAX_VALUE - 2 (나중 실행)
// 즉: Bulkhead(바깥) → CircuitBreaker(안쪽) → 실제 호출
```

### 4. 어노테이션 조합

```java
@Service
public class InventoryService {

    // Bulkhead + Circuit Breaker 조합
    @Bulkhead(name = "inventory", type = Bulkhead.Type.SEMAPHORE,
              fallbackMethod = "bulkheadFallback")
    @CircuitBreaker(name = "inventory", fallbackMethod = "circuitBreakerFallback")
    public Inventory getInventory(Long productId) {
        return inventoryClient.get(productId);
    }

    // BulkheadFullException Fallback (동시 호출 제한 초과)
    private Inventory bulkheadFallback(Long productId, BulkheadFullException e) {
        log.warn("Bulkhead full for inventory {}: {} concurrent calls",
            productId, bulkheadRegistry.bulkhead("inventory")
                .getMetrics().getMaxAllowedConcurrentCalls());
        return Inventory.unavailable("일시적으로 많은 요청이 처리 중입니다.");
    }

    // Circuit Breaker Fallback (장애 감지)
    private Inventory circuitBreakerFallback(Long productId, Exception e) {
        return Inventory.unavailable("재고 서비스가 일시적으로 중단됐습니다.");
    }
}
```

---

## 💻 실전 구성

### Semaphore vs ThreadPool 설정 비교

```yaml
resilience4j:
  bulkhead:
    instances:
      # Semaphore Bulkhead (동시 호출 수 제한)
      external-api:
        max-concurrent-calls: 10     # 동시 최대 10개 호출
        max-wait-duration: 100ms     # 슬롯 없을 때 100ms 대기 후 BulkheadFullException

  thread-pool-bulkhead:
    instances:
      # ThreadPool Bulkhead (전용 스레드 풀)
      blocking-api:
        core-thread-pool-size: 5     # 기본 스레드 수
        max-thread-pool-size: 10     # 최대 스레드 수
        queue-capacity: 20           # 큐 크기
        keep-alive-duration: 20ms    # 유휴 스레드 유지 시간
        # 동시 처리 가능: maxThreadPoolSize(10) + queueCapacity(20) = 30
        # 30개 초과 시 BulkheadFullException
```

### 동시 호출 부하 테스트

```bash
# 동시 30개 요청으로 Bulkhead 한계 테스트
# maxConcurrentCalls: 10, maxWaitDuration: 0

ab -n 100 -c 30 http://localhost:8080/api/inventory/1

# 예상 결과:
# 10개: 즉시 처리 (슬롯 확보)
# 20개: BulkheadFullException (100ms 대기 후 실패)

# Actuator로 Bulkhead 상태 확인
curl http://localhost:8080/actuator/bulkheads
# {
#   "bulkheads": {
#     "inventory": {
#       "availableConcurrentCalls": 3,
#       "maxAllowedConcurrentCalls": 10
#     }
#   }
# }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 다중 외부 서비스 격리

```
DataAggregator 서비스:
  → Payment API (느림: 3~5초)
  → Shipping API (정상: 50ms)
  → Notification API (정상: 20ms)

Without Bulkhead:
  Payment API 3초 대기 × 100 동시 요청
  → 100개 스레드 모두 점유
  → Shipping, Notification도 스레드 없어서 대기
  → 전체 서비스 응답 없음

With Bulkhead:
  [Payment Bulkhead: 20 slots] → 20개 요청만 동시 처리
    21번째 요청: BulkheadFullException → Fallback 즉시 반환
    → 나머지 80개 스레드 자유

  [Shipping Bulkhead: 30 slots] → Shipping 정상 처리
  [Notification Bulkhead: 10 slots] → Notification 정상 처리

결과:
  Payment가 느려도 Shipping, Notification 완전 정상
  Payment의 일부(20개)는 처리, 나머지는 Fallback
  서비스 A 전체 장애 없음
```

---

## ⚖️ 트레이드오프

| 방식 | 스레드 오버헤드 | 타임아웃 처리 | Reactive | 적합한 케이스 |
|------|--------------|-------------|---------|--------------|
| **Semaphore** | 없음 | 어려움 | ✅ | Non-Blocking, WebFlux |
| **ThreadPool** | 있음 | 쉬움 | ❌ | Blocking I/O, Spring MVC |

```
maxConcurrentCalls 값 결정 방법:
  1. 업스트림 처리 용량 파악
     payment-service: 초당 10 TPS
     평균 응답 시간: 500ms
     동시 처리 = TPS × 응답시간(초) = 10 × 0.5 = 5개 정도
     여유: × 2 = 10개 설정

  2. 전체 스레드 풀의 일정 비율로 제한
     스레드 풀 200개 → payment: 20%, shipping: 30%, others: 50%
     → payment: 40, shipping: 60, others: 100

  3. 실험적 접근
     점진적으로 줄이면서 BulkheadFullException 발생률 모니터링
     허용 가능한 오류율(예: 1%) 이하로 유지하는 최솟값 선택
```

---

## 📌 핵심 정리

```
Bulkhead 패턴 핵심:

목적:
  Circuit Breaker: 장애 서비스 감지 후 차단 (사후)
  Bulkhead: 동시 호출 수 사전 제한 (사전) → 장애 파급 범위 제한

Semaphore Bulkhead:
  java.util.concurrent.Semaphore 기반
  max-concurrent-calls: 동시 최대 호출 수
  max-wait-duration: 슬롯 대기 시간 (0 = 즉시 거부)
  tryAcquire() 실패 → BulkheadFullException

ThreadPool Bulkhead:
  전용 ExecutorService (ThreadPoolExecutor)
  CompletableFuture<T> 반환 필수
  큐 꽉 참 → RejectedExecutionException → BulkheadFullException

어노테이션 데코레이터 순서 (자동):
  Bulkhead(바깥, Order: MAX_VALUE-3)
    → CircuitBreaker(안쪽, Order: MAX_VALUE-2)
      → 실제 호출

선택 기준:
  WebFlux / Non-Blocking → Semaphore
  Spring MVC / Blocking → ThreadPool (스레드 격리 필요 시)
  기본: Semaphore가 더 가볍고 단순
```

---

## 🤔 생각해볼 문제

**Q1.** Semaphore Bulkhead에서 `maxWaitDuration`을 0이 아닌 100ms로 설정하면 어떤 동작 차이가 있는가? 언제 0이 더 나은가?

<details>
<summary>해설 보기</summary>

`maxWaitDuration: 0` (즉시 거부): 슬롯이 없으면 즉각 `BulkheadFullException`을 발생시킵니다. 클라이언트에게 빠른 응답(Fallback)을 제공하고 대기 스레드를 만들지 않습니다.

`maxWaitDuration: 100ms` (대기 후 거부): 100ms 동안 슬롯이 해제되기를 기다립니다. 이 시간 내에 슬롯이 생기면 정상 처리됩니다. 단, 100ms 동안 해당 스레드가 대기(블로킹)합니다.

`maxWaitDuration: 0`이 더 나은 경우:
- Reactive 환경 (이벤트 루프 블로킹 절대 금지)
- 빠른 Fail-Fast가 중요한 경우 (클라이언트가 대기 안 하길 원함)
- 업스트림 처리 속도가 빠를 때 (슬롯이 금방 해제됨 = 대기 의미 없음)

`maxWaitDuration > 0`이 나은 경우:
- 일시적 트래픽 스파이크가 있을 때 (대기하면 처리 가능)
- 업스트림 처리 시간이 예측 가능하고 짧을 때

</details>

---

**Q2.** ThreadPool Bulkhead에서 `coreThreadPoolSize: 5`, `maxThreadPoolSize: 10`, `queueCapacity: 20`일 때 동시에 35개 요청이 들어오면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

ThreadPoolExecutor의 동작 순서:

1. 첫 5개 요청: 코어 스레드(5개)에서 즉시 처리
2. 다음 20개 요청: 코어 스레드 모두 사용 중 → 큐(20개)에 적재
3. 큐가 꽉 찬 후 추가 요청: 최대 스레드(10개)까지 추가 생성 → 6번째 ~ 10번째 요청 처리
4. 35번째 요청이 들어올 때: 스레드 10개 모두 사용 중 + 큐 20개 꽉 참 → `RejectedExecutionException` → Resilience4j가 `BulkheadFullException`으로 변환

정리: 동시 처리 가능 = maxThreadPoolSize(10) + queueCapacity(20) = 30개. 31번째부터 `BulkheadFullException` 발생.

주의: Java ThreadPoolExecutor는 `coreSize` 미달일 때만 코어 스레드 생성, 코어 스레드 다 사용 중이면 먼저 큐에 넣고, 큐도 꽉 차면 그때 maxSize까지 스레드 생성합니다.

</details>

---

**Q3.** Bulkhead와 Circuit Breaker가 함께 적용됐을 때, `BulkheadFullException`은 Circuit Breaker의 실패율에 집계되는가?

<details>
<summary>해설 보기</summary>

기본적으로 집계되지 않습니다. `BulkheadFullException`은 업스트림 서비스의 오류가 아니라 Bulkhead 자체의 거부이기 때문입니다. Circuit Breaker의 `recordExceptions` 또는 `ignoreExceptions`에 `BulkheadFullException`이 포함되어 있지 않으면 Circuit Breaker는 이를 "무시된 예외"로 처리합니다.

명시적으로 집계하려면:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        record-exceptions:
          - io.github.resilience4j.bulkhead.BulkheadFullException
```

실무적으로는 `BulkheadFullException`을 Circuit Breaker 실패로 집계하지 않는 것이 일반적입니다. Bulkhead는 "용량 초과" 거부이고, Circuit Breaker는 "서비스 장애" 감지로 역할이 다릅니다. 두 예외를 혼합하면 Circuit Breaker가 실제 서비스 장애와 단순 과부하를 구분하지 못합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Fallback 메서드 작성](./05-fallback-methods.md)** | **[홈으로 🏠](../README.md)** | **[다음: Rate Limiter 통합 ➡️](./07-rate-limiter.md)**

</div>
