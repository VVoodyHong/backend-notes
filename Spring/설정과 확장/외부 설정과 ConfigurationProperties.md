---
category: "Spring/설정과 확장"
tags: ["spring", "spring-boot", "configuration", "validation"]
aliases: ["ConfigurationProperties", "외부화 설정", "Externalized Configuration"]
updated: 2026-09-22
verified: 2026-09-22
---

# 외부 설정과 ConfigurationProperties

## 핵심 정의

외부화 설정(Externalized Configuration)은 같은 애플리케이션 아티팩트에 환경별 값을 주입하는 방식이다. Spring의 `Environment`가 여러 프로퍼티 소스(Property Source)를 우선순위로 조회하고, Spring Boot의 `@ConfigurationProperties`가 관련 값을 타입 있는 객체로 바인딩한다.

`@Value`는 개별 값·표현식 주입에, `@ConfigurationProperties`는 설정 묶음의 변환·검증에 적합하다. 설정 객체를 만들었다고 HTTP 클라이언트·연결 풀에 자동 적용되지는 않는다. 사용하는 컴포넌트를 실제로 그 값으로 구성해야 한다.

## 동작 원리 / 구조

### 설정값의 출처와 우선순위

Boot 4.1.1에서 흔히 쓰는 운영 입력을 낮은 순서부터 요약하면 다음과 같다. JNDI·Servlet 초기화 값·테스트·Devtools 등 전체 목록은 공식 문서를 따른다.

| 낮음 → 높음 | 입력 |
|---|---|
| 1 | `SpringApplication.setDefaultProperties(...)`로 제공한 기본값 |
| 2 | Config Data: application.properties/yaml 및 import |
| 3 | 운영체제 환경 변수 |
| 4 | Java 시스템 프로퍼티(`-D...`) |
| 5 | `SPRING_APPLICATION_JSON` |
| 6 | 명령행 인자(`--...`, 명령행 바인딩 활성화 시) |

같은 위치의 `.properties`와 YAML이 함께 있으면 `.properties`가 우선한다. 기본 검색에서는 외부 설정 파일이 JAR 내부보다, 같은 계층에서는 프로파일별 파일이 일반 파일보다 우선한다. import·location 그룹을 사용하면 해당 규칙도 함께 적용된다.

`spring.config.location`은 기본 검색 위치를 **교체**하고, `spring.config.additional-location`은 위치를 **추가**한다. 둘은 설정 파일을 찾기 전에 읽으므로 환경 변수·시스템 프로퍼티·명령행처럼 초기 입력으로 전달한다. `optional:`은 파일 부재를 허용하는 선택이며 필수 운영 설정에 무조건 붙이지 않는다.

### 불변 설정 객체와 기동 시 검증

아래 예제는 Boot 4.1.1과 Jakarta Validation 구현체가 포함된 `spring-boot-starter-validation`을 전제로 한다. import는 `jakarta.validation.constraints.*`, `org.springframework.boot.context.properties.*`, `org.springframework.boot.context.properties.bind.DefaultValue`, `org.springframework.validation.annotation.Validated`를 사용한다.

```java
@Validated
@ConfigurationProperties("app.client")
public record ClientProperties(
        @NotNull URI baseUrl,
        @DefaultValue("2s") Duration connectTimeout,
        @DefaultValue("32") @Min(1) int maxInFlight) {
}

@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(ClientProperties.class)
class ClientConfiguration {
}
```

```yaml
app:
  client:
    base-url: https://api.example.com
    connect-timeout: 2s
    max-in-flight: 32
```

단일 생성자나 일반 record는 생성자 바인딩(Constructor Binding)을 사용할 수 있다. 등록은 `@EnableConfigurationProperties` 또는 `@ConfigurationPropertiesScan`으로 한다. 같은 타입을 `@Component` 등 일반 빈 생성 경로로도 중복 등록하지 않는다. 일반 클래스의 생성자 매개변수 이름에는 `-parameters` 컴파일 설정을 확인한다.

`@DefaultValue`는 설정 객체를 바인딩할 때 사용하는 기본값이며 `Environment`에 새 프로퍼티를 등록하지 않는다. 따라서 다른 빈의 `@Value`, placeholder, 조건 평가에서도 같은 기본값이 보인다고 가정하지 않는다.

## 실무 관점

