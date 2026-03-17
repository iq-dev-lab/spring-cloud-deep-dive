# CLOSED → OPEN → HALF_OPEN 상태 전이 — CircuitBreakerStateMachine 내부 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CircuitBreakerStateMachine`이 상태를 관리하는 자료구조와 원자적 전이 보장 방법은?
- `COUNT_BASED` 슬라이딩 윈도우와 `TIME_BASED` 슬라이딩 윈도우의 내부 자료구조 차이는?
- HALF_OPEN 상태에서 `permittedNumberOfCallsInHalfOpenState` 제한을 구현하는 방법은?
- 상태 전이 이벤트(`StateTransition`)를 구독해 모니터링·알람을 설정하는 방법은?
- `waitDurationInOpenState`가 정확히 어느 시점부터 카운트되는가?
- 수동 상태 전이(`transitionToOpenState()`)와 자동 전이의 차이점은?

---

## 🔍 왜 MSA에서 필요한가

### 단순 On/Off가 아닌 3가지 상태가 필요한 이유

```
단순 On/Off (CLOSED/OPEN만 있다면):

  service-b 다운
  → Circuit OPEN (요청 차단)
  
  이후 service-b 복구
  → 누가 언제 CLOSED로 돌아가는지 알 수 없음
  → 운영자가 수동으로 CLOSED 전환해야 함
  → 운영 복잡도 급상승

  또는 자동으로 일정 시간 후 CLOSED로?
  → 바로 CLOSED되면 아직 불안정한 service-b에 트래픽 몰림
  → 다시 OPEN → CLOSED → OPEN 반복 (Flapping)

HALF_OPEN이 해결하는 것:
  OPEN 일정 시간 후 → HALF_OPEN (탐색 상태)
  → 제한된 요청만 service-b로 전달 (예: 3개)
  → 3개 모두 성공 → 복구 확인 → CLOSED
  → 1개라도 실패 → 아직 불안정 → 다시 OPEN

  자동 복구 + 안전한 탐색 = 운영자 개입 없이 자가 치유
```

---

## 😱 잘못된 구성

### Before: HALF_OPEN 허용 요청 수를 너무 많이 설정

```yaml
# ❌ HALF_OPEN 탐색 요청이 너무 많음
resilience4j:
  circuitbreaker:
    instances:
      my-service:
        permittedNumberOfCallsInHalfOpenState: 100
        # service-b가 아직 불안정한데 100개 요청을 보냄
        # 100개 중 60개 실패 → OPEN 복귀
        # 이 60개 실패 동안 클라이언트는 오류 응답
```

### Before: waitDurationInOpenState를 너무 짧게 설정

```yaml
# ❌ 복구 시간보다 짧은 OPEN 유지 시간
resilience4j:
  circuitbreaker:
    instances:
      db-service:
        waitDurationInOpenState: 1s  # 1초 후 HALF_OPEN
        # DB 재시작에 15초 걸리는데 1초 만에 탐색
        # → HALF_OPEN → 실패 → OPEN → 1초 후 HALF_OPEN → 실패...
        # → 불필요한 OPEN/HALF_OPEN 반복 (오버헤드)
```

---

## ✨ 올바른 패턴

### After: 서비스 특성에 맞는 상태 전이 설정

```yaml
resilience4j:
  circuitbreaker:
    instances:
      # 일반 외부 API
      external-api:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s      # API 재시작 시간 고려
        permittedNumberOfCallsInHalfOpenState: 3   # 소량 탐색

      # DB 연결 (재시작에 30초 이상)
      database:
        slidingWindowType: TIME_BASED
        slidingWindowSize: 30             # 30초 윈도우
        minimumNumberOfCalls: 10
        failureRateThreshold: 60
        waitDurationInOpenState: 30s      # DB 재시작 대기
        permittedNumberOfCallsInHalfOpenState: 5

      # 핵심 결제 서비스 (매우 보수적)
      payment:
        slidingWindowSize: 20
        failureRateThreshold: 30          # 더 민감하게 (30%만 실패해도 OPEN)
        waitDurationInOpenState: 60s      # 1분 대기
        permittedNumberOfCallsInHalfOpenState: 2
```

---

## 🔬 내부 동작 원리

### 1. CircuitBreakerStateMachine — 상태 관리 핵심

