---
category: "Spring/데이터 접근 심화"
tags: ["spring", "jpa", "querydsl", "동적쿼리"]
updated: 2026-10-04
verified: 2026-09-08
---

# QueryDSL과 동적 쿼리

## 핵심 정의
Querydsl은 자바 코드로 타입 안전(type-safe)하게 쿼리를 작성하는 도구다. 애노테이션 프로세서가 엔티티의 Q타입 소스를 생성하고 컴파일하며, 애플리케이션은 이 클래스로 실행 시점에 조건을 조립한다. 문자열 속성명 대신 생성된 필드를 참조해 리팩터링 오류를 컴파일 단계에서 발견할 수 있다.

Spring Data JPA 4.1.1 공식 문서는 원본 com.querydsl과 OpenFeign 포크 io.github.openfeign.querydsl의 설정을 모두 안내하며 포크 연동을 best-effort로 지원한다. 원본의 Jakarta 대응 classifier와 포크의 classifier가 다르므로 groupId만 바꾸면 안 된다. 선택한 배포판의 릴리스·Java/JPA 호환성을 확인하고 버전을 고정한다.

## 동작 원리 / 구조

### Q타입 생성 과정
```mermaid
flowchart LR
    A["@Entity 클래스"] --> B["APT/annotationProcessor\n(querydsl-apt)"]
    B --> C["QMember, QTeam 등\nQ타입 소스 생성 (build 시점)"]
    C --> D["JPAQueryFactory로\n쿼리 조립"]
    D --> E["JPQL 생성 → Hibernate가\nSQL로 변환·실행"]
```

Gradle 기준 설정 예시(OpenFeign 포크, querydslVersion은 프로젝트에서 호환성을 검증한 버전으로 정의):
```gradle
dependencies {
    implementation "io.github.openfeign.querydsl:querydsl-jpa:${querydslVersion}"
    annotationProcessor "io.github.openfeign.querydsl:querydsl-apt:${querydslVersion}:jpa"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}
```

### 동적 쿼리 조합 - BooleanExpression과 null 반환
```java
public List<Member> search(String name, Integer minAge) {
    return queryFactory
        .selectFrom(member)
        .where(
            nameEq(name),
            ageGoe(minAge)
        )
        .fetch();
}

private BooleanExpression nameEq(String name) {
    return StringUtils.hasText(name) ? member.name.eq(name) : null;
}

private BooleanExpression ageGoe(Integer age) {
    return age != null ? member.age.goe(age) : null;
}
```
`where()`에 전달된 조건 중 `null`은 JPQL 조립 시 자동으로 무시된다. 이 특성이 동적 쿼리를 가능하게 하는 핵심 메커니즘이다. 조건을 `BooleanExpression`으로 메서드화해두면 다른 쿼리에서도 재사용(`.and()`, `.or()`로 조합)할 수 있어, `if`문으로 문자열 JPQL을 누덕누덕 이어붙이는 방식보다 컴파일 타임에 오류를 잡을 수 있다.

### Querydsl-JPA와 Spring Data 리포지토리 결합
QuerydslPredicateExecutor로 기본 Predicate 조회를 사용하거나 사용자 정의 리포지토리 fragment와 JPAQueryFactory를 결합한다. 조인·DTO·별도 count를 명시할 때는 후자가 제어 범위를 드러낸다. QuerydslRepositorySupport는 Spring Data API지만 Querydsl4RepositorySupport라는 이름은 별도 사용자 유틸리티일 수 있으므로 공식 클래스로 가정하지 않는다.

