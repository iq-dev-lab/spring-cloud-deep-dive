# Slow Call 탐지 — 응답 지연이 Circuit Breaker를 OPEN하는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `slowCallDurationThreshold`와 `slowCallRateThreshold`는 서로 어떻게 연동되는가?
- Slow Call이 성공이더라도 Circuit Breaker를 OPEN시킬 수 있는 이유와 조건은?
- Slow Call 탐지가 `Outcome.SLOW_SUCCESS`와 `Outcome.SLOW_ERROR`로 기록되는 시점은?
- `TimeLimiter`와 Slow Call 탐지는 어떤 관계인가? 함께 사용할 때 주의점은?
- 실제 네트워크 지연 시뮬레이션으로 Slow Call을 재현하는 방법은?
- Slow Call 메트릭을 Prometheus/Grafana로 시각화하는 설정은?

---

## 🔍 왜 MSA에서 필요한가

### 실패하지 않아도 느린 서비스가 시스템을 무너뜨리는 방법

```
시나리오: payment-service 응답이 느려진 상황 (10초)

"실패"로 간주하지 않는 이유:
  HTTP 200 OK 반환 (정상 응답 코드)
  결제 처리는 성공
  단, 응답 시간이 10초

실패 기반 Circuit Breaker만 있을 때:
  모든 요청이 성공 → 실패율 0% → CLOSED 유지
  그러나 응답 시간 10초 × 동시 요청 100개
  = 서비스 A 스레드 풀 100개 모두 10초간 점유
  → 새 요청 처리 불가 → 서비스 A 장애

Slow Call 탐지:
  응답 시간 > slowCallDurationThreshold → SLOW_SUCCESS로 기록
  Slow Call 비율 >= slowCallRateThreshold → OPEN!
  → 느린 payment-service 호출 즉시 차단
  → 스레드 고갈 방지
  → 서비스 A 정상 유지

핵심 인사이트:
  "느리게 성공하는 것"도 시스템 관점에서는 실패와 같다
```

---

## 😱 잘못된 구성

### Before: slowCallRateThreshold만 설정하고 threshold를 100%로

```yaml
# ❌ slowCallRateThreshold: 100% → 모든 요청이 느려야 OPEN
resilience4j:
  circuitbreaker:
    instances:
      payment:
        slowCallDurationThreshold: 5s
        slowCallRateThreshold: 100   # 100%가 느릴 때만 OPEN
```

```
결과:
  100개 요청 중 99개가 5초 초과 → 99% < 100% → OPEN 안 됨
  전체 스레드가 5초씩 대기하면서 시스템 영향
  
  합리적 설정: slowCallRateThreshold: 50 (50% 이상 느리면 OPEN)
```

### Before: TimeLimiter 없이 slowCallDurationThreshold만 설정

```yaml
# ❌ TimeLimiter 없이 Slow Call만 설정
resilience4j:
  circuitbreaker:
    instances:
      payment:
        slowCallDurationThreshold: 3s   # 3초 이상 = Slow Call
        # TimeLimiter 없음!

# 문제:
# 응답이 3초를 초과해도 실제 완료까지 기다림
# 응답이 30초 걸리면 30초 동안 스레드 점유
# Slow Call로만 기록되고 실제 호출은 계속 진행됨
# → TimeLimiter로 강제 타임아웃 필요
```

---

## ✨ 올바른 패턴

### After: Slow Call + TimeLimiter 조합

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        slidingWindowSize: 20
        minimumNumberOfCalls: 5
        failureRateThreshold: 50

        # Slow Call 설정
        slowCallDurationThreshold: 2s    # 2초 이상 = Slow Call
        slowCallRateThreshold: 60        # 60% 이상 느리면 OPEN

  timelimiter:
    instances:
      payment:
        timeoutDuration: 3s             # 3초 후 강제 타임아웃
        cancelRunningFuture: true       # 타임아웃 시 Future 취소

# 동작:
# 2s < 응답 시간 < 3s: Slow Call로 기록 + 완료 대기
# 응답 시간 >= 3s: TimeLimiter가 TimeoutException 발생
#                  → CB가 실패로 기록 (SLOW_ERROR)
```

---

## 🔬 내부 동작 원리

### 1. Slow Call 탐지 시점 — onResult() 메서드

```java
// CircuitBreakerStateMachine.java
// 호출 완료 후 응답 시간 측정 및 Outcome 결정

