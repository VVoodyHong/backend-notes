---
category: "컴퓨터 과학/네트워크 프로그래밍 기초"
tags: ["computer-science", "network", "io멀티플렉싱", "epoll", "논블로킹"]
aliases: ["I/O 멀티플렉싱", "I/O Multiplexing"]
updated: 2026-09-23
verified: 2026-09-08
---

# I/O 멀티플렉싱 (select/poll/epoll)

## 핵심 정의
I/O 멀티플렉싱(I/O multiplexing)은 하나의 스레드가 여러 개의 파일 디스크립터(file descriptor, 대부분 [[소켓 프로그래밍 기초]]로 만든 소켓)를 동시에 감시하다가, 그중 실제로 읽기/쓰기가 가능해진 것만 골라 처리하는 기법이다. `select`, `poll`, `epoll`은 모두 이 목적을 위한 시스템 콜(system call)이며, 순서대로 등장해 확장성 문제를 개선해 온 계보를 이룬다.

이 기법이 없다면 커넥션마다 스레드를 하나씩 붙이거나(thread-per-connection), 매 커넥션을 순회하며 상태를 직접 확인(busy polling)해야 한다. I/O 멀티플렉싱은 "커널에게 감시를 맡기고, 준비된 것만 통지받는다"는 방식으로 이 문제를 해결해 소수의 스레드로 대량의 커넥션을 처리하는 이벤트 기반(event-driven) 서버의 기반이 된다.

## 동작 원리 / 구조

### select

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

- 감시할 fd 집합을 비트마스크(`fd_set`)로 커널에 전달하고 블로킹된다.
- 이벤트가 발생하면 커널은 준비된 fd 정보를 같은 `fd_set`에 덮어써서 반환한다. 애플리케이션은 매번 fd 집합을 재구성해야 한다.
- 한계: Linux glibc fd_set은 fd 번호가 FD_SETSIZE(1024) 미만이어야 하고, 매 호출마다 fd 집합 전체를 커널 공간으로 복사(O(n))해야 하며, 커널도 매번 전체 집합을 선형 탐색(O(n))한다.

### poll

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

- `fd_set` 비트마스크 대신 `pollfd` 구조체 배열을 사용해 fd 개수 제한(`FD_SETSIZE`)이 사라진다.
- 그러나 매 호출마다 배열 전체를 커널로 복사하고 커널이 선형 탐색하는 구조는 select와 동일해, fd 수가 많아지면 여전히 O(n) 비용이 든다.

### epoll (Linux)

epoll은 "관심 등록"과 "이벤트 대기"를 분리해 select/poll의 매 대기마다 전체 관심 집합을 전달하는 비용을 줄인다.

```c
int epfd = epoll_create1(0);
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &event);  // 한 번만 등록
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout); // 준비된 것만 반환
```

```mermaid
flowchart TB
    subgraph "select/poll (매 호출마다 반복)"
        A1[전체 fd 목록을 커널로 복사] --> A2[커널이 전체 순회하며 상태 확인 O n]
        A2 --> A3[결과를 사용자 공간에 복사]
        A3 -->|다음 호출| A1
    end
    subgraph "epoll (등록은 1회, 대기만 반복)"
        B1[epoll_ctl로 fd 관심사 등록 - 1회] --> B2[커널 내부에 red-black tree로 관리]
        B2 --> B3[소켓상태 변화 등에 따라 ready list 갱신]
        B3 --> B4[epoll_wait 호출 - ready list만 반환 O ready]
        B4 -->|다음 호출| B4
    end
```

- 커널 내부에서 등록된 fd를 red-black tree로 관리하고, 각 fd에 콜백을 걸어둔다. 네트워크 카드 인터럽트 등으로 데이터가 도착하면 해당 fd를 준비 완료 리스트(ready list)에 추가한다.
- `epoll_wait()`는 이 ready list만 반환하므로 시간 복잡도가 감시 대상 전체 개수가 아니라 실제로 준비된 개수(O(ready))에 비례한다. 유휴 연결이 많은 대규모 집합에서 이점이 크지만 항상 더 빠르거나 API가 정확한 복잡도를 보장하는 것은 아니다.

### Level-Triggered vs Edge-Triggered