- **완화된 바인딩(Relaxed Binding)**: YAML은 `connect-timeout`처럼 kebab-case로 작성한다. 환경 변수는 점을 밑줄로, 하이픈은 제거하고 대문자로 바꿔 `APP_CLIENT_CONNECTTIMEOUT`으로 표현한다. 임의로 `CONNECT_TIMEOUT`을 넣어 같은 키라고 가정하지 않는다.
- **단위**: `Duration`의 단위 없는 숫자는 기본 밀리초다. `2s`, `500ms`처럼 명시하고 기존 숫자 타입에서 변경할 때 의미가 달라지지 않는지 검사한다.
- **컬렉션 병합**: 높은 우선순위의 리스트는 낮은 리스트 전체를 교체한다. Map은 여러 소스의 항목을 합치며 같은 세부 프로퍼티는 우선순위로 결정한다. 값이 객체인 Map에서는 같은 키 아래의 서로 다른 필드도 각각 병합될 수 있다. 프로파일에 항목 하나를 적으면 기존 리스트에 추가될 것이라는 기대가 흔한 설정 오류다.
- **검증 범위**: 필수값과 숫자 범위뿐 아니라 양수 timeout·허용 URI scheme 등 업무상 제약도 필요하다. 중첩 객체 검증에는 `@Valid`, 중첩 객체 자체의 필수 여부에는 `@NotNull`을 별도로 적용한다. 선언한 제약을 바인딩 시 실제로 실행하는지 실패 기동 테스트로 확인한다.
- **진단과 비밀**: 값이 틀리면 최종값과 출처를 함께 확인한다. Actuator `env`·`configprops`는 인증·노출 정책 안에서 사용하고 비밀값을 로그에 출력하지 않는다. 프로파일은 비밀 저장소가 아니다.
- **갱신 경계**: 디스크 파일이나 환경 변수를 바꿨다고 기존 설정 객체와 클라이언트가 자동 재구성되지는 않는다. 실행 중 갱신은 [[Spring Cloud Config와 서비스 디스커버리]]의 지원 범위와 이미 실행 중인 작업의 설정 일관성을 따로 설계한다.

## 심화 Q&A

### Q. 운영 YAML의 값과 실제 애플리케이션 값이 다른 이유는?
환경 변수·시스템 프로퍼티·명령행·외부 프로파일 파일 등이 덮어썼을 수 있다. 실제 시작 인자와 활성 프로파일, Config Data 위치·import, 사용자 PropertySource를 확인한다. 설정 파일 한 개를 읽어 최종값이라고 판단하지 않는다.

### Q. `@PropertySource`로 로깅이나 spring.main 설정을 추가하면 왜 늦을 수 있는가?
해당 프로퍼티 소스는 컨텍스트 refresh 중 추가되지만 일부 로깅·SpringApplication 설정은 그 전에 읽는다. 기동 초기부터 필요한 값은 Config Data나 초기 환경 입력으로 전달한다.

### Q. 생성자 바인딩 record에 서비스 빈을 같이 주입하면 되는가?
설정 바인딩 생성자는 프로퍼티 값을 받는 경계다. 다른 빈이 필요한 구성 로직은 설정 객체를 주입받는 별도 빈으로 옮긴다. 설정 객체가 인프라 초기화까지 수행하면 바인딩·검증·빈 생성 순서가 섞여 실패 원인을 찾기 어려워진다.

### Q. 설정 키 오타도 검증 애노테이션이 모두 잡아주는가?
아니다. 기본 `ignoreUnknownFields=true`는 모르는 키를 허용하며, 필수 제약이 없거나 기본값이 있으면 의도와 다른 값으로 기동할 수 있다. 필요하면 해당 prefix에 `ignoreUnknownFields=false`를 사용하되 같은 prefix를 여러 타입이 나눠 쓰는 구성도 검토한다. 필수값 누락·오타·잘못된 단위·프로파일 덮어쓰기를 각각 시험한다.

## 관련 개념

- [[ApplicationContext]]
- [[Conditional과 조건부 빈]]
- [[자동 구성 원리]]
- [[Bean Validation]]
- [[시크릿 관리]]

## 참고 자료

검증일: 2026-09-22. 적용 범위: Spring Boot 4.1.1 외부 설정·바인딩. Boot Binder 실행으로 소스 우선순위, 리스트 교체, Map의 객체 필드 병합, Duration 단위 변환을 확인했다. 실제 컨텍스트 기동으로 record 기본값과 Environment의 분리, 필수값 누락·최솟값 위반의 기동 실패도 확인했다. 실제 배포의 환경 변수·원격 Config Server까지 실행한 검증은 아니다.

- [Boot Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) — 입력 우선순위, 파일 검색, 생성자 바인딩, 컬렉션·단위·검증.
- [ConfigurationProperties API](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/context/properties/ConfigurationProperties.html) — prefix·ignoreUnknownFields와 바인딩 계약.
- [Binder API](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/context/properties/bind/Binder.html) — 여러 ConfigurationPropertySource에서 타입으로 바인딩.
- [Boot Validation](https://docs.spring.io/spring-boot/reference/io/validation.html) — Jakarta Validation 구현체와 검증 통합.
