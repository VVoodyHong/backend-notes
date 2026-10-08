---
category: "Java/동시성과 멀티스레딩"
tags: ["java", "synchronized", "lock", "concurrency", "jmm"]
updated: 2026-10-04
verified: 2026-09-08
---

# synchronized와 Lock

## 핵심 정의
`synchronized`는 자바 언어 차원에서 제공하는 상호 배제(mutual exclusion) 키워드로, 객체의 모니터(monitor, intrinsic lock)를 이용해 임계 구역(critical section)에 대한 접근을 한 번에 하나의 스레드로 제한한다. `java.util.concurrent.locks.Lock` 인터페이스(대표 구현체 `ReentrantLock`)는 같은 목적을 라이브러리 수준에서 제공하되, 타임아웃, 인터럽트 가능 대기, 공정성(fairness) 옵션 등 `synchronized`가 지원하지 않는 세밀한 제어를 제공한다.

두 방식 모두 상호 배제와 가시성(visibility)을 함께 보장하지만, 락 획득/해제의 유연성과 실패 처리 방식에서 근본적인 차이가 있다.

## 동작 원리 / 구조

| 구분 | synchronized | ReentrantLock |
|---|---|---|
| 락 확보 방식 | JVM이 자동 관리 (블록/메서드 진입·탈출) | `lock()`/`unlock()` 명시 호출 |
| 락 해제 보장 | 예외 발생 시에도 자동 해제 | `finally`에서 수동 해제 필요 |
| 락 획득 타임아웃 | 불가능 | `tryLock(timeout)` 가능 |
| 락 획득 인터럽트 | 불가능 | `lockInterruptibly()` 가능 |
| 공정성(fairness) | 미지원(비공정, 순서 보장 없음) | 생성자에서 공정 모드 선택 가능 |
| 조건 변수 | 객체당 1개 (`wait`/`notify`) | `newCondition()`으로 여러 개 생성 가능 |
| 성능 | 경량 락(lightweight lock) 최적화 적용(바이어스드 락킹은 JDK 15부터 기본 비활성화), 최신 JVM에서 경합 없을 때 매우 저렴 | AQS 기반; 성능 우열은 경합·임계 구역·JDK에 따라 측정 |

```java
// synchronized
public synchronized void increment() {
    count++;
}

// ReentrantLock
private final ReentrantLock lock = new ReentrantLock();
public void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock(); // 반드시 finally에서 해제
    }
}
```

`synchronized`는 JVM 내부적으로 락 상태를 무경합(no contention) 상황에서 경량 락(lightweight lock), 경합이 심해지면 중량 락(heavyweight lock, OS 뮤텍스)으로 승격시키는 락 팽창(lock inflation) 전략을 쓴다. HotSpot에서는 여기에 바이어스드 락킹(biased locking) 최적화도 있었으나 JDK 15부터 기본 비활성화되었고 JDK 25 HotSpot에는 이 최적화가 없다. `ReentrantLock`은 `AbstractQueuedSynchronizer`(AQS)의 상태값(state)과 CAS(Compare-And-Swap) 연산, 대기 스레드의 FIFO 큐로 구현되어 있어 대기자를 관리한다. 기본 비공정 모드는 FIFO 획득을 보장하지 않는다.

## 실무 관점
- 단순한 임계 구역 보호는 `synchronized`로 충분하며, 코드가 간결하고 락 해제를 잊을 위험이 없다. 복잡한 락 제어(타임아웃, 여러 조건 변수, 공정성)가 필요할 때만 `ReentrantLock`으로 전환한다.
- `ReentrantLock`을 쓸 때 `unlock()`을 `finally` 블록 밖이나 `lock()` 이전에 두는 실수가 흔하다. 특히 `lockInterruptibly()`를 try 안에 넣고 획득 실패 후에도 무조건 `unlock()`을 호출하면 `IllegalMonitorStateException`이 발생하는 패턴도 자주 본다.
- `tryLock(timeout)`은 데드락(deadlock) 회피 전략으로 유용하다. 여러 자원을 순서 없이 잠가야 하는 상황에서 타임아웃 후 재시도하는 방식으로 순환 대기를 끊을 수 있다.
- `ReadWriteLock`(`ReentrantReadWriteLock`)은 읽기가 압도적으로 많은 캐시성 데이터 구조에서 `synchronized`보다 처리량을 크게 높일 수 있다. 다만 쓰기 스레드가 굶주리는(writer starvation) 문제를 공정 모드로 완화해야 할 때가 있다.
- 락 경합이 심한 코드는 `synchronized` 블록의 범위를 최소화하거나, `java.util.concurrent.atomic` 패키지의 CAS 기반 클래스(`AtomicInteger`, `LongAdder`)로 대체해 락 자체를 없애는 것이 더 효과적인 경우가 많다. 특히 통계 누적에는 `LongAdder`가 경합을 줄일 수 있지만 `sum()`은 동시 갱신에 대한 원자적 스냅샷이 아니므로 정밀한 동기화 카운터를 대체하지 않는다.
- 모니터링 관점에서 `synchronized`는 스레드 덤프에서 `locked <주소>`로 소유 스레드를 바로 알 수 있지만, `ReentrantLock`은 `Locked ownable synchronizers` 섹션을 따로 봐야 한다.

