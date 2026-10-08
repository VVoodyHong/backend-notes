---
category: Spring/웹 심화
tags: [spring, spring-boot, http-client]
aliases: [HTTP Interface, HttpExchange, HTTP Service Client]
updated: 2026-09-23
verified: 2026-09-23
---

# HTTP Service Client와 선언형 호출

## 핵심 정의

HTTP Service Client는 Java 인터페이스에 HTTP 경로·메서드·요청 인자를 선언하고, 프록시가 실제 HTTP 클라이언트를 호출하도록 하는 Spring 기능이다. `@HttpExchange`와 `@GetExchange` 등으로 계약을 표현하며, 전송은 `RestClient`·`WebClient` 같은 하위 클라이언트가 담당한다.

인터페이스 호출 문법이 로컬 메서드와 같아도 네트워크 지연·실패·재시도·직렬화 경계는 남는다. 아래 예시는 Spring Framework 7.0.9·Spring Boot 4.1.1의 동기 `RestClient` 구성을 기준으로 한다.

## 동작 원리 / 구조

### 인터페이스와 클라이언트 프록시

```java
@HttpExchange("/items")
public interface InventoryApi {
    @GetExchange("/{id}")
    ResponseEntity<String> find(@PathVariable("id") long id);
}
```

타입의 공통 경로와 메서드 경로를 합치고 인자로 URI 변수를 확장한다. 본문은 하위 클라이언트의 메시지 컨버터가 변환한다. Java 타입이 일치한다는 사실만으로 원격 서버의 JSON 필드·미디어 타입·상태 코드 계약이 검증되지는 않는다.

직접 프록시를 만들 때는 실제 클라이언트를 어댑터(Adapter)로 연결한다. Boot에서는 자동 구성된 `RestClient.Builder`를 주입받아 관측·커스터마이징 설정을 이어받는다.

```java
// builder는 Boot가 구성해 주입한 RestClient.Builder다.
RestClient client = builder.baseUrl("https://inventory.example.com").build();
InventoryApi inventory = HttpServiceProxyFactory
    .builderFor(RestClientAdapter.create(client))
    .build()
    .createClient(InventoryApi.class);
```

`RestClientAdapter`는 동기 반환을 지원한다. `WebClientAdapter`는 `Mono`·`Flux` 등 리액티브 반환도 지원하지만 동기 반환 타입을 사용하면 기다리는 경계가 생긴다. 인터페이스로 감쌌다는 이유로 블로킹 호출이 논블로킹으로 바뀌지는 않는다.

### Boot 4.1.1의 그룹 등록

여러 인터페이스의 공통 설정은 HTTP 서비스 그룹(Group)으로 묶을 수 있다. 동기 클라이언트에는 `spring-boot-starter-restclient`를 포함하고 다음처럼 등록한다.

```java
@Configuration(proxyBeanMethods = false)
@ImportHttpServices(group = "inventory", types = InventoryApi.class)
class InventoryClients {
}
```

```yaml
spring:
  http:
    serviceclient:
      inventory:
        base-url: https://inventory.example.com
        connect-timeout: 1s
        read-timeout: 2s
```

이 방식은 프록시를 빈으로 등록하므로 같은 인터페이스를 수동 `@Bean`으로 중복 등록하지 않는다. 그룹을 지정하지 않으면 이름은 `default`다. 같은 호스트라도 인증·제한이 다르면 그룹을 나눌 수 있으며, 연결 풀의 실제 공유·격리는 하위 요청 팩토리 설정을 확인한다. 위 시간은 예시이며 전체 호출 데드라인을 의미하지 않는다.

## 실무 관점

- **오류 계약**: 기본 하위 클라이언트는 4xx·5xx를 예외로 처리한다. `ResponseEntity<T>` 반환 타입으로 선언해도 404가 자동으로 정상 반환되는 것은 아니다. `RestClient.Builder.defaultStatusHandler` 등으로 오류 응답을 도메인 예외에 매핑하고 실제 프록시 호출로 확인한다.
- **클라이언트별 경계**: 인증 헤더·TLS·연결/읽기 제한·관측은 하위 클라이언트와 그룹 구성의 책임이다. 인터페이스마다 호출 계약을 복사하기보다 같은 원격 시스템의 공통 정책을 묶는다.
- **동적 주소**: `URI` 인자는 애노테이션의 URL을 재정의할 수 있다. 사용자 입력을 그대로 전달하는 범용 프록시를 만들지 말고 허용 대상과 주소 생성 책임을 제한한다.
- **재시도**: 상태 코드·예외뿐 아니라 요청 본문 재전송 가능성·멱등성·남은 시간 예산을 확인한다. 관련 원리는 [[HTTP 호출 타임아웃과 재시도 예산]]에 둔다.
- **계약 테스트**: 실제 HTTP 경로·쿼리·헤더·미디어 타입과 정상/오류 본문을 검사한다. 인터페이스를 Mockito로 대체한 서비스 테스트만으로 직렬화·상태 매핑을 검증할 수 없다.

