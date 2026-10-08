---
category: "Spring/웹 심화"
tags: ["spring", "http-client", "webclient", "timeout", "retry", "안정성"]
aliases: ["HTTP 타임아웃", "재시도 예산"]
updated: 2026-09-22
verified: 2026-09-22
---

# HTTP 호출 타임아웃과 재시도 예산

## 핵심 정의

타임아웃(Timeout)은 특정 대기 단계의 허용 시간이며, 데드라인(Deadline)은 작업 전체가 끝나야 하는 시점이다. 연결 제한 하나만으로 풀 대기·DNS·TLS·응답 수신·재시도 전체 시간이 제한되지는 않는다.

재시도 예산(Retry Budget)은 요청 하나 또는 서비스 전체에서 재시도에 사용할 수 있는 횟수·시간·부하의 한도다. 실패율이 높은 상황에서 재시도를 늘리면 복구 중인 서버의 부하도 함께 늘어난다. 호출자의 남은 시간과 중복 실행 안전성을 먼저 정한다.

## 동작 원리 / 구조

### 제한을 거는 위치

| 구간 | 제한 대상 | 다른 제한으로 대체할 수 없는 이유 |
|---|---|---|
| 풀 획득 | 사용 가능한 연결을 기다리는 시간 | 이미 연결된 풀이라 TCP 연결 제한이 작동하지 않을 수 있음 |
| DNS·TCP·TLS | 주소 해석·연결·보안 핸드셰이크 | 각각의 제한 범위와 시작 시점이 다름 |
| 응답 읽기 | 응답 도착 또는 데이터 읽기 사이의 대기 | 데이터가 조금씩 계속 오면 전체 수신은 오래 걸릴 수 있음 |
| 시도별 제한 | 한 번의 호출을 완료할 때까지 | 재시도할 때마다 타이머가 새로 시작할 수 있음 |
| 전체 제한 | 재시도와 백오프를 포함한 논리 요청 | 상위 요청 처리·응답 직렬화 시간도 별도 여유가 필요함 |

Reactor Netty 1.3.7의 `responseTimeout(Duration)`은 요청 전송 후 응답의 **네트워크 읽기 사이 간격**을 제한한다. 응답 전체 다운로드 시간이나 풀 획득 시간을 뜻하지 않는다. 해당 설정은 기본 미설정이며, TCP 연결 제한 기본 30초·풀 획득 대기 기본 45초와 구별한다. 기본값은 목표 지연 시간에 맞춰 조정한다.

`bodyToMono(...)`는 응답을 하나의 결과로 변환하므로 그 뒤의 `Mono.timeout(...)`으로 결과 대기를 제한할 수 있다. 반면 스트리밍 `Flux.timeout(...)`은 다음 항목 사이의 대기 제한이므로 스트림 전체 수명을 같은 방식으로 제한한다고 해석하면 안 된다.

### 연산자 순서와 예산

다음은 이미 구성된 `WebClient`로 멱등한 조회를 수행하는 예다. 숫자는 설명용이며, 상대 서버의 계약과 지연 분포에 맞춰 정한다.

```java
Mono<String> response = webClient.get()
    .uri("/inventory/{id}", productId)
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofMillis(700)) // 한 시도의 결과 대기
    .retryWhen(Retry.backoff(2, Duration.ofMillis(100))
        .maxBackoff(Duration.ofMillis(200))
        .jitter(0.5)
        .filter(error -> error instanceof WebClientResponseException ex
            && (ex.getStatusCode().value() == 502
                || ex.getStatusCode().value() == 503
                || ex.getStatusCode().value() == 504)))
    .timeout(Duration.ofMillis(2500)); // 재시도·백오프를 포함한 전체 대기
```

`Retry.backoff(2, ...)`는 최초 시도에 더해 최대 두 번 재시도한다. 위 예제는 상태 코드 오류만 재시도하며 타임아웃 예외까지 재시도하도록 만들지는 않았다. 재시도 조건을 넓힐 때도 읽기 요청인지, 같은 작업을 다시 보내도 되는지 판단해야 한다. `Retry-After`가 있는 응답은 그 의미와 남은 예산을 반영하는 정책을 별도로 구현한다.

전체 `timeout`을 `retryWhen` 앞에만 두면 재구독마다 제한 시간이 다시 시작한다. 논리 요청 전체를 제한하려면 재시도 연산자의 바깥쪽에 제한을 둔다. 타이머는 구독 시 시작하므로 서비스 진입부터 이미 소비한 시간을 포함하려면 남은 예산을 계산해 전달한다.

## 실무 관점

