---
category: "Java/동시성과 멀티스레딩"
tags: ["java", "동시성", "scopedvalue", "가상스레드", "threadlocal"]
aliases: ["범위 지정 값"]
updated: 2026-10-04
verified: 2026-09-22
---

# ScopedValue

## 핵심 정의
`ScopedValue`는 메서드 실행의 동적 범위(dynamic scope) 동안 값을 바인딩해, 중간 메서드마다 인자를 넘기지 않고 아래 호출 계층에서 읽게 하는 API다. Java 25에서 정식화되었으며 이 노트는 Java SE 27 계약을 기준으로 한다. 플랫폼 스레드와 가상 스레드 모두에서 사용할 수 있다.

값 객체 전체를 불변으로 바꾸는 기능은 아니다. 스코프 동안 **키와 값의 바인딩**을 유지하고 실행이 정상·예외로 끝나면 이전 바인딩을 복원한다. 요청 식별자·인증된 사용자·테넌트처럼 호출자가 정하고 하위 계층이 읽는 문맥(context)에 적합하다.

## 동작 원리 / 구조

```java
final class RequestContext {
    private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

    static String handle(String id) {
        return ScopedValue.where(REQUEST_ID, id).call(() -> service());
    }

    static String service() {
        return "request=" + REQUEST_ID.get();
    }
}
```

`where()`가 만든 carrier에 바인딩 후보를 담고, `run()` 또는 `call()`이 실제 실행 범위를 연다. 키는 정적 필드여도 바인딩은 스레드별이므로 두 요청이 같은 키를 다른 값에 바인딩할 수 있다. 범위 밖의 `get()`은 `NoSuchElementException`을 던진다. 값이 선택적이면 `isBound()`나 `orElse()`로 처리한다.

### Carrier 구성과 수명
`Carrier`는 명시적으로 담은 키·값 매핑이며 현재 스레드의 전체 문맥을 자동 캡처하는 객체가 아니다. `carrier.where(KEY, value)`는 새 Carrier를 반환하므로 조건부로 키를 추가할 때 반환값을 버리면 기존 매핑은 바뀌지 않는다. `carrier = carrier.where(KEY, value)`처럼 새 객체를 사용한다(Java SE 25·27).

Carrier 자체는 불변이며 스레드 안전하지만 그 안의 값까지 깊은 복사·불변으로 만드는 것은 아니다. 요청별 Carrier를 정적 필드나 오래 살아 있는 대기 작업에 보관하면 값도 그 참조를 통해 남는다. `run/call` 종료가 해제하는 것은 실행 스레드의 바인딩이며, 보관한 Carrier의 키·값 매핑은 그대로다.

### 중첩과 전파
- 중첩한 `where(KEY, otherValue).run(...)` 안에서는 새 값이 보이고 종료 후 바깥 값이 복원된다. 안쪽에서 예외가 발생해도 복원된다. 바깥 바인딩을 영구 갱신하는 `set()`은 없다.
- `Thread.start()`, `Thread.ofVirtual().start()`로 시작한 스레드는 현재 바인딩을 자동 상속하지 않는다. 실행기의 다른 작업 스레드로 제출할 때도 자동 전파를 기대하지 않는다. 호출 스레드에서 즉시 실행하는 작업이 현재 스코프의 값을 읽는 것과는 구분한다.
- `StructuredTaskScope`는 생성할 때의 바인딩을 캡처하고 그 범위에서 fork한 자식에게 상속한다. **ScopedValue는 정식 API지만 StructuredTaskScope는 Java 27에서도 일곱 번째 프리뷰**다. 두 API의 안정성·컴파일 옵션을 혼동하지 않는다.

## 실무 관점
- 키에 접근할 수 있는 코드만 값을 읽거나 중첩 바인딩을 만들 수 있다. 키는 필요한 범위에만 노출하고, 필요하면 값을 읽는 메서드만 공개한다.
- `record` 안에 가변 Map/List를 넣어 공유하면 바인딩이 고정되어도 데이터 경쟁은 생긴다. 복사한 불변 값이나 적절한 동기화로 값 자체를 보호한다.
- 명시적 인자 전달이 간단한 관계에서는 인자를 우선 사용한다. ScopedValue도 숨은 입력이므로 깊은 호출 계층에서 반복 전달이 실제 부담인 문맥에 적용한다.
- Spring의 기존 `ThreadLocal` 기반 트랜잭션·보안·MDC가 자동 전환되는 것은 아니다. 비동기 경계에서 어떤 문맥을 전파할지 각각의 통합 API와 수명을 확인한다.

