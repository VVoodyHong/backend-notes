---
category: Spring/테스트 전략
tags: [spring, testing, transaction]
aliases: [TestTransaction, 테스트 관리 트랜잭션]
updated: 2026-09-23
verified: 2026-09-23
---

# Spring 테스트 트랜잭션과 커밋 검증

## 핵심 정의

테스트 관리 트랜잭션(Test-managed Transaction)은 Spring TestContext가 테스트 메서드 전후에 시작하고 종료하는 트랜잭션이다. 테스트의 `@Transactional`은 기본적으로 종료 시 롤백하여 데이터를 정리한다. 서비스 프록시가 시작하는 [[Transactional 동작 원리|애플리케이션 트랜잭션]]과 관리 주체가 다르다.

자동 롤백으로 통과한 테스트가 실제 커밋, 별도 스레드, 외부 시스템까지 검증한 것은 아니다. 저장 SQL 검증, 커밋 검증, 격리·경합 검증을 나눠 설계한다. 아래 API 설명은 Spring Framework 7.0.9, 웹 테스트 경계는 Spring Boot 4.1.1 기준이다.

## 동작 원리 / 구조

### 테스트와 서비스가 공유하는 경계

`TransactionalTestExecutionListener`가 `PlatformTransactionManager`로 테스트 트랜잭션을 연다. 같은 스레드의 `REQUIRED` 서비스 호출은 보통 이 트랜잭션에 참여하므로 서비스 메서드가 반환해도 물리적 커밋은 아직 없다.

| 실행 위치 | 기본 테스트 트랜잭션과의 관계 |
|---|---|
| 테스트 메서드, `@BeforeEach`, `@AfterEach` | 테스트 트랜잭션 안에서 실행 |
| `@BeforeTransaction`, `@AfterTransaction` | 테스트 트랜잭션 시작 전·종료 후 실행 |
| `@BeforeAll`, `@AfterAll` | 테스트 메서드의 트랜잭션 밖 |
| 별도 executor·비동기 작업 | 스레드에 묶인 테스트 트랜잭션을 상속하지 않음 |
| `REQUIRES_NEW` 서비스 | 외부 테스트 트랜잭션과 독립된 커밋·롤백 |

테스트에 `NOT_SUPPORTED`나 `NEVER`를 지정하면 테스트 트랜잭션을 만들지 않는다. 이때 서비스 프록시의 `@Transactional`까지 해제되는 것은 아니다. `@BeforeTransaction`·`@AfterTransaction`도 트랜잭션 테스트에만 호출된다.

### flush, clear, commit은 검증 대상이 다르다

- **flush**: 영속성 컨텍스트의 변경을 SQL로 반영해 SQL 실행 시점 오류를 드러낸다. 커밋은 아니다.
- **clear 후 재조회**: 1차 캐시의 동일 객체를 다시 보는 문제를 줄인다. 2차 캐시를 사용하는 경우 그 영향도 별도로 통제한다.
- **commit 후 별도 경계에서 조회**: 실제 커밋 성공과 이후 가시성을 검증한다. 커밋 시점 검사·동기화 콜백은 롤백만 하는 테스트로 대체할 수 없다.

`@Commit` 또는 `@Rollback(false)`로 테스트 종료 시 커밋을 선택할 수 있다. 테스트 중간에 경계를 제어하려면 `TestTransaction`을 사용한다.

```java
@Test
@Transactional
void committedOrderCanBeRead() {
    Long id = orderService.createOrder();
    entityManager.flush();
    TestTransaction.flagForCommit();
    TestTransaction.end(); // 실제 커밋을 여기서 시도

    TestTransaction.start(); // 새 테스트 트랜잭션
    entityManager.clear();
    assertNotNull(entityManager.find(Order.class, id));
    // 커밋한 fixture는 별도 정리 정책으로 삭제한다.
}
```

위 코드는 JPA 서비스·엔티티가 있는 테스트의 경계 예시다. `flagForCommit()`만 호출해서는 즉시 커밋되지 않는다. `start()`는 이전 트랜잭션 종료 후 호출하며 롤백 여부를 테스트의 기본 정책으로 다시 초기화한다.

### 문서의 지원 계약과 관찰된 구현을 구분

7.0.9 공식 지원표는 테스트의 `@Transactional`에서 `isolation`·`timeout`·`readOnly`를 지원하지 않는 속성으로 명시한다. 이를 **항상 무시된다**고 바꿔 읽으면 안 된다. 2026-09-23에 7.0.9의 `DataSourceTransactionManager`로 실행한 테스트에서는 격리 수준·readOnly 플래그·타임아웃이 전달됐다. 소스도 속성을 위임해 트랜잭션 매니저에 넘긴다.

이는 해당 조합의 구현 관찰이며 지원 계약을 확장하지 않는다. 격리 수준·타임아웃 자체를 검증할 때는 테스트 바깥 트랜잭션을 제거한 뒤 서비스 경계나 명시적인 `TransactionTemplate`으로 조건을 통제한다. 테스트의 `rollbackFor`를 서비스 예외 규칙 검증 수단으로 쓰지 않는다. 테스트 종료 정책은 `@Rollback`·`@Commit`·`TestTransaction`으로 설정한다.

## 실무 관점