| 모드 | 동작 | 특징 |
|---|---|---|
| Level-Triggered(LT, 기본값) | 버퍼에 읽을 데이터가 남아있는 한 매번 `epoll_wait()`가 통지 | 구현이 쉽고 데이터 일부만 읽어도 안전(다음 호출에서 또 통지됨), select/poll과 동일한 의미 |
| Edge-Triggered(ET, `EPOLLET`) | 준비 상태 관련 변화에 통지; 이벤트가 병합될 수 있음 | 일반적으로 EAGAIN까지 처리. 중간에 양보하면 준비 상태를 자체 보존해 이어 처리해야 하며 새 통지만 기다리면 안 됨 |

Edge-Triggered에서는 소켓을 논블로킹(non-blocking)으로 설정하고 `read()`가 `EAGAIN`/`EWOULDBLOCK`을 반환할 때까지 처리하는 방식이 일반적이다. 공정성을 위해 중간에 양보한다면 애플리케이션의 준비 목록에 남겨 재처리한다. 데이터를 남긴 채 준비 상태까지 잊고 다음 통지만 기다리면 데이터 누락처럼 보이는 버그가 발생한다.

### Thundering Herd와 EPOLLEXCLUSIVE

여러 스레드/프로세스가 같은 리스닝 소켓을 여러 epoll 인스턴스에 등록해 감시하면, 연결 하나가 들어올 때 모든 스레드가 깨어났다가 단 하나만 `accept()`에 성공하고 나머지는 헛되이 깨어나는 현상(thundering herd)이 발생할 수 있다. 리눅스 4.5부터 도입된 `EPOLLEXCLUSIVE` 플래그는 이벤트 발생 시 등록된 대기자 중 일부만(과도한 동시 웨이크업 없이) 깨워 이 문제를 완화한다. `SO_REUSEPORT`와 스레드별 epoll 인스턴스를 조합하는 구성도 흔히 쓰인다.

readable에는 데이터뿐 아니라 EOF·오류도 포함되며 write-ready가 전체 메시지 전송 완료를 보장하지 않는다. ET는 nonblocking read/write를 EAGAIN까지 진행하되 EOF·에러·partial I/O를 처리한다. 큰 입력 하나가 루프를 독점하지 않도록 읽기 예산과 사용자 공간 ready queue를 둘 수 있다. EPOLLONESHOT은 처리 후 MOD로 재활성화해야 하고 EPOLLEXCLUSIVE는 허용 flag·ADD 전용 등 제약이 있다. 예제는 event 초기화·반환값/errno·close를 생략한 API 흐름이다.

## 실무 관점
- **언제 select/poll을 여전히 쓰는가**: select/poll은 이식성(portability)이 높아 macOS/BSD 등 epoll이 없는 환경을 함께 지원해야 하는 저수준 라이브러리에서 여전히 쓰인다. macOS/BSD는 epoll 대신 유사한 역할의 `kqueue`를 제공한다. Java NIO의 `Selector`는 JDK/플랫폼에 맞는 SelectorProvider 구현을 사용한다(자세한 매핑은 [[Java NIO]] 참고).
- **왜 직접 다룰 일이 드문가**: 대부분의 백엔드 개발에서는 Netty, Tomcat NIO 커넥터, Nginx, Redis 같은 프레임워크/미들웨어가 이미 epoll 기반 이벤트 루프(event loop)를 내장하고 있어 애플리케이션 코드가 `epoll_wait()`를 직접 호출할 일은 거의 없다. 다만 "왜 스레드 하나로 이렇게 많은 커넥션을 처리할 수 있는가", "왜 CPU 사용률이 낮은데 레이턴시가 튀는가" 같은 질문에 답하려면 이 내부 동작 이해가 필요하다.
- **흔한 실수/장애 패턴**: 이벤트 루프 스레드(Netty의 `EventLoop` 등) 안에서 블로킹 호출(동기 DB 쿼리, 블로킹 I/O, `Thread.sleep`)을 실행하면 그 순간 해당 스레드가 감시하던 모든 커넥션의 이벤트 처리가 지연된다. 이는 "이벤트 루프를 절대 블로킹하지 마라"는 리액티브 프로그래밍의 핵심 규칙이자, 실무에서 리액티브 스택 성능 저하의 가장 흔한 원인이다. [[Spring WebFlux와 리액티브 스트림]] 참고.
- **Edge-Triggered 오용**: ET 모드에서 읽을 데이터가 남았는데도 처리 대상에서 빼고 새 통지만 기다리면 커넥션이 멈춘 것처럼 보일 수 있다. 일반적으로 EAGAIN까지 처리하거나, 준비 상태를 애플리케이션 내부에 보존해 다음 턴에 이어 처리한다. 프레임워크는 자체의 LT/ET·읽기 예산·재등록 정책을 사용하므로 설정과 구현을 확인한다.
- **관련 설정/튜닝**: 파일 디스크립터 한도(`ulimit -n`), 실제 UID별 전체 epoll 등록 상한 max_user_watches, `EPOLLEXCLUSIVE`/`SO_REUSEPORT` 조합으로 멀티코어 accept 분산, `strace -e epoll_wait`로 이벤트 루프의 대기/처리 패턴 관찰.

