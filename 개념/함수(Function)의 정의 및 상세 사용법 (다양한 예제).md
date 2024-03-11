## 1\. PostgreSQL Function이란?

-   SQL 함수는 임의의 SQL문들을 실행하고 마지막 쿼리의 결과를 반환한다. 단순한 형태의 함수는 마지막 쿼리의 첫 번째 row가 리턴된다. (order by 를 사용하지 않는 경우 다중 row의 첫 번째 행은 별도 정의되지 않기에 결과 row가 매번 다를 수 있다.)
-   마지막 쿼리가 row를 하나도 반환하지 않을 경우 null이 리턴된다.
-   SQL 함수는 함수의 리턴 유형을 특정 타입의 집합 (SET)으로 선언하거나, 테이블로 선언하여 반환할 수 있다. 이 경우에는 마지막 쿼리의 모든 ROW가 리턴된다.
-   SQL함수의 body는 세미콜론(;)으로 구분된 SQL구문의 집합이어야만 한다.
-   마지막 구문 뒤의 세미콜론(;)은 생략하여도된다.
-   함수가 void를 리턴하는 것으로 선언되지 않았다면, 마지막 구문은 반환절이 존재하는 select, insert, update, delete 여야만 한다.
-   모든 종류의 SQL 언어의 명령 모음은 패키징 되어 함수로 정의될 수 있다.
-   select쿼리 외에도 insert, update, delete, merge 등의 데이터 수정쿼리 및 기타 SQL을 포함할 수 있지만, 트랜잭션 제어 명령( ex. commit, savepoint) 및 vacutaion 등의 일부 유틸리티 명령은 사용할 수 없다.
-   SQL이 작동은 하지만 특정 값을 리턴하지 않는 SQL 함수를 정의하고 싶다면, void를 리턴하는 것으로 정의할 수 있다.

### ▶ 1-1. Function 간단 예시

다음은 emp 테이블에서 음수의 salary를 삭제하는 함수이다.

```
CREATE FUNCTION clean_emp() RETURNS void AS '
    DELETE FROM emp
        WHERE salary < 0;
' LANGUAGE SQL;

SELECT clean_emp();

 clean_emp
-----------

(1 row)
```

리턴 타입이 없는 문제를 해결하기 위해 프러시저로도 실행이 가능하다.

```
CREATE PROCEDURE clean_emp() AS '
    DELETE FROM emp
        WHERE salary < 0;
' LANGUAGE SQL;

CALL clean_emp();
```

이처럼 단순한 케이스에서 리턴값이 없는 함수와 프러시저는 작성 스타일 외에는 없어보인다. 하지만 프로시져는 transaction 컨트롤 등 추가적인 기능을 제공한다. 또한 프러시저가 SQL 표준이므로 return 값이 없는 경우 프러시저를 사용해야 한다.

함수와 프러시저의 차이는 다음 포스트에서 확인이 가능하다.

[\[PostgreSQL\] Trigger, Procedure, Function (history 관리하기)](https://github.com/junhkang/postgresql/blob/main/Trigger%2C%20Procedure%2C%20Function%20(history%20%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0).md)

## 2\. SQL Function Arguments

-   SQL의 함수 인자는 이름이나 숫자를 사용하여 함수 body에서 참조할 수 있다.
-   이름을 사용하려면 함수 인자에 이름이 있는 것으로 선언한 다음 함수 본문에 해당 이름을 사용하면 된다.
-   인자 이름이 함수 내 SQL 테이블의 칼럼과 하나라도 일치한다면, 해당 칼럼이 지정 인자 보다 우선순위를 가진다.
-   이를 극복하기 위해서 인자의 이름을 함수자체의 이름을 명시한 것으로 지정한다. (ex function\_name.argument\_name) 이 상태에서 칼럼명과 다시 충돌이 난다면(function\_name.argument\_name 같은 칼럼명이 있다면), 실제 칼럼이 또다시 우선순위를 가진다.)
-   $n형태의 오래된 방식의 숫자형태의 인자 접근법에 따르면, $1는 첫 번째 인자를 말하며, $2는 두 번째 $n은 n번째 인자를 의미한다.
-   SQL 함수 인자는 식별자가 아닌 데이터 값으로만 사용할 수 있다.

```
-- 가능
INSERT INTO mytable VALUES ($1);
-- 불가
INSERT INTO $1 VALUES (42);
```

## 3\. PostgreSQL Function 예제

### ▶ 3-1. 기본 타입 반환 Function

가장 간단한 인자가 없는 integer(기본 타입)을 반환하는 함수이다.

```
CREATE FUNCTION one() RETURNS integer AS $$
    SELECT 1 AS result;
$$ LANGUAGE SQL;

-- Alternative syntax for string literal:
CREATE FUNCTION one() RETURNS integer AS '
    SELECT 1 AS result;
' LANGUAGE SQL;

SELECT one();

 one
-----
   1
```

### ▶ 3-2. 기본 타입을 인자로 사용하는 Function