## 실무 관점
- **선택 조건과 필수 범위 분리**: 검색어가 없을 때 조건을 생략하는 패턴을 테넌트·소유자 조건에 그대로 쓰지 않는다. 원본 Querydsl JPA 5.1.0에서 `where((Predicate) null, new BooleanBuilder())`만 남은 삭제 쿼리는 조건 없는 전체 삭제가 된다. 인증 정보에서 얻은 필수 범위가 없으면 쿼리 실행 전에 거부하고, 선택 조건이 모두 비어도 필수 조건은 유지한다. 수정·삭제 테스트에는 다른 테넌트의 행이 그대로 남는지 포함한다.
- **언제 쓰는가**: 가변 검색 조건과 재사용할 조건 표현식이 많을 때 검토한다. 조건 개수만으로 도입을 결정하지 않는다. 단순 CRUD나 고정 쿼리는 파생 메서드와 @Query로 충분할 수 있다.
- **Specification과의 비교**: JPA Criteria의 정적 메타모델을 사용하면 Specification도 타입 안전하게 작성할 수 있다. 문자열 get("name") 사용 여부와 DSL 가독성, 생성 코드·의존성 관리 비용을 비교한다.
- **DTO 프로젝션**: `Projections.constructor()`, `Projections.fields()`, `@QueryProjection`(생성자에 애노테이션을 붙여 Q타입을 생성) 세 가지 방식이 있다. `@QueryProjection`은 컴파일 타임에 생성자 시그니처를 검증하지만 DTO가 QueryDSL에 의존하게 되는 트레이드오프가 있어, 계층 분리를 중시하는 프로젝트는 `Projections.constructor()`를 더 선호하기도 한다.
- **count와 페이징**: 원본 Querydsl의 fetchCount()/fetchResults()는 deprecated이며 여러 GROUP BY 요소나 HAVING이 있으면 전체 결과를 메모리에서 세는 경로가 있다. 이 상태를 모든 포크 버전에 동일하게 일반화하지 않는다. 조회 대상의 중복·그룹 의미를 반영한 count를 별도로 작성한다. 컬렉션 fetch join의 페이징은 ORM 버전·DB에 따라 달라진다. Hibernate 7.4부터 지원 DB에서는 SQL로 부모를 제한하므로 과거의 메모리 페이징 설명을 모든 버전에 적용하지 않는다. [[N+1 문제#Q. fetch join과 페이징(`Pageable`)은 Hibernate 버전에 따라 어떻게 다른가?|버전별 조회 계획]]을 확인한다.
- **빌드 설정**: 생성 소스 디렉터리는 버전 관리에서 제외하고 annotationProcessor를 빌드에 선언한다. IDE가 Gradle에 빌드를 위임하는지 자체 컴파일하는지에 따라 설정이 다르다. 생성된 Q클래스가 필요한 모듈의 컴파일·실행 클래스패스에 포함되도록 한다.
- **마이그레이션**: Jakarta JPA용 원본 설정은 querydsl-jpa와 querydsl-apt에 jakarta classifier를 쓰지만, 공식 문서의 OpenFeign 설정은 querydsl-jpa에 classifier 없이 querydsl-apt에 jpa를 사용한다. JPA EntityManager의 javax/jakarta 혼용과 두 groupId의 중복 포함을 의존성 트리에서 확인한다.

## 심화 Q&A

### Q. `where()`에 여러 `BooleanExpression`을 콤마로 나열하는 방식과 `.and()`로 체이닝하는 방식은 어떤 차이가 있는가?
where(a, b)는 null 조건을 건너뛰며 AND로 결합한다. a.and(null)은 a가 null이 아니라면 허용되고 원래 표현식을 반환한다. NPE는 nameEq(null).and(other)처럼 메서드를 호출하는 수신 객체 자체가 null일 때 발생한다. 따라서 null을 반환하는 조건 메서드들은 where의 인자로 전달하는 방식이 간단하다.

### Q. `@QueryProjection` 방식과 `Projections.constructor()` 방식 중 어떤 걸 선택해야 하는가?
`@QueryProjection`은 DTO 생성자에 애노테이션을 붙여 전용 Q타입(QMemberDto 등)을 만들고, 이를 통해 생성자 인자 개수·타입이 컴파일 타임에 검증된다. 대신 DTO 모듈이 QueryDSL에 의존성을 갖게 되어, DTO를 여러 계층(API 응답, 배치 등)에서 공유하는 구조라면 의존성 오염이 문제가 될 수 있다. `Projections.constructor()`는 DTO가 순수 POJO로 남지만 생성자 시그니처 불일치가 런타임에야 발견된다. 계층 분리가 중요한 대규모 프로젝트는 후자를, 조회 전용 모듈이 명확히 분리된 프로젝트는 전자를 선호하는 경향이 있다.

### Q. QueryDSL로 작성한 동적 쿼리가 카티션 곱(cartesian product)을 유발하는 상황은 언제이고 어떻게 피하는가?
같은 부모의 독립적인 두 to-many 연관을 병렬로 `fetchJoin()`하면 자식 조합만큼 결과 행이 늘어난다. 여러 to-one이나 하나의 연관 경로를 따라가는 중첩 fetch를 모두 같은 문제로 묶지는 않는다. QueryDSL 문법이 유효해도 실제 SQL 비용과 Hibernate의 복수 bag fetch 제약을 확인해야 한다. 한 컬렉션만 fetch하고 나머지를 배치 로딩·별도 쿼리로 분리하는 방식이 흔한 대안이며, “모든 컬렉션 fetch는 반드시 하나만 가능하다”는 문법 규칙은 아니다. 자세한 원리는 [[N+1 문제]] 참고.

### Q. `BooleanBuilder`와 조건 메서드(`BooleanExpression` 반환)를 조합하는 방식 중 무엇이 더 권장되는가?
BooleanBuilder는 변경 가능한 조건 누적기이며 반복문이나 중첩 그룹을 구성할 때 유용하다. BooleanExpression 메서드는 조건 재사용과 조합을 드러내기 좋다. 둘 다 유효한 방식이며 조건의 괄호와 null 처리를 읽기 쉽게 표현하는 쪽을 선택한다.

### Q. Native SQL이 필요한 복잡한 통계 쿼리도 QueryDSL로 대체할 수 있는가?
`querydsl-sql` 모듈을 쓰면 JPA 엔티티가 아닌 테이블 메타데이터 기반으로 순수 SQL에 가까운 쿼리를 타입 안전하게 작성할 수 있지만, JPQL 기반 `querydsl-jpa`와는 별도 모듈이며 영속성 컨텍스트와 무관하게 동작한다. DB 종속적인 윈도우 함수, CTE, 벤더 전용 함수가 필요한 복잡한 리포트성 쿼리는 QueryDSL로 무리하게 표현하려 하기보다 `@Query`의 native query나 MyBatis 같은 별도 도구로 분리하는 것이 유지보수 측면에서 더 낫다는 것이 실무 판단 기준이다.

### Q. 생성된 Q타입을 CI/CD에서 어떻게 관리해야 하는가?
Q타입은 생성된 Java 소스이며 컴파일된 클래스는 쿼리 조립을 위해 런타임에도 필요하다. 매번 무조건 재생성하는 것은 아니며 정상적인 증분 빌드·캐시는 입력 변경을 추적한다. CI에서는 깨끗한 체크아웃에서도 생성→컴파일→패키징이 성공하는지 확인하고, 생성물을 누락한 아티팩트나 잘못된 캐시 입력 설정을 점검한다.

## 관련 개념
- [[N+1 문제]]
- [[지연 로딩과 즉시 로딩]]
- [[JPA 영속성 컨텍스트]]
- [[페이징 최적화]]

## 참고 자료

부분 재검증: 2026-10-04. 원본 Querydsl JPA 5.1.0의 Jakarta 배포판·Hibernate 7.4.5.Final·H2 2.4.240에서 3건을 실행했다. 빈/null 조건 삭제가 3행 전체에 적용되는 경우, 필수 테넌트 조건이 다른 테넌트 2행을 보존하는 경우, 필수 범위 누락을 실행 전에 거부하는 경우를 확인했다. OpenFeign 포크와 APT 설정은 이번 시험 범위가 아니다.

- [DefaultQueryMetadata 5.1.0 소스](https://raw.githubusercontent.com/querydsl/querydsl/QUERYDSL_5_1_0/querydsl-core/src/main/java/com/querydsl/core/DefaultQueryMetadata.java), [JPADeleteClause 5.1.0 소스](https://raw.githubusercontent.com/querydsl/querydsl/QUERYDSL_5_1_0/querydsl-jpa/src/main/java/com/querydsl/jpa/impl/JPADeleteClause.java) — null·빈 조건 제거와 삭제 쿼리 실행.

검증일: 2026-09-08. 적용 범위: Spring Data JPA 4.1.1의 Querydsl 연동 및 원본 Querydsl 5 계열 API.

- [Spring Data Querydsl 연동](https://docs.spring.io/spring-data/jpa/reference/repositories/core-extensions.html) — 원본과 OpenFeign 의존성 설정 및 best-effort 지원.
- [BooleanExpression 소스](https://raw.githubusercontent.com/querydsl/querydsl/master/querydsl-core/src/main/java/com/querydsl/core/types/dsl/BooleanExpression.java) — and/or null 인자 처리.
- [AbstractJPAQuery 소스](https://raw.githubusercontent.com/querydsl/querydsl/master/querydsl-jpa/src/main/java/com/querydsl/jpa/impl/AbstractJPAQuery.java) — 원본 fetchCount/fetchResults의 메모리 집계 제약.
- [Hibernate ORM 7.1 User Guide](https://docs.hibernate.org/orm/7.1/userguide/html_single/) — fetch join과 페이징.

부분 재검증: 2026-09-23. 변경 범위는 Querydsl이 생성한 JPQL 이후 Hibernate의 컬렉션 페이징과 병렬 to-many fetch의 행 증폭이다. to-one·중첩 fetch와 구분해 공식 HQL 설명을 대조했다. Querydsl 배포판·APT 설정 전체를 다시 검증한 것은 아니다.

- [Hibernate ORM 7.4 What's New](https://docs.hibernate.org/orm/7.4/whats-new/) — 지원 DB의 컬렉션 fetch 페이징 변경.
- [Hibernate Query Language 7.4](https://docs.hibernate.org/orm/7.4/querylanguage/html_single/Hibernate_Query_Language.html) — Association fetching, 병렬 to-many와 중첩 fetch의 구분.
