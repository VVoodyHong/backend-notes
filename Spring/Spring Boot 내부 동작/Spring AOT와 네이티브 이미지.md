---
category: Spring/Spring Boot 내부 동작
tags: [spring, spring-boot, aot, graalvm]
aliases: [Spring AOT, GraalVM Native Image]
updated: 2026-09-23
verified: 2026-09-23
---

# Spring AOT와 네이티브 이미지

## 핵심 정의

Spring AOT(Ahead-of-Time) 처리는 런타임에 수행하던 빈 구성 분석 일부를 빌드 시점에 수행하고 초기화 코드·런타임 힌트(Runtime Hints)를 만드는 과정이다. GraalVM 네이티브 이미지(Native Image)는 도달 가능한 코드와 필요한 메타데이터를 분석해 특정 OS·아키텍처용 실행 파일을 만드는 별도 컴파일 단계다.

Spring AOT 결과는 JVM에서도 실행할 수 있다. 따라서 “AOT 처리에 성공했다”, “AOT 모드 JVM에서 동작했다”, “네이티브 실행 파일을 검증했다”는 서로 다른 검증 결과다. 아래는 Spring Boot 4.1.1·Spring Framework 7.0.9의 적용 범위다.

## 동작 원리 / 구조

### 두 단계 빌드

1. **Spring 분석**: 빌드 시점 클래스패스·프로파일·조건부 설정으로 사용할 빈 정의를 결정한다.
2. **Spring 산출물 생성**: 컨텍스트 초기화 Java 코드, 필요한 프록시 바이트코드, 리플렉션·리소스 등의 힌트를 만든다.
3. **네이티브 컴파일**: GraalVM이 애플리케이션과 라이브러리의 도달 가능한 코드 및 힌트를 분석해 실행 파일을 만든다.
4. **실행 검증**: 대상 플랫폼에서 실제 기동·요청·직렬화·인증·종료 경로를 확인한다.

GraalVM 네이티브 이미지의 폐쇄 세계 가정(Closed-world Assumption) 때문에 빌드 후 임의의 클래스를 추가해 로딩하는 플러그인 구조는 일반 JVM과 같은 방식으로 기대할 수 없다. 동적으로 이름을 조합해 접근하는 클래스·리소스는 정적 분석만으로 필요성을 알아내기 어렵다.

### 빌드 시 고정되는 것과 실행 시 읽는 것

| 설정 | AOT 적용 시 의미 |
|---|---|
| `@Profile`, 빈 존재 여부에 영향을 주는 `@ConditionalOnProperty` | 빌드 시점 빈 선택에 사용 |
| 클래스패스와 빈 정의 집합 | AOT 처리 시 고정된 구성을 전제로 함 |
| 이미 존재하는 빈이 읽는 URL·자격 증명 등의 값 | 그 값이 빈 선택에 관여하지 않는다면 런타임 외부 설정으로 제공 가능 |

예를 들어 `app.feature-enabled=true`로 기능 빈을 생성한 AOT 산출물에 런타임 `false`를 전달해도 해당 빈이 사라지지 않는다. 런타임에 바꿔야 하는 기능 플래그는 빈 생성 조건과 실제 요청 처리 분기를 구분해 설계한다. 비밀 값을 빌드 인수나 이미지에 포함할 이유는 없다.

### 런타임 힌트

Spring이 컨트롤러·설정 속성 등에서 추론하는 힌트 외에 애플리케이션 고유의 동적 접근이 있으면 `RuntimeHintsRegistrar`와 `@ImportRuntimeHints`로 보완할 수 있다. 예를 들어 이름을 조합해 읽는 템플릿 리소스가 실제로 패키징되는지 확인한다.

```java
final class ReceiptHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader loader) {
        hints.resources().registerPattern("receipts/*.txt");
    }
}
```

이 등록기를 필요한 구성 클래스에서 `@ImportRuntimeHints(ReceiptHints.class)`로 연결한다. 실제 리소스 파일도 빌드 입력에 있어야 한다. `RuntimeHintsPredicates`로 등록 결과를 검사할 수 있지만, 그 테스트만으로 모든 실행 경로의 힌트가 충분하다고 증명되지는 않는다.

### JVM에서 먼저 확인

Maven의 `spring-boot:process-aot`는 AOT 결과를 생성한다. 빌드에 해당 goal을 연결해 결과를 포함한 실행 JAR를 만든 뒤 다음과 같이 확인한다.

```shell
java -Dspring.aot.enabled=true -jar target/app.jar
```

`spring.aot.enabled`는 JVM 시스템 속성이다. AOT 산출물 없이 이 옵션만 붙여도 필요한 코드가 생성되는 것은 아니다. Maven에서는 생성 코드를 `target/spring-aot/main/sources`, 힌트 리소스를 `target/spring-aot/main/resources`에서 확인할 수 있다.

