# Rate Limiter 통합 — 요청 속도를 제어하는 토큰 버킷 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AtomicRateLimiter`가 토큰 버킷을 주기적으로 갱신하는 내부 알고리즘은?
- `limitForPeriod`와 `limitRefreshPeriod`의 조합으로 RPS를 계산하는 공식은?
- `SemaphoreBasedRateLimiter`와 `AtomicRateLimiter`의 구현 차이와 각각의 적합한 사용 케이스는?
- `RequestNotPermitted` 예외 발생 조건과 `timeoutDuration` 설정의 역할은?
- Circuit Breaker → Bulkhead → Rate Limiter 데코레이터 조합의 올바른 실행 순서는?
- Rate Limiter를 분산 환경(여러 인스턴스)에서 공유하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### Circuit Breaker, Bulkhead와 다른 Rate Limiter의 역할

```
세 패턴의 역할 비교:

Circuit Breaker:
  "이 서비스가 현재 장애 중인가?"
  → 장애 감지 후 차단 (사후적)

Bulkhead:
  "이 순간 몇 개의 요청이 동시에 처리 중인가?"
  → 동시 실행 수 제한 (현재 상태 기반)

Rate Limiter:
  "단위 시간당 얼마나 많은 요청을 허용할 것인가?"
  → 시간 기반 요청 속도 제한 (주기적 제한)

Rate Limiter가 필요한 케이스:

1. 외부 API 요금제 한도 준수
   "외부 결제 API: 초당 100 요청 한도"
   → Rate Limiter: 100 RPS 제한
   → 한도 초과 시 즉시 거부 (429 에러 방지)

2. 공정한 자원 배분
   "모든 테넌트(tenant)는 초당 최대 10 요청"
   → 특정 테넌트의 폭발적 요청이 다른 테넌트에 영향 없도록

3. DDoS 방어
   "동일 IP에서 초당 5 요청 초과 시 차단"

4. 내부 서비스 보호
   "배치 작업이 DB에 과도한 부하를 주지 않도록 50 TPS 제한"

5. 비용 제어
   "LLM API 호출 분당 60회 제한 (비용 관리)"
```

---

## 😱 잘못된 구성

### Before: limitForPeriod을 너무 낮게 설정한 Rate Limiter

```yaml
# ❌ 너무 엄격한 Rate Limiter
resilience4j:
  ratelimiter:
    instances:
      external-api:
        limit-for-period: 1         # 1초당 1개 요청만 허용
        limit-refresh-period: 1s
        timeout-duration: 0s        # 대기 없이 즉시 거부
```

```
결과:
  10 RPS 서비스에서 9개는 RequestNotPermittedException
  클라이언트 오류율 90%
  Rate Limiter가 서비스 가용성을 낮춤

적절한 설정:
  외부 API 한도가 100 RPS → 90 RPS로 설정 (10% 여유)
  limit-for-period: 90
  limit-refresh-period: 1s
```

### Before: 분산 환경에서 인스턴스별 Rate Limiter 적용

```yaml
# ❌ 3개 인스턴스 각각 100 RPS → 실제 300 RPS
# 외부 API 한도는 100 RPS인데 3배 초과
resilience4j:
  ratelimiter:
    instances:
      external-api:
        limit-for-period: 100   # 각 인스턴스 100 RPS
        # 3개 인스턴스 × 100 = 300 RPS → 외부 API 한도 초과
```

---

## ✨ 올바른 패턴

### After: 올바른 Rate Limiter 설정

```yaml
resilience4j:
  ratelimiter:
    instances:
      # 외부 결제 API 보호 (한도: 100 RPS)
      payment-api:
        limit-for-period: 30         # 1초에 30개 (100/3 인스턴스, 10% 여유)
        limit-refresh-period: 1s     # 1초마다 토큰 갱신
        timeout-duration: 500ms      # 슬롯 대기 최대 500ms
        register-health-indicator: true

      # 내부 배치 작업 (DB 보호)
      batch-db:
        limit-for-period: 10         # 100ms당 10개
        limit-refresh-period: 100ms  # 100ms마다 갱신 → 100 TPS
        timeout-duration: 0s         # 즉시 거부 (배치는 대기 OK 아님)

      # 테넌트별 Rate Limiter (동적 생성)
      # 코드에서 동적으로 생성: rateLimiterRegistry.rateLimiter("tenant-" + tenantId)
```

---

## 🔬 내부 동작 원리

### 1. AtomicRateLimiter — 토큰 버킷 갱신 알고리즘

