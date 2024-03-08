## PostgreSQL 제약조건 (Constrant)란?

데이터베이스는 데이터 타입 외에 제약조건들을 통해 데이터의 무결성을 유지한다.

제약조건에는 여러 가지 종류가 있으며 DMBS에 마다 다양하지만, 이번 포스트는 PostgreSQL의 5가지 제약 조건들을 설명하겠다.

1\. Primary Keys(PK)

2\. Foreign Keys(FK)

3\. Check

4\. Not-null

5\. Unique

## 1\. Primary Keys (PK)

-   Primary Keys는 테이블의 각 ROW를 구분하는 유니크한 컬럼 혹은 컬럼의 조합이다.
-   Not- null, Unique Constraints의 조합이다. 테이블인 단 1개의 PK만 가질 수 있다.
-   PK 생성 시 Postgresql은 B-tree 인덱스를 자동으로 부여한다.
-   B-tree 인덱스를 사용하기 때문에 컬럼의 조합으로 PK를 설정 시 순서가 중요하다. (상세 내용은 다음 포스트에서 확인이 가능하다.)

[\[PostgreSQL\] B-tree 인덱스의 원리 및 특징](https://github.com/junhkang/postgresql/blob/main/B-tree%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%EC%9D%98%20%EC%9B%90%EB%A6%AC%20%EB%B0%8F%20%ED%8A%B9%EC%A7%95.md)

###           1-1. 테이블 생성 시 PK 부여

```
-- 단일 설정
CREATE TABLE po_headers (
	po_no INTEGER PRIMARY KEY,
	vendor_no INTEGER,
	description TEXT,
	shipping_address TEXT
);
-- 복합설정
CREATE TABLE TABLE (
	column_1 data_type,
	column_2 data_type,
	… 
        PRIMARY KEY (column_1, column_2)
);
```

###           1-2. 기존 테이블에 PK 속성 부여

```
ALTER TABLE table_name ADD PRIMARY KEY (column_1, column_2);

-- 자동 증가하는 PK 설정
ALTER TABLE vendors ADD COLUMN ID SERIAL PRIMARY KEY;
```

###           1-3. PK 삭제

```
ALTER TABLE table_name DROP CONSTRAINT primary_key_constraint;
```

## 2\. Foreign Keys

> 외래키(Foreign Keys)는 다른 테이블의 Primary Key에 참조된 컬럼 혹은 컬럼의 조합이다.  
> 다른 테이블과의 관계에 따라 다양한 FK를 가질 수 있다.   
> 외래키 설정 후 parent 컬럼의 상태에 따라 다음 액션을 지정할 수 있다.  
>   
> a. SET NULL  
> b. SET DEFAULT  
> c. RESTRICT  
> d. NO ACTION  
> e. CASCADE  
>   
> Postgresql에서는 다음 5가지 parent데이터 변경에 대한 옵션을 제공한다. 다음 FK 설정 예제는 parent데이터가 삭제될 경우 종속된 데이터를 null로 업데이트한다. Cascade의 경우 parent 데이터가 삭제될 경우 종속된 데이터들도 같이 전체 삭제된다.

###           2-1. FK 생성

```
CREATE TABLE customers(
   customer_id INT GENERATED ALWAYS AS IDENTITY,
   customer_name VARCHAR(255) NOT NULL,
   PRIMARY KEY(customer_id)
);

CREATE TABLE contacts(
   contact_id INT GENERATED ALWAYS AS IDENTITY,
   customer_id INT,
   contact_name VARCHAR(255) NOT NULL,
   phone VARCHAR(15),
   email VARCHAR(100),
   PRIMARY KEY(contact_id),
   CONSTRAINT fk_customer
      FOREIGN KEY(customer_id) 
	  REFERENCES customers(customer_id)
		-- 다음 설정은 parent 데이터가 삭제될시 참조데이터를 null로 업데이트한다.
	  ON DELETE SET NULL
);
```

## 3\. Check

Boolean 타입으로 컬럼에 제약을 줘서 insert 혹은 update 전에 테이블에 유효한 데이터인지를 검증한다.

(맞지 않는다면 Constraint violation error를 발생시킨다.)

###           3-1. Check Constraint 부여한 채로 테이블 생성

```
CREATE TABLE employees (
	id SERIAL PRIMARY KEY,
	first_name VARCHAR (50),
	last_name VARCHAR (50),
	birth_date DATE CHECK (birth_date > '1900-01-01'),
	joined_date DATE CHECK (joined_date > birth_date),
	salary numeric CHECK(salary > 0)
);
```

다음 테이블에는 2가지 Constraint이 걸려있다. birth\_date는 1900-01-01 이후 날짜여야 하며, joined\_date는 birth\_date 이후 날짜여야만 한다.

###           3-2. 기존에 테이블에 Check Constraint 추가

```
ALTER TABLE prices_list 
ADD CONSTRAINT price_discount_check 
CHECK (
	price > 0
	AND discount >= 0
	AND price > discount
);
```

## 4\. Not null

특정 컬럼에 Null 제약을 줘서 insert 혹은 update시 해당 값이 null이 아닌지를 검증한다.

###           4-1. Not null Constraint 부여

```
CREATE TABLE table_name(
   ...
   column_name data_type NOT NULL,
   ...
);
```

check와 Not null을 동시에 적용 가능하다.

```
CREATE TABLE invoices(
  id SERIAL PRIMARY KEY,
  product_id INT NOT NULL,
  qty numeric NOT NULL CHECK(qty > 0),
  net_price numeric CHECK(net_price > 0) 
);
```

###           4-2. 기존 테이블에 not null 속성을 추가

            해당 컬럼에 null 값이 없어야 적용 가능하다.

```
ALTER TABLE table_name
ALTER COLUMN column_name SET NOT NULL;

-- 여러개
ALTER TABLE table_name
ALTER COLUMN column_name_1 SET NOT NULL,
ALTER COLUMN column_name_2 SET NOT NULL,
...;
```

종종 두 컬럼 중 적어도 1개는 null이 아니게 설정해야 할 경우가 있다.

```
CREATE TABLE users (
 id serial PRIMARY KEY,
 username VARCHAR (50),
 password VARCHAR (50),
 email VARCHAR (50),
 CONSTRAINT username_email_notnull CHECK (
   NOT (
     ( username IS NULL  OR  username = '' )
     AND
     ( email IS NULL  OR  email = '' )
   )
 )
);
```

## 5\. Unique

insert 혹은 update 시 해당 컬럼에 유니크한 값이 들어있는지를 확인한다. 단일 컬럼 혹은 컬럼의 조합으로 설정이 가능하며 Unique index가 자동으로 부여된다.

###           5-1. Unique Constraint 적용한 테이블 생성

```
CREATE TABLE person (
	id SERIAL PRIMARY KEY,
	first_name VARCHAR (50),
	last_name VARCHAR (50),
	email VARCHAR (50) UNIQUE
);
```

컬럼의 조합에 설정하고 싶을 때는 

```
CREATE TABLE table (
    c1 data_type,
    c2 data_type,
    c3 data_type,
    UNIQUE (c2, c3)
);
```

###           5-2. 기존 테이블에 Unique Constraint 추가

```
CREATE UNIQUE INDEX CONCURRENTLY equipment_equip_id 
ON equipment (equip_id);

ALTER TABLE equipment 
ADD CONSTRAINT unique_equip_id 
UNIQUE USING INDEX equipment_equip_id;
```

참고

[https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-unique-constraint/](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-unique-constraint/)

[https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-check-constraint/](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-check-constraint/)