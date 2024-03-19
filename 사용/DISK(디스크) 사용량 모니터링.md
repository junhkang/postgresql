## 1\. Disk 영역

PostgreSQL의 저장 영역은 크게 Heap, Index, Toast 3개로 나뉜다. 각 테이블의 대부분의 데이터를 메인 heap 영역에 저장한고, 테이블 칼럼 중 매우 큰 데이터를 받을 수 있는 칼럼은 TOAST 영역에 별도 저장한다. TOAST 테이블에는 실제 데이터를 저장하고, 본래 테이블에는 해당 데이터를 가리키는 포인터만 남게 된다. TOAST 테이블의 유효한 인덱스는 1개뿐이고, 기준 테이블에는 더 많은 인덱스가 존재할 수 있다. 테이블과 인덱스는 각각의 디스크파일에 저장 되며 각 파일이 1G가 넘으면 별도 파일로 분리된다.

-   **Heap -** 실제 메인 데이터가 저장되는 곳
-   **Index -** 데이터에 빠르게 접근할 수 있도록 돕는 색인정보, 검색속도를 높이고 데이터의 무결성을 유지하는데 도움
-   **Toast -** 테이블의 한 행이 페이지 크기 (일반적으로 8KB)를 초과할 경우 별도 분리된 공간인 TOAST에 저장하여 데이터베이스의 효율성 유지

## 2\. Disk 용량 모니터링

현재 산군 서비스는 AWS, Datadog의 모니터링을 사용하고 있지만 공식문서에 따르면 디스크용량을 모니터링할 수 있는 방법은 3가지가 있다.

-   SQL 함수를 사용
-   oid2name
-   시스템 카탈로그의 수동 검사

### 2-1. SQL 기본 함수 사용

SQL 기본 함수를 사용하여 실시간으로 조회하는 것이 가장 쉽고 보편적인 방법이다. 쿼리를 사용하여 필요한 항목, 조건에 따라 조회하는 방식이다. (예제의 PG\_SIZE\_PRETTY -> 가독성 좋은 단위로 변경)

#### 2-1-1. 테이블의 디스크 사용량 확인 

**PG\_RELATION\_SIZE =** 지정한 테이블의 크기를 바이트 단위로 반환

```
SELECT pg_size_pretty(pg_relation_size('테이블명'));
```

#### 2-2-2. 전체 데이터베이스 디스크 확인

**PG\_DATABASE\_SIZE** **\=** 전체 데이터베이스의 크기를 바이트 단위로 반환

```
SELECT pg_size_pretty(pg_database_size('데이터베이스명'));
```

#### 2-2-3. 인덱스 디스트 사용량 확인

**PG\_INDEX\_SIZE =** 특정 테이블에 연결된 인덱스 크기를 반환

```
SELECT pg_size_pretty(pg_indexes_size('테이블명'));
```

#### 2-2-4. 테이블스페이스 크기확인

**PG\_TABLESPACE\_SIZE =** 테이블 스페이스의 디스크 사용량을 바이트단위로 반환

```
SELECT pg_size_pretty(pg_tablespace_size('테이블스페이스명'));
```

### 2-2. oid2 name

PostgreSQL에 사용되는 파일구조를 검사하는 유틸리티 프로그램으로, 대상 데이터베이스에 연결하여 OID, 파일노드 및 테이블 정보를 추출한다. 테이블 OID와 파일 노드 간의 차이를 반드시 이해하고 사용해야 한다.

### 2-3. 시스템 카탈로그에서 직접 확인 (Psql)

 데이터베이스의 메타데이터 (테이블, 뷰, 인덱스 등)을 직접 조회하고 분석

최근 vacuum 되거나 재집계된 데이터베이스를 대상으로 psql을 통해 디스크 사용량을 확인하기 위한 쿼리를 실행시킬 수 있다.

#### 2-3-1. 테이블 용량 확인

```
SELECT pg_relation_filepath(oid), relpages FROM pg_class WHERE relname = '테이블명';
```

<p align="center"><img src="/img/disk.png"/></p>

각 "page"는 일반적으로 8kb이며 relpage는 VACUUM, ANALYZE, CREATE INDEX와 같은 몇 가지 DDL 명령문에 의해서만 업데이트된다. 파일 경로 이름은 테이블의 디스크 파일을 직접 확인하고 싶을 때 필요한 정보이다. 예제의 테이블은 인덱스와 파티션이 포함되어 있는 1300만 ROW에 대한 정보로 926,896페이지 용량의(926,896 \* 8kb로 약 7.07GB) 테이블이다.

#### 2-3-2. TOAST 용량확인

TOAST 테이블을 사용하는 경우 다음과 같다.

```
SELECT relname, relpages
FROM pg_class,
     (SELECT reltoastrelid
      FROM pg_class
      WHERE relname = '테이블명') AS ss
WHERE oid = ss.reltoastrelid OR
      oid = (SELECT indexrelid
             FROM pg_index
             WHERE indrelid = ss.reltoastrelid)
ORDER BY relname;
```

<p align="center"><img src="/img/disk2.png"/></p>

#### 2-3-3. 인덱스 용량 확인

```
SELECT c2.relname, c2.relpages
FROM pg_class c, pg_class c2, pg_index i
WHERE c.relname = '테이블명' AND
      c.oid = i.indrelid AND
      c2.oid = i.indexrelid
ORDER BY c2.relname;
```

#### 2-3-4. 최대 용량 테이블, 인덱스 찾기

각 영역의 페이지 용량으로 정렬하면, 최대 용량의 테이블 및 인덱스를 조회할 수 있다.

```
SELECT relname, relpages
FROM pg_class
ORDER BY relpages DESC;
```

참고

[https://www.postgresql.org/docs/16/disk-usage.html](https://www.postgresql.org/docs/16/disk-usage.html)