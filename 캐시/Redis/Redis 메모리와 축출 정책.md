---
category: "캐시/Redis"
tags: ["cache", "redis", "memory", "eviction", "운영"]
aliases: ["Redis Eviction", "maxmemory"]
updated: 2026-10-04
verified: 2026-09-22
---

# Redis 메모리와 축출 정책

## 핵심 정의

키 축출(eviction)은 Redis가 설정한 메모리 한도에 맞추기 위해 키를 제거하는 동작이다. 유효 기간(Time To Live, TTL)이 끝나 제거되는 만료(expiration)와 다르다. TTL이 남아 있거나 TTL이 없는 키도 정책에 따라 축출될 수 있으므로 캐시 조회 실패는 언제든 처리해야 한다.

`maxmemory`는 키 축출 판단에 사용하는 메모리 한도다. 프로세스의 실제 상주 메모리(Resident Set Size, RSS)나 컨테이너 메모리 상한과 같지 않다. 아래 정책·기본값은 **Redis Open Source 8.4**의 `redis.conf`와 공식 명령 문서를 기준으로 한다. 관리형 제품의 기본값은 별도로 확인한다.

## 동작 원리 / 구조

### 정책의 대상과 실패 방식

| 정책 | 제거 후보 | 선택 기준·주의점 |
|---|---|---|
| `noeviction` | 없음 | 기본 정책. 한도 초과 시 메모리를 더 필요로 하는 쓰기가 오류를 반환하며 읽기는 가능 |
| `allkeys-lru` / `allkeys-lfu` | TTL 유무와 무관한 키 | 최근 사용 시점 / 사용 빈도를 근사해 선택 |
| `volatile-lru` / `volatile-lfu` | TTL이 설정된 키 | 해당 후보 안에서 LRU / LFU 적용 |
| `allkeys-random` / `volatile-random` | 전체 키 / TTL이 있는 키 | 무작위 선택 |
| `volatile-ttl` | TTL이 설정된 키 | 남은 TTL이 짧은 후보를 근사해 선택 |

LRU(Least Recently Used)와 LFU(Least Frequently Used)는 모든 키를 매번 완전 정렬하는 방식이 아니다. 샘플링과 근사 정보로 CPU·메모리 비용을 줄인다. `volatile-*`에서도 제거할 적합한 키가 없으면 메모리가 필요한 쓰기가 실패할 수 있다. 정책을 설정했다는 이유만으로 쓰기 성공이 보장되지는 않는다.

### maxmemory 아래에서도 OOM이 발생하는 이유

축출 판단에서 제외되는 일부 복제·AOF 버퍼가 있으며 `INFO memory`의 `mem_not_counted_for_evict`로 확인한다. 여기에 allocator가 보유한 페이지, 프로세스 오버헤드, 영속화 작업 중 쓰기 시 복사(Copy-on-Write, CoW) 비용이 더해진다. 따라서 물리·컨테이너 한도까지 `maxmemory`로 채우면 축출보다 먼저 운영체제의 메모리 부족(Out Of Memory, OOM)에 노출될 수 있다.

Redis 8.4의 기본 복제본은 `replica-ignore-maxmemory yes`로 자신의 한도에 맞춰 독자적으로 키를 축출하지 않고 원본이 보낸 제거를 따른다. 원본과 복제본의 버퍼·메모리 사용량이 같다고 가정하지 말고 양쪽 여유를 관측한다.

### 관측 명령

```text
INFO memory
INFO stats
INFO persistence
MEMORY USAGE cache:product:123 SAMPLES 5
```

| 지표 | 해석 |
|---|---|
| `used_memory`, `used_memory_rss` | Redis allocator 할당량과 OS가 보는 RSS를 구분 |
| `mem_not_counted_for_evict` | 축출 한도 계산에서 제외한 메모리 |
| `mem_fragmentation_bytes`, `allocator_frag_bytes` | 비율만 보지 않고 실제 추가 바이트·allocator 단편화를 구분 |
| `evicted_keys`, `expired_keys` | 메모리 정책에 의한 축출과 TTL 만료의 누적 건수; 일정 구간 증가량으로 관측 |
| `keyspace_hits`, `keyspace_misses` | Redis 키 조회의 적중·미스; 애플리케이션 업무 성공률과는 별개 |
| `current_cow_size`, `current_cow_peak` | 자식 프로세스가 동작하는 동안의 CoW 메모리 |

