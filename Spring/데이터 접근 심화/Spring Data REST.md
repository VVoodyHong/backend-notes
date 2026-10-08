---
category: "Spring/데이터 접근 심화"
tags: ["spring", "spring-data", "rest", "hateoas", "jpa"]
updated: 2026-10-04
verified: 2026-09-08
---

# Spring Data REST

## 핵심 정의
Spring Data REST는 지원되는 Spring Data 리포지토리를 HAL(Hypertext Application Language) 기반 하이퍼미디어 API로 노출하는 모듈이다. 사용 가능한 CRUD·페이징 메서드와 노출 설정에 따라 엔드포인트가 생성된다. Spring Data 3.0부터 PagingAndSortingRepository만 상속하면 CRUD 메서드는 포함되지 않는다. 아래 JPA 예제는 JpaRepository를 사용한다.

## 동작 원리 / 구조

### 요청 처리 흐름
```mermaid
flowchart LR
    A[HTTP 요청 GET /members/1] --> B[RepositoryRestHandlerMapping]
    B --> C["RepositoryEntityController\n(내장 컨트롤러)"]
    C --> D["JpaRepository\n구현체 호출"]
    D --> E[JPA/Hibernate]
    E --> F["PersistentEntityResourceAssembler\n→ HAL JSON 변환"]
    F --> G["_links 포함 응답\n(self, member, 연관 리소스)"]
```

Spring MVC의 `DispatcherServlet`은 그대로 쓰이지만, 개발자가 만든 `@RestController` 대신 Spring Data REST가 제공하는 범용 컨트롤러(`RepositoryEntityController`, `RepositoryPropertyReferenceController` 등)가 요청을 가로채 리포지토리 메서드로 위임한다. 엔티티 노출 여부, 경로명, 페이지 크기 등은 `@RepositoryRestResource`, `@RestResource` 애노테이션이나 `RepositoryRestConfigurer`로 커스터마이징한다.

### 자동 생성되는 리소스 예시
```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByTeamId(@Param("teamId") Long teamId); // 자동으로 /members/search/findByTeamId?teamId=1 노출
}
```
응답 예시(HAL):
```json
{
  "_embedded": { "members": [ { "name": "홍길동", "_links": { "self": {"href": "/members/1"} } } ] },
  "_links": { "self": {"href": "/members"}, "profile": {"href": "/profile/members"} },
  "page": { "size": 20, "totalElements": 1, "totalPages": 1, "number": 0 }
}
```
- 쿼리 메서드는 `/search` 하위 경로로 자동 노출되며, `@RestResource(exported = false)`로 개별 메서드나 리포지토리를 숨길 수 있다.
- 연관관계는 `_links`로 표현되고 별도 요청으로 지연 탐색한다(HAL의 하이퍼미디어 원칙).
- ALPS(Application-Level Profile Semantics) 메타데이터를 `/profile`(루트에서는 단일 링크, 리소스별로는 `/profile/{리포지토리명}`) 경로에서 제공해 클라이언트가 리소스 구조를 발견할 수 있다.