```java
// CircuitBreakerStateMachine.java
final class CircuitBreakerStateMachine implements CircuitBreaker {

    // ① 현재 상태를 원자적으로 관리 (AtomicReference)
    // 동시 상태 전이 시 race condition 방지
    private final AtomicReference<CircuitBreakerState> stateReference;

    // ② 초기 상태: CLOSED
    CircuitBreakerStateMachine(String name, CircuitBreakerConfig config) {
        this.stateReference = new AtomicReference<>(
            new ClosedState(this));  // CLOSED 상태 객체
    }

    // ③ 상태별 동작 위임 패턴 (State 패턴)
    @Override
    public boolean tryAcquirePermission() {
        // 현재 상태 객체에게 permission 획득 위임
        return stateReference.get().tryAcquirePermission();
    }

    @Override
    public void onError(long duration, TimeUnit unit, Throwable throwable) {
        // 실패 기록 → 상태 전이 트리거 가능
        stateReference.get().onError(duration, unit, throwable);
    }

    @Override
    public void onSuccess(long duration, TimeUnit unit) {
        stateReference.get().onSuccess(duration, unit);
    }

    // ④ 상태 전이 메서드 (원자적)
    void transitionToOpenState() {
        CircuitBreakerState currentState = stateReference.get();

        // CAS로 원자적 전이 (동시 전이 방지)
        if (stateReference.compareAndSet(currentState,
                new OpenState(this, currentState.getMetrics()))) {

            // 전이 성공 시 이벤트 발행
            publishStateTransitionEvent(
                StateTransition.CLOSED_TO_OPEN);
        }
        // CAS 실패 = 다른 스레드가 이미 전이 완료 → 무시
    }

    void transitionToHalfOpenState() {
        CircuitBreakerState currentState = stateReference.get();
        if (stateReference.compareAndSet(currentState,
                new HalfOpenState(this))) {
            publishStateTransitionEvent(
                StateTransition.OPEN_TO_HALF_OPEN);
        }
    }

    void transitionToClosedState() {
        CircuitBreakerState currentState = stateReference.get();
        if (stateReference.compareAndSet(currentState,
                new ClosedState(this))) {
            publishStateTransitionEvent(
                StateTransition.HALF_OPEN_TO_CLOSED);
        }
    }
}
```

### 2. 각 State 객체의 역할

```java
// ClosedState: 정상 운영
class ClosedState extends CircuitBreakerState {

    private final CircuitBreakerMetrics circuitBreakerMetrics;

    @Override
    boolean tryAcquirePermission() {
        // CLOSED에서는 항상 허용
        return true;
    }

    @Override
    void onError(long duration, TimeUnit unit, Throwable throwable) {
        // 슬라이딩 윈도우에 실패 기록
        checkIfThresholdsExceeded(
            circuitBreakerMetrics.onError(duration, unit));
        // 실패율 >= threshold → transitionToOpenState()
    }

    @Override
    void onSuccess(long duration, TimeUnit unit) {
        checkIfThresholdsExceeded(
            circuitBreakerMetrics.onSuccess(duration, unit));
        // 성공도 기록 (Slow Call 탐지 포함)
    }

    private void checkIfThresholdsExceeded(Result result) {
        if (Result.hasExceededThresholds(result)) {
            // 임계값 초과 → OPEN으로 전이
            circuitBreaker.transitionToOpenState();
        }
    }
}

// OpenState: 차단 상태
class OpenState extends CircuitBreakerState {

    private final long retryAfterWaitDuration;  // 전환 가능 시점

    OpenState(CircuitBreakerStateMachine sm, CircuitBreakerMetrics metrics) {
        // OPEN 상태 진입 시각 기록
        // waitDurationInOpenState 이후 HALF_OPEN 가능
        this.retryAfterWaitDuration = System.nanoTime()
            + sm.getCircuitBreakerConfig()
                .getWaitDurationInOpenState()
                .toNanos();
    }

    @Override
    boolean tryAcquirePermission() {
        // waitDuration 경과했으면 HALF_OPEN으로 전이 후 허용
        if (System.nanoTime() >= retryAfterWaitDuration) {
            circuitBreaker.transitionToHalfOpenState();
            return acquirePermissionInHalfOpenState();
        }
        // 아직 대기 중 → 차단
        circuitBreaker.publishCallNotPermittedEvent();
        return false;
    }

    @Override
    void onError(long duration, TimeUnit unit, Throwable throwable) {
        // OPEN 중에는 메트릭 기록 안 함 (통과 요청 없으므로)
    }
}

// HalfOpenState: 탐색 상태
class HalfOpenState extends CircuitBreakerState {

    // 허용된 요청 수 추적 (Semaphore)
    private final AtomicInteger allowedCalls;

    HalfOpenState(CircuitBreakerStateMachine sm) {
        this.allowedCalls = new AtomicInteger(
            sm.getCircuitBreakerConfig()
                .getPermittedNumberOfCallsInHalfOpenState());
    }

    @Override
    boolean tryAcquirePermission() {
        // 허용 요청 수 남아있으면 감소 후 허용
        int permittedCalls = allowedCalls.decrementAndGet();
        if (permittedCalls >= 0) {
            return true;
        }
        // 허용 수 초과 → 차단
        circuitBreaker.publishCallNotPermittedEvent();
        return false;
    }

    @Override
    void onError(long duration, TimeUnit unit, Throwable throwable) {
        checkIfThresholdsExceeded(
            circuitBreakerMetrics.onError(duration, unit));
        // HALF_OPEN 중 실패 → 다시 OPEN
    }

    @Override
    void onSuccess(long duration, TimeUnit unit) {
        checkIfThresholdsExceeded(
            circuitBreakerMetrics.onSuccess(duration, unit));
        // 허용된 요청 모두 성공 → CLOSED
    }
}
```

