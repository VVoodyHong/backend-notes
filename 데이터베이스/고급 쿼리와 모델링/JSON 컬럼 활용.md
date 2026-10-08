---
category: "데이터베이스/고급 쿼리와 모델링"
tags: ["database", "json", "jsonb", "mysql", "postgresql", "schema-design"]
updated: 2026-10-04
verified: 2026-09-08
---

# JSON 컬럼 활용

## 핵심 정의

JSON 컬럼은 관계형 DB 안에 반정형(semi-structured) 데이터를 JSON 문서 형태로 저장하고, 내부 필드를 SQL 함수/연산자로 조회·인덱싱할 수 있게 하는 데이터 타입이다. 완전한 스키마가 불필요하거나 자주 바뀌는 속성 집합(예: 상품 옵션, 이벤트 로그의 가변 메타데이터)을 별도 테이블/컬럼으로 정규화하지 않고도 다룰 수 있게 해주며, 관계형 모델과 문서형 모델의 중간 지점 역할을 한다.

## 동작 원리 / 구조

### MySQL의 JSON 타입

MySQL 5.7부터 네이티브 `JSON` 타입을 지원한다. 아래 예제는 `JSON_VALUE()`를 제공하는 MySQL 8.4 기준이다(이 함수는 8.0.21 도입). 텍스트가 아니라 내부적으로 파싱된 바이너리 형식으로 저장돼, 매번 문자열을 파싱하지 않고 빠르게 특정 값에 접근할 수 있다.

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(200),
  attributes JSON
);

INSERT INTO products VALUES
  (1, '노트북', '{"cpu": "i7", "ram_gb": 16, "tags": ["office", "gaming"], "tag_ids": [10, 20]}');

SELECT id, JSON_VALUE(attributes, '$.cpu') AS cpu
FROM products
WHERE JSON_VALUE(attributes, '$.ram_gb' RETURNING UNSIGNED) >= 16;
```

JSON 컬럼 자체는 직접 인덱싱할 수 없다. 대신 두 가지 방법으로 인덱싱한다.

1. **생성 컬럼(Generated Column) + 일반 인덱스**: JSON에서 값을 추출해 가상/저장 컬럼으로 만들고 그 컬럼에 인덱스를 건다.
   ```sql
   ALTER TABLE products
     ADD COLUMN ram_gb INT GENERATED ALWAYS AS (JSON_VALUE(attributes, '$.ram_gb' RETURNING SIGNED)) VIRTUAL,
     ADD INDEX idx_ram_gb (ram_gb);
   ```
2. **함수형 인덱스(Functional Index)**: `JSON_VALUE()` 표현식에 직접 인덱스를 걸면 내부적으로 숨겨진 가상 컬럼이 자동 생성된다. 배열 원소에 대해서는 다중값 인덱스(Multi-Valued Index, MVI, MySQL 8.0.17+)로 `MEMBER OF()`, `JSON_CONTAINS()`, `JSON_OVERLAPS()` 조건을 인덱스로 가속할 수 있다.
   ```sql
   ALTER TABLE products
     ADD INDEX idx_tag_ids ((CAST(attributes->'$.tag_ids' AS UNSIGNED ARRAY)));

   SELECT * FROM products WHERE 20 MEMBER OF (attributes->'$.tag_ids');
   ```

### PostgreSQL의 JSON vs JSONB

| | `json` | `jsonb` |
|---|---|---|
| 저장 방식 | 입력 텍스트 그대로 저장 | 파싱된 바이너리 구조로 저장 |
| 입력 속도 | 빠름(파싱만) | 상대적으로 느림(파싱+변환) |
| 조회/연산 속도 | 매번 재파싱 필요 | 빠름 |
| 키 순서/중복 키 | 원본 그대로 보존 | 정규화됨(중복 키 제거, 순서 변경) |
| 인덱싱 | 문서 전체 GIN은 지원하지 않지만 추출한 스칼라의 표현식 인덱스는 가능 | GIN 및 표현식 인덱스 지원 |

실무에서는 조회·인덱싱이 필요한 거의 모든 경우 `jsonb`를 사용한다. `json`은 원본 텍스트를 그대로 보존해야 하는 로그성 저장 용도 정도에만 의미가 있다.

```sql
CREATE TABLE events (
  id BIGSERIAL PRIMARY KEY,
  payload JSONB
);