```java
// AtomicRateLimiter.java
// AtomicReference<State>로 상태를 원자적으로 관리
// 별도 스케줄러 스레드 없음 (lazy 갱신)

public class AtomicRateLimiter implements RateLimiter {

    // 현재 상태: 마지막 갱신 시각 + 남은 토큰 수
    private final AtomicReference<State> state;

    // 토큰 갱신 주기 (nanos 단위)
    private final long cyclePeriodInNanos;

    // 주기당 토큰 수
    private final int permissionsPerCycle;

    @Override
    public boolean acquirePermission(final int permits) {
        // ① 현재 나노초 시각
        long timeoutInNanos = rateLimiterConfig.getTimeoutDuration().toNanos();
        State currentState = state.get();

        // ② 허가를 얻을 수 있는 최소 시간 계산 (Lazy 갱신 핵심)
        long nanosToWait = nanosToWaitForPermissions(permits, currentState);

        // ③ 대기 시간이 timeout 내이면 대기 후 획득
        boolean reachable = nanosToWait <= timeoutInNanos;

        if (reachable) {
            // ④ State 원자적 업데이트 (CAS)
            //    획득된 토큰 반영 + 다음 갱신 시각 업데이트
            waitForPermissionIfNeeded(nanosToWait);
            return true;
        }
        // ⑤ timeout 내에 획득 불가 → 거부
        return false;
    }

    private long nanosToWaitForPermissions(int permits, State activeState) {
        long cyclePeriodInNanos = this.cyclePeriodInNanos;
        long permissionsPerCycle = this.permissionsPerCycle;

        // ⑥ 현재 주기에서 사용 가능한 권한 수 계산 (핵심 로직)
        long currentNanos = currentNanoTime();

        // 마지막 갱신 이후 경과한 주기 수
        long elapsedNanos = currentNanos - activeState.activeCycleStart;
        long completedCycles = elapsedNanos / cyclePeriodInNanos;

        // 완료된 주기 × 주기당 토큰으로 획득 가능한 토큰 추가
        long freshPermissions = completedCycles * permissionsPerCycle;

        // 현재 가용 토큰 = 기존 잔여 + 새로 갱신된 토큰
        long availablePermissions = Math.min(
            activeState.activePermissions + freshPermissions,
            permissionsPerCycle  // 최대 1주기분 이상 축적 안 됨
        );

        if (availablePermissions >= permits) {
            return 0;  // 즉시 획득 가능
        }

        // 부족한 경우: 다음 갱신까지 대기 시간 계산
        long deficitPermissions = permits - availablePermissions;
        long cyclesToWait = (long) Math.ceil(
            (double) deficitPermissions / permissionsPerCycle);
        return cyclesToWait * cyclePeriodInNanos;
    }
}

// State: 불변 객체, AtomicReference로 원자적 교체
@Value
class State {
    long activeCycleStart;     // 현재 주기 시작 나노초
    long activePermissions;    // 남은 토큰 수
    long nanosToWait;          // 다음 토큰 획득까지 대기 시간
}
```

### 2. RPS 계산 공식

```
RPS 계산:
  RPS = limitForPeriod / limitRefreshPeriod(초)

예시:
  limitForPeriod: 100, limitRefreshPeriod: 1s
  → 100 / 1 = 100 RPS

  limitForPeriod: 50, limitRefreshPeriod: 500ms
  → 50 / 0.5 = 100 RPS (동일한 RPS)

  차이:
  첫 번째 (1초 주기): 1초에 100개 한꺼번에 허용 → 초반 버스트 가능
  두 번째 (500ms 주기): 500ms마다 50개 → 더 균일한 분배

버스트 허용:
  limitForPeriod: 100, limitRefreshPeriod: 1s, timeout: 0s
  → 1초의 첫 순간에 100개 요청이 모두 들어오면 전부 허용
  → 이후 1초 동안은 모두 거부
  → Burst-then-block 패턴

균일한 분배:
  limitForPeriod: 1, limitRefreshPeriod: 10ms, timeout: 0s
  → 10ms마다 1개 허용 → 100 RPS
  → 동시에 들어오는 요청은 거부 (더 엄격)
```

### 3. SemaphoreBasedRateLimiter vs AtomicRateLimiter