@Override
public void onSuccess(long duration, TimeUnit unit) {
    if (!isCallRecorded()) return;

    // ① 응답 시간 측정
    long durationInMs = unit.toMillis(duration);

    // ② slowCallDurationThreshold와 비교
    long slowCallThresholdMs = circuitBreakerConfig
        .getSlowCallDurationThreshold()
        .toMillis();

    // ③ Outcome 결정
    Outcome outcome;
    if (durationInMs > slowCallThresholdMs) {
        outcome = Outcome.SLOW_SUCCESS;    // 느린 성공
    } else {
        outcome = Outcome.SUCCESS;          // 정상 성공
    }

    // ④ 슬라이딩 윈도우에 기록
    stateReference.get().onSuccess(duration, unit);
}

@Override
public void onError(long duration, TimeUnit unit, Throwable throwable) {
    if (!isCallRecorded()) return;

    long durationInMs = unit.toMillis(duration);
    long slowCallThresholdMs = circuitBreakerConfig
        .getSlowCallDurationThreshold()
        .toMillis();

    // ⑤ 실패인데 느리기도 하면 SLOW_ERROR
    Outcome outcome;
    if (durationInMs > slowCallThresholdMs) {
        outcome = Outcome.SLOW_ERROR;
    } else {
        outcome = Outcome.ERROR;
    }

    stateReference.get().onError(duration, unit, throwable);
}
```

### 2. Slow Call Rate 계산 및 OPEN 트리거

```java
// TotalAggregation에서 Slow Call Rate 계산

float getSlowCallRate() {
    int bufferedCalls = numberOfCalls;    // 슬라이딩 윈도우의 전체 호출 수
    if (bufferedCalls == 0) return -1f;

    // slowCallRate = (SLOW_SUCCESS + SLOW_ERROR) / 전체 × 100
    return (float) numberOfSlowCalls / bufferedCalls * 100;
}

// OPEN 트리거 조건 (OR 조건):
// ① failureRate >= failureRateThreshold
// ② slowCallRate >= slowCallRateThreshold

Result checkIfThresholdsExceeded(TotalAggregation aggregation) {
    float failureRate = aggregation.getFailureRate();
    float slowCallRate = aggregation.getSlowCallRate();

    // minimum calls 충족 여부 확인
    if (aggregation.getNumberOfCalls() < minimumNumberOfCalls) {
        return Result.BELOW_MINIMUM_CALLS_THRESHOLD;
    }

    // 실패율 또는 Slow Call율 초과 시 OPEN
    if (failureRate >= failureRateThreshold
            || slowCallRate >= slowCallRateThreshold) {
        return Result.ABOVE_THRESHOLDS;
    }

    return Result.BELOW_THRESHOLDS;
}
```

### 3. TimeLimiter와의 연동

```java
// TimeLimiter: 실제 호출을 강제 타임아웃
// CircuitBreaker: 결과를 기록 및 상태 관리
// 두 개가 함께 사용될 때의 상호작용

@CircuitBreaker(name = "payment")
@TimeLimiter(name = "payment")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
    // CompletableFuture 반환 = TimeLimiter가 필요한 형태
    return CompletableFuture.supplyAsync(() ->
        paymentClient.process(req));
}

// TimeLimiter 타임아웃 발생 시:
// 1. Future.cancel() 호출 (cancelRunningFuture: true)
// 2. TimeoutException 발생
// 3. CircuitBreaker.onError() 호출 (타임아웃 시간 = timeoutDuration)
//    → 응답 시간 = timeoutDuration (3s)
//    → slowCallDurationThreshold(2s) 초과 → SLOW_ERROR

// 중요한 순서:
// TimeLimiter(바깥) → CircuitBreaker(안쪽) → 실제 호출
// TimeLimiter가 바깥에 있어야 타임아웃을 CircuitBreaker 실패로 기록 가능
```

### 4. Slow Call 메트릭 구조

```java
// CircuitBreaker.Metrics 인터페이스
public interface Metrics {
    // 전체 호출 수 (슬라이딩 윈도우)
    int getNumberOfBufferedCalls();

    // 성공 호출 수 (SLOW_SUCCESS 포함)
    int getNumberOfSuccessfulCalls();

    // 실패 호출 수 (SLOW_ERROR 포함)
    int getNumberOfFailedCalls();

    // Slow Call 수 (SLOW_SUCCESS + SLOW_ERROR)
    int getNumberOfSlowCalls();

    // Slow 성공 수
    int getNumberOfSlowSuccessfulCalls();

    // Slow 실패 수
    int getNumberOfSlowFailedCalls();

    // 실패율 (%)
    float getFailureRate();

    // Slow Call율 (%)
    float getSlowCallRate();

    // OPEN 중 차단된 호출 수
    long getNumberOfNotPermittedCalls();
}
```

---

## 💻 실전 구성

### 네트워크 지연 시뮬레이션 실험

```java
// 업스트림 서비스에 지연 주입 (테스트용)
@RestController
public class SlowController {

