---
category: "Java/JVM과 메모리 관리"
tags: ["java", "jit", "hotspot", "성능최적화", "바이트코드"]
aliases: ["Just-In-Time Compiler", "JIT 컴파일"]
updated: 2026-09-23
verified: 2026-09-08
---

# JIT 컴파일러

## 핵심 정의
JIT(Just-In-Time) 컴파일러는 인터프리터가 바이트코드를 한 줄씩 해석 실행하는 방식의 느린 속도를 보완하기 위해, 실행 중 자주 호출되는 코드(hot spot)를 탐지해 해당 부분만 네이티브 머신 코드로 컴파일해 캐싱하는 HotSpot JVM의 핵심 최적화 기술이다.

정적 컴파일과 달리 실제 실행 시점의 프로파일링 정보(호출 빈도, 분기 확률, 실제 타입 등)를 활용할 수 있어 더 공격적인 최적화가 가능하지만, 그만큼 최적화 효과가 나타나기까지 예열(warm-up) 시간이 필요하다는 특성이 있다.

## 동작 원리 / 구조

### 계층형 컴파일 (Tiered Compilation)
```mermaid
graph LR
    A[인터프리터 실행] -->|호출 카운트 임계값 도달| B[C1 컴파일러]
    B -->|프로파일링 지속| C[C2 컴파일러]
    C -->|가정 위반 감지| D[역최적화 Deoptimization]
    D --> A
```
- **인터프리터**: 초기 실행, 바이트코드를 그대로 해석하며 프로파일링 데이터 수집
- **C1 (Client Compiler)**: 빠른 컴파일, 가벼운 최적화. 시작 속도가 중요한 경우 유리
- **C2 (Server Compiler)**: 느린 컴파일이지만 인라이닝, 루프 최적화 등 훨씬 공격적인 최적화 수행. 오래 실행되는 서버 애플리케이션에 유리

기본값인 Tiered Compilation은 C1과 C2를 단계적으로 적용해 초기 반응성과 장기 처리량을 모두 잡는다.

### 주요 최적화 기법
- **메서드 인라이닝(inlining)**: 짧고 자주 호출되는 메서드 호출을 호출부에 직접 삽입해 호출 오버헤드 제거
- **OSR(On-Stack Replacement)**: 메서드 호출 없이 오래 도는 루프 자체를 실행 도중 컴파일된 코드로 교체
- **Escape Analysis(탈출 분석)**: 객체가 메서드 밖으로 참조가 빠져나가지 않으면 스칼라 치환(scalar replacement)으로 Heap 할당을 제거하거나 락 제거(lock elision)로 최적화
- **역최적화(Deoptimization)**: 인라이닝이나 타입 추측의 근거가 된 가정이 런타임에 깨지면(예: 다형성 호출 지점에 새로운 타입 등장) 컴파일된 코드를 폐기하고 인터프리터로 되돌아감

## 실무 관점
- **Warm-up 문제**: 서버 재기동 직후나 무중단 배포 롤링 직후에는 JIT이 아직 핫스팟을 컴파일하지 못해 응답 지연이 평소보다 크게 나타난다. 헬스체크 통과 직후 바로 실 트래픽을 전량 투입하면 초반 지연 스파이크가 발생하는 이유다.
- **완화 방법**:
  - **CDS/AppCDS (Class Data Sharing)**: 클래스 메타데이터를 미리 아카이브해 클래스 로딩 시간 단축
  - **점진적 트래픽 유입(warm-up ramp-up)**: 로드밸런서에서 신규 인스턴스에 트래픽을 서서히 늘려 붓기
  - **Project Leyden(AOT 캐시 등)**: Java 25의 JEP 514는 AOT 캐시 생성 명령을 단순화하고 JEP 515는 이전 훈련 실행의 메서드 프로파일을 재사용해 JIT이 일찍 최적화하도록 한다. 네이티브 코드 전체를 저장하는 AOT 컴파일과 구분한다
- **디버깅/관측**: `-XX:+PrintCompilation`으로 어떤 메서드가 언제 어느 티어로 컴파일되는지 확인 가능하며, JIT 관련 이상(예상보다 느린 처리량)은 JFR(Java Flight Recorder)의 컴파일 이벤트로 분석한다.

### 컴파일 티어와 관측의 경계

OpenJDK 25 HotSpot의 계층형 정책은 항상 인터프리터 → C1 → C2를 한 번씩 통과하는 직선 경로가 아니다. 레벨 0은 인터프리터, 1–3은 프로파일 수집 정도가 다른 C1, 4는 C2다. 호출·루프 횟수와 컴파일 대기열 부하 등에 따라 중간 티어를 건너뛰거나 다른 경로를 선택할 수 있다. 고정된 호출 횟수를 넘으면 반드시 C2로 바뀐다는 설명은 피한다.

