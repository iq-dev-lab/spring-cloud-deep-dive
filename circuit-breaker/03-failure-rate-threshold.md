# Failure Rate Threshold 계산 — SlidingWindow가 실패율을 판단하는 내부 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `FixedSizeSlidingWindow`의 원형 배열(Ring Buffer)이 실패율을 O(1)로 계산하는 방법은?
- `SlidingTimeWindow`의 시간 버킷(Epoch Second 버킷)이 만료된 데이터를 제거하는 방식은?
- `minimumNumberOfCalls` 없이 첫 번째 호출이 실패하면 실패율이 100%가 되는 문제는?
- `recordExceptionPredicate`로 특정 예외를 실패로 기록하지 않도록 설정하는 방법은?
- `ignoreExceptionPredicate`와 `recordExceptionPredicate`의 차이는?
- `Outcome` 열거형에 `SUCCESS`, `ERROR`, `SLOW_SUCCESS`, `SLOW_ERROR`가 있는 이유는?

---

## 🔍 왜 MSA에서 필요한가

### 실패율 계산이 Circuit Breaker의 핵심인 이유

```
Circuit Breaker의 근본 질문:
  "이 서비스를 계속 호출해도 되는가?"

판단 근거:
  최근 10번 중 5번 실패 → 50% 실패율 → 위험
  최근 10번 중 1번 실패 → 10% 실패율 → 허용 가능

단순 실패 카운트가 아닌 실패율인 이유:
  트래픽 100 RPS: 10번 실패 = 10% → 허용 가능
  트래픽 10 RPS: 10번 실패 = 100% → 위험

  절대값이 아닌 비율로 판단해야
  트래픽 변화에 관계없이 일관된 기준 적용 가능

실패율 계산의 정확성이 CB 신뢰성을 결정
  오탐(False Positive): 정상 서비스를 OPEN → 불필요한 서비스 중단
  오탐 없음(False Negative): 장애 서비스를 CLOSED → 지속적 오류
```

---

## 😱 잘못된 구성

### Before: minimumNumberOfCalls 없이 순간 오탐

```yaml
# ❌ minimumNumberOfCalls 설정 없음 (기본값: 100)
# 기본값이 100이므로 실제로는 괜찮지만, 명시적으로 낮게 설정할 때 문제
resilience4j:
  circuitbreaker:
    instances:
      my-service:
        slidingWindowSize: 10
        failureRateThreshold: 50
        minimumNumberOfCalls: 1    # ❌ 1개만 실패해도 OPEN 가능!
```

```
시나리오:
  서비스 방금 배포됨 (완전히 정상)
  첫 번째 요청: 네트워크 순간 불안정 → 실패
  
  minimumNumberOfCalls: 1이면:
  실패 1개 / 총 1개 = 100% > 50% threshold
  → 즉시 OPEN!
  
  정상 서비스가 단 한 번의 네트워크 이상으로 차단됨
  minimumNumberOfCalls: 5 이상으로 설정하는 이유
```

### Before: 비즈니스 예외를 실패로 잘못 기록

```java
// ❌ 404 Not Found를 서킷 브레이커 실패로 기록
// 404는 서비스 장애가 아닌 정상적인 비즈니스 응답
// 그런데 HttpClientErrorException을 예외로 감지해 OPEN

@CircuitBreaker(name = "order-service")
public Order getOrder(Long id) {
    // 404 Not Found → HttpClientErrorException.NotFound 발생
    // CB가 이를 실패로 기록 → 실패율 증가 → OPEN
    // 실제로는 "주문 없음"이라는 정상 응답
    return restTemplate.getForObject("/orders/" + id, Order.class);
}
```

---

## ✨ 올바른 패턴

### After: 올바른 예외 구분 설정

```yaml
resilience4j:
  circuitbreaker:
    instances:
      order-service:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5     # 최소 5개 이상 수집 후 판단
        failureRateThreshold: 50

        # 실패로 기록할 예외 (서비스 장애)
        record-exceptions:
          - java.io.IOException
          - java.net.ConnectException
          - org.springframework.web.client.HttpServerErrorException  # 5xx

        # 무시할 예외 (비즈니스 로직 오류, 실패 아님)
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException  # 4xx
          - com.example.exception.BusinessException  # 도메인 예외
```

---

