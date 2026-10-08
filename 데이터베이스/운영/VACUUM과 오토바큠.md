---
category: "데이터베이스/운영"
tags: ["database", "postgresql", "mvcc", "vacuum", "운영"]
aliases: ["PostgreSQL VACUUM", "Autovacuum", "오토바큠"]
updated: 2026-10-04
verified: 2026-09-22
---

# VACUUM과 오토바큠

## 핵심 정의

VACUUM은 PostgreSQL에서 더 이상 보존할 필요가 없는 행 버전을 정리하고 공간을 재사용 가능하게 만드는 유지보수 작업이다. 다중 버전 동시성 제어(Multi-Version Concurrency Control, MVCC)는 갱신·삭제 직후에도 이전 버전을 남기므로, 삭제 쿼리의 성공과 물리 공간 회수는 별개다. 오토바큠(autovacuum)은 변경량·트랜잭션 ID 나이 등을 보고 VACUUM과 통계 수집(ANALYZE)을 자동 실행한다.

VACUUM은 공간뿐 아니라 가시성 맵(visibility map)과 트랜잭션 ID 동결(freezing)도 관리한다. 가시성 맵은 인덱스 전용 스캔(index-only scan)의 힙 접근을 줄이고, 동결은 오래된 행이 트랜잭션 ID 순환(wraparound) 때문에 미래 행으로 오인되는 것을 막는다. 아래 설정·뷰·SQL은 **PostgreSQL 18** 기준이다.

## 동작 원리 / 구조

### 갱신에서 재사용까지

1. `UPDATE`는 새 행 버전을 만들고 `DELETE`는 기존 버전을 삭제된 상태로 표시한다.
2. 이전 버전을 볼 수 있는 스냅샷이나 보존 요구가 남으면 VACUUM도 그 버전을 제거할 수 없다.
3. 제거 가능한 버전을 정리하고 인덱스 항목·빈 공간·가시성 정보를 갱신한다.
4. 확보한 공간을 다음 삽입·갱신에 재사용한다. 일반 VACUUM은 주로 파일 내부 공간을 돌려주며, 파일 끝의 빈 페이지를 잘라낼 수 있는 경우를 제외하면 OS에 파일 크기를 반환하지 않는다.

### 작업별 차이

| 작업 | 목적 | 운영상 경계 |
|---|---|---|
| `VACUUM` | 오래된 버전 정리·재사용·가시성·동결 | 일반 조회·DML과 병행할 수 있으나 I/O와 일부 잠금 영향을 남김 |
| `ANALYZE` | 옵티마이저용 데이터 분포 통계 수집 | dead tuple 정리나 파일 축소를 대신하지 않음 |
| `VACUUM (ANALYZE)` | 정리와 통계 수집을 함께 요청 | 명시적 트랜잭션 블록 안에서 실행할 수 없음 |
| `VACUUM FULL` | 테이블을 새 파일로 재작성해 축소 | `ACCESS EXCLUSIVE` 락과 추가 디스크 필요; 정기 작업의 기본 선택이 아님 |

오토바큠은 `VACUUM FULL`을 실행하지 않는다. 일반 VACUUM에도 파일 끝 축소 과정의 잠금이 있을 수 있어 "온라인이므로 잠금이 없다"고 해석하지 않는다.

### 자동 실행 조건과 조절 지점

PostgreSQL 18의 UPDATE·DELETE 기반 실행 임계치는 다음과 같다. 행 수는 `pg_class.reltuples`의 추정치다.

```text
min(autovacuum_vacuum_max_threshold,
    autovacuum_vacuum_threshold
      + autovacuum_vacuum_scale_factor × 추정 행 수)
```

`autovacuum_vacuum_max_threshold=-1`이면 상한을 적용하지 않는다. INSERT 기반 임계치와 XID/MXID 나이에 따른 실행은 별도다. 따라서 갱신·삭제량이 적다는 이유만으로 VACUUM이 필요 없다고 판단하지 않는다.

큰 테이블은 전역 scale factor만 낮추기보다 해당 테이블의 변경량·정리 시간을 보고 저장 매개변수를 조정한다. 임계치를 넘었다고 즉시 시작되는 것은 아니다. worker 여유와 다른 테이블 작업을 함께 확인한다. PostgreSQL 18에서 `autovacuum_max_workers`를 `autovacuum_worker_slots`보다 높여도 실제 worker 수는 슬롯 수를 넘지 못한다.