### 3. SlidingWindowType — COUNT_BASED vs TIME_BASED

```java
// COUNT_BASED: 마지막 N번의 호출 기반
// 자료구조: 크기 N의 원형 배열 (Ring Buffer)

class FixedSizeSlidingWindow implements Metrics {
    // 예: slidingWindowSize = 10
    // [성공, 실패, 성공, 성공, 실패, 성공, 성공, 실패, 성공, 성공]
    // 실패 3개 / 총 10개 = 30% 실패율

    private final TotalAggregation totalAggregation;
    private final Outcome[] ring;         // 원형 배열
    private int headIndex;                // 다음 기록 위치

    Result record(long duration, TimeUnit unit, Outcome outcome) {
        Outcome oldOutcome = ring[headIndex];    // 가장 오래된 기록 제거
        ring[headIndex] = outcome;               // 새 기록 추가
        headIndex = (headIndex + 1) % ring.length;  // 다음 위치

        // 전체 집계 업데이트 (O(1))
        totalAggregation.record(duration, unit, outcome);
        totalAggregation.remove(oldOutcome);     // 제거된 것 빼기
        return checkIfThresholdsExceeded(totalAggregation);
    }
}

// TIME_BASED: 최근 N초 동안의 호출 기반
// 자료구조: 시간 버킷 배열 (Epoch Second 단위)

class SlidingTimeWindow implements Metrics {
    // 예: slidingWindowSize = 30 (30초 윈도우)
    // 각 초마다 버킷 생성 → 30개 버킷
    // 현재 시각 기준 30초 이전 버킷은 제거

    private final PartialAggregation[] timeSlots;  // 초당 버킷 배열
    private final int timeSlotSize;                 // = slidingWindowSize (초)

    Result record(long duration, TimeUnit unit, Outcome outcome) {
        long currentEpochSecond = clock.wallTime() / 1000;
        int slotIndex = (int)(currentEpochSecond % timeSlotSize);

        // 현재 초 버킷에 기록
        // 만료된 버킷(30초 이전) 있으면 제거 후 집계 재계산
        return evictOldEntriesAndRecord(slotIndex, duration, unit, outcome);
    }
}
```

### 4. StateTransition 이벤트 구독

```java
// CircuitBreaker 이벤트 구독 (모니터링, 알람, 로깅)

CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("inventory");

// 상태 전이 이벤트 구독
cb.getEventPublisher()
    .onStateTransition(event -> {
        log.info("Circuit Breaker [{}] state changed: {} → {}",
            event.getCircuitBreakerName(),
            event.getStateTransition().getFromState(),
            event.getStateTransition().getToState());

        // OPEN 전환 시 알람
        if (event.getStateTransition() ==
                StateTransition.CLOSED_TO_OPEN) {
            alertManager.sendAlert(
                "Circuit Breaker OPEN: " + event.getCircuitBreakerName());
        }
    });

// 차단된 요청 이벤트 구독
cb.getEventPublisher()
    .onCallNotPermitted(event -> {
        metrics.increment("circuit_breaker.calls.not_permitted",
            Tags.of("name", event.getCircuitBreakerName()));
    });

// 성공/실패 이벤트 구독
cb.getEventPublisher()
    .onSuccess(event -> log.debug("Success: {}ms",
        event.getElapsedDuration().toMillis()))
    .onError(event -> log.warn("Error: {} - {}ms",
        event.getThrowable().getMessage(),
        event.getElapsedDuration().toMillis()));
```