## 🔬 내부 동작 원리

### 1. Outcome 열거형 — 호출 결과의 4가지 분류

```java
// Outcome.java
enum Outcome {
    SUCCESS,       // 성공 (응답 시간 < slowCallDurationThreshold)
    ERROR,         // 실패 (예외 발생)
    SLOW_SUCCESS,  // 느린 성공 (성공이지만 응답 시간 >= slowCallDurationThreshold)
    SLOW_ERROR     // 느린 실패 (예외 + 느린 응답)
}

// 실패율 계산:
// failureRate = (ERROR + SLOW_ERROR) / (SUCCESS + ERROR + SLOW_SUCCESS + SLOW_ERROR)
//              × 100

// Slow Call 실패율 계산 (Ch5-04에서 상세):
// slowCallRate = (SLOW_SUCCESS + SLOW_ERROR) / 전체 × 100

// OPEN 조건:
// failureRate >= failureRateThreshold
// OR slowCallRate >= slowCallRateThreshold
```

### 2. FixedSizeSlidingWindow — COUNT_BASED 구현

```java
// FixedSizeSlidingWindow.java
// 원형 배열(Ring Buffer) 기반 O(1) 실패율 계산

final class FixedSizeSlidingWindow implements Metrics {

    private final int size;
    private final TotalAggregation totalAggregation;  // 집계 합산값 (O(1) 조회용)
    private final Outcome[] measurements;             // 원형 배열
    private int headIndex;                            // 다음 기록 위치

    FixedSizeSlidingWindow(int size) {
        this.size = size;
        this.measurements = new Outcome[size];
        Arrays.fill(measurements, Outcome.SUCCESS);   // 초기값: 성공
        this.headIndex = 0;
        this.totalAggregation = new TotalAggregation();
    }

    @Override
    public synchronized Result record(long duration, TimeUnit unit, Outcome outcome) {
        // ① 현재 headIndex 위치의 가장 오래된 기록 읽기
        Outcome oldOutcome = measurements[headIndex];

        // ② 새 결과 기록 (가장 오래된 것 덮어쓰기)
        measurements[headIndex] = outcome;

        // ③ headIndex 이동 (원형)
        headIndex = (headIndex + 1) % size;

        // ④ 집계값 업데이트 (O(1))
        totalAggregation.record(duration, unit, outcome);
        totalAggregation.remove(oldOutcome);    // 제거된 것 빼기

        // ⑤ 임계값 초과 여부 반환
        return Result.of(
            totalAggregation.getFailureRate(),
            totalAggregation.getSlowCallRate());
    }

    // 예시: size=5, 기록 순서: 성공, 성공, 실패, 성공, 실패
    // headIndex=0: [SUCCESS, SUCCESS, SUCCESS, SUCCESS, SUCCESS]
    // headIndex=1: [SUCCESS, SUCCESS, SUCCESS, SUCCESS, SUCCESS] (첫 SUCCESS 기록)
    // ...
    // headIndex=0 (wrap): [ERROR, SUCCESS, ERROR, SUCCESS, SUCCESS]
    // totalAggregation: 실패 2개, 성공 3개 → 실패율 40%
}
```

### 3. TotalAggregation — O(1) 집계 유지

```java
// TotalAggregation.java
// 슬라이딩 윈도우의 집계값을 항상 최신 상태로 유지
// → 실패율 계산 시 O(1) 조회 가능 (전체 배열 순회 불필요)

class TotalAggregation {
    private long totalDurationInMillis = 0;
    private int numberOfSlowCalls = 0;
    private int numberOfSlowFailedCalls = 0;
    private int numberOfFailedCalls = 0;
    private int numberOfCalls = 0;

    void record(long duration, TimeUnit unit, Outcome outcome) {
        numberOfCalls++;
        totalDurationInMillis += unit.toMillis(duration);
        switch (outcome) {
            case SLOW_SUCCESS -> numberOfSlowCalls++;
            case SLOW_ERROR -> {
                numberOfSlowCalls++;
                numberOfSlowFailedCalls++;
                numberOfFailedCalls++;
            }
            case ERROR -> numberOfFailedCalls++;
        }
    }

    // 윈도우에서 제거된 오래된 Outcome 반영
    void remove(Outcome outcome) {
        numberOfCalls--;
        switch (outcome) {
            case SLOW_SUCCESS -> numberOfSlowCalls--;
            case SLOW_ERROR -> {
                numberOfSlowCalls--;
                numberOfSlowFailedCalls--;
                numberOfFailedCalls--;
            }
            case ERROR -> numberOfFailedCalls--;
        }
    }

    float getFailureRate() {
        if (numberOfCalls == 0) return -1f;
        return (float) numberOfFailedCalls / numberOfCalls * 100;
    }
}
```

