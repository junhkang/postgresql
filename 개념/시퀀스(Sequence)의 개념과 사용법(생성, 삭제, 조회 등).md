## 1\. 시퀀스(Sequence)의 사용

###           1-1. 생성, 삭제, 조회

```
-- 101부터 시작하는 기본 시퀀스 생성
CREATE SEQUENCE serial START 101;
-- 시퀀스 다음값 조회
SELECT nextval('serial');
-- 시퀀스 현재값 조회
select currval('serial');
-- 시퀀스 삭제
DROP SEQUENCE serial;

-- 시퀀스로 INSERT하기
INSERT INTO distributors VALUES (nextval('serial'), 'nothing');
-- COPY FROM 후에 시퀀스 시작값 변경하기
BEGIN;
COPY distributors FROM 'input_file';
SELECT setval('serial', max(id)) FROM distributors;
END;


-- Synopsis
CREATE [ TEMPORARY | TEMP ] SEQUENCE [ IF NOT EXISTS ] 이름
    [ AS 자료형 ]
    [ INCREMENT [ BY ] 증가값 ]
    [ MINVALUE 최소값 | NO MINVALUE ] [ MAXVALUE 최대값 | NO MAXVALUE ]
    [ START [ WITH ] 시작값 ] [ CACHE 캐시 ] [ [ NO ] CYCLE ]
    [ OWNED BY { 테이블이름.칼럼이름 | NONE } ]
```

###           1-2. 사용 중인 시퀀스 확인

```
select n.nspname as sequence_schema, 
          c.relname as sequence_name,
          u.usename as owner
from pg_class c 
     join pg_namespace n on n.oid = c.relnamespace
     join pg_user u on u.usesysid = c.relowner
where c.relkind = 'S'
     and u.usename = current_user;
```

## 2\. 시퀀스 생성시 상세 옵션

**\- TEMPORARY or TEMP**

        현재 세션에서만 사용하는 시퀀스를 생성하며, 세션이 종료되면 시퀀스는 자동 삭제된다.

**\- IF NOT EXISTS**

        동일명의 시퀀스가 있다면 알림만 보여주고 작업은 생략한다.

**\- AS 자료형**

        시퀀스의 자료형을 설정한다. smallint, integer, and bigint 세 종류로 bigint형이 기본값이다.

**\- INCREMENT BY 증가값**

        시퀀스 채번식 증가값을 더하여 구한다. 양수라면 증가 시퀀스, 음수면 감소시퀀스이다. default는 1이다.

**\- MINVALUE / NO MINVALUE**

        해당 시퀀스의 최솟값을 설정한다. 기본 값은 1이며, 감소 시퀀스의 경우 해당 자료형의 최솟값이다.

**\- MAXVALUE / NO MAXVALUE**

        해당 시퀀스의 최댓값을 설정한다. 기본값은 해당 자료형의 최댓값이다. 감소 시퀀스의 경우 -1이다.

**\- START WITH**

        시퀀스의 시작값을 설정한다. 기본값은 증가시퀀스의 경우 최솟값이며, 감소 시퀀스는 최댓값이다.

**\- CACHE**

        시퀀스 채번을 빠르게 하기 위해 메모리에서만 처리하는 캐시값이다.

        기본값은 1이며 캐시를 사용하지 않고 매번 디스크를 사용함을 뜻한다.

**\- CYCLE / NO CYCLE**

        시퀀스가 최댓값/최솟값에 도달했을 때 순환하며 다시 시작한다.

        NO CYCLE의 경우 최솟값/최댓값에 도달하면 에러로 처리한다.

        기본 설정은 NO CYCLE이다.

**\- OWNED BY / OWNED BY NONE**

        해당 칼럼과 시퀀스의 의존관계를 생성한다.

        테이블/칼럼이 삭제되면 시퀀스는 자동으로 삭제되며, 테이블/시퀀스의 소유자가 같아야 한다.

        OWNED BY NONE이 기본 옵션이며 어떠한 의존관계도 없는 상태이다.

## 3\. 시퀀스의 개념

CREATE SEQUENCE는 일련번호 생성기인 SEQUENCE를 생성한다. 시퀀스를 생성하면, 내부적으로 지정한 명칭으로 단일 로우의 특수 테이블을 만들고, 그 로우의 값을 초기화한다. 시퀀스는 특수 용도의 "테이블" 이기 때문에 다음과 같은 쿼리를 사용할 수 있지만,

```
SELECT * FROM seq_name;
```

<p align="center"><img src="/img/seq.png"/></p>

이 테이블을 직접 조작하면 안되며, 결과에서 last\_value (nextval)은 "실행 시점"의 최신 값이다. (그 후로는 다른 세션에서 호출되어 바뀌었을 수도 있는 값이다.)

## 4\. 시퀀스의 특징

-   시퀀스명은 시퀀스 / 테이블 / 인덱스 / 뷰 명과 겹칠 수 없다.
-   시퀀스는 기본적으로 bigint형으로 계산한다. (-9223372036854775808 ~ 9223372036854775807)
-   nextval, setval은 취소가 되지 않는다. 때문에 연속되지 않는 일련번호를 처리하는 용도로 사용할 수 없고, 락을 통해 구현은 가능하지만 시퀀스를 사용하지 않는 것보다 더 높은 코스트가 소모된다. 특히 동시 접속 많은 서비스라면 더 비효율적이다.

## 5\. CACHE 옵션

###           5-1. CACHE 옵션이란?

CACHE 옵션을 사용하는 경우 다중 세션에서 시퀀스가 순차적으로 채번 되지 않는다. 각세션 별로 캐시값만큼 생략 후 그다음 세션시작번호를 채번 한다. (last\_Value +캐시-1) 즉 시퀀스 캐시는 한 세션 내의 캐시를 의미한다. 이를 통해 다른 세션에서 이전 세션에서 미사용 한 일련번호를 사용할 수 없도록 하고, 그렇기 때문에 시퀀스가 연속적이지 않는 경우가 종종 발생한다.

###           5-2. CACHE 옵션 사용시 주의 사항 

캐시를 사용하면 시퀀스가 크다는 의미가 꼭 나중에 생성된 데이터라는 보장이 없다. 그렇기에 순차적 시퀀스를 사용해야 하는 경우라면 캐시값을 1로 설정해야 만한다. 예를 들어

> 캐시값은 10으로 설정할 경우  
> \- A 세션이 1-10 시퀀스를 선점  
> \- B 세션이 11-20 시퀀스를 선점  
> 이 상태에서 B세션에서 더 빨리 시퀀스를 호출하더라도 A세션의 1~10 시퀀스보다 낮은 값을 가질 수 없기 때문이다.  
>   
> (CACHE를 사용하더라도 유니크한 시퀀스를 채번함에는 전혀 지장이 없기에, 순차적 의미로써 시퀀스를 사용할 것이 아니라면 사용하여도 무관하다.)

참고[https://www.postgresql.kr/docs/10/sql-createsequence.html](https://www.postgresql.kr/docs/10/sql-createsequence.html)