`PrintCompilation`에서 OSR 엔트리와 일반 메서드 엔트리를 구분한다. `made not entrant`는 기존 코드로 새로 진입하지 못하게 했다는 표시이며, 상위 티어로 교체될 때도 나와 이것만으로 최적화 가정이 깨진 장애라고 판단할 수 없다. 처리량 저하와 연관된 반복 역최적화인지 컴파일·실행 프로파일을 함께 본다.

### Java 25 AOT 캐시 배포 경계

Java 25에서는 `java -XX:AOTCacheOutput=app.aot -cp app.jar App`으로 훈련 실행과 캐시 생성을 연결하고, 다음 실행은 `-XX:AOTCache=app.aot`를 지정한다. JEP 515의 저장된 프로파일은 운영 중 추가 프로파일링·재최적화를 막지 않는다. [[Spring AOT와 네이티브 이미지]]의 빈 초기화 코드 생성과는 별도 단계다.

캐시는 애플리케이션·JDK·OS/CPU 조합에 맞춰 만든다. 비어 있지 않은 클래스 디렉터리를 classpath에 넣은 개발 실행을 그대로 캐시 생성 명령으로 옮기지 말고 JAR 등 지원되는 패키징을 사용한다. 기본 `AOTMode=auto`는 캐시를 못 읽으면 캐시 없이 계속 실행하므로 기동 성공만으로 적용 여부를 판단하지 않고 `-Xlog:aot`를 확인한다. `AOTMode=on`은 실패 시 종료하는 진단용 선택이다.

JEP 514의 한 단계 명령은 캐시를 조립하는 별도 JVM을 띄운다. 훈련 JVM과 같은 힙 크기가 전달되므로, 두 JVM의 힙과 비힙 메모리를 감당할 생성 환경이 필요하다. 예를 들어 두 JVM 모두 `-Xms4g -Xmx4g`면 힙만 합계 8GB가 필요할 수 있다. 메모리가 빠듯하면 훈련과 생성 단계를 분리하거나 생성 JVM의 `JDK_AOT_VM_OPTIONS`를 검토한다.

## 심화 Q&A

### Q. JIT warm-up 문제를 완화하기 위한 실질적인 방법에는 어떤 것들이 있는가?
근본적으로는 애플리케이션이 실제로 실행되며 프로파일을 쌓아야 하므로 완전히 없앨 수는 없지만, CDS/AppCDS로 클래스 로딩·검증 비용을 줄이고, 배포 파이프라인에서 신규 인스턴스에 트래픽을 점진적으로 유입시켜 실 트래픽 자체를 준비 운동으로 활용하는 방법이 널리 쓰인다. Java 25 AOT 캐시는 로드·링크된 클래스 정보 및 JEP 515의 메서드 프로파일을 다음 실행에 재사용한다. 프로파일이 새 트래픽을 대표하는지 검증해야 하며 Java 25 기능을 네이티브 JIT 코드 캐시로 설명하면 부정확하다.

### Q. 역최적화(Deoptimization)는 어떤 조건에서 발생하며 어떤 비용을 유발하는가?
C2가 "이 호출 지점은 실제로는 항상 특정 구현체 하나만 호출된다"처럼 관찰된 프로파일을 가정으로 인라이닝/타입 추측 최적화를 적용했는데, 이후 실제로 다른 구현체가 나타나 가정이 깨지면 해당 컴파일 코드를 즉시 폐기하고 인터프리터 실행으로 되돌아간다. 이 전환 자체에 비용이 들 뿐 아니라, 폐기된 컴파일 코드는 이후 다시 컴파일 대상이 되어 재컴파일 비용도 발생한다. 다형성이 심한 코드나 팩토리 패턴으로 구현체가 자주 바뀌는 경우 역최적화가 반복되며 성능 저하로 이어질 수 있다.

### Q. Escape Analysis로 인한 할당 제거/락 제거는 실제로 얼마나 신뢰할 수 있는 최적화인가?
Escape Analysis는 JIT 컴파일 시 중간 표현에 수행하는 정적 분석이며 런타임 프로파일과 인라이닝 결과의 영향을 받으므로, 같은 코드라도 실행 경로나 인라이닝 여부에 따라 최적화가 적용될 수도, 안 될 수도 있다. JDK 25 HotSpot C2는 스칼라 치환을 통해 할당을 제거하며 객체를 통째로 스택에 옮기지 않는다. 할당 제거 여부도 코드 작성 시점에 보장할 수 없다. 다만 `synchronized` 블록 내에서 스레드 로컬로 확실히 한정되는 락 객체(예: `StringBuffer` 내부 락)에 대한 락 제거(lock elision)는 비교적 흔하게 관찰되는 최적화다.