---

## 💻 실전 구성

### 상태 전이 완전 실험

```java
// 상태 전이 수동 제어 (테스트 및 운영)
@RestController
@RequestMapping("/admin/circuit-breaker")
public class CircuitBreakerAdminController {

    private final CircuitBreakerRegistry registry;

    // 강제 OPEN (긴급 차단)
    @PostMapping("/{name}/open")
    public String forceOpen(@PathVariable String name) {
        registry.circuitBreaker(name).transitionToOpenState();
        return "Circuit Breaker [" + name + "] forced to OPEN";
    }

    // 강제 CLOSED (복구 후 수동 활성화)
    @PostMapping("/{name}/close")
    public String forceClose(@PathVariable String name) {
        registry.circuitBreaker(name).transitionToClosedState();
        return "Circuit Breaker [" + name + "] forced to CLOSED";
    }

    // 현재 상태 조회
    @GetMapping("/{name}/state")
    public Map<String, Object> getState(@PathVariable String name) {
        CircuitBreaker cb = registry.circuitBreaker(name);
        Metrics metrics = cb.getMetrics();
        return Map.of(
            "state", cb.getState(),
            "failureRate", metrics.getFailureRate(),
            "slowCallRate", metrics.getSlowCallRate(),
            "bufferedCalls", metrics.getNumberOfBufferedCalls(),
            "failedCalls", metrics.getNumberOfFailedCalls()
        );
    }
}
```

```bash
# 상태 확인
curl http://localhost:8080/actuator/circuitbreakers
# → state: CLOSED, failureRate: 30.0%

# 장애 시뮬레이션 후 OPEN 확인
for i in {1..10}; do
  curl -s http://localhost:8080/api/inventory/1 | jq '.error'
done
curl http://localhost:8080/actuator/circuitbreakers
# → state: OPEN, failureRate: 100.0%

# waitDuration 후 HALF_OPEN 전환 확인
sleep 10
curl http://localhost:8080/api/inventory/1  # 탐색 요청
curl http://localhost:8080/actuator/circuitbreakers
# → state: HALF_OPEN
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: COUNT_BASED vs TIME_BASED 선택 기준

```
COUNT_BASED 적합한 경우:
  요청 빈도가 균일한 서비스
  예: 내부 API (초당 100건 이상 안정적)
  
  장점: 직관적 (최근 N번 중 X번 실패)
  단점: 요청 빈도가 낮을 때 오래된 데이터 오래 유지
        자정에 갑자기 요청 없어지면 오래된 실패 데이터 유지

  예: slidingWindowSize=10, 오전 2시에 요청 드물음
      10번 채우는 데 1시간 → 지난 1시간의 데이터로 판단
      → 새벽에 실패해도 낮에 그 영향이 남음

TIME_BASED 적합한 경우:
  요청 빈도가 시간대별로 크게 다른 서비스
  예: 외부 사용자 API (낮에 많고 새벽에 적음)
  
  장점: 항상 최근 N초 데이터만 반영
  단점: 요청이 드물면 minimumNumberOfCalls 조건 충족 어려움
        "최근 30초에 3건 이하" → CB 발동 안 됨

실무 선택:
  내부 서비스 간 호출 → COUNT_BASED (균일한 빈도)
  외부 사용자 API → TIME_BASED (시간대별 편차)
```

---

## ⚖️ 트레이드오프

| 설정 | 빠른 장애 감지 | 오탐 가능성 | 적합한 케이스 |
|------|--------------|------------|--------------|
| **slidingWindowSize 작게** | 빠름 | 높음 | 장애 즉각 대응 중요 |
| **slidingWindowSize 크게** | 느림 | 낮음 | 안정적 판단 중요 |
| **waitDurationInOpenState 짧게** | - | - | 빠른 복구 시도 |
| **waitDurationInOpenState 길게** | - | - | 안정적 복구 대기 |

```
COUNT_BASED vs TIME_BASED 내부 비용:

COUNT_BASED (원형 배열):
  기록: O(1) (인덱스 증가 + 원소 교체)
  조회: O(1) (집계값 유지)
  메모리: slidingWindowSize × (크기 상수)

