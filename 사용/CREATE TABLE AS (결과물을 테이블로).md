## 1\. CREATE TABLE AS 사용

```
-- 기본
CREATE TABLE films_recent AS
  SELECT * FROM films WHERE date_prod >= '2002-01-01';

--Synopsis
CREATE [ [ GLOBAL | LOCAL ] { TEMPORARY | TEMP } ] TABLE table_name [ (column_name [, ...] ) ] [ [ WITH | WITHOUT ] OIDS ]
    AS query
```

## 2\. CREATE TABLE AS 옵션

**\- TEMPORARY / TEMP**

        임시 테이블로 생성되며 세션이 종료될 시 삭제된다.

**\- WITH OIDS / WITHOUT OIDS**

        CREATE TABLE AS로 생성된 테이블이 OID를 포함하는지 여부이다. 사용하기 위해서는 default\_with\_oids 설정이 필요하다.

## 3\. CREATE TABLE AS란?

###           3-1. 장점

데이터 작업을 하다 보면 종종 특정 쿼리의 결과물로 별도 테이블을 생성하는 경우가 있다. 일반 테이블을 생성 후 SELECT-INSERT를 사용해도 되지만, 컬럼 명이나 data type을 별도로 맞춰 넣는 작업을 해야 한다. 하지만 CREATE TABLE AS는 테이블을 생성 후 SELECT 쿼리의 결과물을 바로 채워 넣고 테이블의 각 컬럼들은 SELECT 결과의 명칭 (alias를 통해 명시적으로 컬럼명을 재정의 할 수 있다.)과 data type을 자동으로 생성한다. 

###           3-2. VIEW, SELECT INTO 와 다른 점

VIEW를 만드는 것과 유사지만 다르다. VIEW는 매 조회 시 데이터를 새로 조회하지만, CREATE TABLE AS는 테이블을 만들기 위해 SELECT 쿼리를 최초 1회만 수행한다. 그러므로 신규 테이블은 기존 테이블들의 변화를 참조하지 않는다.

SELECT INTO와도 유사한 기능이지만, SELECT INTO 문법보다 직관적이며 더 많은 기능을 제공한다.

###           3-3. 주의사항

PostgreSQL 8.0 이전에는 CREATE TABLE AS는 OID를 항상 포함했고, 8.0 이후부터 OID 포함여부를 선택할 수 있게 되었다.

OID 포함 여부를 명시적으로 지정하지 않은 경우 default\_with\_oids 파라미터를 사용한다. default\_with\_oids는 기본값이 true이다.

CREATE TABLE AS에서 생성한 테이블에 OID를 필요로 하는 응용 프로그램은 PostgreSQL 향후 버전과의 호환성을 보장하기 위해 WITH OID를 명시적으로 지정해야 한다. 

참고

[https://www.postgresql.org/docs/8.0/sql-createtableas.html](https://www.postgresql.org/docs/8.0/sql-createtableas.html)