## 심화 Q&A

### Q. 같은 인터페이스를 서버 컨트롤러와 클라이언트가 공유하면 호환성이 자동 보장되는가?
A. `@Controller`가 HTTP 인터페이스를 구현하는 구성은 가능하다. 다만 서버·클라이언트의 배포 시점과 직렬화 설정이 다를 수 있어 공통 Java 타입만으로 실행 중 계약까지 보장되지는 않는다. API 변경의 하위 호환성과 실제 HTTP 응답을 별도로 검증한다.

### Q. 반환 타입을 `ResponseEntity<String>`으로 바꿨는데 왜 404 예외가 계속 발생하는가?
A. 응답을 반환 타입으로 만드는 과정에도 하위 클라이언트의 기본 상태 처리가 적용되기 때문이다. 404를 정상 분기로 취급할지는 업무 계약에 따라 결정하고 해당 상태만 명시적으로 처리한다. 모든 오류를 억제해 인증 실패나 서버 장애까지 정상 데이터처럼 읽지 않는다.

### Q. `WebClientAdapter`로 바꾸기만 하면 기존 동기 인터페이스도 이벤트 루프에서 안전한가?
A. 아니다. 반환 타입도 확인해야 한다. 동기 결과를 반환하는 메서드는 내부에서 결과를 기다릴 수 있다. 리액티브 호출 경계에서는 `Mono`·`Flux`를 반환해 상위 흐름으로 연결하고 타임아웃·취소·컨텍스트 전파를 함께 검증한다.

### Q. 서비스 그룹 하나가 곧 장애 격리 단위인가?
A. 그룹은 클라이언트 구성과 프록시 생성 단위다. 필요한 장애 격리는 연결 풀·동시 호출 제한·대기열·타임아웃 같은 실제 자원 정책으로 확인한다. 이름만 나눠 같은 하위 자원에 무제한 요청을 보내면 격리 효과를 얻을 수 없다.

## 관련 개념

- [[WebClient와 RestTemplate]]
- [[HTTP 호출 타임아웃과 재시도 예산]]
- [[Micrometer와 분산 트레이싱 연동]]
- [[멱등성 키 설계]]
- [[Bulkhead 패턴]]

## 참고 자료

검증일: 2026-09-23. Framework 7.0.9·Boot 4.1.1 공식 문서와 API를 대조했다. Java 25.0.2의 로컬 HTTP 서버로 경로/본문/헤더 매핑, 기본 404 예외, 사용자 정의 상태 처리, Boot 그룹 등록과 base-url 바인딩 4건을 실행했다. WebClient의 리액티브 경로·실제 TLS/인증·시간 제한 발동·운영 서비스 호출은 실행하지 않았다.

- [HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-service-client) — Framework 7.0.9, 어댑터·반환 타입·오류 처리·그룹.
- [Boot Calling REST Services](https://docs.spring.io/spring-boot/reference/io/rest-client.html) — Boot 4.1.1, `@ImportHttpServices`와 `spring.http.serviceclient` 설정.
- [HttpExchange 7.0.9 API](https://docs.spring.io/spring-framework/docs/7.0.9/javadoc-api/org/springframework/web/service/annotation/HttpExchange.html) — 타입/메서드 선언과 동적 URL 인자.
- [RestClientAdapter 7.0.9 API](https://docs.spring.io/spring-framework/docs/7.0.9/javadoc-api/org/springframework/web/client/support/RestClientAdapter.html) — 동기 요청·응답 어댑터.
- [ImportHttpServices 7.0.9 API](https://docs.spring.io/spring-framework/docs/7.0.9/javadoc-api/org/springframework/web/service/registry/ImportHttpServices.html) — 타입·패키지 선택과 그룹 등록.