-- 포함 관계(containment) 검색
SELECT * FROM events WHERE payload @> '{"type": "click"}';

-- GIN 인덱스: 기본(jsonb_ops) - 키 존재(?)/포함(@>)/jsonpath(@?, @@) 모두 지원, 인덱스 크기 큼
CREATE INDEX idx_payload_gin ON events USING GIN (payload);

-- jsonb_path_ops - @>, @?, @@ 만 지원하지만 인덱스가 더 작고 빠름
CREATE INDEX idx_payload_gin_pathops ON events USING GIN (payload jsonb_path_ops);

-- 특정 키만 자주 조회한다면 표현식 B-tree 인덱스가 더 효율적
CREATE INDEX idx_payload_type ON events ((payload->>'type'));
```

PostgreSQL 17부터는 SQL 표준 SQL/JSON 함수(`JSON_TABLE`, `JSON_VALUE`, `JSON_QUERY`, `JSON_EXISTS`)가 추가돼, JSON 배열을 관계형 행 집합으로 펼치는 작업(`JSON_TABLE`)을 표준 문법으로 처리할 수 있다.

```sql
SELECT t.*
FROM events, JSON_TABLE(payload, '$.items[*]' COLUMNS (
  sku TEXT PATH '$.sku',
  qty INT PATH '$.qty'
)) AS t;
```

### 필드 누락·JSON null·SQL NULL과 CHECK

PostgreSQL 18에서 `payload->'qty'`는 없는 키를 SQL NULL로 반환하지만, 존재하는 JSON null은 JSON 값으로 남긴다. `->>`로 텍스트를 추출하면 JSON null도 SQL NULL이 되어 누락과 구별되지 않는다. `jsonb_typeof(payload->'qty')`를 사용하면 JSON null은 문자열 `'null'`, 누락은 SQL NULL로 구분할 수 있다.

CHECK 제약은 결과가 TRUE뿐 아니라 SQL NULL이어도 통과한다. 따라서 `CHECK (jsonb_typeof(payload->'qty') = 'number')`만으로 필수 숫자 필드를 강제하면 키가 없는 문서가 통과한다. 객체 안에 숫자 필드가 반드시 있어야 한다면 전체 판정을 `IS TRUE`로 감싼다.

```sql
-- PostgreSQL 18: SQL NULL, 비객체, 누락, JSON null, 문자열 숫자를 거부
ALTER TABLE events ADD CONSTRAINT events_numeric_qty
CHECK ((jsonb_typeof(payload) = 'object'
        AND jsonb_typeof(payload -> 'qty') = 'number') IS TRUE);
