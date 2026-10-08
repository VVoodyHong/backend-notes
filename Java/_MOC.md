# Java

Java 언어 자체와 JVM 내부 동작, 동시성, 컬렉션/스트림, 예외·IO를 다루는 노트 모음.

## 추천 읽기 순서

- 언어와 컬렉션: [[equals와 hashCode]] → [[불변 객체]] → [[제네릭]] → [[HashMap 내부 구조]] → [[ConcurrentHashMap]] → [[Stream API]]
- JVM과 진단: [[JVM 구조와 클래스 로더]] → [[런타임 데이터 영역]] → [[가비지 컬렉션 알고리즘]] → [[GC 튜닝]] → [[JFR과 프로파일링]] → [[힙덤프와 스레드덤프 분석]]
- 동시성: [[스레드 생명주기와 상태]] → [[Java Memory Model과 happens-before]] → [[volatile과 가시성]] → [[synchronized와 Lock]] → [[ExecutorService와 스레드 풀]] → [[인터럽트와 작업 취소]] → [[CompletableFuture]] → [[가상 스레드]] → [[ScopedValue]]

## 언어 핵심
Java 언어 핵심 8개 개념(제네릭, equals/hashCode, String Pool, 불변 객체, Optional, 애노테이션/리플렉션, Record, 함수형 인터페이스와 람다)을 다룬다.
- [[제네릭]]
- [[equals와 hashCode]]
- [[String과 String Pool]]
- [[불변 객체]]
- [[Optional]]
- [[애노테이션과 리플렉션]]
- [[Record]]
- [[함수형 인터페이스와 람다]]

## JVM과 메모리 관리
JVM 구조, 런타임 데이터 영역, GC 알고리즘/튜닝, JIT 컴파일러, 메모리 누수를 다룬다.
- [[JVM 구조와 클래스 로더]]
- [[런타임 데이터 영역]]
- [[가비지 컬렉션 알고리즘]]
- [[GC 튜닝]]
- [[JIT 컴파일러]]
- [[메모리 누수]]

## 동시성과 멀티스레딩
스레드 생명주기와 메모리 모델, 락과 동기화 도구, 실행기와 비동기 작업, 가상 스레드를 다룬다.
- [[스레드 생명주기와 상태]]
- [[Java Memory Model과 happens-before]]
- [[volatile과 가시성]]
- [[synchronized와 Lock]]
- [[데드락과 경쟁 상태]]
- [[Semaphore와 CountDownLatch와 CyclicBarrier]]
- [[BlockingQueue]]
- [[StampedLock과 낙관적 읽기]]
- [[ExecutorService와 스레드 풀]]
- [[Fork-Join 프레임워크]]
- [[CompletableFuture]]
- [[ThreadLocal]]
- [[ScopedValue]]
- [[인터럽트와 작업 취소]]
- [[가상 스레드]]

## 컬렉션과 스트림
HashMap/ConcurrentHashMap 내부 구조, ArrayList/LinkedList 비교, Stream API, 컬렉션 동기화를 다룬다.
- [[HashMap 내부 구조]]
- [[ConcurrentHashMap]]
- [[ArrayList와 LinkedList]]
- [[Stream API]]
- [[컬렉션 동기화]]

## 예외 처리와 IO
Checked/Unchecked 예외, try-with-resources, Java NIO를 다룬다.
- [[Checked와 Unchecked 예외]]
- [[try-with-resources]]
- [[Java NIO]]

## 언어 심화
Sealed Interface/Pattern Matching, Switch Expression, Text Block, Enum 활용, 오토박싱, Object/Comparable/Iterator, 내부 클래스 종류를 다룬다.
- [[Sealed Interface와 Pattern Matching]]
- [[Switch Expression]]
- [[Text Block]]
- [[Enum 활용법]]
- [[가변인자와 오토박싱]]
- [[Object 클래스와 메서드 오버라이드]]
- [[Comparable과 Comparator]]
- [[Iterator와 Iterable]]
- [[내부 클래스 종류]]

## JVM 심화
클래스 파일 구조와 바이트코드, 직렬화(Serializable)를 다룬다.
- [[클래스 파일 구조와 바이트코드]]
- [[직렬화와 Serializable]]

## 테스트와 진단
JUnit5 심화, Mockito, JFR 프로파일링, 힙덤프/스레드덤프 분석을 다룬다.
- [[JUnit5 심화와 테스트 전략]]
- [[Mockito 활용법]]
- [[JFR과 프로파일링]]
- [[힙덤프와 스레드덤프 분석]]

## 성능 최적화
Escape Analysis, String 연산 최적화, 컬렉션 초기 용량과 로드 팩터를 다룬다.
- [[Escape Analysis와 스칼라 치환]]
- [[String 연산과 StringBuilder 최적화]]
- [[컬렉션 초기 용량과 로드 팩터]]

## 다른 분야 연결

- 운영체제와 메모리: [[프로세스와 스레드]], [[캐시 지역성]]
- Spring의 실행과 자원 관리: [[Bean 생명주기]], [[프록시 기반 AOP 동작 원리]], [[Transactional 동작 원리]], [[Spring Event와 비동기 처리]]
- I/O와 웹 처리: [[소켓 프로그래밍 기초]], [[Spring WebFlux와 리액티브 스트림]], [[WebClient와 RestTemplate]]