### 4. SlidingTimeWindow — TIME_BASED 구현

```java
// SlidingTimeWindow.java
// 초(Epoch Second) 단위 버킷 배열

final class SlidingTimeWindow implements Metrics {

    private final int timeSlotSize;                  // 윈도우 크기 (초)
    private final PartialAggregation[] timeSlots;    // 초당 버킷
    private final TotalAggregation totalAggregation;
    private long headEpochSecond;                    // 현재 시각(초)

    SlidingTimeWindow(int timeSlotSizeInSeconds) {
        this.timeSlotSize = timeSlotSizeInSeconds;
        this.timeSlots = new PartialAggregation[timeSlotSizeInSeconds];
        // 각 버킷 초기화
        IntStream.range(0, timeSlotSizeInSeconds)
            .forEach(i -> timeSlots[i] = new PartialAggregation());
        this.headEpochSecond = epochSecond();
        this.totalAggregation = new TotalAggregation();
    }

    @Override
    public synchronized Result record(long duration, TimeUnit unit, Outcome outcome) {
        long currentEpochSecond = epochSecond();

        // ① 만료된 버킷 제거 (현재 시각 - 윈도우 크기보다 오래된 것)
        long secondsToEvict = currentEpochSecond - headEpochSecond;
        if (secondsToEvict > 0) {
            evictOldEntries(secondsToEvict);
            headEpochSecond = currentEpochSecond;
        }

        // ② 현재 초 버킷에 기록
        int slotIndex = (int)(currentEpochSecond % timeSlotSize);
        timeSlots[slotIndex].record(duration, unit, outcome);
        totalAggregation.record(duration, unit, outcome);

        return Result.of(
            totalAggregation.getFailureRate(),
            totalAggregation.getSlowCallRate());
    }

    private void evictOldEntries(long secondsToEvict) {
        // 만료된 초만큼 버킷 순회하여 집계에서 제거
        long evictUntil = Math.min(secondsToEvict, timeSlotSize);
        for (int i = 1; i <= evictUntil; i++) {
            int slotIndex = (int)((headEpochSecond + i) % timeSlotSize);
            PartialAggregation evictedSlot = timeSlots[slotIndex];
            // 만료 버킷의 집계값을 totalAggregation에서 제거
            totalAggregation.removeBucket(evictedSlot);
            // 버킷 초기화 (재사용)
            evictedSlot.reset();
        }
    }
}
```

### 5. 예외 분류 — record vs ignore

```java
// CircuitBreakerConfig에서 예외 분류 설정

CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    // 실패로 기록할 예외 (OPEN 트리거에 기여)
    .recordExceptions(
        IOException.class,
        TimeoutException.class,
        HttpServerErrorException.class)  // 5xx

    // 무시할 예외 (실패로 기록 안 함, 성공으로도 안 함)
    .ignoreExceptions(
        HttpClientErrorException.class,  // 4xx (비즈니스 오류)
        ValidationException.class)

    // 조건부: Predicate로 세밀한 제어
    .recordException(throwable -> {
        if (throwable instanceof HttpClientErrorException ex) {
            // 400, 404는 무시, 429는 실패로 기록
            return ex.getStatusCode() == HttpStatus.TOO_MANY_REQUESTS;
        }
        return true;  // 나머지 예외는 실패로 기록
    })
    .build();

// 우선순위:
// ignoreExceptions → recordExceptions → recordExceptionPredicate
// ignoreExceptions에 있으면 항상 무시 (Predicate보다 우선)
```

---

## 💻 실전 구성

### minimumNumberOfCalls 적절한 값 선택

