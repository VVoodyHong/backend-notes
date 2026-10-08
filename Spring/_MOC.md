# Spring

Spring Framework / Spring Boot의 IoC, AOP, MVC, 데이터 접근, 보안, 내부 동작, 테스트 전략을 다루는 노트 모음.

## 추천 읽기 순서

- 컨테이너와 프록시: [[ApplicationContext]] → [[Bean 생명주기]] → [[Bean 스코프]] → [[프록시 기반 AOP 동작 원리]] → [[Transactional 동작 원리]]
- 웹과 보안: [[DispatcherServlet 요청 처리 흐름]] → [[Filter와 Interceptor]] → [[Security Filter Chain]] → [[Method Security와 CSRF 방어]] → [[CORS 설정]]
- 데이터 접근: [[JPA 영속성 컨텍스트]] → [[지연 로딩과 즉시 로딩]] → [[N+1 문제]] → [[트랜잭션 전파와 격리]] → [[JPA 낙관적 락과 비관적 락]] → [[다중 데이터소스와 라우팅]]
- 설정과 기동: [[외부 설정과 ConfigurationProperties]] → [[Conditional과 조건부 빈]] → [[자동 구성 원리]] → [[Spring AOT와 네이티브 이미지]] → [[Actuator와 헬스체크]]
- 배치: [[Spring Scheduling]] → [[Spring Batch 아키텍처]]
- 테스트: [[단위 테스트와 통합 테스트 경계]] → [[Spring 테스트 트랜잭션과 커밋 검증]] → [[MockMvc]] → [[TestContainers]]

## IoC와 DI
Bean 생명주기, 스코프, ApplicationContext, 순환 참조 문제를 다룬다.
- [[Bean 생명주기]]
- [[Bean 스코프]]
- [[ApplicationContext]]
- [[순환 참조 문제]]

## AOP
프록시 기반 AOP 동작 원리, JDK Dynamic Proxy vs CGLIB, Advice 종류와 실행 순서를 다룬다.
- [[프록시 기반 AOP 동작 원리]]
- [[JDK Dynamic Proxy와 CGLIB]]
- [[Advice 종류와 실행 순서]]

## Spring MVC
DispatcherServlet 요청 처리 흐름, Filter/Interceptor, Bean Validation을 다룬다.
- [[DispatcherServlet 요청 처리 흐름]]
- [[Filter와 Interceptor]]
- [[Bean Validation]]

## 데이터 접근과 트랜잭션
JPA 영속성 컨텍스트, 지연/즉시 로딩, N+1 문제, 트랜잭션 전파/격리, @Transactional 동작 원리를 다룬다.
- [[JPA 영속성 컨텍스트]]
- [[지연 로딩과 즉시 로딩]]
- [[N+1 문제]]
- [[트랜잭션 전파와 격리]]
- [[Transactional 동작 원리]]
- [[JPA 낙관적 락과 비관적 락]]

## Spring Security
Security Filter Chain, JWT 인증, OAuth2와 소셜 로그인, 메서드 보안과 CSRF 방어를 다룬다.
- [[Security Filter Chain]]
- [[JWT 인증]]
- [[OAuth2와 소셜 로그인]]
- [[Method Security와 CSRF 방어]]

## Spring Boot 내부 동작
자동 구성 원리, 내장 WAS, Actuator와 헬스체크를 다룬다.
- [[자동 구성 원리]]
- [[Spring AOT와 네이티브 이미지]]
- [[내장 WAS]]
- [[Actuator와 헬스체크]]

## 테스트 전략
단위/통합 테스트 경계, MockMvc, TestContainers를 다룬다.
- [[단위 테스트와 통합 테스트 경계]]
- [[Spring 테스트 트랜잭션과 커밋 검증]]
- [[MockMvc]]
- [[TestContainers]]

## 웹 심화
HTTP 클라이언트와 선언형 호출, WebFlux, 전역 예외 처리, CORS 설정, 파일 업로드를 다룬다.
- [[WebClient와 RestTemplate]]
- [[HTTP Service Client와 선언형 호출]]
- [[HTTP 호출 타임아웃과 재시도 예산]]
- [[Spring WebFlux와 리액티브 스트림]]
- [[전역 예외 처리와 ControllerAdvice]]
- [[CORS 설정]]
- [[파일 업로드 처리]]

## 데이터 접근 심화
QueryDSL, Spring Data REST, 다중 데이터소스 라우팅을 다룬다.
- [[QueryDSL과 동적 쿼리]]
- [[Spring Data REST]]
- [[다중 데이터소스와 라우팅]]

## 인프라 연동
Spring Cloud Config/서비스 디스커버리, Spring Retry, Spring Event를 다룬다.
- [[Spring Cloud Config와 서비스 디스커버리]]
- [[Spring Retry와 재시도 전략]]
- [[Spring Event와 비동기 처리]]

## 배치와 스케줄링
주기적인 작업 실행과 Spring Batch의 Job/Step 구조, 청크 처리, 재시작을 다룬다.
- [[Spring Scheduling]]
- [[Spring Batch 아키텍처]]

## 설정과 확장
Bean 정의 방법 비교, @Conditional 조건부 빈을 다룬다.
- [[Bean 정의 방법 비교]]
- [[Conditional과 조건부 빈]]
- [[외부 설정과 ConfigurationProperties]]

## API 설계와 문서화
OpenAPI/Swagger 문서화와 Spring HATEOAS를 다룬다.
- [[OpenAPI와 Swagger 문서화]]
- [[Spring HATEOAS]]

## 캐시와 관측성
Spring Cache 추상화, Micrometer와 분산 트레이싱 연동을 다룬다.
- [[Spring Cache 추상화]]
- [[Micrometer와 분산 트레이싱 연동]]

## 다른 분야 연결

- Java의 실행 모델: [[ThreadLocal]], [[ExecutorService와 스레드 풀]], [[가상 스레드]]
- HTTP와 데이터베이스: [[HTTP 메서드와 상태 코드]], [[트랜잭션 격리 수준]], [[MVCC]]
- 캐시와 이벤트: [[캐시와 DB 정합성]], [[트랜잭셔널 아웃박스]]
- 운영과 진단: [[관측 가능성 3요소]], [[분산 트레이싱]]