TIME_BASED (시간 버킷):
  기록: O(1) (현재 버킷에 추가)
  조회: O(W) (만료 버킷 제거 시, W=윈도우 크기(초))
  메모리: slidingWindowSize(초) × 버킷 크기
  → 윈도우가 클수록 메모리와 계산 비용 증가
```

---

## 📌 핵심 정리

```
상태 전이 핵심:

CircuitBreakerStateMachine:
  AtomicReference<CircuitBreakerState>: 현재 상태 원자적 관리
  CAS(compareAndSet)으로 동시 전이 방지
  State 패턴: 각 상태 객체가 동작 정의

전이 조건:
  CLOSED → OPEN: 실패율 >= threshold (Ch5-03 상세)
  OPEN → HALF_OPEN: waitDurationInOpenState 경과
  HALF_OPEN → CLOSED: 허용 요청 모두 성공
  HALF_OPEN → OPEN: 실패 발생

SlidingWindow:
  COUNT_BASED: 원형 배열, O(1), 요청 빈도 균일 시 적합
  TIME_BASED: 시간 버킷, O(W), 빈도 변동 클 때 적합

이벤트 구독:
  onStateTransition: 상태 변경 감지 → 알람
  onCallNotPermitted: OPEN 중 차단 카운트
  onError/onSuccess: 개별 호출 결과 추적
```

---

## 🤔 생각해볼 문제

**Q1.** `transitionToHalfOpenState()`가 `OpenState.tryAcquirePermission()` 내부에서 호출된다. 여러 스레드가 동시에 `waitDurationInOpenState` 경과 시점에 요청하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`CircuitBreakerStateMachine.transitionToHalfOpenState()`는 `AtomicReference.compareAndSet()`으로 구현되므로, 여러 스레드가 동시에 호출해도 오직 하나의 스레드만 성공합니다. 나머지 스레드들은 CAS 실패 후 다시 현재 상태(`stateReference.get()`)를 조회합니다. 이미 HALF_OPEN 상태로 전환됐으므로 `HalfOpenState.tryAcquirePermission()`이 호출됩니다. `HalfOpenState`는 `AtomicInteger`로 허용 카운트를 관리하므로, 설정된 `permittedNumberOfCallsInHalfOpenState` 이내로만 요청이 통과합니다. 즉, 동시 요청이 많아도 허용된 수만큼만 HALF_OPEN 탐색에 참여합니다.

</details>

---

**Q2.** HALF_OPEN에서 `permittedNumberOfCallsInHalfOpenState: 3`으로 설정했을 때, 3개 중 2개 성공, 1개 실패하면 어떤 상태로 전환되는가?

<details>
<summary>해설 보기</summary>

다시 OPEN 상태로 전환됩니다. HALF_OPEN의 실패율을 계산할 때 `failureRateThreshold`와 비교합니다. 3개 중 1개 실패 = 33% 실패율입니다. `failureRateThreshold: 50`이면 33% < 50% → CLOSED로 전환됩니다. `failureRateThreshold: 30`이면 33% >= 30% → OPEN으로 재전환됩니다. HALF_OPEN은 허용된 요청 전체가 완료된 후 실패율을 계산합니다. 3개 모두 처리되기 전에는 판단을 미룹니다. 

결론: HALF_OPEN 결과는 `failureRateThreshold` 설정에 따라 달라지며, 반드시 "모두 성공해야 CLOSED"가 아닙니다.

</details>

---

**Q3.** `DISABLED` 상태의 Circuit Breaker에 `transitionToOpenState()`를 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`DISABLED` 상태에서의 수동 전이는 상태에 따라 다릅니다. `DISABLED` → `CLOSED` 전이는 `transitionToClosedState()`로 가능합니다. 하지만 `DISABLED` → `OPEN` 전이는 내부 상태 머신의 전이 규칙에 따라 허용될 수도, 거부될 수도 있습니다. Resilience4j 구현에서는 `DISABLED` → `FORCED_OPEN`을 `transitionToForcedOpenState()`로 전환하는 것을 권장합니다. `FORCED_OPEN`은 "모든 요청 영구 차단" 상태로 운영자 의도가 명확합니다. `transitionToOpenState()`는 자동 전이(실패율 초과 시) 경로에서만 사용하고, 수동 차단은 `transitionToForcedOpenState()`를 사용하는 것이 의미상 올바릅니다.

</details>

---

<div align="center">

**[⬅️ 이전: Circuit Breaker 패턴과 필요성](./01-circuit-breaker-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: Failure Rate Threshold 계산 ➡️](./03-failure-rate-threshold.md)**

</div>