## 심화 Q&A

### Q. ThreadLocal의 remove 누락이 사라지면 메모리 누수도 없어지는가?
A. 바인딩 수명은 run/call 종료에 맞춰 정리되지만, 값이나 값에서 꺼낸 참조를 정적 컬렉션·캐시·미완료 작업에 저장하면 계속 살아 있다. 끝나지 않는 스코프도 값을 오래 보유한다. 자동 바인딩 정리와 객체의 모든 참조 제거는 다르다.

### Q. 중첩 바인딩을 허용하는데 왜 단방향 데이터 전달이라고 하는가?
A. 안쪽 호출은 자기 하위 범위에 새 값을 제공할 수 있지만, 실행을 마치고 돌아오면 호출자의 값이 복구된다. 호출자에게 결과를 돌려줘야 하면 반환값이나 명시적 결과 객체를 쓴다. 공유 가변 값의 필드를 바꾸어 이 원칙을 우회하면 동기화·수명 문제도 다시 생긴다.

### Q. 작업을 비동기 실행기에 던지기만 하면 무엇이 실패하는가?
A. 실행기 스레드에 해당 바인딩이 없으므로 `get()`이 실패할 수 있다. 전달할 값을 제출 전에 읽어 작업 안에서 새 스코프로 명시적으로 바인딩할 수 있지만, 작업이 요청보다 오래 살아도 되는지와 취소 책임은 별도로 설계해야 한다. 자동 상속만을 위해 프리뷰 API를 도입하지 말고 실행 수명 요구까지 함께 판단한다.

### Q. 작업마다 수정하는 스레드 로컬 캐시도 바꾸면 좋은가?
A. ScopedValue는 호출 문맥을 아래로 읽기 전달하는 용도다. 스레드마다 재사용하며 내용을 바꾸는 버퍼·캐시와는 목적이 다르다. 특히 가상 스레드에서 큰 캐시를 스레드마다 만들면 수가 많아져 메모리 비용이 커질 수 있으므로, 캐시 범위 자체를 먼저 검토한다.

## 관련 개념
- [[ThreadLocal]]
- [[가상 스레드]]
- [[불변 객체]]
- [[인터럽트와 작업 취소]]

## 참고 자료
확인: 2026-09-22. 정식화 시점과 Java SE 27 API의 바인딩·예외·재바인딩·상속 계약을 확인했다. Oracle JDK 27+35에서 중첩 값, 예외 후 복원, 종료 후 미바인딩, 일반 가상 스레드의 비상속을 실행 검사했다.

- [Java SE 27 ScopedValue](https://docs.oracle.com/en/java/javase/27/docs/api/java.base/java/lang/ScopedValue.html) — Since 25, 동적 스코프, 키 접근, 상속·값 동기화 조건.
- [Java SE 27 ScopedValue.Carrier](https://docs.oracle.com/en/java/javase/27/docs/api/java.base/java/lang/ScopedValue.Carrier.html) — where·run·call과 바인딩 복원.
- [Java SE 27 Preview API 목록](https://docs.oracle.com/en/java/javase/27/docs/api/preview-list.html) — JEP 533 Structured Concurrency (Seventh Preview).
- [Java SE 27 StructuredTaskScope](https://docs.oracle.com/en/java/javase/27/docs/api/java.base/java/util/concurrent/StructuredTaskScope.html) — 생성 시 캡처·자식 fork·수명 제약.

부분 재검증: 2026-10-04. 아래 적용 버전·범위만 확인했으며 노트 전체의 기존 verified는 유지한다.

- [Java SE 25 ScopedValue.Carrier](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ScopedValue.Carrier.html)와 [Java SE 27 ScopedValue.Carrier](https://docs.oracle.com/en/java/javase/27/docs/api/java.base/java/lang/ScopedValue.Carrier.html) — 명시적 매핑, where의 새 Carrier 반환, 불변·스레드 안전성, run/call의 현재 스레드 바인딩 복원. Carrier를 별도로 보관하는 수명 위험은 이 계약에서 도출했다.

실행 확인: Oracle JDK 25.0.4+7-LTS-189에서 Carrier.where의 비변경·새 매핑, 범위 종료 후 보관한 매핑, 명시한 키만 포함되는 점. 실행 검사는 위 부분 재검증 범위에 한한다.