- **중복 실행**: 응답을 못 받았어도 원격 서버가 커밋했을 수 있다. 주문·결제·상태 변경은 [[멱등성 키 설계]]처럼 중복 요청을 구별할 계약이 필요하다.
- **재시도 위치**: 게이트웨이·서비스·SDK가 각자 재시도하면 시도 수가 곱해질 수 있다. 기본 자동 재시도까지 포함해 한 요청의 실제 시도 수를 측정한다. Reactor Netty도 특정 연결 reset에 한 번 재시도하는 동작을 제공한다.
- **부하 제한**: 풀 크기·대기열·동시 호출 제한을 함께 둔다. 모든 요청이 즉시 재시도하면 실패한 서버에 부하가 집중되므로 백오프(Backoff)와 지터(Jitter)를 사용한다.
- **취소의 범위**: 로컬 구독 취소는 원격 트랜잭션 롤백 명령이 아니다. 호출자가 포기한 뒤에도 원격 작업이 완료될 수 있다. 완료 여부 조회와 재조정 경로를 마련한다.
- **관측**: 풀 대기·연결·응답 지연·시도 횟수·전체 지연을 나눠 본다. 최종 성공률만 보면 재시도가 가린 성능 저하를 놓친다.
- **구현별 차이**: RestClient도 같은 설계 원칙을 따르지만 요청 팩토리별 설정 API와 취소 효과가 다르다. Reactor의 연산자 이름을 동기 클라이언트의 설정값에 그대로 대응시키지 않는다.

## 심화 Q&A

### Q. 읽기 제한을 1초로 걸었는데 다운로드가 10초 넘게 지속될 수 있는가?
A. 읽기 간격을 제한하는 구현에서는 가능하다. 서버가 1초보다 짧은 간격으로 데이터를 계속 보내면 제한에 걸리지 않는다. 큰 응답의 전체 수신 시간을 제한하려면 집계 결과의 제한 시간 또는 별도의 데드라인이 필요하다.

### Q. 재시도 대상을 모든 5xx와 타임아웃으로 지정하면 충분한가?
A. 오류 코드만으로 실패 시점이나 부수 효과를 알 수 없다. 영구 오류인지, 작업이 이미 반영됐는지, 본문을 다시 전송할 수 있는지, 남은 예산이 있는지를 함께 본다. 멱등한 메서드라도 서버 계약과 클라이언트의 본문 재생 가능성을 확인한다.

### Q. 전체 제한이 시도별 제한보다 짧아도 되는가?
A. 가능하다. 전체 제한이 먼저 끝나면 현재 시도와 뒤따를 재시도를 포기한다. 다만 대부분의 요청이 시도별 제한에 도달하기도 전에 중단된다면 설정 의도를 다시 확인한다. 하위 호출에 상위 요청의 전체 시간을 모두 배정하지 않는다.

### Q. 타임아웃 예외를 잡아 즉시 다시 호출하면 어떤 문제가 생기는가?
A. 원격 작업이 아직 실행 중이면 같은 작업이 겹치고, 풀 대기가 원인이면 대기 요청만 더 늘어난다. 실패를 숨기기보다 제한된 재시도·동시성 제한·명시적인 실패 응답을 조합한다. 필요한 경우 [[서킷 브레이커와 장애 격리]]와 함께 적용한다.

## 관련 개념
- [[WebClient와 RestTemplate]]
- [[Spring Retry와 재시도 전략]]
- [[멱등성 키 설계]]
- [[Bulkhead 패턴]]
- [[서킷 브레이커와 장애 격리]]
- [[인터럽트와 작업 취소]]

## 참고 자료

검증일: 2026-09-22. 적용 범위: Spring Framework 7.0.9, Reactor Netty 1.3.7, Reactor Core 3.8.7, HTTP 의미론 RFC 9110. 로컬 HTTP 서버의 느린 청크 응답과 Reactor 가상 시간 테스트로 읽기 간격·전체 제한·재시도 위치의 차이를 확인했다. 본문 예제를 추출해 502·503·504의 재시도, 500·타임아웃의 미재시도, 재시도 소진 시 원인 예외 보존도 검사했다. 외부 서비스의 처리 취소는 보장 범위에 포함하지 않는다.

- [Reactor Netty HTTP Client](https://projectreactor.io/docs/netty/release/reference/http-client.html#timeout-configuration) — 연결·풀·응답 제한의 범위와 기본값.
- [HttpClient API](https://projectreactor.io/docs/netty/release/api/reactor/netty/http/client/HttpClient.html) — responseTimeout과 disableRetry.
- [Mono API](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html) / [Flux API](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html) — timeout과 재구독.
- [RetryBackoffSpec](https://projectreactor.io/docs/core/release/api/reactor/util/retry/RetryBackoffSpec.html) — 횟수·필터·백오프·지터.
- [RFC 9110 §9.2.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2) — 멱등성과 자동 재시도 조건.