## 심화 Q&A

### Q. epoll이 select/poll보다 빠른 근본적인 이유를 한 문장 이상으로 설명한다면?
A. select/poll은 매 호출마다 "감시할 전체 목록"을 사용자-커널 공간 간에 복사하고 커널이 그 전체를 순회하며 상태를 확인하는 방식(O(n), n = 감시 대상 전체)이라 fd 수가 늘수록 매 호출 비용이 선형으로 증가한다. epoll은 관심 등록(`epoll_ctl`)을 최초 한 번만 하고 커널 내부에 fd별 콜백을 걸어두어, 이벤트가 실제로 발생한 fd만 별도의 ready list에 쌓아둔다. `epoll_wait()`는 이 ready list만 반환하므로 비용이 준비된 이벤트 수(O(ready))에 비례하고, 전체 관심 집합 복사를 피한다. 등록·갱신·락경합·준비 상태 재검사 비용도 남는다.

### Q. Level-Triggered와 Edge-Triggered 중 어느 것을 기본으로 선택해야 하는가?
A. 대부분의 범용 애플리케이션은 Level-Triggered가 안전하다. 데이터를 일부만 읽고 다음 이벤트 루프 턴으로 넘어가도 커널이 "아직 읽을 데이터가 남아있다"고 계속 알려주므로 로직이 단순해진다. Edge-Triggered는 통지 횟수를 줄일 수 있지만, EAGAIN까지 처리하거나 미처리 준비 상태를 자체 목록에 유지하는 등 구현 부담과 버그 가능성이 커서, 그 복잡성을 감당할 명확한 성능 이유가 있을 때 선택하는 것이 합리적이다.

### Q. epoll 기반 이벤트 루프에서 워커 스레드가 하나뿐인데 왜 CPU 코어를 여러 개 쓰는 서버보다 특정 워크로드에서 더 유리할 수 있는가?
A. 요청 처리 자체가 I/O 대기가 대부분이고 CPU 연산이 짧은 워크로드(단순 프록시, 캐시 조회 등)에서는 스레드 간 컨텍스트 스위칭과 락 경합(lock contention) 비용이 오히려 처리량을 깎아먹는다. 단일 스레드 이벤트 루프는 이런 경합 자체가 없어 캐시 지역성(cache locality)이 좋고 오버헤드가 적다. Redis, Nginx의 워커 프로세스, Node.js가 이 모델을 채택한 이유다. 다만 CPU 바운드 연산이 섞이면 이벤트 루프가 블로킹되므로, 그런 워크로드는 별도 스레드 풀로 분리하거나 멀티 이벤트 루프(코어당 1개)로 확장한다.

### Q. epoll의 red-black tree 구조는 왜 하필 이 자료구조를 선택했는가?
A. epoll은 `epoll_ctl(ADD/MOD/DEL)`로 fd를 매우 빈번하게 추가/수정/삭제하면서, 동시에 특정 fd가 이미 등록되어 있는지 빠르게 조회해야 한다. red-black tree는 삽입/삭제/탐색이 모두 O(log n)으로 균형 잡혀 있어, 수만 개의 fd가 드나드는 상황에서도 등록 관리 비용이 완만하게 증가한다. 이벤트 통지 자체는 별도의 연결 리스트(ready list)로 관리해 통지 비용과 등록 관리 비용의 역할을 분리한 것도 설계상 핵심이다.

