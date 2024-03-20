## 1\. 기본 트랜잭션의 개념 및 원리

트랜잭션의 기본 개념과 사용 방법은 다음 포스트에서 확인이 가능하다.

[[PostgreSQL] 트랜잭션(Transaction)의 개념 및 사용](https://github.com/junhkang/postgresql/blob/main/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98(Transaction)%EC%9D%98%20%EA%B0%9C%EB%85%90%20%EB%B0%8F%20%EC%82%AC%EC%9A%A9.md)

기본적으로 트랜잭션이 어떤 것인지, COMMIT, ROLLBACK도 익숙하게 사용하고 있다면, PostgreSQL 내부의 트랜잭션이 어떤 구조로 작동하며 세부 단계를 각각 어떻게 확인 가능한지 자세히 알아보자.

## 2\. 트랜잭션과 식별자 (Transactions and Identifiers)

기본 개념에서 확인했듯이 트랜잭션은 명시적으로 실행(BEGIN, START TRANSACTION), 종료 (COMMIT, ROLLBACK) 할 수 있다. 명시적 트랜잭션 외의 SQL 구문들은 단일 트랜잭션이 자동으로 적용된다. 그렇다면 각각의 트랜잭션이 어떻게 구분되는지 먼저 살펴보자.

<p align="center"><img src="/img/tx.png"/></p>

### 2-1. Virtual Transaction Id

모든 트랜잭션은 유니크한 **Virtual Transaction Id(virtualXID, vxid)**로 식별된다. 이 **Virtual Transaction Id**는 **Backend ID**와 각 백앤드에 순차적으로 부여된 로컬 아이디 (**LocalXID**)로 구성되어 있다. 캡처의 **virtualxid**를 확인해 보면 다음과 같다.

-   Virtual Transaction ID = 115/10798
-   Backend Id = 115
-   LocalXID = 10798

### 2-2. Non-Virtual TransactionIds

그 외 가상이 아닌 **Non-Virtual TransactionIds**(or xid) (캡처의 transactionId = 114016445)들은 PostgreSQL 클러스터의 모든 데이터베이스에서 공통으로 사용하는 global counter를 순차적으로 사용한다. 

이러한 TransactionID는 트랜잭션이 처음 데이터베이스에 write 할 때 적용되며, xids가 낮을수록 먼저 수행된 트랜잭션임을 의미한다. 하지만 트랜잭션이 처음 데이터베이스 write를 실행한 순서와 트랜잭션이 시작된 순서는 다를 수 있다. 특히 트랜잭션이 데이터베이스를 read만으로 시작할 때 그렇다.

각 xid는 32비트이고 20억 건의 트랜잭션마다 wraps around 한다.

> **wraps around** - 트랜잭션 번호가 40억 건이 넘으면 0에서부터 다시 시작하며, 중복된 트랜잭션이 부여될 수 있다. 이 경우 데이터는 존재하지만 접근하지 못하는 데이터가 발생하며, 치명적인 데이터 손실로 이어질 수 있다. 이를 방지하기 위해 20억 트랜잭션이 일어날 때마다 모든 데이터베이스, 모든 테이블에 최소 1번의 vacuum을 실행한다. 

해당 내용에서 트랜잭션의 40억 건이 넘으면 0부터 다시 실행되는데 왜 40억이 아닌 20억 트랜잭션마다 최소 1번의 Vacuum이 일어날까? 에 대한 궁금증이 들어 PostgreSQL 측에 문의하였고, PostgreSQL 측 공식 답변은 다음과 같다.

> 트랜잭션 ID를 "순환 카운터(Circular Counter)"로 취급하기 때문이다. 특정 트랜잭션 (A)와 트랜잭션 (B)를 비교할 때 (A의 트랜잭션 ID-B의 트랜잭션 ID)의 연산 후 부호를 포함한 정수로 연산을 한다. 만약 결과가 음수라면 A가 더 최근 트랜잭션이고, 공간이 순환적이고 서로 다른 트랜잭션 A, B의 ID가 다를 수 없기에 부호를 가진 정수로써 모두 표현이 가능하다.   
>   
> 순환 공간에서는 1G(10억 건의 트랜잭션)이 3G보다 이전에 있을 수도 있고, 이후에 있을 수도 있다. (1G가 3G의 이전 사이클에서 생성된 트랜잭션 ID인지, 이후 사이클에서 생성된 트랜잭션 ID인지 부호 없이는 구분이 불가능하다.)
>   
> **1.5G와 2.5G의 비교 :** 1.5G는 2.5G보다 10억 트랜잭션 이전이므로, 부호 있는 비교에서는 1.5G는 2.5G보다 이전 트랜잭션  
> **0.5G 와 3.5G 비교 :** 순환 공간을 생각할 때 4G가 최대치인 상황에서 0.5G가 3.5G보다 이후로 간주된다. (0이 최대치(4G) 이후에 존재하기 때문) 그렇기 때문에 부호 있는 비교에서는 0.5G가 3.5G 이후로 판단됨.  
>   
> 그렇기 때문에 순환 카운터에서는 최대치(4G)의 절반인 2G 트랜잭션 제한을 통해 +/-2G의 범위 내에서 트랜잭션 ID의 "전후" 관계를 파악한다.

매 wraps around 마다  32bit epoch가 증가하며 64비트의 xid8 유형도 존재한다. (pg\_current\_xact\_id () → 로 현재 xid8 조회 가능) Xid들은 PostgreSQL의 MVCC의 동시성, 및 스트리밍 복제의 근간으로 사용된다. 최상위 트랜잭션의 non-virtual xid가 커밋되면, **pg\_xact** 디렉터리에 커밋으로 기록되며 추가 정보들은 **pg\_commit\_ts** 디렉터리에 기록된다. (track\_commit\_timestamp가 활성화 필요) virtual xid, non-virtual xid에, PREPARED TRANSACTION의  경우에는 Global Transaction Identifiers(GID)가 추가로 부여된다. GID는 200바이트의 문자열로 현재의 다른 PREPARED TRANSACTION과 중복되지 않아야 한다. (GID와 xid의 매핑관계는 **pg\_prepared\_xacts**에서 확인가능하다.)

## 3\. 트랜잭션과 락 (Transactions and Locking)

현재 진행 중인 트랜잭션의 Transaction ID는 pg\_locks의 virtualxid와 transactionId 칼럼에서 확인할 수 있다.

```
SELECT LOCKTYPE, VIRTUALXID, TRANSACTIONID FROM PG_LOCKS;
```

두 칼럼 모두 read/write 트랜잭션에 존재하지만 Read-only 트랜잭션에는 virtualxid는 있으나 transactionId는 null이다. 

<p align="center"><img src="/img/tx2.png"/></p>
<p align="center"><img src="/img/tx.png"/></p>

Row-level의 read/write locks는 잠긴 row에 바로 기록되며 pgrowlocks extension을 통해 확인 가능하다.

## 4\. 서브트랜잭션 (Subtransactions)

서브 트랜잭션(Subtransactions, subxact)은 트랜잭션 안에서 시작되며 큰 트랜잭션을 더 작은 트랜잭션 단위로 분리할 수 있게 해 준다. 서브 트랜잭션은 부모 트랜잭션이 영향을 받지 않고 계속 진행되게 한 채로 commit 혹은 rollback 할 수 있다. 서브 트랜잭션은 SAVEPOINT라는 커맨드로 명시적 실행이 가능하지만 PL/pgSQL의 Exception 구문으로도 실행 가능하다. 서브트랜잭션은 다른 서브트랜잭션으로부터도 시작 가능하다. 최상위 트랜잭션과 그 자식 서브트랜잭션은 계층구조 혹은 트리 구조를 형성하며 그렇기 때문에 메인 트랜잭션을 top-level 트랜잭션이라고 한다.

서브트랜잭션이 non-virtual TransactionalId를 할당받았다면, 그 트랜잭션아이디는 subxid라고 한다. Read-only 서브트랜잭션은 subxids가 부여되지 않지만 write를 하려고 하는 순간 1개를 부여받는다. 또한 상위 레벨 트랜잭션을 포함한 하위 xid의 부모 모두에게 non-virtual 트랜잭션 ID가 할당되며, 부모 xid가 자식 xid보다 항상 낮도록 유지한다.

-   각 subxid의 바로 윗 부모 xid는 pg\_subtrans디렉터리에 기록된다. 최상위 xid의 경우 부모 트랜잭션이 없기에 기록되지 않으며 읽기 전용 하위 트랜잭션에 대해서도 기록하지 않는다.
-   서브 트랜잭션이 커밋되면, subxid에 포함된 모든 하위 트랜잭션도 해당 트랜잭션에서 임시로 commit 된 것으로 고려된다. 서브 트랜잭션이 취소되면 모든 자신의 서브 트랜잭션도 취소된다.
-   최상위 트랜잭션의 xid가 커밋되면, 모든 하위 트랜잭션의 임시 커밋들이 pg\_xact 디렉터리에 기록된다. 최상위 트랜잭션이 취소되면, 임시 커밋된 트랜잭션을 포함한 모든 서브 트랜잭션이 또한 취소된다.
-   각 트랜잭션이 서브트랜잭션을 많이 열어둘수록 트랜잭션 관리 오버해드가 증가한다. 각 백앤드마다 서브트랜잭션을 64개까지는 공유메모리에 캐시 하지만, 그 후로는 pg\_subtrans의 subxid를 추가로 찾으면서 저장소 I/O 오버헤드가 극도로 증가한다. 

## 5\. 2단계 트랜잭션 (Two-Phase Transactions)

PostgreSQL은 two-phase commit (2PC) 프로토콜을 지원한다. 다수의 분산 시스템 환경에서 모든 데이터베이스가 정상적으로 수정되었음을 보장하는 두 단계 커밋 프로토콜로 분산 트랜잭션에 참여한 모든 데이터베이스가 모두 함께 커밋되거나 롤백되는 것을 보장한다.
상세 내용은 다음 포스트에서 확인가능
[[PostgreSQL] 2단계 커밋 프로토콜(Two-Phase Commit Protocol), Prepare transaction](https://github.com/junhkang/postgresql/blob/main/2%EB%8B%A8%EA%B3%84%20%EC%BB%A4%EB%B0%8B%20%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C(Two-Phase%20Commit%20Protocol)%2C%20Prepare%20transaction.md)

참고

[https://www.postgresql.org/docs/16/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND](https://www.postgresql.org/docs/16/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)

[https://www.postgresql.org/docs/16/view-pg-prepared-xacts.html](https://www.postgresql.org/docs/16/view-pg-prepared-xacts.html)

[https://www.postgresql.org/docs/16/two-phase.html](https://www.postgresql.org/docs/16/two-phase.html)

[https://www.postgresql.org/docs/16/subxacts.html](https://www.postgresql.org/docs/16/subxacts.html)

[https://www.postgresql.org/docs/16/xact-locking.html](https://www.postgresql.org/docs/16/xact-locking.html)

[https://www.postgresql.org/docs/16/transaction-id.html](https://www.postgresql.org/docs/16/transaction-id.html)