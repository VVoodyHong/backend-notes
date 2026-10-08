---
category: "Spring/데이터 접근과 트랜잭션"
tags: ["spring", "jpa", "hibernate", "동시성", "transaction"]
aliases: ["JPA Lock", "Optimistic Locking", "Pessimistic Locking"]
updated: 2026-09-22
verified: 2026-09-22
---

# JPA 낙관적 락과 비관적 락

## 핵심 정의

낙관적 락(Optimistic Locking)은 엔티티의 버전 등을 비교해 다른 트랜잭션이 먼저 바꾼 데이터를 덮어쓰는 충돌을 검출한다. 비관적 락(Pessimistic Locking)은 DB 잠금을 획득해 경쟁하는 갱신을 조율한다. JPA는 `@Version`과 `LockModeType`으로 이를 표현하고 실제 SQL·잠금 범위는 구현체와 DB에 위임한다.

낙관적 락도 UPDATE 실행 중에는 DB의 쓰기 잠금을 사용할 수 있다. 두 방식은 “DB 잠금을 전혀 쓰는가”보다 충돌을 언제 확인하고 어떻게 처리하는가로 구분한다. DB 잠금 자체는 [[락의 종류]], 격리 수준은 [[트랜잭션 격리 수준]]에서 다룬다.

## 동작 원리 / 구조

### 버전 기반 충돌 검출

```java
@Entity
class Stock {
    protected Stock() {} // JPA 표준의 public/protected 무인자 생성자

    @Id
    private Long id;

    @Version
    private long version;

    private int quantity;
}
```

관리 엔티티 변경을 반영할 때 provider는 읽은 버전을 검사하고 버전을 갱신한다. 숫자 버전의 대표적인 SQL 형태는 다음과 같다. 정확한 SQL·검사 시점은 provider에 따라 다르다.

```sql
UPDATE stock
SET quantity = ?, version = ?
WHERE id = ? AND version = ?;
```

두 트랜잭션이 같은 버전을 읽었다면 먼저 반영한 쪽 이후의 오래된 갱신은 충돌한다. 예외는 API 호출·flush·commit 때 드러날 수 있다. 표준 `OptimisticLockException`은 트랜잭션을 rollback-only로 만든다. Spring 경계에서는 예외 변환으로 다른 상위 예외 타입이 보일 수도 있다.

### 잠금 모드 선택

| 모드 | 의미 | 확인할 점 |
|---|---|---|
| `OPTIMISTIC` | 버전 검사를 요청 | 읽기 의존성 검출에도 사용하며 실제 검사 시점은 지연될 수 있음 |
| `OPTIMISTIC_FORCE_INCREMENT` | 낙관적 검사와 버전 증가 요청 | 일반 필드 변경 없이도 집합 루트의 버전을 바꿀 필요가 있는 경우 |
| `PESSIMISTIC_READ` | 동시 갱신을 막는 비관적 읽기 잠금 | DB가 지원하지 않으면 더 강한 잠금을 쓸 수 있음 |
| `PESSIMISTIC_WRITE` | 경쟁 갱신을 직렬화하는 비관적 잠금 | 일반 MVCC 읽기까지 전부 차단한다는 뜻은 아님 |
| `PESSIMISTIC_FORCE_INCREMENT` | 비관적 잠금과 버전 증가 | 버전 엔티티에 사용 |

Spring Data JPA는 조회 메서드에 `@Lock`으로 잠금 모드를 지정한다. 잠금 조회와 변경을 같은 서비스 트랜잭션 안에 둔다.

```java
public interface StockRepository extends JpaRepository<Stock, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from Stock s where s.id = :id")
    Optional<Stock> findForUpdate(@Param("id") Long id);
}
```

`@Lock` 자체가 서비스 전체의 트랜잭션 경계를 만들지는 않는다. 잠금은 트랜잭션이 끝날 때 해제되며, 그 밖에서 상태를 판단·갱신하면 보호하려던 작업이 잠금 범위를 벗어난다.

## 실무 관점

- 충돌이 적고 재시도·충돌 응답을 수용할 수 있으면 낙관적 검사를 검토한다. 빈번한 충돌은 반복 재시도 비용을 키운다. 비관적 잠금은 대기·데드락·연결 점유 비용이 있으므로 임계 구역을 짧게 유지한다.
- 낙관적 충돌 후에는 실패한 트랜잭션과 엔티티를 계속 사용하지 않는다. 새 트랜잭션에서 다시 읽고 업무 조건을 재평가한다. 외부 결제·메시지 전송은 재시도 중 중복될 수 있으므로 멱등성·아웃박스를 함께 설계한다.
- “재고가 충분하면 차감”처럼 한 SQL로 표현할 수 있으면 `UPDATE ... SET quantity = quantity - :n WHERE quantity >= :n`과 변경 행 수 확인도 선택지다. 같은 테이블의 다른 JPA 경로가 버전을 사용한다면 버전 증가·관리 객체 정합성도 맞춘다.
- JPQL/Criteria 벌크 갱신은 `@Version` 검사·증가를 자동 수행하지 않는다. 필요한 버전 조건·증가를 직접 표현하고, `flush()`·`clear()` 순서와 미반영 변경 유실을 확인한다.
- `jakarta.persistence.lock.timeout`은 밀리초 단위 힌트다. 모든 DB·드라이버가 동일하게 지원하지 않으므로 실제 잠금 대기와 예외를 검증한다. DB 잠금 대기, 쿼리 타임아웃, HTTP 요청 기한은 서로 다르다.