integer(기본 타입)을 인자로 사용하는 함수이다. 

```
-- argument name(인자 명)을 사용하는 경우
CREATE FUNCTION add_em(x integer, y integer) RETURNS integer AS $$
    SELECT x + y;
$$ LANGUAGE SQL;

-- 숫자형태의 인자를 사용하는 경우
CREATE FUNCTION add_em(integer, integer) RETURNS integer AS $$
    SELECT $1 + $2;
$$ LANGUAGE SQL;

SELECT add_em(1, 2) AS answer;

 answer
--------
      3
```

### ▶ 3-3. 이름을 arguments로 사용하는 Function

은행 계좌에서 금액을 차감하는 함수 예제이다. (argument명과 테이블 칼럼이 일치하는 경우)

```
CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS numeric AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno;
    SELECT 1;
$$ LANGUAGE SQL;
```

컬럼 명 ccountno와 인자 accountno의 명칭이 동일하기에 함수 명을 명시해서 tf1.accountno로 사용해야 한다.

지금은 1을 반환하지만 좀 더 유용하게, 잔액을 반환하게 변경하면

```
CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS numeric AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno;
    SELECT balance FROM bank WHERE accountno = tf1.accountno;
$$ LANGUAGE SQL;
```

혹은 returning을 사용하면

```
CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS numeric AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno
    RETURNING balance;
$$ LANGUAGE SQL;
```

마지막 select 혹은 Returning이 함수에서 정의된 Return 타입과 동일하지 않다면, PostgreSQL에서 자동으로 return 타입을 맞춰 캐스팅한다.

### ▶ 3-4. 복합 유형의 Function

복합적 타입을 인자로 사용하는 함수를 작성할 때, 정확히 어떤 인자를 사용하는지 뿐만 아니라 해당 인자의 속성(필드)도 같이 정의해야 한다. 다음은 단일 직원 정보를 보여주는 함수이다.

```
CREATE FUNCTION new_emp() RETURNS emp AS $$
    SELECT ROW('None', 1000.0, 25, '(2,2)')::emp;
$$ LANGUAGE SQL;

-- 1. row 자체로 select
SELECT new_emp();

         new_emp
--------------------------
 (None,1000.0,25,"(2,2)")
 
 -- 2. 테이블 형태로 select
 SELECT * FROM new_emp();
 
 name | salary | age | cubicle
------+--------+-----+---------
 None | 1000.0 |  25 | (2,2)
 
 -- 3. 속성별 select
 SELECT (new_emp()).name;

 name
------
 None
```

### ▶ 3-5. 출력 매개변수(Output Parameters)를 활용한 Function 

다음과 같이 출력 매개변수를 정의하여 원하는 출력값을 조합하는 방식의 함수도 생성 가능하다. 이 경우 출력 매개변수의 이름은 각 칼럼 명을 결정한다.

```
CREATE FUNCTION sum_n_product (x int, y int, OUT sum int, OUT product int)
AS 'SELECT x + y, x * y'
LANGUAGE SQL;

 SELECT * FROM sum_n_product(11,42);
 sum | product
-----+---------
  53 |     462
(1 row)
```

출력 매개변수(Output Parameters)를 활용한 프러시저

함수와 동일하게 출력 매개변수를 사용할 수 있지만, 호출 시에 출력 매개변수를 포함시켜야 하는 점이 다르다.

```
CREATE PROCEDURE tp1 (accountno integer, debit numeric, OUT new_balance numeric) AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tp1.accountno
    RETURNING balance;
$$ LANGUAGE SQL;

-- NULL로 파라미터를 포함시켜야한다.
CALL tp1(17, 100.0, NULL);
```

### ▶ 3-6. 다수의 인자를 받는 Function

같은 데이터 타입의 다수의 인자를 VARIADIC을 명시한 배열 형태로 받는 함수를 정의할 수 있다. 

```
CREATE FUNCTION mleast(VARIADIC arr numeric[]) RETURNS numeric AS $$
    SELECT min($1[i]) FROM generate_subscripts($1, 1) g(i);
$$ LANGUAGE SQL;

SELECT mleast(10, -1, 5, 4.4);
 mleast
--------
     -1
(1 row)
```

### ▶ 3-7. Default 값을 인자로 가지는 Function

함수는 일부 혹은 모든 입력 인수에 default 값을 설정할 수 있다. 순차적으로 적용되기 때문에 default 설정이 된 변수 뒤의 변수들은 모두 default 설정이 되어야 한다. (ex. a, b, c의 인자 중 b가 default 값이 정의되면 c도 default 값 정의가 되어야만 한다.)

```
CREATE FUNCTION foo(a int, b int DEFAULT 2, c int DEFAULT 3)
RETURNS int
LANGUAGE SQL
AS $$
    SELECT $1 + $2 + $3;
$$;

SELECT foo(10, 20, 30);
 foo
-----
  60
(1 row)

SELECT foo(10, 20);
 foo
-----
  33
(1 row)

SELECT foo(10);
 foo
-----
  15
(1 row)

SELECT foo();  -- fails since there is no default for the first argument
ERROR:  function foo() does not exist
```