## 실무 관점
- **적합한 상황**: 내부 관리자 도구, 프로토타입, CRUD 위주의 백오피스처럼 API 설계 유연성보다 개발 속도가 중요한 경우에 유리하다. 짧은 시간에 CRUD 엔드포인트 뼈대를 갖추고 싶을 때 채택한다.
- **부적합한 상황**: 도메인 로직(검증, 트랜잭션 조합, 권한 세분화)이 복잡한 서비스 API에는 적합하지 않다. 엔티티 구조가 API 스펙에 그대로 노출되므로 엔티티 변경이 곧 API 스펙 변경이 되어 버전 관리와 하위 호환성 유지가 어렵다. 실무에서는 이런 이유로 공개 API나 도메인 복잡도가 높은 서비스에는 잘 채택하지 않고, 직접 `@RestController` + 서비스 계층을 작성하는 방식을 선호한다.
- **보안**: 리포지토리·메서드 노출 설정과 실제 인증·인가를 별도로 설계한다. 공개할 저장소만 명시적으로 내보내는 정책을 검토하고, URL·메서드·행/필드별 권한을 검증한다. exported=false는 해당 REST 노출을 막는 설정이지 다른 코드 경로의 인가를 대체하지 않는다.
- **이벤트 훅**: 저장/삭제 전후 로직이 필요하면 `@RepositoryEventHandler` + `@HandleBeforeSave`/`@HandleAfterCreate` 등을 사용한다. 다만 이런 훅이 늘어날수록 "자동 생성 API에 수동 로직을 끼워 넣는" 구조가 되어 결국 일반 컨트롤러 방식보다 코드 흐름 추적이 어려워지는 역설이 발생하기 쉽다.
- **프로젝션과 Excerpt**: excerpt는 목록·연관 리소스의 기본 표현에 적용되며 개별 리소스에는 자동 적용되지 않는다. 프로젝션은 @JsonIgnore로 숨긴 필드도 getter로 다시 노출할 수 있어 보안 allowlist로 간주하면 안 된다. 연관 필드를 펼치면 N+1도 발생할 수 있다.
- **클라이언트 계약**: 링크를 따라 리소스를 발견하는지, 고정 URL과 필드만 사용하는지 합의한다. 생성 API를 쓰더라도 페이지·정렬·표현 형식과 변경 호환성을 계약으로 테스트한다.
- **ETag 제공과 조건부 수정 강제는 별개**: Data REST 5.1.1은 JPA `@Version`을 ETag로 제공하고, 요청의 `If-Match`가 현재 버전과 다르면 412로 거부한다. 하지만 헤더가 없는 수정까지 자동으로 거부하지 않는다. 클라이언트가 오래전에 읽은 값을 덮어쓰지 못하게 해야 한다면 수정 요청에 이전 ETag를 보내도록 계약을 정하고 누락 처리도 별도로 구현·시험한다. `@Version`의 DB 저장 충돌 검사가 사용자 조회 시점부터 이어진 수정 의도를 대신 기억해 주지는 않는다. 412를 받았을 때는 새 버전을 조회해 충돌을 해결하며 최신 ETag로 무조건 재전송하지 않는다.

## 심화 Q&A

### Q. Spring Data REST와 일반 `@RestController` + `@Service` 계층 구조 중 어떤 기준으로 선택하는가?
판단 기준은 "API가 도메인 로직의 얇은 창구인가, 아니면 리포지토리의 거의 직접적인 노출인가"이다. 단순 마스터 데이터 관리처럼 비즈니스 규칙이 거의 없는 CRUD는 Spring Data REST로 개발 속도를 얻을 수 있다. 반면 여러 리포지토리를 조합하거나 트랜잭션 경계를 명시적으로 제어해야 하는 API는 서비스 계층이 필수이므로 일반 MVC 방식이 맞다. 실무에서는 두 방식을 혼합해, 내부 관리 도구는 Spring Data REST로, 외부 공개 API는 커스텀 컨트롤러로 분리하는 경우가 많다.

### Q. 엔티티 필드를 변경했더니 API 응답 구조가 바로 바뀌어 버리는 문제는 어떻게 완화하는가?
엔티티와 API 계약의 결합을 줄이려면 RepresentationModelProcessor 등으로 표현을 가공할 수 있지만, 프로젝션만 추가해 모든 엔드포인트의 노출 필드가 고정되지는 않는다. 목록·단건·검색·연관관계·프로젝션별 응답을 함께 확인한다. 독립된 계약과 복잡한 쓰기 규칙이 필요하면 DTO 기반 컨트롤러와 서비스 경계가 더 명확하다.

### Q. `findByTeamId` 같은 파생 쿼리 메서드가 `/search` 경로로 노출될 때 발생할 수 있는 성능 문제는?
파생 쿼리 메서드는 QueryDSL 같은 세밀한 조인 제어 없이 자동 생성되므로, 연관 엔티티를 포함한 응답을 만들 때 지연 로딩 접근이 각 리소스 직렬화 과정에서 개별적으로 발생해 [[N+1 문제]]가 그대로 나타날 수 있다. Spring Data REST는 이 문제를 자동으로 해결해주지 않으므로, `@EntityGraph`를 파생 쿼리 메서드에 함께 선언하거나 커스텀 프로젝션으로 완화해야 한다.

