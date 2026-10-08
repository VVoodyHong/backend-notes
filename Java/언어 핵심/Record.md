---
category: "Java/언어 핵심"
tags: ["java", "record", "불변객체", "값타입", "jep"]
updated: 2026-09-23
verified: 2026-09-08
---

# Record

## 핵심 정의
`record`는 얕은 불변성을 가진 데이터 홀더(data carrier)를 간결하게 선언하기 위한 클래스 종류로, JEP 395로 Java 16에서 정식 기능으로 도입됐다. 필드(컴포넌트, component) 목록만 선언하면 컴파일러가 생성자, `getter`(접근자, accessor), `equals()`, `hashCode()`, `toString()`을 자동으로 생성해줘 DTO(Data Transfer Object), 값 객체(value object)를 작성할 때의 보일러플레이트(boilerplate)를 크게 줄인다.

## 동작 원리 / 구조
```java
public record Point(int x, int y) { }
```

위 한 줄은 대략 다음과 동일한 클래스를 컴파일러가 생성한 것과 같다.

```java
public final class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    // equals(), hashCode(), toString()은 모든 컴포넌트 기준으로 자동 생성
}
```

- `record`는 java.lang.Record를 암묵적으로 상속하고 final이며 다른 부모 클래스를 지정할 수 없다(단 인터페이스 구현은 가능).
- 모든 컴포넌트는 `private final` 필드가 되고, 접근자 메서드 이름은 `getX()`가 아니라 컴포넌트명 그대로인 `x()`다.
- **컴팩트 생성자(compact constructor)** 로 검증/정규화 로직을 추가할 수 있다.

```java
public record Range(int min, int max) {
    public Range {
        if (min > max) throw new IllegalArgumentException("min > max");
    }
}
```