### ▶ 3-8. 테이블 자원으로써의 Function

함수결과 칼럼을 일반 테이블의 열과 동일하게 사용 가능하다. (SETOF를 사용하지 않았기 때문에 1개의 열만 선택하였다.)

```
CREATE TABLE foo (fooid int, foosubid int, fooname text);
INSERT INTO foo VALUES (1, 1, 'Joe');
INSERT INTO foo VALUES (1, 2, 'Ed');
INSERT INTO foo VALUES (2, 1, 'Mary');

CREATE FUNCTION getfoo(int) RETURNS foo AS $$
    SELECT * FROM foo WHERE fooid = $1;
$$ LANGUAGE SQL;

SELECT *, upper(fooname) FROM getfoo(1) AS t1;

 fooid | foosubid | fooname | upper
-------+----------+---------+-------
     1 |        1 | Joe     | JOE
(1 row)
```

### ▶ 3-9. 집합을 반환하는 Function

SETOF some type 형태를 반환하는 함수로 선언되면 함수가 출력하는 각 행은 결과 set의 각 요소로 반환된다.

```
CREATE FUNCTION getfoo(int) RETURNS SETOF foo AS $$
    SELECT * FROM foo WHERE fooid = $1;
$$ LANGUAGE SQL;

SELECT * FROM getfoo(1) AS t1;

 fooid | foosubid | fooname
-------+----------+---------
     1 |        1 | Joe
     1 |        2 | Ed
(2 rows)
```

또한 output parameters에 정의된 칼럼에 맞춰 여러 개의 행을 반환할 수도 있다.

```
CREATE TABLE tab (y int, z int);
INSERT INTO tab VALUES (1, 2), (3, 4), (5, 6), (7, 8);

CREATE FUNCTION sum_n_product_with_tab (x int, OUT sum int, OUT product int)
RETURNS SETOF record
AS $$
    SELECT $1 + tab.y, $1 * tab.y FROM tab;
$$ LANGUAGE SQL;

SELECT * FROM sum_n_product_with_tab(10);
 sum | product
-----+---------
  11 |      10
  13 |      30
  15 |      50
  17 |      70
(4 rows)
```

### ▶ 3-10. 트리구조 요소를 나열하기 위한 SET 반환 Function

```
SELECT * FROM nodes;
   name    | parent
-----------+--------
 Top       |
 Child1    | Top
 Child2    | Top
 Child3    | Top
 SubChild1 | Child1
 SubChild2 | Child1
(6 rows)

CREATE FUNCTION listchildren(text) RETURNS SETOF text AS $$
    SELECT name FROM nodes WHERE parent = $1
$$ LANGUAGE SQL STABLE;

SELECT * FROM listchildren('Top');
 listchildren
--------------
 Child1
 Child2
 Child3
(3 rows)

SELECT name, child FROM nodes, LATERAL listchildren(name) AS child;
  name  |   child
--------+-----------
 Top    | Child1
 Top    | Child2
 Top    | Child3
 Child1 | SubChild1
 Child1 | SubChild2
(5 rows)
```

### ▶ 3-11. 테이블을 리턴하는 Function

집합을 반환하는 다른 방법은 RETURNS TABLE을 사용하는 것이다. (SETOF와 동일) 이 표기법은 최근 버전 PostgreSQL 표준에 정의되어 있기 때문에 SETOF보다 뛰어날 수 있다.

```
CREATE FUNCTION sum_n_product_with_tab (x int)
RETURNS TABLE(sum int, product int) AS $$
    SELECT $1 + tab.y, $1 * tab.y FROM tab;
$$ LANGUAGE SQL;
```

OUT, INOUT 인자를 사용할 수 없고, 모든 결과 칼럼을 TABLE에 정의해야 한다.

### ▶ 3-12. 다형성(Polymorphic)  Function

인자, 반환값의 형태에 관계없이 함수를 정의할 수 있다.

```
CREATE FUNCTION anyleast (VARIADIC anyarray) RETURNS anyelement AS $$
    SELECT min($1[i]) FROM generate_subscripts($1, 1) g(i);
$$ LANGUAGE SQL;

SELECT anyleast(10, -1, 5, 4);
 anyleast
----------
       -1
(1 row)

SELECT anyleast('abc'::text, 'def');
 anyleast
----------
 abc
(1 row)

CREATE FUNCTION concat_values(text, VARIADIC anyarray) RETURNS text AS $$
    SELECT array_to_string($2, $1);
$$ LANGUAGE SQL;

SELECT concat_values('|', 1, 4, 2);
 concat_values
---------------
 1|4|2
(1 row)
```

참고 + 예제 출처

[https://www.postgresql.org/docs/16/xfunc-sql.html#XFUNC-SQL-BASE-FUNCTIONS](https://www.postgresql.org/docs/16/xfunc-sql.html#XFUNC-SQL-BASE-FUNCTIONS)