## 심화 Q&A

### Q. `@Version`만 붙이면 여러 행에 걸친 불변식도 보장되는가?
아니다. 서로 다른 행을 갱신하면 각 행의 버전 검사를 통과하면서 집합 조건을 깨뜨리는 쓰기 편향(Write Skew)이 가능하다. 공통 집합 루트의 버전·잠금, DB 제약, 적절한 SERIALIZABLE과 재시도 중 불변식에 맞는 방법을 선택한다. 버전 필드 하나가 임의의 조회 조건 전체를 보호하지 않는다.

### Q. 부모 엔티티를 비관적으로 잠그면 자식 엔티티도 전부 잠기는가?
일반적으로 그렇지 않다. 표준 잠금 범위는 잠근 엔티티의 DB 상태를 기준으로 하며 참조한 다른 엔티티까지 자동 확장되지 않는다. 확장 잠금 범위도 임의의 연관 그래프 전체를 잠그는 기능은 아니다. 실제 조회·잠금 SQL과 DB의 인덱스·범위 잠금을 확인한다.

### Q. 이미 관리 중인 엔티티를 비관적 잠금으로 다시 find하면 최신 필드로 바뀌는가?
`find(..., PESSIMISTIC_WRITE)`는 영속성 컨텍스트에 있는 객체를 반환하고 잠금을 획득하는 동작이지 자동 `refresh`가 아니다. 버전 엔티티라면 잠금 획득 시에도 버전을 검사하므로 다른 트랜잭션이 버전을 바꿨다면 `OptimisticLockException`이 발생할 수 있다. 버전을 올리지 않은 벌크 갱신은 잠금에 성공해도 관리 객체의 오래된 필드를 남길 수 있다. 다시 읽어야 한다면 미반영 변경을 덮어쓴다는 점을 고려해 `refresh(entity, lockMode)` 등 갱신 경계를 명시한다.

### Q. 조회 뒤 오랫동안 사용자가 편집하는 화면에 DB 잠금을 유지해야 하는가?
사용자 편집 시간 동안 트랜잭션과 연결을 유지하지 않는다. 응답의 버전 또는 ETag를 후속 변경 요청에서 대조하는 충돌 계약을 설계한다. 충돌 시 최신 상태를 보여주고 재입력·병합 여부를 결정한다. 서버가 새 버전을 재조회한 뒤 오래된 화면 값을 무조건 덮어쓰면 클라이언트 편집 충돌을 놓칠 수 있다.

### Q. 잠금 타임아웃과 낙관적 충돌을 catch한 뒤 같은 트랜잭션에서 재시도해도 되는가?
낙관적 충돌이나 트랜잭션 전체가 실패한 경우에는 새 경계가 필요하다. JPA는 문장 수준 잠금 타임아웃의 `LockTimeoutException`과 트랜잭션 수준 실패의 `PessimisticLockException`을 구분하지만 DB가 실제로 남긴 상태도 확인해야 한다. 예외를 삼켰다는 이유로 트랜잭션이 복구되지는 않는다.

## 관련 개념

- [[JPA 영속성 컨텍스트]]
- [[트랜잭션 전파와 격리]]
- [[락의 종류]]
- [[트랜잭션 격리 수준]]
- [[멱등성 키 설계]]

## 참고 자료

검증일: 2026-09-22. 적용 범위: Jakarta Persistence 3.2의 잠금 계약, Spring Data JPA 4.1 계열의 `@Lock`. Hibernate 7.1.0.Final/H2 2.3.232에서 두 EntityManager의 버전 충돌, 벌크 갱신의 버전 검사·컨텍스트 동기화 부재, 관리 엔티티의 비관적 잠금과 refresh의 차이를 실행 확인했다. PostgreSQL/MySQL의 잠금 SQL·대기 시간은 이 실험의 검증 범위가 아니다.

- [Jakarta Persistence 3.2 §3.5·§4.11](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2) — 버전 검사, 잠금 범위·예외·timeout 힌트, 벌크 갱신.
- [Spring Data JPA Locking](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html) — 조회 메서드와 CRUD 재정의의 잠금 모드.
- [Spring Data JPA Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html) — 선언한 쿼리 메서드와 서비스 트랜잭션 경계.
- [Hibernate ORM 7.1 Locking](https://docs.hibernate.org/orm/7.1/userguide/html_single/Hibernate_User_Guide.html#locking) — 낙관적·비관적 잠금과 provider 구현.