### Q. io_uring은 epoll을 완전히 대체하는가?
A. 아니다. epoll은 "준비 상태를 통지"할 뿐 실제 `read()`/`write()`는 별도의 시스템 콜로 수행해야 한다. io_uring은 읽기/쓰기 요청 자체를 커널과 공유하는 링 버퍼에 제출하고 완료를 비동기로 통지받는 방식이라 통지와 실행을 통합한다. 다수의 소규모 I/O가 몰리는 워크로드에서 시스템 콜 횟수를 크게 줄이는 이점이 있지만, 생태계 성숙도(라이브러리 지원)와 커널 버전 요구사항 때문에 네트워크 서버 영역에서 epoll을 즉시 대체하기보다 당분간 공존하는 추세다. 자세한 배경은 [[시스템 콜과 인터럽트]] 참고.

### Q. 리스닝 소켓 하나를 여러 스레드의 epoll 인스턴스에 등록했을 때 왜 thundering herd가 발생하고, 어떻게 완화하는가?
A. 연결 하나가 도착하면 커널은 그 소켓을 감시 중인 모든 epoll 인스턴스에 이벤트를 통지하는데, 실제 `accept()`에 성공할 수 있는 것은 하나뿐이라 나머지 스레드는 깨어났다가 헛수고만 하고 다시 잠든다. 이 깨움-실패 반복이 트래픽이 몰릴수록 CPU 낭비로 이어진다. 리눅스 4.5+에서는 `EPOLLEXCLUSIVE` 플래그로 이벤트 발생 시 깨어나는 대기자 수를 제한할 수 있고, 더 널리 쓰이는 대안은 `SO_REUSEPORT`로 스레드/프로세스마다 별도의 리스닝 소켓을 만들어 커널이 애초에 연결을 분산 배정하게 하는 방식이다.

### Q. ET에서 EAGAIN까지 한 소켓을 계속 읽으면 공정한가?
A. 대량 데이터를 계속 보내는 연결 하나가 이벤트 루프를 독점할 수 있다. Linux epoll(7)은 애플리케이션의 준비 목록(ready list)에 아직 처리할 fd를 남겨 두고 순환 처리하는 방법을 설명한다. 연결별 바이트·작업 예산에 도달해 양보하더라도 준비 상태를 잊지 않아야 한다. "매번 끝까지 처리"와 "새 edge만 기다리기" 사이에 재스케줄링 정책이 필요하다.

### Q. EPOLLONESHOT을 쓰면 처리 이후에도 자동으로 다시 통지되는가?
A. 아니다. 한 번 통지한 뒤에는 `epoll_ctl(EPOLL_CTL_MOD)`로 다시 활성화(rearm)해야 한다. 워커에게 처리를 넘기는 구조에서 동시 처리를 제한할 수 있지만, 오류·부분 처리 경로에서 rearm을 누락하면 연결이 멈춘다. 등록된 오류·종료 이벤트와 아직 남은 수신 데이터도 구분해 처리한다.

## 관련 개념
- [[소켓 프로그래밍 기초]]
- [[Java NIO]]
- [[Spring WebFlux와 리액티브 스트림]]
- [[시스템 콜과 인터럽트]]

## 참고 자료

- [Linux select(2)](https://man7.org/linux/man-pages/man2/select.2.html) — glibc FD_SETSIZE·준비상태. 확인: 2026-09-08.
- [Linux epoll(7)](https://man7.org/linux/man-pages/man7/epoll.7.html) — LT/ET·ready list·per-UID watches. 확인: 2026-09-08.
- [Linux epoll_ctl(2)](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html) — EPOLLEXCLUSIVE4.5+·등록제약. 확인: 2026-09-08.
- [Linux io_uring_setup(2)](https://man7.org/linux/man-pages/man2/io_uring_setup.2.html) — completion model. 확인: 2026-09-08.

### 2026-09-23 부분 재검증

Linux man-pages 6.19의 epoll(7) ET 기아 방지·준비 목록과 epoll_ctl(2)의 EPOLLONESHOT 재활성화 계약을 확인했다. 기존 전체 검증일은 유지한다.