```java
// SemaphoreBasedRateLimiter:
// 실제 스케줄러 스레드가 주기마다 세마포어 갱신

public class SemaphoreBulkhead implements RateLimiter {
    private final Semaphore semaphore;
    private final ScheduledExecutorService refreshScheduler;

    SemaphoreBulkhead(String name, RateLimiterConfig config) {
        this.semaphore = new Semaphore(config.getLimitForPeriod());

        // 별도 스케줄러 스레드가 주기마다 세마포어 리셋
        this.refreshScheduler = Executors.newSingleThreadScheduledExecutor();
        refreshScheduler.scheduleAtFixedRate(
            () -> semaphore.release(config.getLimitForPeriod()
                - semaphore.availablePermits()),
            config.getLimitRefreshPeriod().toNanos(),
            config.getLimitRefreshPeriod().toNanos(),
            TimeUnit.NANOSECONDS);
    }

    // 단점:
    // 별도 스케줄러 스레드 항상 실행 중
    // Rate Limiter 수 × 스케줄러 스레드
    // 다수의 Rate Limiter 인스턴스 시 리소스 낭비
}

// AtomicRateLimiter (기본값):
// 스케줄러 없음, 요청 시점에 lazy 계산
// 단점: 높은 트래픽에서 CAS 경쟁
// 장점: 별도 스레드 없음, 메모리 효율적

// 선택 기준:
// Rate Limiter 인스턴스 수가 많음 → AtomicRateLimiter
// Rate Limiter 인스턴스 수가 적고 정확성 중요 → SemaphoreBasedRateLimiter
```

### 4. CB → Bulkhead → Rate Limiter 데코레이터 조합 순서

```java
// Resilience4j Decorators로 전체 조합

CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("api");
Bulkhead bulkhead = bulkheadRegistry.bulkhead("api");
RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter("api");
Retry retry = retryRegistry.retry("api");

// 데코레이터 실행 순서 (바깥 → 안쪽):
// Retry(바깥) → CircuitBreaker → Bulkhead → RateLimiter → 실제 호출

Supplier<Inventory> decoratedSupplier =
    Decorators.ofSupplier(() -> inventoryClient.get(productId))
        .withRateLimiter(rateLimiter)   // 안쪽 (가장 나중에 등록)
        .withBulkhead(bulkhead)
        .withCircuitBreaker(cb)
        .withRetry(retry)               // 바깥쪽 (가장 먼저 등록)
        .decorate();

// 실행 흐름:
// Retry: 실패하면 재시도 (CB, Bulkhead, RL 전체 재실행)
//   → CircuitBreaker: OPEN이면 즉시 Fallback
//     → Bulkhead: 슬롯 없으면 BulkheadFullException
//       → RateLimiter: 토큰 없으면 RequestNotPermitted
//         → 실제 API 호출

// 어노테이션 방식 (자동 Order):
@RateLimiter(name = "api")          // Order: MAX_VALUE - 1 (가장 안쪽)
@Bulkhead(name = "api")             // Order: MAX_VALUE - 3
@CircuitBreaker(name = "api")       // Order: MAX_VALUE - 2
@Retry(name = "api")                // Order: MAX_VALUE - 4 (가장 바깥)
public Inventory getInventory(Long id) { ... }
```

### 5. 분산 환경에서 Rate Limiter 공유

```java
// 단일 인스턴스 Rate Limiter의 한계:
// 인스턴스 3개 × 각 100 RPS = 실제 300 RPS
// 외부 API 한도 초과 가능

// 분산 Rate Limiter: Redis 기반
@Bean
public RateLimiter redisBasedRateLimiter(RedisTemplate<String, String> redisTemplate) {
    // Lua 스크립트로 원자적 토큰 버킷 구현
    // Redis에서 공유 토큰 버킷 관리
    return new RedisRateLimiter(
        redisTemplate,
        "external-api-rate-limit",
        100,    // 전체 클러스터 기준 100 RPS
        Duration.ofSeconds(1));
}

// 또는 Spring Cloud Gateway의 RedisRateLimiter 활용
// (Ch3에서 Gateway 레벨 Rate Limiting)
spring:
  cloud:
    gateway:
      routes:
        - id: api-route
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@ipKeyResolver}"
# → Redis에서 IP별 토큰 버킷 공유
# → 모든 Gateway 인스턴스에서 동일한 Rate Limit 적용
```

---

## 💻 실전 구성

### 테넌트별 Rate Limiter (동적 생성)