### 조건 대기의 확인 순서

Java SE 25의 `Condition.await()`는 연결된 락을 놓고 기다린 뒤, 반환 전에 다시 획득한다. 깨어남은 조건 충족의 증명이 아니므로 조건 검사는 `if` 대신 `while`로 반복한다. 허위 깨어남(spurious wakeup)뿐 아니라 다른 소비자가 먼저 조건을 바꿀 수도 있다. `signal()` 호출자는 신호만 보낸다고 즉시 락을 양도하지 않는다.

시간 제한 `await`도 반환 전에 연결된 락을 다시 얻어야 한다. 50ms의 조건 대기가 끝났어도 다른 스레드가 그 락을 오래 보유하면 호출 전체는 50ms보다 늦게 반환할 수 있다. `Object.wait(timeout)`도 모니터를 재획득한 뒤 반환하므로, 조건 대기 시간을 요청 전체 종료 시각으로 해석하지 않는다. 조건이 여전히 거짓이라 재대기할 때는 원래 시간을 매번 다시 주지 않고 `awaitNanos`의 남은 시간 등으로 예산을 유지한다.

`await`는 연결된 락만, `Object.wait`는 호출한 객체의 모니터만 놓는다. 바깥에서 잡은 별도의 락까지 해제하지 않는다. 조건을 만족시킬 스레드가 그 바깥 락을 필요로 하면 서로 진행하지 못할 수 있으므로, 여러 락을 보유한 채 조건을 기다리는 경로를 점검한다.

`synchronized`의 모니터 진입 대기는 인터럽트로 취소할 수 없지만, 이미 모니터를 보유한 상태에서 호출하는 `Object.wait()`는 인터럽트에 반응한다. 두 대기를 구분한다. JDK 24의 JEP 491 이후에는 모니터 사용 때문에 가상 스레드가 carrier에 고정되는 제약이 제거되었으므로, 핀닝 회피만을 이유로 `ReentrantLock`으로 전환하지 않는다. 잠금을 보유한 채 느린 I/O를 수행해 다른 작업을 직렬화하는 문제는 남는다.

## 심화 Q&A

### Q. `synchronized`에서 예외가 발생해도 락이 항상 해제되는 이유는 무엇이며, 이것이 `ReentrantLock`과 어떤 차이를 만드는가?
`synchronized` 블록은 `monitorenter`/`monitorexit`과 예외 처리 경로로 구현되고, synchronized 메서드는 `ACC_SYNCHRONIZED` 플래그에 따라 JVM이 진입·종료를 처리한다. JVM은 블록 내부에서 예외가 발생해도 스택을 풀면서(unwind) `monitorexit`을 반드시 실행하도록 보장한다. 반면 `ReentrantLock`은 일반 메서드 호출이므로 `lock()`과 `unlock()` 사이에서 예외가 발생하면 개발자가 `finally`로 명시적으로 해제하지 않는 한 락이 영원히 반환되지 않는다. 즉 안전성은 `synchronized`가 언어 차원에서 보장하고, `ReentrantLock`은 사용자 책임이라는 차이가 있다.

### Q. 공정 모드(fair mode) `ReentrantLock`을 항상 쓰는 것이 안전하지 않은 이유는?
공정 모드는 경합 시 오래 기다린 스레드를 우선하지만 스레드 스케줄러의 공정성을 보장하지 않으며, 무인자 `tryLock()`은 공정 모드에서도 끼어들 수 있다. 대기 순서를 존중하는 인계가 경합 상황의 처리량(throughput)을 낮출 수 있지만, 매 획득마다 컨텍스트 스위칭이 강제되거나 항상 느린 것은 아니다. 짧은 임계 구역을 매우 빈번하게 잠그는 상황에서는 비공정 모드가 이미 CPU를 점유 중인 스레드에게 락을 연속으로 넘겨(barging) 전체 처리량을 높이는 경우가 많다. 따라서 공정성은 실제로 기아 문제가 관측되거나 응답 시간 형평성이 중요한 경우에만 선택적으로 적용해야 한다.

### Q. 여러 개의 `Condition`이 하나의 `ReentrantLock`에 필요한 실무 상황은 어떤 경우인가?
생산자-소비자(producer-consumer) 패턴에서 버퍼가 가득 찼을 때(`notFull`)와 비었을 때(`notEmpty`)를 구분해서 대기시켜야 하는 경우가 대표적이다. `synchronized` + `wait()`/`notifyAll()`은 객체당 조건 변수가 하나뿐이라 관련 없는 대기자까지 모두 깨워야 하지만, `Condition`을 두 개 만들면 `notFull.signal()`과 `notEmpty.signal()`을 분리해 불필요한 깨어남(spurious wakeup 외의 불필요 경쟁)을 줄이고 성능을 개선할 수 있다.