## 실무 관점

- 기동 시간·메모리·처리량·빌드 시간을 함께 측정한다. 네이티브 이미지가 모든 워크로드에서 더 높은 처리량을 낸다고 가정하지 않는다. 짧게 실행되는 작업과 장시간 고부하 서버의 판단 기준이 다르다.
- JSON 직렬화, 리플렉션 기반 라이브러리, 리소스 로딩, 동적 프록시, JNI 등 동적 접근 경로를 확인한다. “서버가 떴다”만으로 충분하지 않다.
- 실행 JAR를 각 플랫폼에서 네이티브로 변환할 수 있지만, 동일 네이티브 바이너리를 모든 OS·아키텍처에서 실행하는 것은 아니다. 대상 빌드·테스트 행렬을 명시한다.
- AOT 때 선택한 프로파일과 조건을 빌드 기록에 남긴다. 일반 JVM 테스트만 통과시키면 AOT 전용 구성 오류를 놓칠 수 있다.
- 추적 에이전트(Tracing Agent)가 모은 힌트는 실제로 실행한 경로의 관찰 결과다. 드문 예외·다른 DTO·언어별 리소스 경로가 빠질 수 있다.

## 심화 Q&A

### Q. AOT JVM 실행이 성공하면 네이티브 검증을 생략할 수 있는가?
A. 아니다. 생성된 Spring 초기화 코드의 문제를 조기에 찾는 데 유용하지만, JVM에는 가능한 동적 접근이 네이티브에서는 힌트 부족으로 실패할 수 있다. 대상 네이티브 실행 파일로 주요 경로를 확인한다.

### Q. 환경마다 다른 DB URL 때문에 매번 다시 빌드해야 하는가?
A. 빈 구성 자체가 같고 URL을 런타임에 읽는다면 보통 그럴 필요가 없다. 반면 프로파일·속성으로 JDBC/JPA 등 빈 집합 자체를 전환하려 하면 AOT 빌드 시 선택과 충돌한다. 연결 값과 빈 선택 조건을 구분한다.

### Q. 모든 타입에 넓게 리플렉션 힌트를 주면 해결되는가?
A. 누락 일부를 가릴 수 있지만 필요한 접근 범위를 설명하기 어렵고 포함 코드·메타데이터가 늘 수 있다. 사용 라이브러리의 지원 상태를 먼저 확인하고 실제 경로에 필요한 힌트를 등록·검증한다.

### Q. Java의 JVM AOT 캐시와 같은 기능인가?
A. 아니다. Spring AOT는 프레임워크 빈 구성과 초기화 코드를 빌드 시 생성하는 기능이다. JVM의 클래스 로딩·링킹 관련 AOT 캐시는 별도 JVM 기능이다. 이름이 같아도 활성화 방법과 제한, 생성 산출물을 혼용하지 않는다.

## 관련 개념

- [[자동 구성 원리]]
- [[Conditional과 조건부 빈]]
- [[외부 설정과 ConfigurationProperties]]
- [[ApplicationContext]]
- [[애노테이션과 리플렉션]]
- [[JIT 컴파일러]]

## 참고 자료

검증일: 2026-09-23. Boot 4.1.1·Framework 7.0.9 공식 문서의 AOT 제한·힌트·빌드 흐름을 확인했다. OpenJDK 25.0.2와 Maven 3.9.11에서 `process-aot`·패키징 및 일반/AOT JVM 실행을 비교했다. 빌드 때 활성화한 빈이 AOT JVM에서 유지되고 일반 URL 값은 런타임 값으로 바뀌는 것을 확인했다. `RuntimeHintsPredicates`로 예시 리소스 패턴의 포함·제외 경로도 검사했다. GraalVM 네이티브 컴파일·성능 비교는 실행하지 않았다.

- [Introducing GraalVM Native Images](https://docs.spring.io/spring-boot/reference/packaging/native-image/introducing-graalvm-native-images.html) — Boot 4.1.1, 폐쇄 세계 가정과 Spring 생성 산출물.
- [Ahead of Time Optimizations](https://docs.spring.io/spring-framework/reference/core/aot.html) — Framework 7.0.9, 빈 구성 고정·힌트 API·AOT JVM 실행.
- [Maven AOT plugin](https://docs.spring.io/spring-boot/maven-plugin/aot.html) — Boot 4.1.1, process-aot와 빌드 시 환경 인수.
- [Advanced Native Images Topics](https://docs.spring.io/spring-boot/reference/packaging/native-image/advanced-topics.html) — Boot 4.1.1, 실행 JAR의 플랫폼별 변환과 추적 기반 검증.