```java
@Service
public class TenantAwareRateLimiter {

    private final RateLimiterRegistry rateLimiterRegistry;
    private final Map<String, Integer> tenantLimits;

    // 테넌트별 Rate Limiter 동적 조회/생성
    public boolean tryAcquireForTenant(String tenantId) {
        int rpsLimit = tenantLimits.getOrDefault(tenantId, 10); // 기본 10 RPS

        // RateLimiterRegistry: 이름으로 조회, 없으면 생성
        RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter(
            "tenant-" + tenantId,
            () -> RateLimiterConfig.custom()
                .limitForPeriod(rpsLimit)
                .limitRefreshPeriod(Duration.ofSeconds(1))
                .timeoutDuration(Duration.ZERO)  // 즉시 거부
                .build());

        return rateLimiter.acquirePermission();
    }
}

// 사용:
@Aspect
@Component
public class TenantRateLimitAspect {
    @Around("@annotation(rateLimited)")
    public Object checkTenantRateLimit(ProceedingJoinPoint pjp,
            TenantRateLimited rateLimited) throws Throwable {
        String tenantId = getTenantId();
        if (!tenantRateLimiter.tryAcquireForTenant(tenantId)) {
            throw new TooManyRequestsException("Rate limit exceeded for tenant: " + tenantId);
        }
        return pjp.proceed();
    }
}
```

```bash
# Rate Limiter 상태 모니터링
curl http://localhost:8080/actuator/ratelimiters
# {
#   "rateLimiters": {
#     "payment-api": {
#       "availablePermissions": 45,
#       "numberOfWaitingThreads": 0
#     }
#   }
# }

# 부하 테스트로 Rate Limiter 확인
ab -n 200 -c 50 http://localhost:8080/api/payment
# 100개 성공, 100개 RequestNotPermitted (429)
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 외부 결제 API 한도 준수

```
외부 결제 API 계약:
  100 RPS 한도 (초당 100 요청)
  초과 시 429 Too Many Requests + 15분 차단

서비스 구성:
  order-service 인스턴스 3개
  각 인스턴스: RPS = 100 / 3 ≈ 33 RPS

설정:
  limit-for-period: 33
  limit-refresh-period: 1s

모니터링:
  requests_rate{service="payment"} < 100
  → 3개 인스턴스 합계 < 100 RPS 유지

피크 트래픽 시나리오:
  동시 1000 RPS 요청
  → 각 인스턴스 Rate Limiter: 33개 통과, 나머지 거부
  → 클라이언트: 429 → 재시도 (backoff 포함)
  → 외부 API: 정확히 100 RPS 이하로 수신
  → 429 차단 없음, 서비스 안정 유지
```

---

## ⚖️ 트레이드오프

| 방식 | 정확도 | 오버헤드 | 분산 지원 | 적합한 케이스 |
|------|--------|---------|---------|--------------|
| **AtomicRateLimiter** | 중간 (CAS 경쟁 시 오차) | 낮음 | 인스턴스별 | 단일 인스턴스, 고트래픽 |
| **SemaphoreBasedRateLimiter** | 높음 | 중간 (스케줄러) | 인스턴스별 | 정확도 중요 |
| **Redis RateLimiter** | 높음 | 높음 (Redis 호출) | 클러스터 공유 | 분산 환경, 외부 API 한도 |

```
limitRefreshPeriod 선택:
  1s 주기: 간단, 버스트 허용 (초반에 몰릴 수 있음)
  100ms 주기: 더 균일, 버스트 제한
  1ms 주기: 거의 선형 분배, 오버헤드 증가

  외부 API 한도 준수: 100ms 또는 짧은 주기 권장
  (1s 주기이면 초 시작에 100개 폭발적 요청 가능)
  
  timeout-duration:
  0: 즉시 거부 (클라이언트 빠른 응답)
  > 0: 지정 시간 대기 (트래픽 스파이크 흡수)
  
  배치 작업: timeout-duration: 1s (잠시 대기 허용)
  실시간 API: timeout-duration: 0 (즉시 거부)
```

---

## 📌 핵심 정리

```
Rate Limiter 핵심:

역할:
  단위 시간당 요청 수 제한 (시간 기반)
  CB(장애 감지), Bulkhead(동시 수 제한)와 다른 차원의 보호

AtomicRateLimiter (기본):
  AtomicReference<State>로 lazy 갱신
  스케줄러 없음 → 요청 시점에 경과 시간으로 토큰 계산
  높은 동시성에서 CAS 경쟁 발생 가능

RPS 계산:
  RPS = limitForPeriod / limitRefreshPeriod(초)
  분산: 각 인스턴스 RPS = 전체 한도 / 인스턴스 수

데코레이터 순서 (바깥→안쪽):
  Retry → CircuitBreaker → Bulkhead → RateLimiter → 실제 호출
  
  실패율: RateLimiter(RequestNotPermitted) 거부
          → Bulkhead(BulkheadFullException) 거부
          → CircuitBreaker(CallNotPermittedException) 차단
          → Retry: 재시도