    @GetMapping("/slow")
    public ResponseEntity<String> slow(
            @RequestParam(defaultValue = "0") long delayMs) throws Exception {
        Thread.sleep(delayMs);
        return ResponseEntity.ok("Done after " + delayMs + "ms");
    }
}
```

```bash
# Slow Call 시나리오 재현

# 1. 정상 요청 5개 (warmup)
for i in {1..5}; do
  curl "http://localhost:8080/api/payment?delay=100"
done

# 2. Slow Call 요청 (2.5초 지연 = slowCallDurationThreshold 2초 초과)
for i in {1..10}; do
  curl "http://localhost:8080/api/payment?delay=2500" &
done
wait

# 3. Slow Call Rate 확인
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.slow.call.rate
# → value: 66.7%

# 4. CB 상태 확인 (60% threshold 초과 → OPEN)
curl http://localhost:8080/actuator/circuitbreakers
# → state: OPEN
```

### Grafana 대시보드 쿼리

```promql
# Slow Call Rate 시각화
resilience4j_circuitbreaker_slow_call_rate{
  application="order-service",
  name="payment"
}

# Slow Call 수 (초당)
rate(resilience4j_circuitbreaker_calls_total{
  application="order-service",
  name="payment",
  kind="slow_successful"
}[1m])

# Circuit Breaker 상태 (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
resilience4j_circuitbreaker_state{
  application="order-service"
}

# 알람 규칙 (Slow Call Rate >= 50% 시 알람)
ALERT CircuitBreakerSlowCallHigh
  IF resilience4j_circuitbreaker_slow_call_rate > 50
  FOR 1m
  LABELS { severity="warning" }
  ANNOTATIONS {
    summary = "High slow call rate on {{ $labels.name }}"
  }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: DB 커넥션 풀 고갈로 인한 Slow Call 폭증

```
상황:
  order-service → DB 쿼리 응답 시간 정상: 50ms
  DB 커넥션 풀 고갈 시작: 쿼리 대기 시간 증가
  
  t=0s: 응답 시간 50ms → SLOW_SUCCESS(아님), SUCCESS 기록
  t=10s: 커넥션 풀 80% 점유 → 응답 시간 1.5s
         slowCallDurationThreshold: 2s → 아직 아님
  t=20s: 커넥션 풀 100% 점유 → 응답 시간 3s (대기)
         3s > 2s → SLOW_SUCCESS/SLOW_ERROR 기록
  t=30s: Slow Call Rate 70% > 60% threshold → OPEN!
         → order-service CB OPEN
         → 새 요청: 즉시 Fallback (DB 대기 없음)
         → DB 커넥션 요청 감소 → 풀 회복 시작
  t=35s: HALF_OPEN → DB 응답 시간 500ms
         2초 미만 → SLOW_SUCCESS 아님 → CLOSED
         정상 운영 재개

결과:
  DB 문제가 order-service 전체 장애로 번지기 전에 CB가 차단
  DB 커넥션 풀이 자연스럽게 회복 (CB가 새 요청 차단)
```

---

## ⚖️ 트레이드오프

| 설정 | 장점 | 단점 |
|------|------|------|
| `slowCallDurationThreshold` 낮게 | 빠른 Slow Call 탐지 | 정상 범위 요청도 Slow로 분류 |
| `slowCallDurationThreshold` 높게 | 오탐 낮음 | 심각한 지연만 탐지 |
| `slowCallRateThreshold` 낮게 | 빠른 OPEN | 일부 느린 요청에 과민 반응 |
| `slowCallRateThreshold` 높게 | 안정적 | 지연 지속 시간 길어짐 |

```
TimeLimiter + CircuitBreaker 조합 시 임계값 관계:

timeoutDuration(3s) > slowCallDurationThreshold(2s) 권장:
  2s~3s 사이 응답: Slow Call로 기록 (타임아웃 안 됨)
  3s 이상 응답: TimeLimiter가 TimeoutException 발생
              → CircuitBreaker가 실패로 기록

timeoutDuration(2s) = slowCallDurationThreshold(2s)이면:
  2s에서 동시에 타임아웃 + Slow Call threshold
  → TimeoutException이 발생하므로 SLOW_ERROR로 기록
  → 동작은 하지만 의미 구분이 모호

timeoutDuration(1s) < slowCallDurationThreshold(2s):
  1s에서 이미 타임아웃 → 모든 "Slow Call"이 실제로는 ERROR
  slowCallRate 항상 0% → Slow Call 탐지 무의미
```

---

## 📌 핵심 정리