### Q. `synchronized`의 락 팽창(lock inflation) 과정에서 성능 저하가 발생하는 지점은 어디인가?
경합이 없을 때는 HotSpot의 경량 잠금 경로를 사용해 대기·스케줄링 비용을 줄일 수 있다. 경합·대기 등 조건에 따라 모니터가 팽창하고 parking과 스케줄링 비용이 늘 수 있다. 두 번째 스레드의 시도마다 즉시 팽창하는 것은 아니며 HotSpot은 사용하지 않는 모니터를 비동기로 deflate할 수 있다. CAS 횟수나 상태 전환은 JDK 구현 세부이므로 고정 비용으로 가정하지 않는다.

### Q. `ReentrantLock`과 `synchronized`를 같은 객체에 섞어 쓰면 어떤 문제가 생기는가?
`ReentrantLock`은 객체의 모니터(intrinsic lock)와 완전히 별개의 락 메커니즘이다. 한 스레드가 `synchronized`로 모니터를 획득한 상태에서 다른 스레드가 같은 자원을 `ReentrantLock`으로 보호하려 하면 두 락은 서로를 인지하지 못해 상호 배제가 깨진다. 하나의 자원에는 반드시 하나의 락 전략만 일관되게 사용해야 한다.

### Q. `tryLock()`을 이용한 데드락 회피 전략은 왜 완전한 해결책이 아닌가?
`tryLock(timeout)`으로 일정 시간 안에 두 번째 락을 못 얻으면 첫 번째 락을 풀고 재시도하는 방식은 순환 대기(circular wait)를 끊어 데드락을 회피할 수 있지만, 여러 스레드가 동시에 타임아웃 후 재시도를 반복하면 라이브락(livelock, 서로 양보만 하다 진행이 안 되는 상태)에 빠질 수 있다. 재시도 간격에 무작위 지연(jitter)을 주는 등 추가 조치가 없으면 근본적으로 해결됐다고 보기 어렵다.

## 관련 개념
- [[스레드 생명주기와 상태]]
- [[데드락과 경쟁 상태]]
- [[volatile과 가시성]]
- [[ExecutorService와 스레드 풀]]

## 참고 자료

확인 날짜: 2026-09-08. 아래 버전과 범위의 공식 문서·소스로 본문을 대조했다. 성능 비교와 운영 선택은 워크로드에 따라 달라지는 판단 기준이다.

- [JLS 25 §17](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html) — 모니터·자동 해제·wait set.
- [Java SE 25 ReentrantLock](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html) — 공정성·tryLock 예외·인터럽트·Condition.
- [JVMS 25 §2.11.10](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-2.html#jvms-2.11.10) — synchronized 메서드와 블록 바이트코드 차이.
- [JEP 491](https://openjdk.org/jeps/491) — JDK 24+ 모니터 구현과 가상 스레드.

### 2026-09-23 부분 재검증

Java SE 25 ReentrantLock·Condition의 공정성·재획득과 JEP 491(JDK 24 도입)의 모니터 핀닝 변경을 확인했다. 기존 전체 검증일은 유지한다.

- [Java SE 25 Condition](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/Condition.html) — await의 락 해제·재획득·조건 반복 검사.

### 부분 재검증: 2026-10-04

Java SE 25 Condition/Object의 시간 제한 후 재획득, 남은 대기 예산, 다른 락을 해제하지 않는 범위를 확인했다. 기존 전체 `verified`는 유지한다.

- [Java SE 25 Condition](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/Condition.html) — awaitNanos·시간 제한 await의 반환 전 재획득과 남은 시간.
- [Java SE 25 Object.wait](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Object.html#wait(long,int)) — 대상 모니터만 해제, 타임아웃 후 재획득.

실행 확인: Oracle JDK 25.0.4+7-LTS-189(macOS AArch64)에서 50ms await 중 다른 스레드가 연결된 락을 250ms 보유하도록 하여 반환까지 254ms 걸림을 관찰했다. 반환 시 락 재획득과 별도로 잡은 락의 유지도 확인했다. 관측 시간은 스케줄러 상한 보장이 아니며, Object.wait 또는 운영 환경 교착을 직접 실행한 것은 아니다.

- 도식·표 대조(2026-10-04): [Java SE 25 Object.wait](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Object.html#wait(long)) — 표의 불가능 항목을 모니터 진입 대기로 한정해 시간 제한·인터럽트 가능한 wait 조건 대기와 구분했다. 실행 시험 추가 없이 명세·소스와 기존 본문을 대조했으며 verified는 유지한다.