`MEMORY USAGE`는 값의 문자열 길이만이 아니라 키·자료구조의 관리 비용도 포함한다. 중첩 자료구조는 기본적으로 일부 항목을 샘플링해 추정한다. `SAMPLES 0`은 전수 조사하므로 큰 컬렉션에 반복 실행하는 비용을 고려한다.

## 실무 관점

- **캐시와 필수 상태를 구분한다.** 재생성 가능한 조회 캐시에는 축출을 허용할 수 있다. 분산 락, 중복 처리 표식, 세션, 큐를 같은 인스턴스에 넣고 `allkeys-*`를 쓰면 아직 유효한 필수 상태가 제거될 수 있다. 장애 영향이 다르면 인스턴스·한도를 나누고 각 유실 허용 범위를 설계한다. 논리 DB 번호를 나누는 것만으로 인스턴스의 메모리 정책이 분리되지는 않는다.
- **noeviction도 오류 경로가 필요하다.** 락·큐 키를 축출하지 않아도 메모리 부족 때 신규 등록이 실패한다. 애플리케이션이 쓰기 오류를 무시하면 락 획득·메시지 보관이 성공했다고 오판한다.
- **축출 급증을 DB 부하와 연결한다.** 적중률 하락 → 원본 조회 증가 → 지연 → 재시도 증가가 이어질 수 있다. 용량 확대뿐 아니라 키 크기·입력 제한·TTL·동시 재적재 제한을 함께 검토한다.
- **큰 키부터 원인을 나눈다.** 직렬화 크기, 자료구조 원소 수, 키 수 증가 중 무엇이 메모리를 쓰는지 본다. `MEMORY USAGE` 표본을 업무 키 종류별로 모으면 평균·상위 크기를 비교하기 쉽다.
- **운영 한도는 실측으로 정한다.** 평상시뿐 아니라 full resync, BGSAVE, AOF rewrite, 트래픽 급증 시의 RSS·CoW·버퍼 최대치를 보고 여유를 남긴다. 고정 비율 하나를 모든 인스턴스에 적용하지 않는다.

### WATCH 재시도도 만료·축출의 영향을 받는다

Redis의 `WATCH`는 다른 클라이언트의 명시적 쓰기만 감시하지 않는다. 감시한 키가 만료되거나 축출되어도 이후 `EXEC`가 null 응답으로 중단될 수 있다. 만료가 이 중단 조건에 포함되는 것은 Redis 6.0.9 이후이며, 8.10.2 소스에서도 EXEC 직전 만료 검사를 확인할 수 있다. 애플리케이션 쓰기 경쟁이 없다는 이유만으로 재시도 원인을 찾지 못하는 경우를 피해야 한다.

중단된 시도는 업무 성공으로 처리하지 않는다. 다시 WATCH하고 값을 읽어 현재 상태에서 계산하며, 오래전에 읽은 값을 그대로 재전송하지 않는다. TTL이 계산 시간보다 짧거나 메모리 압력으로 자주 축출된다면 재시도만 늘리기보다 상태 수명·인스턴스 분리·작업 시간과 재시도 예산을 확인한다. EXEC의 null(조건 실패)과 실행 결과 배열 안의 명령 오류는 다른 경로다. 실행 중 오류의 롤백 부재는 [[싱글 스레드 모델과 이벤트 루프]]에서 다룬다.

## 심화 Q&A

### Q. TTL을 충분히 짧게 설정했는데 왜 쓰기가 실패하는가?
A. 유입량이 만료·축출 속도보다 크거나, `volatile-*`에서 TTL 없는 키가 대부분이면 적합한 제거 후보가 부족할 수 있다. TTL은 메모리 용량 예약이 아니다. 실제 정책, TTL 누락, 큰 키와 버퍼 사용량을 확인한다.