```
Slow Call 탐지 핵심:

Outcome 분류:
  응답 시간 > slowCallDurationThreshold → SLOW_SUCCESS or SLOW_ERROR
  
  slowCallRate = (SLOW_SUCCESS + SLOW_ERROR) / 전체 × 100

OPEN 조건:
  failureRate >= failureRateThreshold (실패 기반)
  OR slowCallRate >= slowCallRateThreshold (지연 기반)
  → 두 조건 중 하나만 충족해도 OPEN

TimeLimiter와 역할 분리:
  TimeLimiter: 실제 호출 강제 타임아웃 (Future.cancel())
  CircuitBreaker: 결과 기록 및 상태 관리
  조합: TimeLimiter(바깥) → CircuitBreaker(안쪽) → 업스트림
  TimeLimiter 타임아웃 → TimeoutException → CB 실패(SLOW_ERROR)

메트릭:
  getNumberOfSlowCalls() = SLOW_SUCCESS + SLOW_ERROR
  getSlowCallRate() = slowCalls / 전체 × 100
  Actuator: resilience4j.circuitbreaker.slow.call.rate
```

---

## 🤔 생각해볼 문제

**Q1.** `slowCallDurationThreshold: 2s`와 `failureRateThreshold: 50%` 모두 설정된 상태에서, 응답 시간이 2.5초인 호출 10개가 모두 성공했다. Circuit Breaker 상태는 어떻게 되는가? 어떤 설정이 관여하는가?

<details>
<summary>해설 보기</summary>

모든 10개 호출이 2.5s > 2s(threshold)이므로 `SLOW_SUCCESS`로 기록됩니다. 이때:
- `failureRate = 0/10 = 0%` → failureRateThreshold(50%) 미달 → OPEN 아님
- `slowCallRate = 10/10 = 100%` → `slowCallRateThreshold`와 비교

`slowCallRateThreshold`를 설정하지 않으면 기본값이 100%입니다. 100% >= 100% → OPEN! `slowCallRateThreshold: 80`이면 100% >= 80% → OPEN! `slowCallRateThreshold: 100`이면 100% >= 100% → OPEN!

결론: `slowCallRateThreshold`를 설정하지 않으면 기본값(100%)이 적용되어 모든 호출이 Slow일 때만 OPEN됩니다. 적절한 threshold 설정이 중요합니다.

</details>

---

**Q2.** `@TimeLimiter`와 `@CircuitBreaker`를 동시에 사용할 때 어노테이션 순서가 중요한가?

<details>
<summary>해설 보기</summary>

중요합니다. 어노테이션은 AOP 프록시에서 적용 순서가 있습니다. 올바른 순서는:

```java
@CircuitBreaker(name = "payment")  // 안쪽 (나중에 적용)
@TimeLimiter(name = "payment")    // 바깥쪽 (먼저 적용)
public CompletableFuture<Result> process() { ... }
```

TimeLimiter가 바깥에 있어야 타임아웃이 발생하면 CircuitBreaker의 `onError()`가 호출됩니다. CircuitBreaker가 바깥에 있으면 TimeLimiter 타임아웃 예외를 CircuitBreaker가 포착하지 못할 수 있습니다.

그러나 Resilience4j Spring Boot Starter에서는 `@Order`가 자동으로 적용되어 대부분 올바른 순서가 보장됩니다. TimeLimiter는 Order 1, CircuitBreaker는 Order 2로 설정되어 TimeLimiter가 바깥에서 실행됩니다. 명시적으로 제어하려면 `resilience4j.timelimiter.instances.payment.order=1`, `resilience4j.circuitbreaker.instances.payment.order=2`로 설정합니다.

</details>

---

**Q3.** Slow Call 탐지와 타임아웃을 조합할 때, 클라이언트가 경험하는 최대 응답 시간은 `slowCallDurationThreshold`인가 `timeoutDuration`인가?

<details>
<summary>해설 보기</summary>

`timeoutDuration`입니다. `slowCallDurationThreshold`는 호출 결과를 분류하는 기준이지 실제로 호출을 중단시키지 않습니다. 응답이 2.5초에 도달해도 `slowCallDurationThreshold(2s)`를 넘겼을 뿐 호출은 계속 진행됩니다. `TimeLimiter.timeoutDuration(3s)`이 실제로 3초에 `Future.cancel()`을 호출하고 `TimeoutException`을 발생시킵니다. 따라서 클라이언트는 최대 3초(timeoutDuration)를 기다립니다.

실무 권장 관계: `timeoutDuration >= slowCallDurationThreshold + 여유시간`으로 설정해 의미 있는 구분을 만듭니다.

</details>

---

<div align="center">

**[⬅️ 이전: Failure Rate Threshold 계산](./03-failure-rate-threshold.md)** | **[홈으로 🏠](../README.md)** | **[다음: Fallback 메서드 작성 ➡️](./05-fallback-methods.md)**

</div>