## 실무 관점

### 먼저 볼 지표

```sql
-- 현재 DB에서 정리 지연 후보 찾기. n_dead_tup은 정확한 실측 건수가 아니다.
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- 현재 DB에서 실행 중인 VACUUM 단계. heap 스캔 비율은 전체 완료율이 아니다.
SELECT pid, relid::regclass AS relation, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed,
       index_vacuum_count
FROM pg_stat_progress_vacuum
WHERE datname = current_database();

-- 데이터베이스별 가장 오래된 미동결 XID의 나이
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
```

`pg_stat_progress_vacuum`은 다른 DB의 작업도 보여줄 수 있다. `relid::regclass`는 현재 DB의 카탈로그에서 이름을 해석하므로 예제는 현재 DB로 제한했다. 클러스터 전체를 볼 때는 `datname`·`relid`를 함께 표시하고 각 DB에서 테이블 이름을 확인한다.

- **정리가 안 되는 원인부터 확인한다.** 오래 열린 트랜잭션, 오래된 `backend_xmin`, prepared transaction, 복제 슬롯의 `xmin`/`catalog_xmin`을 조사한다. worker를 늘려도 보존 경계를 넘어서 삭제하지 못한다.
- **테이블 팽창과 WAL 보존을 구분한다.** 슬롯의 `xmin`/`catalog_xmin`은 행·카탈로그 정리에, `restart_lsn`은 필요한 WAL 보존에 관련된다. CDC 소비 중단으로 WAL이 쌓였다고 VACUUM FULL을 실행해도 WAL 보존 원인은 해결되지 않는다.
- **XID 나이와 증가 속도를 함께 감시한다.** 트랜잭션 ID 나이는 날짜가 아니라 트랜잭션 수다. 정리 실패가 누적되면 쓰기용 XID 할당이 차단될 수 있다. 오래된 트랜잭션·슬롯을 해소하고 일반 VACUUM으로 복구하는 공식 절차를 따른다.
- **파티션 부모 통계를 챙긴다.** 오토바큠은 각 파티션을 처리하지만 파티션 부모에 자동 ANALYZE를 수행하지 않는다. 파티션 추가·대량 적재로 분포가 바뀌면 부모의 통계 수집도 계획한다.
- **정리 속도와 서비스 지연을 함께 측정한다.** 실행 빈도, cost delay/limit, worker 수를 바꿀 때 I/O·지연·dead tuple 추세를 비교한다. 임계치만 낮춰 이미 가득 찬 worker 대기를 해결하려 하지 않는다.

## 심화 Q&A

### Q. 원본 DB에 장기 조회가 없는데도 오래된 행 버전이 남는다면?
A. 복제본의 조회가 `hot_standby_feedback`으로 보존 경계를 원본에 전달하는지 확인한다. PostgreSQL 18의 `pg_stat_replication.backend_xmin`은 복제본이 보고한 xmin 경계다. 원본에서 다음 값을 보면 해당 경계를 오래 유지하는 복제 연결을 찾는 데 도움이 된다.

```sql
SELECT pid, application_name, state, backend_xmin,
       age(backend_xmin) AS xmin_age
FROM pg_stat_replication
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;
```