- Java 21에서 **Record Patterns**(JEP 440)와 **Pattern Matching for switch**(JEP 441)가 함께 정식화되어, `switch`나 `instanceof`에서 record를 분해(deconstruct)해 컴포넌트를 바로 꺼낼 수 있다.

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle(double r) -> Math.PI * r * r;
        case Rectangle(double w, double h) -> w * h;
    };
}
```

`sealed` 인터페이스(Java 17, JEP 409)와 record를 결합하면 컴파일러가 `switch`의 분기 누락을 검사해주는 exhaustiveness check까지 얻을 수 있다.

자동 생성된 equals는 참조형 컴포넌트의 equals를 사용한다. 따라서 배열 컴포넌트는 내용이 아닌 배열 참조로 비교된다. 배열 방어적 복사와 접근자 재정의를 도입할 때는 record 복사 불변식(`new R(r.x(), ...)`와 r의 동등성)도 점검한다.

## 실무 관점
- DTO, API 요청/응답 바디, 값 객체, 불변 설정값 등 "데이터를 담기만 하는" 타입에 record를 쓰면 코드량이 크게 줄고 기본 멤버 구현 누락을 줄인다. 가변 컴포넌트나 잘못된 사용자 재정의까지 방지하지는 않는다.
- JPA 엔티티(entity)는 record로 만들 수 없다. Jakarta Persistence 3.2 엔티티는 public/protected 무인자 생성자와 비final 클래스·영속 멤버를 요구하고 record를 제외한다. record는 조회 전용 프로젝션(projection)이나 서비스 계층 간 데이터 전달용으로 적합하다.
- 컴포넌트가 가변 객체(예: `List`, `Date`)라면 record라도 얕은 불변성만 보장된다. 완전한 불변을 원하면 입력·접근자에서 방어적 복사를 검토한다. List.copyOf는 요소까지 복제하지 않으므로 가변 요소도 별도 처리해야 한다.
- Bean Validation(`@NotNull`, `@Valid` 등)을 record 컴포넌트에 적용할 때는 애노테이션을 컴포넌트 선언부에 붙이면 컴파일러가 필드/생성자 파라미터/접근자에 적절히 전파해준다(대상 애노테이션의 `@Target` 설정에 따라 다를 수 있음).
- Jackson 등 JSON 라이브러리는 record를 기본 지원하지만, 커스텀 생성자나 이름이 다른 필드 매핑이 필요하면 `@JsonCreator`, `@JsonProperty` 설정이 추가로 필요할 수 있다.

## 심화 Q&A

### Q. record와 일반 불변 클래스(`final` 클래스 + `private final` 필드 수동 구현)의 실질적 차이는?
동작 자체는 유사하지만 record는 `equals()`/`hashCode()`/`toString()`을 컴파일러가 컴포넌트 전체 기준으로 자동 생성해 실수 여지를 없앤다. 대신 상속이 완전히 금지되고 접근자 이름 규칙(`x()`)이 고정되어 있어 커스터마이징 폭이 좁다. 세밀한 제어(일부 필드만 `equals` 대상으로 삼기, 상속 구조 필요)가 필요하면 일반 불변 클래스가 더 적합하다.

### Q. record는 왜 다른 클래스를 상속할 수 없는가?
record는 "이 타입은 정확히 선언된 컴포넌트들로 표현되는 값이다"라는 강한 보장(canonical representation)을 언어 차원에서 제공하려는 설계 의도를 가진다. 상속을 허용하면 하위 클래스가 상태를 추가하거나 `equals`/`hashCode` 의미를 훼손할 수 있어 이 보장이 깨진다. 그래서 record는 암묵적으로 `final`이며 다른 클래스 확장이 금지되고, 인터페이스 구현만 허용된다.

### Q. 컴팩트 생성자에서 필드 할당을 생략해도 되는 이유는?
컴팩트 생성자는 파라미터 목록을 다시 쓰지 않고 검증/정규화 로직만 작성하면, 컴파일러가 로직 실행 후 암묵적으로 각 파라미터를 동일한 이름의 필드에 대입하는 코드를 자동 삽입한다. 즉 개발자는 "무엇을 검증/변형할지"만 신경 쓰면 되고 대입 자체는 컴파일러가 보장한다.

### Q. Record Patterns와 sealed 타입을 함께 쓸 때 얻는 이점은 무엇인가?
`sealed` 인터페이스는 허용된 구현체 목록을 컴파일 타임에 고정하므로, `switch`에서 모든 구현체를 다루면 `default` 분기 없이도 컴파일러가 완전성(exhaustiveness)을 검증해준다. 여기에 Record Patterns를 결합하면 `case Circle(double r) ->`처럼 타입 분기와 값 분해(destructuring)를 한 번에 표현할 수 있어, 방문자 패턴(visitor pattern)이나 `instanceof` 연쇄보다 훨씬 간결하고 안전한 분기 처리가 가능하다.

### Q. record를 JPA 엔티티로 사용할 수 없는 근본적인 이유는?
Jakarta Persistence 3.2의 엔티티 타입 제약이 record를 허용하지 않는다. record는 추가 생성자를 선언할 수 있지만 반드시 canonical 생성자로 위임해야 하며, final 클래스·컴포넌트 필드는 엔티티 계약과 맞지 않는다. 같은 명세의 embeddable record 지원과 엔티티 지원은 구분해야 한다. 대신 조회 전용 DTO 프로젝션이나 QueryDSL/JPQL의 생성자 표현식 대상으로는 record가 적합하다.

### Q. record의 `equals()`가 모든 컴포넌트를 기준으로 자동 생성되는 것이 부적절한 경우는 언제인가?
캐시 타임스탬프, 계산된 파생 값처럼 "정체성 비교에는 포함하고 싶지 않은" 컴포넌트가 있는 경우 자동 생성된 `equals()`가 원하는 동등성 기준과 맞지 않을 수 있다. 이때는 `equals()`/`hashCode()`를 직접 재정의해 원하는 필드만 비교 대상으로 삼을 수 있지만, 이렇게 되면 record가 주는 "보일러플레이트 제거"라는 이점이 줄어들므로 애초에 일반 클래스로 설계하는 것이 더 명확할 수 있다.

### Q. 컴팩트 생성자에서 접근자로 읽으면 정규화한 인자가 보이는가?
A. 컴포넌트 필드는 컴팩트 생성자 본문이 정상 종료한 뒤 인자로부터 대입된다. 따라서 본문에서는 `name`처럼 인자를 검증·정규화해야 하며, `name()`으로 아직 할당되지 않은 필드를 읽는 코드는 기본값을 볼 수 있다. `items = List.copyOf(items)`처럼 인자를 교체하면 그 결과가 필드에 저장된다. 생성자 안에서 this를 콜백에 넘기는 것도 피한다.

### Q. 자동 equals는 컴포넌트 접근자의 반환값을 비교하는가?
A. 아니다. JLS 25는 접근자 호출 대신 컴포넌트 필드를 직접 기준으로 삼는다. 접근자가 값을 가공하면 복사 불변식과 어긋날 수 있다. 기본형 비교도 단순한 `==`로 일반화하지 않는다. double 컴포넌트는 대응 래퍼의 compare 의미를 사용하므로 NaN끼리는 같고 +0.0과 -0.0은 구분된다. 직접 equals를 재정의할 때 해시와 복사 불변식까지 함께 검토한다.

## 관련 개념
- [[불변 객체]]
- [[equals와 hashCode]]
- [[제네릭]]

## 참고 자료

확인 날짜: 2026-09-08. 아래 버전과 범위의 공식 문서·소스로 본문을 대조했다. 성능 비교와 운영 선택은 워크로드에 따라 달라지는 판단 기준이다.

- [JLS 25 §8.10](https://docs.oracle.com/javase/specs/jls/se25/html/jls-8.html#jls-8.10) — record 멤버·생성자·애노테이션 전파.
- [Java SE 25 Record](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Record.html) — equals와 복사 불변식.
- [JEP 395](https://openjdk.org/jeps/395) — Java 16 정식화.
- [JEP 440](https://openjdk.org/jeps/440) — Java 21 record pattern.
- [Jakarta Persistence 3.2](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2.html) — 엔티티 제외 및 embeddable record 지원.

### 2026-09-23 부분 재검증

JLS 25 §8.10.3·8.10.4.2의 필드 기반 동등성·기본형 compare·컴팩트 생성자 후 자동 대입을 확인했다. 기존 전체 검증일은 유지한다.

실행 확인: 2026-09-23, OpenJDK 25.0.2+10-69에서 컴팩트 생성자에서 접근자의 기본값·인자 정규화·NaN/부호 있는 0·배열 동일성을 재현했다. 구현 관측은 해당 버전 범위로 한정한다.
