## \* 가장 보편적으로 쓰이는 간단한 history 저장 트리거 생성 예제

특정 테이블에 insert, update가 수행될 경우 무조건 내역에 “insert”를 하는 간단한 트리거 생성 예제이다.

#### 1-1. 함수를 실행할 트리거 생성

```
create trigger trigger_save_history
after insert or update on A
for each row
execute procedure trigger_insert();
```

#### 1-2. 실제 insert문이 실행되는 함수 

```
CREATE OR REPLACE FUNCTION trigger_insert()
returns trigger
AS $$
DECLARE
BEGIN
    insert into B
        (id, values, date)
    values
        (new.id, new.values, current_timestamp());
    return NULL;
END; $$
LANGUAGE 'plpgsql';
```

하지만 특정 table에 insert, delete, update에 따라 서로 다른 테이블에 이력을 보관하거나, 기존 이력을 업데이트하는 등

서로 다른 행위에 대한 트리거가 필요한 경우가 있다.

그럴경우 다음 예제처럼 TG\_OP을 통해 데이터베이스에 UPDATE, INSERT, DELETE를 분기하는 것이 가능하다. 또한 old, new를 통해 delete의 삭제 전 값, update의 업데이트 전, 후 값을 각각 사용할 수 있다.

## \* 수행되는 SQL에 따라 별도 history 저장 트리거 생성 예제

```
CREATE OR REPLACE FUNCTION TRIGGER_INSERT()
    RETURNS TRIGGER
    LANGUAGE PLPGSQL
AS
$function$
BEGIN
    IF (TG_OP = 'UPDATE') THEN
        insert into update_history
            (id, values, date)
        values
            (new.id, new.values, current_timestamp());
        return NULL;
    ELSIF (TG_OP = 'INSERT') THEN
        insert into insert_history
            (id, values, date)
        values
            (new.id, new.values, current_timestamp());
        return NULL;

    ELSIF (TG_OP = 'DELETE') THEN
        insert into delete_history
            (id, values, date)
        values
            (new.id, old.values, current_timestamp());
        return NULL;
    END IF;

    RETURN NULL;

END

$function$
;
```

## 1\. 트리거(Trigger) 란 무엇일까?

특정 SQL이 실행될 때 자동으로 실행되는 객체이다. 테이블의 변경 감지 및 로깅에 많이 사용되며, 데이터만 전달 후 연산 자체를 DB에 넘기기에 부하 및 확장성을 고려하여 적용하여야한다. 단일 함수를 생각할 경우 서버 로직보다 트리거 함수가 빠르고 쉽게 적용되는 경우가 있지만, 트리거가 너무 많은 경우 문제발생 원인파악과 유지보수가 힘들다. 또한 코드가 복잡하여 작성자 외 트리거 내용을 분석하기 힘들다. 또한, 트리거 함수가 동작할 때, 트리거의 영향을 받는 모든 개체들은 트랜잭션이 열린 상태로 유지된다. 즉, 트리거 연산 시간만큼 트랜잭션 lock 타임도 길어진다. 부적합한 트리거는 성능을 크게 저하시킬 수 있다. 그래서 복잡한 로직을 처리하기보다는, 간단한 실행에 사용하고, 트리거 작성 시 정확한 목적과 동작방식을 문서화하는 것이 중요하다.

> \- 특정 SQL이 실행될 때 자동으로 실행되는 객체이다  
> \- 트리거 내에서 COMMIT / ROLLBACK 사용불가하다.  
> \- 쿼리문장별, ROW별로 실행가능하다.  
> \- OLD / NEW 를 통해 실행된 쿼리의 이전, 이후값 접근 가능하다.  
> \- 특정 PROCEDURE / FUNCTION을 실행시킬 수 있다.

## 2\. 프로시저(Procedure)와 함수(Function)는 무엇일까?

자주 사용되는 특정기능을 모듈화 시켜놓은 것을 함수(function) 또는 프로시저(procedure)라고 하는 것을 알고 있다. PostgreSQL에서 정확히 어떻게 사용되며 차이점은 무엇일까?

### 2-1. FUNCTION

\- 주로 클라이언트에서 실행 (어플리케이션에서 호출)  
\- 리턴값 필수  
\- 저장해서 쓰는 프로시져 ( 인자만 변경하여 자유롭게 재사용 가능 )\- 반복작업을 줄여주며 여러개의 쿼리문을 묶어서 실행 가능  
\- 특정 계산을 수행

### 2-2. PROCEDURE

\- 리턴값은 필요에 따라 반환  
\- DB서버에서 실행(처리속도 빠름)- 미리 컴파일된 SQL 명령어의 집합  
\- 특정 작업을 수행

### 2-3. 차이점 비교

| 함수(Function)              | 프로시저(Procedure) |
| ------------------------- | --------------- |
| 특정 계산 수행                  | 특정 작업 수행        |
| 리턴값 필수 O                  | 리턴값 필수 X        |
| 리턴값이 1개여야만 함              | 리턴값이 여러개일 수 있음  |
| Client에서 실행 (어플리케이션에서 호출) | Server에서 실행(DB) |
| 단독 문장 구성 불가               | 단독 문장 구성 가능     |
| 수식내에서만 사용 가능              | 수식 내에서 사용 불가    |