- `@SpringBootTest(webEnvironment = RANDOM_PORT)`로 HTTP 요청을 보내면 서버 처리 스레드는 테스트 스레드와 다르다. 테스트의 롤백으로 서버가 커밋한 데이터를 지울 수 없으므로 고유 fixture와 명시적인 정리가 필요하다.
- 선점형 타임아웃(Preemptive Timeout)은 테스트 본문을 다른 스레드에서 실행할 수 있다. `assertTimeoutPreemptively` 안의 DB 작업이 테스트 트랜잭션 밖에서 커밋되는지 확인한다.
- 롤백만 하는 테스트에서는 해당 트랜잭션의 `AFTER_COMMIT` 이벤트와 커밋된 아웃박스 데이터의 후속 처리를 확인할 수 없다. `REQUIRES_NEW`의 독립 커밋은 남을 수 있으므로 별도 조회·정리로 검증한다. 커밋 이후 작업에는 결과와 실패 복구를 확인하는 테스트를 둔다.
- 정리도 실패할 수 있다. 커밋 테스트에는 전용 DB·스키마나 고유 키, 실패해도 동작하는 정리를 둔다. `@DirtiesContext`는 DB 레코드 삭제 수단이 아니다.
- H2로 확인한 테스트 트랜잭션 경계를 PostgreSQL/MySQL의 잠금·격리·지연 제약 동작까지 일반화하지 않는다. 이 부분은 실제 대상 DB로 [[TestContainers]] 테스트를 구성한다.

## 심화 Q&A

### Q. 저장 직후 조회가 성공했는데 운영에서는 커밋이 실패하는 이유는?
A. 조회가 1차 캐시를 읽었거나 아직 SQL을 보내지 않았을 수 있다. flush로 SQL 실행을 확인해도 커밋 단계의 실패 가능성은 남는다. 검증하려는 실패 시점에 맞춰 flush·캐시 통제·실제 커밋을 선택한다.

### Q. `REQUIRES_NEW`로 기록한 감사 로그가 테스트 종료 후 남는 이유는?
A. 별도 물리적 트랜잭션이 이미 커밋했기 때문이다. 외부 테스트 롤백은 이를 되돌리지 못한다. 독립 커밋이 요구사항인지 검증하고 해당 데이터도 정리한다. 내부 작업이 외부 트랜잭션이 잠근 데이터를 접근하면 대기·교착 가능성까지 달라진다.

### Q. 서비스의 체크 예외 롤백 규칙은 어떻게 검증하는가?
A. 테스트 전체를 감싼 기본 롤백에 의존하지 않는다. 서비스가 자신의 트랜잭션을 시작하도록 호출하고 예외를 확인한 뒤 새로운 조회 경계에서 데이터가 남았는지 확인한다. 내부 self-invocation 때문에 프록시를 건너뛴 경우도 분리해 확인한다.

### Q. 커밋 후 콜백이 실행됐으면 메시지 발행까지 안전한가?
A. 콜백 실행 사실과 외부 브로커의 저장은 별개다. 커밋 후 프로세스 종료·발행 실패를 견뎌야 하면 [[트랜잭셔널 아웃박스]] 등으로 복구 경로를 설계하고 중복 처리도 검증한다.

## 관련 개념

- [[단위 테스트와 통합 테스트 경계]]
- [[Transactional 동작 원리]]
- [[트랜잭션 전파와 격리]]
- [[JPA 영속성 컨텍스트]]
- [[Spring Event와 비동기 처리]]
- [[TestContainers]]

## 참고 자료

검증일: 2026-09-23. Spring Framework 7.0.9·Spring Boot 4.1.1의 테스트 경계와 API를 확인했다. OpenJDK 25.0.2·Boot 4.1.1 관리 의존성으로 기본 롤백, 명시 커밋과 afterCommit, REQUIRES_NEW, 별도 스레드, 속성 전달을 실행했다. DB 고유 격리·지연 제약 및 위 JPA 예제의 도메인 코드는 실행 범위에 포함하지 않는다.

- [TestContext transaction management](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html) — 기본 롤백, 생명주기, flush, 별도 스레드와 공식 속성 지원표.
- [TestTransaction 7.0.9 API](https://docs.spring.io/spring-framework/docs/7.0.9/javadoc-api/org/springframework/test/context/transaction/TestTransaction.html) — 명시적 종료·재시작 API.
- [TransactionalTestExecutionListener 7.0.9 소스](https://github.com/spring-projects/spring-framework/blob/v7.0.9/spring-test/src/main/java/org/springframework/test/context/transaction/TransactionalTestExecutionListener.java) — 서비스 AOP와 다른 테스트 경계 관리.
- [TransactionContext 7.0.9 소스](https://github.com/spring-projects/spring-framework/blob/v7.0.9/spring-test/src/main/java/org/springframework/test/context/transaction/TransactionContext.java) — 속성 전달 및 rollback flag에 따른 종료.
- [TestContextTransactionUtils 7.0.9 소스](https://github.com/spring-projects/spring-framework/blob/v7.0.9/spring-test/src/main/java/org/springframework/test/context/transaction/TestContextTransactionUtils.java) — 속성 위임 구현. 공식 지원표와 구현 관찰 차이를 확인했다.
- [Testing Spring Boot Applications](https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html) — Boot 4.1.1, 실제 서버의 별도 스레드·트랜잭션.
- [Jakarta Persistence 3.2 EntityManager](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/entitymanager) — flush·clear·1차/2차 캐시 경계.
