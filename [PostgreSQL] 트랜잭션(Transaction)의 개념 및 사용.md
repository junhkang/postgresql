## 1\. 트랜잭션(Transaction)이란?

트랜잭션은 데이터베이스에서 실행되는 일련의 작업들이다. 트랜잭션은 데이터베이스의 무결성 및 작업 간 충돌방지, 데이터 검증을 위해 필수적인 요소이다. 단순한 DML 작업의 롤백 용도뿐 아니라, 대용량 데이터 처리의 무결성, 에러발생 시, 여러 유저의 동시작업 등에서 사용된다.

## 2\. 트랜잭션 적용

트랜잭션을 사용하는 커맨드 예제이다.

```
--COMMIT 혹은 ROLLBACK으로 트랜잭션을 종료하지 않으면, 해당 업데이트 건은 데이터베이스에 적용되지 않는다.

BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
COMMIT;
```

트랜잭션 COMMIT 전에 다른 유저가 동일한 row의 balance를 업데이트하려고 한다면, 그전 트랜잭션이 commit 혹은 rollback 되는 것을 대기해야하며 이를 lock 상태라고 한다. (Lock의 개념 및 상세는 다음 포스트에 상세하게 정리되어 있다.)

[2023.09.11 - \[Postgresql\] - Postgresql Lock이란? (조회 및 kill, Dead lock)](https://junhkang.tistory.com/4)

 [Postgresql Lock이란? (조회 및 kill, Dead lock)

1\. Lock 확인방법 SELECT PSAT.RELNAME, PL.LOCKTYPE, PL.PID, PL.MODE, PL.GRANTED FROM PG\_LOCKS PL, PG\_STAT\_ALL\_TABLES PSAT WHERE PL.RELATION = PSAT.RELID 2. Lock Kill 방법 SELECT PG\_CANCEL\_BACKEND(\[PID\]) SELECT PG\_TERMINATE\_BACKEND(\[PID\]) Lock 리스

junhkang.tistory.com](https://junhkang.tistory.com/4)

> PostgreSQL에서는 모든 SQL 구문이 트랜잭션 안에서 실행되며, BEGIN을 명시적으로 실행하지 않아도 SQL 명령어를 실행시키면  BIGIN이 되었다고 간주한다. Postgresql의 툴로 사용하는 PgAdmin, Dbeaver, DataGrip 등의 툴에는 각각 트랜잭션을 Manual or Auto로 설정할 수 있다. Manual의 경우 트랜잭션을 commit, rollback을 명시적으로 선언해주지 않을 경우 트랜잭션이 계속 유지되며, Auto의 경우 명시적으로 선언하지 않아도 SQL이 성공적으로 실행된 후 즉시 commit이 실행된 것으로 간주한다.

## 3\. 트랜잭션의 4가지 특성

###      3-1. 원자성 Atomic

-   작업이 최종적으로는 하나로 취급된다.
-   동일 트랜잭션 내의 작업은 전부 취소 혹은 전부 작업된다.
-   트랜잭션 내의 작업에 오류가 발생하면 해당 트랜잭션 내의 모든 작업이 취소된다. 

###     3-2. 내구성 durability

-   트랜잭션이 정상적으로 끝났을 경우, 변경완료된 자료에는 어떠한 간섭도 없이 저장되어야 하고, 손상이 되면 안 된다.
-   트랜잭션 완료까지 간섭을 없애기 위해, 데이터베이스에서는 트랜잭션이 정상종료됨을 전달받기 전에 트랜잭션에서 발생하는 모든 작업들을 영구저장장치(예, 하드디스크)에 기록해 둔다.

###      3-3. 고립성 isolation

-   트랜잭션은 다른 트랜잭션에 의해 간섭받지 않아야 한다.
-   여러 개의 트랜잭션 발생 시, 각각의 트랜잭션은 다른 트랜잭션의 변경 중인 데이터를 참조, 간섭할 수 없어야 한다.

###     3-4. 정합성 consistency

-   트랜잭션은 각강의 명령을 데이터베이스 원데이터에 영향을 주는 것이 아니라 트랜잭션 영역 안에 있는 모든 작업이 끝났을 때, 한 번에 그 변경사항이 데이터베이스에 적용된다.

## 4\. SAVEPOINT

성공/실패에 따라  트랜잭션 내의 작업은 일괄 처리되지만 SAVEPOINT 명령을 사용하여 부분 커밋을 하여 좀 더 유연하게 처리가 가능하다.

-   savepoint로 취소작업을 진행한 뒤에도 트랜잭션 내 작업을 계속 진행할 수 있다.
-   savepoint가 필요 없다고 판단될 시 삭제하여 시스템 자원을 늘릴 수 있다.
-   savepoint로 돌아갈 경우 그지점 이후의 savepoint들도 모두 롤백된다.

```
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
ROLLBACK TO my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Wally';
COMMIT;
```

참고 

[https://www.postgresql.kr/docs/9.6/tutorial-transactions.html](https://www.postgresql.kr/docs/9.6/tutorial-transactions.html)