```yaml
resilience4j:
  circuitbreaker:
    instances:
      high-traffic:     # 초당 1000 RPS
        slidingWindowSize: 100
        minimumNumberOfCalls: 20   # 2% → 빠른 판단
        failureRateThreshold: 50

      low-traffic:      # 초당 1 RPS
        slidingWindowSize: 10
        minimumNumberOfCalls: 5    # 50초 치 → 충분한 데이터
        failureRateThreshold: 50
        # 1 RPS에서 5개 = 5초 데이터
        # slidingWindowSize TIME_BASED 30초 + minimumNumberOfCalls 5 조합 권장

      internal-batch:   # 배치 작업 (간헐적)
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 5       # 최근 5번만 판단 (간헐적이므로)
        minimumNumberOfCalls: 3
        failureRateThreshold: 60   # 더 관대하게
```

### 실패율 실시간 모니터링

```bash
# Micrometer 메트릭으로 실패율 조회
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.failure.rate
# {
#   "name": "resilience4j.circuitbreaker.failure.rate",
#   "measurements": [{"statistic": "VALUE", "value": 30.0}],
#   "availableTags": [{"tag": "name", "values": ["inventory"]}]
# }

# Prometheus 쿼리 (Grafana 대시보드)
# resilience4j_circuitbreaker_state{name="inventory"}
# resilience4j_circuitbreaker_failure_rate{name="inventory"}
# resilience4j_circuitbreaker_calls_total{name="inventory", kind="failed"}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 부분 실패로 인한 실패율 변화 추적

```
설정:
  slidingWindowSize: 10, minimumNumberOfCalls: 5, threshold: 50%

시간순 호출 결과 (O=성공, X=실패):
  [O, O, O, O, O] → 5개, 실패율 0%, minimum 충족
  [O, O, O, O, O, X] → 6개, 실패율 17%
  [O, O, O, O, O, X, X] → 7개, 실패율 29%
  [O, O, O, O, O, X, X, X] → 8개, 실패율 38%
  [O, O, O, O, O, X, X, X, X] → 9개, 실패율 44%
  [O, O, O, O, O, X, X, X, X, X] → 10개, 실패율 50%

  ← 여기서 50% 도달 → OPEN!

이후 오래된 성공이 밀려나면서:
  [O, O, O, O, X, X, X, X, X, (새 요청)] → 윈도우 가득 참
  오래된 O들이 빠지고 X들 비율 증가
  → 더 빠르게 OPEN 유지됨

복구 후:
  HALF_OPEN → 성공 3개 → CLOSED
  새 윈도우 시작 (CLOSED로 전환 시 메트릭 리셋)
```

---

## ⚖️ 트레이드오프

| 설정 조합 | 반응성 | 안정성 | 권장 환경 |
|----------|--------|--------|----------|
| `size=5, minimum=3, threshold=50%` | 매우 빠름 | 낮음 (오탐) | 테스트, 빠른 차단 |
| `size=20, minimum=10, threshold=50%` | 빠름 | 중간 | 일반 서비스 |
| `size=100, minimum=50, threshold=60%` | 느림 | 높음 | 안정성 중요 서비스 |

```
failureRateThreshold 선택 기준:
  30%: 매우 민감 (조기 차단) → 잦은 OPEN, 오탐 위험
  50%: 균형 (기본값 권장)
  70%: 관대 (늦은 차단) → 장애 지속 기간 길어짐

  실무: 서비스별 정상 실패율 × 2 정도를 threshold로 설정
  예: 정상 실패율 10% → threshold: 20~30%

ignoreExceptions vs recordExceptions 선택:
  ignoreExceptions: "이 예외는 서비스 장애가 아님" (비즈니스 예외)
  recordExceptions: "이 예외만 장애로 기록" (whitelist 방식)
  
  단, ignoreExceptions이 recordExceptions보다 우선순위 높음
  둘 다 지정 시 ignoreExceptions 먼저 체크
```

---

## 📌 핵심 정리

```
Failure Rate 계산 핵심:

Outcome 4가지:
  SUCCESS, ERROR, SLOW_SUCCESS, SLOW_ERROR
  failureRate = (ERROR + SLOW_ERROR) / 전체

FixedSizeSlidingWindow (COUNT_BASED):
  원형 배열(Ring Buffer) 크기 N
  새 기록 → 가장 오래된 것 덮어쓰기
  TotalAggregation으로 O(1) 집계 유지
  → 실패율 계산 O(1)