### Q. 인터프리터 실행과 JIT 컴파일된 코드 사이의 성능 차이는 근본적으로 어디서 오는가?
인터프리터는 바이트코드 명령 하나하나를 디스패치(dispatch)하며 해석하는 오버헤드가 매 명령마다 반복되고, 타입 체크와 호출 디스패치 비용도 발생한다. 인터프리터도 캐시를 사용할 수 있으므로 모든 조회를 매번 처음부터 한다는 뜻은 아니다. JIT 컴파일된 네이티브 코드는 이런 반복 해석 오버헤드가 없고, 인라이닝·레지스터 할당·루프 최적화 등 실제 실행 프로파일에 기반한 전역적 최적화가 적용되어 있어 같은 로직도 더 적은 명령으로 실행할 수 있다. 성능 배율은 입력과 JDK에 따라 달라 측정해야 한다.

### Q. Tiered Compilation에서 C1과 C2의 역할 분담은 왜 필요한가?
C2만 사용하면 컴파일 자체에 시간이 오래 걸려 애플리케이션 시작 초반의 응답성이 나빠지고, C1만 사용하면 장기적으로 실행되는 서버 애플리케이션에서 얻을 수 있는 깊은 최적화 기회를 놓친다. C1으로 빠르게 1차 컴파일해 초반 성능을 확보하고, 프로파일링 데이터가 충분히 쌓이면 C2로 재컴파일해 장기 처리량을 극대화하는 단계적 접근이 시작 속도와 정상 상태(steady state) 성능을 모두 확보하는 절충안이다.

## 관련 개념
- [[런타임 데이터 영역]]
- [[가비지 컬렉션 알고리즘]]
- [[JVM 구조와 클래스 로더]]
- [[Spring AOT와 네이티브 이미지]]

## 참고 자료

확인 날짜: 2026-09-08. 아래 버전과 범위의 공식 문서·소스로 본문을 대조했다. 성능 비교와 운영 선택은 워크로드에 따라 달라지는 판단 기준이다.

- [HotSpot JDK 25 Performance Enhancements](https://docs.oracle.com/en/java/javase/25/vm/java-hotspot-virtual-machine-performance-enhancements.html) — 계층형 컴파일·code cache·Escape Analysis.
- [JEP 514](https://openjdk.org/jeps/514) — Java 25 AOT 명령.
- [JEP 515](https://openjdk.org/jeps/515) — Java 25 AOT 메서드 프로파일.
- [JDK 25 java](https://docs.oracle.com/en/java/javase/25/docs/specs/man/java.html) — PrintCompilation·컴파일 옵션.

### 2026-09-23 부분 재검증

OpenJDK jdk-25-ga compilationPolicy.cpp의 티어 전이·부하 적응과 JDK 25 HotSpot 성능 문서의 계층형 컴파일·code cache를 확인했다. 기존 전체 검증일은 유지한다.

- [OpenJDK jdk-25-ga compilationPolicy.cpp](https://raw.githubusercontent.com/openjdk/jdk/jdk-25-ga/src/hotspot/share/compiler/compilationPolicy.cpp) — 0–4 레벨 및 가능한 티어 전이.
- [OpenJDK jdk-25-ga compilationPolicy.hpp](https://raw.githubusercontent.com/openjdk/jdk/jdk-25-ga/src/hotspot/share/compiler/compilationPolicy.hpp) — 레벨별 프로파일링과 중간 티어를 건너뛰는 경로 설명.

실행 확인: 2026-09-23, OpenJDK 25.0.2+10-69의 PrintCompilation에서 동일 hot 메서드의 레벨 3 → 4 컴파일 후 기존 코드의 `made not entrant: not used`, OSR 상위 티어 전환의 `OSR invalidation of lower level`을 관찰했다. 해당 문구 자체를 최적화 가정 실패로 단정할 수 없음을 확인했으며 성능 벤치마크는 아니다.

추가 부분 확인: 2026-09-23, [JEP 514](https://openjdk.org/jeps/514)·[JEP 515](https://openjdk.org/jeps/515)의 JDK 25 훈련/생성 흐름·추가 프로파일링·자식 JVM 메모리와 [JDK 25 java](https://docs.oracle.com/en/java/javase/25/docs/specs/man/java.html)의 classpath 제약·AOTMode를 대조했다. OpenJDK 25.0.2에서 비어 있지 않은 클래스 디렉터리의 캐시 생성 실패, JAR 패키징 후 생성 성공과 `Using AOT-linked classes: true` 로그를 확인했다. 기동 시간 비교나 실제 서비스의 캐시 호환성 검증은 아니다.
