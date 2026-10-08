# 메시지 큐

메시징 기본 개념, Kafka, RabbitMQ, 이벤트 기반 아키텍처를 다루는 노트 모음.

## 추천 읽기 순서

1. [[Pub-Sub와 Point-to-Point]] → [[멱등성과 메시지 순서 보장]] → [[메시지 큐 선택 기준]]
2. Kafka 경로: [[Kafka 아키텍처]] → [[파티션과 컨슈머 그룹]] → [[오프셋 관리와 정확히 한 번 처리]] → [[Kafka 로그 보존과 컴팩션]] → [[컨슈머 랙과 백프레셔]]
3. Kafka 활용: [[Kafka Connect]] → [[Kafka Streams]]
4. RabbitMQ 경로: [[Exchange와 Queue 바인딩]] → [[메시지 확인 ACK]] → [[Dead Letter Queue]]
5. 이벤트 설계: [[트랜잭셔널 아웃박스]] → [[SAGA 패턴]] → [[이벤트 소싱]] → [[CQRS 패턴]]

브로커를 선택한 뒤 메시징 패턴과 운영 노트를 함께 읽는다. CQRS의 원본은 시스템 설계 분야에 있다.

## 메시징 기본 개념
Pub-Sub와 Point-to-Point, 멱등성과 메시지 순서 보장을 다룬다.
- [[Pub-Sub와 Point-to-Point]]
- [[멱등성과 메시지 순서 보장]]

## Kafka
Kafka 아키텍처와 처리 보장, 로그 보존·컴팩션, 소비 지연, 외부 연동과 스트림 처리를 다룬다.
- [[Kafka 아키텍처]]
- [[파티션과 컨슈머 그룹]]
- [[오프셋 관리와 정확히 한 번 처리]]
- [[컨슈머 랙과 백프레셔]]
- [[Kafka 로그 보존과 컴팩션]]
- [[Kafka Connect]]
- [[Kafka Streams]]

## RabbitMQ
Exchange와 Queue 바인딩, 메시지 확인 ACK를 다룬다.
- [[Exchange와 Queue 바인딩]]
- [[메시지 확인 ACK]]

## 이벤트 기반 아키텍처
이벤트 소싱, SAGA 패턴, 트랜잭셔널 아웃박스를 다룬다.
- [[이벤트 소싱]]
- [[SAGA 패턴]]
- [[트랜잭셔널 아웃박스]]

## 메시지 큐 운영
Dead Letter Queue와 메시지 큐 선택 기준을 다룬다.
- [[Dead Letter Queue]]
- [[메시지 큐 선택 기준]]

## 메시징 패턴
Competing Consumers 패턴, 우선순위 큐와 메시지 정렬을 다룬다.
- [[Competing Consumers 패턴]]
- [[우선순위 큐와 메시지 정렬]]

## 다른 분야와 연결

- 읽기·쓰기 모델 설계: [[CQRS 패턴]], [[읽기와 쓰기 분리]]
- API 재시도와 중복 처리: [[멱등성 키 설계]]
- Redis 메시징: [[Redis Pub-Sub와 Streams]]
- Spring 이벤트 처리: [[Spring Event와 비동기 처리]], [[Spring Retry와 재시도 전략]]
- 장애 격리와 추적: [[Bulkhead 패턴]], [[분산 트레이싱]], [[분산 트레이스 ID 전파]]