SlidingTimeWindow (TIME_BASED):
  초당 버킷 배열 (크기 = 윈도우 초 수)
  만료 버킷 제거 후 집계값 갱신
  → 실패율 계산 O(W) (만료 시)

minimumNumberOfCalls:
  이 수 이하에서는 실패율 계산 안 함 (오탐 방지)
  예: 1번 실패 = 100% → OPEN 방지

예외 분류:
  ignoreExceptions: 성공도 실패도 아님 (무시)
  recordExceptions: 실패로 기록 (OPEN 트리거)
  recordException(Predicate): 세밀한 조건부 분류
  우선순위: ignoreExceptions > recordExceptions
```

---

## 🤔 생각해볼 문제

**Q1.** `COUNT_BASED` 슬라이딩 윈도우에서 최초 N개 호출이 채워지기 전의 실패율은 어떻게 처리되는가? 초기화 값이 모두 `SUCCESS`이면 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

`FixedSizeSlidingWindow`의 원형 배열은 `SUCCESS`로 초기화됩니다. 호출이 채워지기 전에는 `TotalAggregation.numberOfCalls`가 실제 호출 수보다 적습니다. 하지만 `minimumNumberOfCalls` 조건이 충족되지 않으면 실패율을 계산해도 임계값과 비교하지 않습니다(`Result.BELOW_MINIMUM_CALLS_THRESHOLD`). 초기값이 SUCCESS인 영향: 첫 몇 번의 실패가 있어도 SUCCESS 초기값들 때문에 실패율이 낮게 계산됩니다. 예를 들어 size=10, minimum=5, 처음 3번 실패 → `3/(3+7_dummy_success) = 30%`가 아닌 실제 `3/3 = 100%`로 계산됩니다. Resilience4j는 실제 기록된 것만 `numberOfCalls`로 카운트하므로 dummy SUCCESS는 집계에 포함되지 않습니다.

</details>

---

**Q2.** `recordException(Predicate)`에서 네트워크 타임아웃(SocketTimeoutException)과 비즈니스 로직 오류(IllegalArgumentException)를 구분해 처리하려면 어떻게 구현하는가?

<details>
<summary>해설 보기</summary>

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .recordException(throwable -> {
        // 네트워크/인프라 오류 → 실패로 기록
        if (throwable instanceof java.net.SocketTimeoutException
                || throwable instanceof java.io.IOException
                || throwable instanceof java.net.ConnectException) {
            return true;
        }
        // HTTP 5xx → 실패로 기록
        if (throwable instanceof HttpServerErrorException ex
                && ex.getStatusCode().is5xxServerError()) {
            return true;
        }
        // 비즈니스 로직 오류 → 무시 (실패 아님)
        if (throwable instanceof IllegalArgumentException
                || throwable instanceof BusinessException) {
            return false;
        }
        // 기본: 예상치 못한 예외는 실패로 기록
        return true;
    })
    .build();
```

중요: `recordException(Predicate)`은 `ignoreExceptions`보다 낮은 우선순위입니다. `ignoreExceptions`에 포함된 예외는 Predicate 평가 없이 무시됩니다.

</details>

---

**Q3.** TIME_BASED 윈도우에서 30초 동안 요청이 전혀 없다가 갑자기 요청이 오면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

30초 동안 요청이 없으면 모든 버킷이 만료됩니다. 새 요청이 올 때 `evictOldEntries()`가 호출되어 만료 버킷들이 정리됩니다. 이때 `totalAggregation`의 모든 카운터가 0으로 리셋됩니다. 결과적으로 `numberOfCalls = 0` → `minimumNumberOfCalls` 조건 미충족 → 실패율 계산 없음(CLOSED 유지). 30초 공백 후 첫 번째 호출은 빈 슬라이딩 윈도우에서 시작하므로, 사실상 새로 시작하는 것과 같습니다. 이것이 TIME_BASED가 트래픽이 간헐적인 서비스에 불리한 이유입니다 — 공백 후 `minimumNumberOfCalls`를 다시 채워야 CB가 동작합니다.

</details>

---

<div align="center">

**[⬅️ 이전: CLOSED → OPEN → HALF_OPEN 상태 전이](./02-state-machine-transitions.md)** | **[홈으로 🏠](../README.md)** | **[다음: Slow Call 탐지 ➡️](./04-slow-call-detection.md)**

</div>
