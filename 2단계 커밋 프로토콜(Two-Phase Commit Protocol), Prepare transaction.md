## 1\. 2단계 커밋 프로토콜(two-phase commit (2PC))

PostgreSQL은 two-phase commit (2PC) 프로토콜을 지원한다. 다수의 분산 시스템 환경에서 모든 데이터베이스가 정상적으로 수정되었음을 보장하는 두 단계 커밋 프로토콜로 분산 트랜잭션에 참여한 모든 데이터베이스가 모두 함께 커밋되거나 롤백되는 것을 보장한다. PostgreSQL의 2단계 트랜잭션은 외부 트랜잭션 관리 시스템에서 사용하기 위해 존재하며 X/Open XA 표준에서 제안된 특징과 모델을 따른다. (사용빈도가 낮은 일부 기능은 구현되지 않았다.) 2단계 커밋은 다음 스탭에 따라 작동된다.

```
Coordinator                                         Cohort
                              QUERY TO COMMIT
                -------------------------------->
                              VOTE YES/NO           prepare*/abort*
                <-------------------------------
commit*/abort*                COMMIT/ROLLBACK
                -------------------------------->
                              ACKNOWLEDGMENT        commit*/abort*
                <--------------------------------  
end
```

**\[커밋 요청단계\]**

-   PREPARE를 통해 Coordinator가 데이터베이스 각 노드에 커밋을 준비 요청
-   각 데이터베이스는 필요 리소스에 LOCK 설정, 로그파일 저장 등 커밋 준비 작업 실행
-   준비 과정의 실패/성공 여부 알림

**\[커밋 단계\]**

-   모든 데이터베이스 노드로부터 완료 메시지를 받을 때까지 대기
-   한 데이터베이스라도 PREPARE OK를 받지 못하면, 모든 데이터베이스 노드에 롤백 메시지를 보내 해당 작업 롤백
-   모든 데이터베이스에서 PREPARE OK를 받으면 모든 데이터베이스 노드에 커밋 메시지를 보내고 모든 작업 커밋

짧은 기간의 PREPARED 트랜잭션은 공유 메모리와 WAL에 저장되며 체크포인트의 트랜잭션은 pg\_twophase 디렉터리에 기록된다.

## 2\. PREPARE TRANSACTION

### 2-1. PREPARE TRANSACTION란?

```
PREPARE TRANSACTION 'foobar';
```

PREPARED TRANSACTION 구문은 2단계 커밋을 위해 현재 트랜잭션을 준비 상태로 변경한다. 이 명령어 후에 해당 트랜잭션은 현재 세션과 완전히 분리되며, 해당 트랜잭션의 상태가 디스크에 완전히 저장된다. 그리고 해당 트랜잭션은 데이터베이스가 커밋요청 전에 충돌하더라도 성공적으로 커밋될 확률이 높다.

PREPARED TRANSACTION이 성공하면 트랜잭션은 COMMIT PREPARED, ROLLBACK PREPARED로만 커밋/롤백된다. 해당 커맨드는 PREPARED를 실행한 세션 외에 어떤 세션에서도 실행 가능하다.

구문을 실행한 세션의 시점에서 봤을 때 PREPARED TRANSACTION과 ROLLBACK 명령어는 큰 차이가 없다. 현재 진행 중인 트랜잭션은 없어지고, PREPARE TRANSACTION의 실제 결과는 보이지 않기 때문이다. (PREPARED COMMIT 후에나 차이를 알 수 있다.) PREPARE TRANSACTION 명령어가 실패한다면, 현재 트랜잭션이 취소되며 ROLLBACK과 동일한 결과를 나타낸다.

지금 실행 중인 prepared 트랜잭션은 pg\_prepared\_xacts에서 확인 가능하다.

**\[pg\_prepared\_xacts\]**

-   **TransactionId -** 트랜잭션 ID
-   **GID -** 유저가 정의한 트랜잭션 이름
-   **Prepared Date -** 트랜잭션 생성일, timestamp with timezone
-   **Owner -** 트랜잭션 생성자
-   **Database -** 해당 데이터베이스 명

### 2-2. 주의사항

-   PREPARE TRANSACTION은 애플리케이션, 혹은 상호 통신하는 세션을 위한 기능이 아니다. 외부 트랜잭션 관리자가 다양한 트랜잭션이나 기타 트랜잭션 리소스에 걸쳐 원자단위의 글로벌 트랜잭션을 수행할 수 있도록 하는 것이 목적이기에, 만약 transaction manager를 쓰는 것이 아니라면 PREPARE TRANSACTION의 사용을 중단해야 한다.
-   트랜잭션 내부에서만 사용해야 한다.
-   임시 테이블이나 세션의 임시 네임스페이스를 포함한 명령어나 WITH HOLD, LITEN, UNLISTEN, NOTIFY를 PREPARE에 사용하는 것은 불가능하다. 해당 기능들은 현재세션과 너무 타이트하기 연결되어 있어 트랜잭션 사용에 유용하지 않다.
-   런타임 파라미터를 변경한다면, PREPARE TRANSACTION 이후에도 적용되며 COMMIT PREPARED, ROLLBACK PREPARED의 영향을 받지 않는다. 이러한 측면에서 PREPARED TRANSACTION은 롤백보다는 커밋에 더 가깝다.
-   클라이언트가 사라지면 종료되지 않은 채로 트랜잭션이 남을 수 있고, PREPARE 스탭만 백업이 복구되면서 트랜잭션을 닫는 스탭이 없어 계속 유지될 수도 있다. PREPARE 구문을 사용한다면, 트랜잭션 매니저를 통해 해당 트랜잭션들을 주기적으로 관리해주어야 한다.
-   PREPARE 구문이 계속 열려있을 경우 Lock 같은 주요 시스템 자원을 계속 잡고 있을 수 있고, 트랜잭션 ID를 계속 유지하고 있기에 vacuum 시에 사용되지 않는 dead tuple 임에도 정리되지 않을 수 있다. 

## 3\. COMMIT/ROLLBACK PREPARE

```
ROLLBACK PREPARED transaction_id;

COMMIT PREPARED 'foobar';
```

-   PREPARE 상태의 트랜잭션을 커밋/롤백시킨다.
-   커밋/롤백을 위해서는 해당 트랜잭션을 실행시킨 유저와 동일하거나, superuser권한이 있어야 한다.
-   트랜잭션이 실행된 세션과 동일한 세션에 있을 필요는 없다. 
-   트랜잭션 블록 안에서 사용이 불가능하다. (PREPARED TRANSACTION이 즉시 커밋/롤백된다.)

PostgreSQL Extension 기능이며, 외부 트랜잭션 관리 시스템에서 사용하기 위해 만들어졌으며, 트랜잭션 매니저와 함께 사용하는 것을 권장한다.

[https://www.postgresql.org/docs/9.6/sql-rollback-prepared.html](https://www.postgresql.org/docs/9.6/sql-rollback-prepared.html)

[https://www.postgresql.org/docs/9.6/sql-prepare-transaction.html](https://www.postgresql.org/docs/9.6/sql-prepare-transaction.html)

[https://www.postgresql.org/docs/9.6/sql-commit-prepared.html](https://www.postgresql.org/docs/9.6/sql-commit-prepared.html)

[https://www.highgo.ca/2020/01/28/understanding-prepared-transactions-and-handling-the-orphans/](https://www.highgo.ca/2020/01/28/understanding-prepared-transactions-and-handling-the-orphans/)