이 값은 벽시계 경과 시간이 아니라 트랜잭션 ID 나이이며, 높다는 사실만으로 특정 테이블의 모든 팽창 원인이 확정되지는 않는다. 로컬 세션·prepared transaction·복제 슬롯도 함께 확인한다. 다른 세션의 전체 통계에는 superuser 또는 `pg_read_all_stats` 권한이 필요할 수 있다. 피드백을 끄면 복제본 조회가 취소될 수 있으므로, 우선 해당 복제본의 장기 조회 용도·시간 제한을 조정하고 정리 추세를 관측한다([[읽기와 쓰기 분리#Q. PostgreSQL 복제본의 조회가 제한 시간보다 훨씬 빨리 취소되는 이유는?]]).

### Q. VACUUM을 실행했는데 디스크 사용량이 그대로인 이유는?
A. 파일 내부의 빈 공간을 다음 쓰기에 재사용하는 것이 일반 VACUUM의 기본 효과다. 파일 크기가 같다는 이유로 실패한 것은 아니다. 실제로 축소가 필요하면 `VACUUM FULL`의 락·추가 공간·재증가 가능성을 비교한다. 곧 다시 채워질 테이블을 매번 축소하면 재확장 비용을 반복한다.

### Q. dead tuple이 적어도 VACUUM이 필요한 이유는?
A. 오래 유지되는 행의 XID/MXID 동결과 가시성 맵 관리가 필요하다. INSERT 위주 테이블도 이 대상이다. `n_dead_tup` 하나를 근거로 오토바큠을 끄면 정리 대상이 없어 보이는 테이블에서 wraparound 방지 작업이 뒤늦게 커질 수 있다.

### Q. 장기 트랜잭션을 종료하면 즉시 모든 공간을 회수하는가?
A. 종료가 보존 경계를 앞으로 움직일 수는 있지만, 다른 세션·prepared transaction·복제 슬롯의 요구가 남을 수 있다. 이후 VACUUM이 처리해야 하며, 반환한 공간도 우선 내부 재사용 대상이다. 세션 종료와 슬롯 삭제는 업무·복제 영향까지 확인한 뒤 결정한다.

### Q. autovacuum을 꺼도 실행되는 작업은 무엇인가?
A. PostgreSQL 18은 ID 순환 방지를 위해 필요한 autovacuum을 일반 autovacuum 비활성화 상태에서도 실행한다. 이 보호 장치가 모든 차단 원인이나 I/O 부족을 해결해 주지는 않는다. 강제 작업에 기대기보다 나이와 정리 추세를 관리한다.

### Q. `heap_blks_scanned / heap_blks_total`이 100%면 끝났는가?
A. 아니다. 힙 스캔 뒤에도 인덱스 정리·힙 정리·파일 끝 축소·마무리 단계가 남을 수 있다. 가시성 맵으로 건너뛴 페이지도 스캔 카운터에 포함된다. `phase`와 인덱스 작업을 함께 보고 완료 시간을 단순 비례 계산하지 않는다. `VACUUM FULL` 진행은 별도 `pg_stat_progress_cluster`에서 본다.

## 관련 개념
- [[MVCC]]
- [[커버링 인덱스]]
- [[실행 계획 읽는 법]]
- [[WAL과 트랜잭션 로그]]
- [[Kafka Connect]]
- [[백업과 복구 전략]]

## 참고 자료

확인일: 2026-09-22. PostgreSQL 18의 유지보수·임계치·잠금·관측 뷰를 공식 문서와 대조했다. SQL은 문서 기반 예시이며 PostgreSQL 서버에서 실행하지 않았다.

- [Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html) — 공간 재사용, VM·동결, 장기 보존 경계, 파티션 부모 ANALYZE.
- [VACUUM](https://www.postgresql.org/docs/18/sql-vacuum.html) — FULL·ANALYZE 옵션, 잠금, 트랜잭션 블록 제한.
- [Vacuuming Configuration](https://www.postgresql.org/docs/18/runtime-config-vacuum.html) — 실행 임계치·worker slots·비용 조절.
- [Statistics Views](https://www.postgresql.org/docs/18/monitoring-stats.html) — `pg_stat_user_tables` 추정치와 세션 관측.
- [Progress Reporting](https://www.postgresql.org/docs/18/progress-reporting.html#VACUUM-PROGRESS-REPORTING) — VACUUM 단계와 카운터의 해석.
- [Replication Slots](https://www.postgresql.org/docs/18/view-pg-replication-slots.html) — `xmin`, `catalog_xmin`, `restart_lsn`의 보존 대상 차이.
- [Object Identifier Types](https://www.postgresql.org/docs/18/datatype-oid.html) — `regclass`의 `pg_class` 이름 해석과 DB별 관측 범위.

부분 재검증: 2026-10-04. PostgreSQL 18의 복제 피드백 xmin 관측 필드와 통계 접근 권한을 확인했다. SQL은 문서 대조만 수행했으며 실제 원본·복제본에서 실행하지 않았다. VACUUM 전체의 기존 `verified`는 유지한다.

- [PostgreSQL 18 Statistics: pg_stat_replication](https://www.postgresql.org/docs/18/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW) — backend_xmin의 피드백 경계, application_name·state, 통계 접근 권한.
- [PostgreSQL 18 hot_standby_feedback](https://www.postgresql.org/docs/18/runtime-config-replication.html#GUC-HOT-STANDBY-FEEDBACK) — 원본 정리 지연과 복제본 cleanup 충돌의 관계.