### Q. Redis 메모리는 줄었는데 RSS가 줄지 않는 것은 누수인가?
A. 해제한 메모리를 allocator가 재사용하려고 보유할 수 있다. `used_memory`와 RSS 차이만으로 누수를 단정하지 말고 `allocator_frag_bytes`, 프로세스 오버헤드와 시간 추세를 본다. 작은 인스턴스는 적은 절대 바이트 차이에도 단편화 비율이 크게 보인다.

### Q. LFU가 LRU보다 항상 좋은가?
A. 반복적으로 인기 있는 키를 보존할 때 LFU가 유리할 수 있지만 접근 패턴이 급변하면 과거 빈도가 새로운 인기를 반영하는 속도가 중요해진다. 같은 메모리 예산에서 적중률·축출량·명령 지연을 비교한다. 정책 이름만으로 우열을 결정하지 않는다.

### Q. 락 키에 TTL이 있으면 volatile 정책에서도 안전한가?
A. 오히려 `volatile-*`의 제거 후보가 된다. TTL이 남은 락이 축출되면 다른 작업자가 새 락을 얻는 동안 이전 작업자가 계속 실행할 수 있다. 인스턴스 분리·쓰기 오류 처리와 함께 [[분산 락]]의 fencing·업무 멱등성을 검토한다.

### Q. 복제본도 원본과 같은 maxmemory를 설정하면 충분한가?
A. 기본 복제본은 독립 축출을 하지 않으며 복제 버퍼와 자료구조 할당량도 다를 수 있다. 원본의 정상 상태만 보고 복제본 용량을 맞추면 재동기화 때 OOM이 날 수 있다. 승격 이후 적용될 정책까지 포함해 메모리 여유와 오류 처리를 확인한다.

## 관련 개념
- [[TTL과 캐시 무효화 전략]]
- [[Redis 자료구조]]
- [[영속화 RDB와 AOF]]
- [[Redis Cluster]]
- [[캐시 스탬피드]]
- [[분산 락]]

## 참고 자료

확인일: 2026-09-22. Redis Open Source 8.4의 정책·복제본 기본 동작을 소스 설정과 대조하고 공식 명령 문서에서 지표·샘플링 의미를 확인했다. Redis 서버에서 명령을 실행하지 않았다.

- [Redis 8.4 redis.conf](https://raw.githubusercontent.com/redis/redis/8.4/redis.conf) — 지원 정책, 기본 `noeviction`, 근사 알고리즘, `replica-ignore-maxmemory`.
- [Key Eviction](https://redis.io/docs/latest/develop/reference/eviction/) — 축출 판단·제외 버퍼·정책 선택. 이 노트는 8.4 정책 범위만 사용한다.
- [INFO](https://redis.io/docs/latest/commands/info/) — allocator·RSS·축출/만료·CoW 지표.
- [MEMORY USAGE](https://redis.io/docs/latest/commands/memory-usage/) — 관리 비용 포함과 `SAMPLES`의 추정·전수 조사.
- [Redis Replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/#maxmemory-on-replicas) — 복제본 독립 축출을 기본적으로 하지 않는 이유.

부분 재검증: 2026-10-04. WATCH의 만료·축출 중단 계약과 Redis 8.10.2 EXEC 구현을 확인했다. 격리 Redis 6.2.11에서 감시 키의 TTL 만료 후 EXEC가 null을 반환하고 예정된 쓰기는 없음을 재현했다. 새 값을 읽어 재시도한 대조 실험은 성공했다. 강제 eviction·Redis 8.10.2 서버 실행은 하지 않았고 본문의 8.4 정책·기본값 `verified`는 유지한다.

- [Redis Transactions: WATCH Explained](https://redis.io/docs/latest/develop/using-commands/transactions/#watch-explained) — 만료·축출 조건과 6.0.9 이전 차이, 실행 오류와 CAS 실패 구분.
- [Redis 8.10.2 multi.c](https://github.com/redis/redis/blob/8.10.2/src/multi.c) — EXEC의 isWatchedKeyExpired·CLIENT_DIRTY_CAS와 null 응답.