### Q. PATCH와 PUT 요청 처리 방식의 차이는 무엇이며 부분 수정에서 무엇을 검증해야 하는가?
PUT은 리소스 교체, PATCH는 부분 변경을 표현한다. PATCH에서도 Content-Type에 따른 JSON Patch 연산과 merge patch의 null·배열 의미를 구분해야 한다. 부분 변경이 권한·필수값·도메인 불변식 검증을 생략하게 해주지는 않는다. 동시 수정 충돌은 버전/ETag와 If-Match 등으로 별도 제어한다.

### Q. 클라이언트가 HAL 링크를 활용하지 않아도 Spring Data REST의 이점이 있는가?
페이징·정렬·검색 API를 자동 제공하는 이점은 남는다. 다만 HAL 링크를 제거하는 것만으로 일반 DTO API 계약이 되지는 않는다. 클라이언트가 요구하는 응답 형식과 도메인 로직이 자동 노출 모델에 맞는지 비교한다.

### Q. 이벤트 핸들러(`@HandleBeforeSave` 등)에서 예외를 던지면 트랜잭션과 HTTP 응답은 어떻게 처리되는가?
내장 컨트롤러는 BeforeSave/Create 이벤트 → repository save 호출 → AfterSave/Create 이벤트 순서로 동기 발행한다. 이 순서는 트랜잭션 시작·커밋 시점과 동일한 정의가 아니다. 저장소 메서드만 로컬 트랜잭션을 여는 일반 JPA 구성에서는 After 이벤트 시 이미 커밋됐을 수 있지만, 바깥 트랜잭션이 있다면 그 안에서 실행될 수 있다. Before 예외는 저장 호출을 막고 After 예외는 이미 저장된 데이터를 되돌리지 못할 수 있다. HTTP 상태는 예외 매핑에 따르며 처리되지 않은 예외는 5xx가 될 수 있다. 원자성이 필요한 검증·저장 로직은 명시적 서비스 트랜잭션으로 묶고, 커밋 후 외부 전송의 내구성은 아웃박스 등으로 설계한다.

## 관련 개념
- [[N+1 문제]]
- [[QueryDSL과 동적 쿼리]]
- [[Bean Validation]]
- [[페이징 최적화]]

## 참고 자료

부분 재검증: 2026-10-04. Spring Data REST 5.1.1의 ETag·If-Match 경계를 확인했다. Boot 4.1.1·Hibernate 7.4.5.Final·H2 2.4.240의 실제 HTTP 통합 테스트에서 GET 200, 잘못된/오래된 If-Match의 PATCH 412, 일치하는 헤더와 헤더 누락의 PATCH 200을 확인하고 저장 결과도 검사했다. 다른 DB의 동시 트랜잭션·격리 동작을 시험한 것은 아니다.

- [Conditional Operations — Data REST 5.1.1](https://docs.spring.io/spring-data/rest/reference/etags-and-other-conditionals.html) — 버전 기반 ETag와 조건부 수정.
- [ETag 5.1.1 소스](https://raw.githubusercontent.com/spring-projects/spring-data-rest/5.1.1/spring-data-rest-webmvc/src/main/java/org/springframework/data/rest/webmvc/support/ETag.java) — NO_ETAG와 버전 비교 조건.

검증일: 2026-09-08. 적용 범위: Spring Data REST 5.1.1, Spring Data Commons 3.0 이후 리포지토리 인터페이스.

- [Repository Resources](https://docs.spring.io/spring-data/rest/reference/repository-resources.html) — CRUD와 PUT/PATCH 리소스.
- [리포지토리 정의](https://docs.spring.io/spring-data/commons/reference/repositories/definition.html) — sorting 인터페이스와 CRUD 분리.
- [Projections and Excerpts](https://docs.spring.io/spring-data/rest/reference/projections-excerpts.html) — excerpt 범위와 숨김 데이터 노출.
- [Security](https://docs.spring.io/spring-data/rest/reference/security.html) — 메서드 인가.
- [RepositoryEntityController 소스](https://raw.githubusercontent.com/spring-projects/spring-data-rest/main/spring-data-rest-webmvc/src/main/java/org/springframework/data/rest/webmvc/RepositoryEntityController.java) — 저장 전후 이벤트 발행 순서.
