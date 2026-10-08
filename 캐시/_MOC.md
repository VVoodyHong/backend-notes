# 캐시

캐시 전략, Redis, 분산 캐시 이슈를 다루는 노트 모음.

## 추천 읽기 순서

1. [[Cache-Aside와 Write-Through-Behind]] → [[TTL과 캐시 무효화 전략]] → [[캐시와 DB 정합성]]
2. [[Redis 자료구조]] → [[싱글 스레드 모델과 이벤트 루프]] → [[Redis 메모리와 축출 정책]] → [[영속화 RDB와 AOF]]
3. [[Redis Sentinel]] → [[Redis Cluster]] → [[Redis Pub-Sub와 Streams]]
4. [[캐시 스탬피드]] → [[로컬 캐시와 멀티레벨 캐시]] → [[Hot Key 대응 전략]] → [[캐시 워밍 전략]]
5. [[분산 락]]을 읽고 정합성·멱등성 관련 노트와 적용 범위를 비교한다.

## 캐시 전략
Cache-Aside/Write-Through-Behind, TTL과 캐시 무효화, 스탬피드, 로컬·멀티레벨 캐시를 다룬다.
- [[Cache-Aside와 Write-Through-Behind]]
- [[TTL과 캐시 무효화 전략]]
- [[캐시 스탬피드]]
- [[로컬 캐시와 멀티레벨 캐시]]

## Redis
Redis 자료구조와 실행 모델, 메모리·축출 정책, 영속화, 고가용성·분산 구성, 메시징 기능을 다룬다.
- [[Redis 자료구조]]
- [[싱글 스레드 모델과 이벤트 루프]]
- [[영속화 RDB와 AOF]]
- [[Redis 메모리와 축출 정책]]
- [[Redis Cluster]]
- [[Redis Sentinel]]
- [[Redis Pub-Sub와 Streams]]

## 분산 캐시 이슈
캐시와 DB 정합성, 분산 락을 다룬다.
- [[캐시와 DB 정합성]]
- [[분산 락]]

## 캐시 실전 패턴
캐시 워밍 전략, Hot Key 대응 전략을 다룬다.
- [[캐시 워밍 전략]]
- [[Hot Key 대응 전략]]

## 다른 분야와 연결

- Spring 캐시 적용: [[Spring Cache 추상화]]
- 데이터베이스와 복제: [[읽기와 쓰기 분리]], [[리플리케이션]], [[데이터 일관성 모델]]
- 메시징의 전달·처리 보장: [[Pub-Sub와 Point-to-Point]], [[멱등성과 메시지 순서 보장]]
- 요청 중복과 장애 대응: [[멱등성 키 설계]], [[Bulkhead 패턴]], [[우아한 성능 저하]]