분산 Rate Limiter:
  인스턴스별: limit / 인스턴스 수
  Redis 공유: Gateway RedisRateLimiter 또는 커스텀 구현
```

---

## 🤔 생각해볼 문제

**Q1.** `limitForPeriod: 10`, `limitRefreshPeriod: 1s`, `timeoutDuration: 500ms`로 설정했을 때, 1초에 15개 요청이 순차적으로 들어오면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

첫 10개 요청은 즉시 토큰을 획득하고 처리됩니다. 11번째 요청부터는 토큰이 없어서 다음 갱신(1초 후)까지 대기합니다. `timeoutDuration: 500ms`이면 최대 500ms를 기다립니다. 1초 주기라면 500ms 내에 갱신이 이루어지지 않아 11~15번째 요청 중 일부는 `RequestNotPermitted`가 발생할 수 있습니다. 정확한 동작은 요청 간 간격에 따라 다릅니다.

`limitRefreshPeriod: 100ms`라면 100ms마다 1개씩 갱신되어 500ms 대기 내에 최대 5개를 추가로 처리할 수 있습니다. 이 경우 15개 모두 처리 가능합니다(10개 즉시 + 5개 대기 후).

</details>

---

**Q2.** `AtomicRateLimiter`의 Lazy 갱신 방식은 `SemaphoreBasedRateLimiter`의 스케줄러 방식과 비교해 어떤 엣지 케이스가 있는가?

<details>
<summary>해설 보기</summary>

AtomicRateLimiter의 엣지 케이스:

1. **시계 오차 누적**: `currentNanoTime()` 기반이므로 시스템 시계가 조정되면(NTP 동기화 등) 토큰 계산이 일시적으로 부정확해질 수 있습니다.

2. **고동시성 CAS 경쟁**: 수백 개 스레드가 동시에 `acquirePermission()`을 호출하면 `compareAndSet` 실패가 많아져 재시도 루프가 반복됩니다. 이로 인해 미세한 지연이 발생할 수 있습니다.

3. **토큰 축적 제한**: 장시간 요청이 없다가 한꺼번에 들어오면, AtomicRateLimiter는 최대 1주기(`limitForPeriod`)만큼만 토큰을 축적합니다. 예를 들어 10분 동안 요청이 없었다가 갑자기 요청이 와도 `limitForPeriod` 개 이상은 처리 안 됩니다.

SemaphoreBasedRateLimiter의 스케줄러는 정확한 주기마다 토큰을 갱신하지만, Rate Limiter 수 × 1개 스케줄러 스레드가 항상 실행 중입니다.

</details>

---

**Q3.** Circuit Breaker + Bulkhead + Rate Limiter가 모두 적용된 메서드에서 순서대로 오류가 발생하는 시나리오를 설명하라. 각 예외 타입이 무엇이며, 최종 클라이언트는 무엇을 받는가?

<details>
<summary>해설 보기</summary>

실행 순서 (바깥 → 안쪽): Retry → CircuitBreaker → Bulkhead → RateLimiter → 업스트림

**시나리오 1: Rate Limiter 거부**
- RateLimiter: 토큰 없음 → `RequestNotPermittedException`
- Bulkhead: 실행 안 됨
- CircuitBreaker: `onError()` 호출 (`ignore-exceptions`에 없으면 실패 카운트)
- Retry: `RequestNotPermittedException`이 retryable이면 재시도, 아니면 전파
- 클라이언트: `RequestNotPermittedException` → `@ExceptionHandler`로 429 변환

**시나리오 2: Bulkhead 거부**
- RateLimiter: 토큰 획득
- Bulkhead: 슬롯 없음 → `BulkheadFullException`
- RateLimiter 토큰: 이미 소비됨 (환불 없음 — 버그 가능성)
- CircuitBreaker: `onError()` 호출
- 클라이언트: `BulkheadFullException` → 503

**시나리오 3: Circuit Breaker OPEN**
- CircuitBreaker: `CallNotPermittedException`
- Bulkhead, RateLimiter 호출 안 됨 (CB가 가장 바깥 아닌 경우)
- Fallback 메서드 실행
- 클라이언트: Fallback 응답 또는 `CallNotPermittedException` → 503

정확한 동작은 어노테이션/데코레이터 순서에 따라 다르므로, 실제 사용 시 Resilience4j Aspect Order를 확인하고 테스트로 검증하는 것이 필수입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Bulkhead 패턴](./06-bulkhead-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — Distributed Tracing ➡️](../distributed-tracing/01-sleuth-trace-span.md)**

</div>
