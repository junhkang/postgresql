## 1\. 뷰(VIEW) 테이블의 사용 (생성, 삭제, 수정)

```
-- 기본 생성
CREATE VIEW comedies AS
    SELECT *
    FROM films
    WHERE kind = 'Comedy';

-- 삭제
DROP VIEW comedies

--Synopsis
CREATE [ OR REPLACE ] VIEW name [ ( column_name [, ...] ) ] AS query
-- or
CREATE VIEW name [ ( column [, ...] ) ]
    AS query
    [ WITH [ CASCADE | LOCAL ] CHECK OPTION ]
```

## 2\. 뷰(VIEW)의 개념 및 특징

-   정의된 쿼리를 실행시켜 가상의 테이블 형태로 보여주며 테이블을 조회하는 것과 같은 방식으로 조회가 가능하다.
-   VIEW는 물리적으로 생성되지 않는다.
-   복잡한 쿼리를 단순화시키거나 반복된 쿼리 작업을 효율적으로 처리할 수 있게 해 준다.
-   VIEW에 참조된 쿼리는 호출 시 매번 새로 실행되기에 실시간 결과물을 조회할 수 있다.
-   CREATE OR REPLACE VIEW로 VIEW를 수정할 시, 완전히 일치하는 컬럼 셋을 조회하는 쿼리로만 대체가 가능하다. (같은 컬럼명과 데이터타입)
-   Schema 명을 명시적으로 작성하면 해당 Schema에, 아니라면 현재 Schema에 생성된다.
-   View, Table, Sequence, Index는 한 스키마에 중복된 명칭을 가질 수 없다.
-   VIEW 결과물은 수정이 불가능하다.
-   테이블의 전체 컬럼 및 정보를 직접적으로 노출시키지 않은 채로 사용이 가능하다.

## 3\. 주의사항

###           3-1. READ-ONLY

VIEW 자체에 insert, update, delete를 실행할 수 없다.

(RULE 생성을 적절하게 사용하여 업데이트 가능한 뷰의 효과를 볼 수는 있지만 설정이 까다롭기에 새로 생성하는 것이 효율적이다.)

###           3-2. 데이터 타입/ 컬럼명 선언

뷰를 생성할 시 데이터타입과 컬럼명을 선언해주어야한다. 그렇지 않을 경우 기본 컬럼명은 ?column?, 데이터 타입은 unknown으로 선언된다. 예를들어 다음 쿼리로 VIEW를 생성하면 컬럼명 = ?column?,  데이터타입 = unknown으로 생성된다.

```
CREATE VIEW vista AS SELECT 'Hello World';
```

이를 방지하기 위해서는 다음과 같이 SQL문을 작성해야 한다.

```
CREATE VIEW vista AS SELECT text 'Hello World' AS hello;
```

추가로, VIEW를 수정할 때 정확히 일치하는 데이터타입 및 컬럼명 세트를 가진 쿼리로만 수정이 가능하다.

###           3-3. 권한

테이블을 참고하는 권한은 VIEW 소유자의 권한을 따른다. 그러나 view 내에서 호출되는 함수는 쿼리에서 직접 호출된 것과 동일한 권한을 가진다. 그래서 VIEW user는 VIEW에서 사용 중인 모든 함수들에 대한 권한을 보유하고 있어야 한다.

참고

[https://www.postgresql.org/docs/8.0/sql-createview.html](https://www.postgresql.org/docs/8.0/sql-createview.html)