```

위 제약은 숫자 존재·타입만 검사하므로 음수·소수의 업무 허용 여부는 별도 조건이다. JSON 컬럼 자체의 NOT NULL은 필드 존재를 보장하지 않는다. 기존 행이 제약을 위반하면 추가 DDL이 실패하므로 배포 전 누락·JSON null·문자열 값을 각각 집계하고 정리한다.

### jsonb_set의 중간 경로와 SQL NULL 인자

PostgreSQL 18의 `jsonb_set(..., create_if_missing=true)`는 마지막 경로 항목을 만들 수 있지만 빠진 중간 객체까지 만들지는 않는다. 예를 들어 `{}`에 경로 `{settings,theme}`로 `"dark"`를 넣으면 원문 `{}`가 그대로 반환된다. 이 식을 UPDATE에 사용하면 영향 행 수가 1이어도 기대한 필드가 생성되지 않을 수 있다. 중간 경로 존재·타입을 검증하고 필요한 부모 객체를 명시적으로 구성한다.

또한 `jsonb_set`에 SQL NULL을 새 값으로 전달하면 전체 함수 결과가 SQL NULL이다. JSON 필드에 null을 넣으려면 `'null'::jsonb`처럼 JSON null을 전달해야 한다. 바인딩 값이 없을 때의 의미를 오류·필드 삭제·JSON null·원문 유지 중 무엇으로 할지 정하고, 필요하면 `jsonb_set_lax`의 `null_value_treatment`를 명시한다. lax의 기본값은 `use_json_null`이므로 이름만 보고 요청 필드 누락을 자동으로 무시한다고 생각하면 안 된다.

## 실무 관점

- **언제 쓰는가**: 속성이 상품 카테고리마다 달라지는 EAV(Entity-Attribute-Value)성 데이터, 외부 API 응답 원문 보관, 이벤트/로그의 가변 메타데이터, 사용자 커스텀 설정(preference) 저장에 적합하다. 반대로 자주 조인·집계·정합성 제약이 필요한 핵심 비즈니스 데이터는 정규화된 컬럼으로 두는 것이 원칙이다.
- **트레이드오프**: JSON은 스키마 유연성을 얻는 대신 DB 레벨의 타입 검증, 외래키 제약, 컬럼 단위 통계 기반 옵티마이저 추정이 약해진다. "일단 JSON에 다 넣고 나중에 정리하자"는 접근은 시간이 지나며 사실상 스키마가 없는 상태로 변질돼 쿼리 복잡도와 데이터 정합성 문제를 키운다.
- **부분 업데이트**: JSON 문서 일부만 바꾸고 싶을 때 애플리케이션에서 전체를 읽고 수정 후 통째로 다시 쓰면 동시성 문제(lost update)가 발생하기 쉽다. 현재 컬럼 값에 PostgreSQL의 `jsonb_set()`이나 MySQL의 `JSON_SET()`을 적용하면 다른 경로까지 통째로 덮어쓰는 위험을 줄일 수 있다. 동일 경로를 경쟁 갱신하거나 애플리케이션에서 읽은 낡은 값으로 새 값을 계산하는 경우에는 조건부 UPDATE·버전 검사 등 별도 충돌 처리가 필요하다. MySQL 8.4는 `JSON_SET()`/`JSON_REPLACE()`/`JSON_REMOVE()`로 기존 값을 덮어쓰기만 하고 기존 항목만 교체·삭제하고 대체 값이 기존 공간 또는 이전 부분 갱신으로 확보한 여유 공간에 들어가는 등의 조건이 맞으면, 문서 전체를 재작성하지 않고 바이너리 표현을 제자리(in-place)에서 부분 갱신하는 최적화를 적용한다(`JSON_STORAGE_FREE()`로 이전 부분 갱신이 만든 여유 공간을 확인할 수 있다).
- **비대화 주의**: 큰 배열이나 중첩 구조를 계속 append하는 패턴(예: 하나의 주문 문서에 이벤트 이력을 무한정 추가)은 문서가 비대해지고, 매번 갱신 시 재작성 비용이 커진다. 이력성 데이터는 별도 테이블로 분리하는 편이 낫다.
- **인덱스 전략 선택**: 특정 키를 자주 등호(`=`) 조건으로 조회한다면 표현식 B-tree 인덱스를, 포함 관계(`@>`) 중심이면 GIN(`jsonb_path_ops`)을 검토한다. 키 존재 검색까지 필요하면 기본 `jsonb_ops`가 필요하다. 인덱스 종류를 잘못 고르면 JSON 컬럼에 인덱스가 있어도 실제로는 순차 스캔이 발생한다. `EXPLAIN`으로 반드시 확인해야 한다.
- **흔한 실수**: MySQL에서 `JSON_UNQUOTE()`/`->>`로 문자열을 추출한 결과의 콜레이션(`utf8mb4_bin`)이 테이블 기본 콜레이션과 달라 함수형 인덱스가 조용히 사용되지 않는 경우가 있다. `CAST(... AS CHAR) COLLATE ...`로 콜레이션을 명시적으로 맞춰야 한다.

## 심화 Q&A

### Q. JSON 컬럼을 쓸지, 완전히 정규화된 테이블로 쪼갤지 판단하는 기준은?
그 속성이 "구조화된 쿼리 대상(자주 필터링·조인·집계·제약 조건이 필요)"인지가 핵심 기준이다. 자주 검색·정렬·집계되고 다른 테이블과 관계를 맺어야 한다면 정규화된 컬럼/테이블이 낫다. 반면 속성 집합이 레코드마다 크게 다르고, 조회 빈도가 낮거나 "가져와서 그대로 보여주는" 용도라면 JSON이 스키마 마이그레이션 비용을 줄여준다. 실무에서는 "핵심 필드는 정규화 컬럼 + 부가 속성은 JSON"인 하이브리드 모델을 가장 많이 쓴다.

### Q. PostgreSQL에서 `jsonb_ops`와 `jsonb_path_ops` GIN 인덱스 중 무엇을 선택해야 하는가?
쿼리 패턴에 달렸다. 키 존재 여부(`?`, `?|`, `?&`) 연산자를 써야 한다면 `jsonb_ops`가 필수다(`jsonb_path_ops`는 이 연산자를 지원하지 않는다). 반면 포함 관계(`@>`)나 jsonpath 매치(`@?`, `@@`) 위주라면 `jsonb_path_ops`가 인덱스 크기가 더 작고(키와 값 각각에 항목을 만드는 `jsonb_ops`와 달리, 값과 상위 키를 해시해 하나의 항목만 만들기 때문) 검색 특이성도 높아 더 빠르다. 다만 `jsonb_path_ops`는 값이 없는 빈 객체/배열 구조는 인덱싱하지 않아 해당 패턴 검색 시 전체 인덱스 스캔이 필요할 수 있다는 점도 고려해야 한다. 두 연산자 그룹을 모두 자주 쓴다면 두 인덱스를 함께 만들 수도 있지만 쓰기·저장 비용이 늘어나므로 신중히 판단해야 한다.

### Q. JSON 컬럼에 외래키(foreign key) 제약을 걸 수 있는가?
표준적으로는 불가능하다. JSON 내부의 필드는 DB 카탈로그가 별도 컬럼으로 인식하지 않으므로 참조 무결성을 DB가 보장해주지 않는다. JSON 안에 다른 테이블의 ID를 저장하는 패턴을 쓴다면, JSON에서 추출한 값을 별도 저장 컬럼으로 만들고 DB가 지원하는 조건에서 FK를 적용하거나, 애플리케이션·트리거로 검증해야 한다. 트리거가 곧 약한 보장이라는 뜻은 아니지만 동시 삭제·갱신까지 올바르게 잠그는 구현 책임이 생긴다. 참조 무결성이 중요한 관계라면 애초에 JSON이 아니라 정규화된 컬럼/조인 테이블로 모델링하는 것이 맞다.

### Q. 같은 JSON 문서 안의 한 필드만 자주 갱신하는데 전체 문서를 매번 다시 쓰면 어떤 문제가 생기는가?
문서가 클수록 갱신 시 I/O·CPU 비용이 커지고, PostgreSQL처럼 MVCC로 동작하는 엔진에서는 UPDATE가 새 튜플(tuple) 버전을 만들기 때문에 큰 JSON 문서를 자주 갱신하면 테이블/인덱스 블로트(bloat)가 빠르게 쌓인다. 자주 바뀌는 필드는 별도 컬럼(또는 별도 테이블)으로 분리하고, JSON은 상대적으로 안정적인 속성 집합에만 쓰는 것이 유리하다. `SET attributes = jsonb_set(attributes, ...)`처럼 DB의 현재 컬럼을 기준으로 갱신하면 다른 경로를 오래된 전체 문서로 덮어쓰는 문제를 줄인다. 다만 같은 경로에 애플리케이션이 미리 계산한 구값을 넣는 경쟁은 남는다. 현재 값 기반 SQL 연산이나 버전 조건을 함께 사용하고 격리 수준에 따른 충돌·재시도를 처리한다. 함수 호출 하나가 업무 차원의 lost update를 모두 막지는 않는다.

### Q. MySQL의 다중값 인덱스(Multi-Valued Index)는 일반 보조 인덱스와 무엇이 다른가?
일반 보조 인덱스는 한 행에 대해 하나의 인덱스 엔트리를 갖지만, MVI는 JSON 배열의 원소마다 별도 인덱스 엔트리를 만들어 한 행이 여러 엔트리에 대응한다. 그 결과 `MEMBER OF()`, `JSON_CONTAINS()`, `JSON_OVERLAPS()`처럼 "배열에 이 값이 포함되는가"를 빠르게 찾을 수 있지만, 엔트리가 흩어져 있어 범위 스캔(range scan)이나 인덱스 온리 스캔은 지원하지 않고, 온라인 생성(`ALGORITHM=INPLACE`)도 불가능하다는 제약이 있다.

### Q. JSON 컬럼에 대한 통계 정보 부족이 실행 계획에 어떤 영향을 주는가?
옵티마이저는 일반 컬럼에 대해서는 히스토그램 등 통계로 카디널리티(cardinality)를 추정하지만 통계 오차가 있으며, JSON 내부 필드는 생성 컬럼·표현식 인덱스 등으로 통계를 수집하지 않으면 내부 필드의 분포를 충분히 추정하지 못할 수 있다. PostgreSQL 18은 `CREATE STATISTICS`로 표현식 통계도 수집할 수 있어 인덱스 생성만이 유일한 방법은 아니다. 그 결과 선택도(selectivity) 추정이 부정확해 부적절한 조인 순서나 스캔 방식을 선택할 위험이 커진다. 자주 조회되는 JSON 필드는 생성 컬럼으로 승격시켜 통계 수집 대상에 포함시키는 것이 실행 계획 안정성 측면에서도 유리하다.

## 관련 개념
- [[NoSQL 데이터 모델 비교]]
- [[정규화와 반정규화]]
- [[인덱스 설계 전략]]
- [[MVCC]]

## 참고 자료

부분 재확인: 2026-09-22. PostgreSQL 18 `jsonb_set`·동시 UPDATE와 MySQL 8.4 `JSON_SET`의 경로 갱신 의미를 대조해 부분 갱신 함수의 lost update 방지 범위를 보완했다. 기존 전체 확인일은 유지한다.

- [MySQL 8.4 JSON Modification Functions](https://dev.mysql.com/doc/refman/8.4/en/json-modification-functions.html) — 현재 JSON 값의 경로 갱신과 반환값.
- [PostgreSQL 18 JSON Functions](https://www.postgresql.org/docs/18/functions-json.html) — `jsonb_set`의 경로 갱신과 반환값.
- [PostgreSQL 18 Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html) — 동시 UPDATE의 대기·재평가·격리 수준별 재시도.

확인일: 2026-09-08. 아래 적용 범위에서 본문·Q&A·예제를 재검증했다. 예제는 문서와 대조했으며 별도 실행 검증은 하지 않았다.

- [MySQL 8.4 JSON Data Type](https://dev.mysql.com/doc/refman/8.4/en/json.html) — JSON 바이너리 저장·부분 갱신 조건.
- [MySQL 8.4 CREATE INDEX](https://dev.mysql.com/doc/refman/8.4/en/create-index.html) — 생성/함수형/MVI 인덱스와 제약.
- [PostgreSQL 18 JSON Types](https://www.postgresql.org/docs/18/datatype-json.html) — json/jsonb·GIN 연산자 클래스.
- [PostgreSQL 17 Release Notes](https://www.postgresql.org/docs/17/release-17.html) — SQL/JSON 기능 도입.
- [PostgreSQL 18 CREATE STATISTICS](https://www.postgresql.org/docs/18/sql-createstatistics.html) — 표현식 통계.

부분 재검증: 2026-09-23. PostgreSQL 18 JSON 추출·타입 함수와 CHECK의 3값 논리 경계를 공식 문서 및 REL_18_STABLE JSON 소스로 대조했다. 해당 jsonb DDL의 PostgreSQL 실행 검증은 하지 않았다. 기존 전체 검증일은 유지한다.

- [PostgreSQL 18 JSON Functions](https://www.postgresql.org/docs/18/functions-json.html) — 누락 경로의 SQL NULL과 jsonb_typeof.
- [PostgreSQL 18 Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) — CHECK의 NULL 통과와 NOT NULL의 별도 역할.
- [PostgreSQL 18 jsonfuncs.c](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/utils/adt/jsonfuncs.c) — jsonb_object_field_text의 JSON null → SQL NULL 반환.

부분 재검증: 2026-10-04. PostgreSQL 18의 jsonb_set 중간 경로·jsonb_set_lax NULL 정책을 확인했다. PostgreSQL 18.6·pgJDBC 42.7.13 격리 실행에서 strict 카탈로그 속성, 부모 누락 시 원문 유지·UPDATE 행 수 1, SQL NULL 문서 반환과 JSON null 필드 유지, lax 기본 변환·raise_exception의 22004를 실제 확인했다(10개 assertion). 기존 CHECK DDL·MySQL·동시 JSON 갱신은 이번 실행 범위가 아니며 전체 `verified`는 유지한다.

- [PostgreSQL 18 JSON Functions](https://www.postgresql.org/docs/18/functions-json.html) — jsonb_set의 중간 경로 요구와 jsonb_set_lax의 네 가지 NULL 처리.
- [PostgreSQL 18.6 pg_proc.dat](https://github.com/postgres/postgres/blob/REL_18_6/src/include/catalog/pg_proc.dat) — jsonb_set과 비엄격 함수 jsonb_set_lax 정의. 실제 서버의 pg_proc.proisstrict도 조회했다.
