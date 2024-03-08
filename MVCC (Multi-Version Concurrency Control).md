## 1.  MVCC란?

동시성 제어를 위해 lock을 사용하는 대부분의 다른 데이터베이스 시스템과 달리 Postgres는 다중 버전 모델(multiversion model)을 사용하여 데이터 일관성을 유지한다. 각 트랜잭션이 데이터베이스를 쿼리 하는 동안 데이터의 현재 상태에 관계없이 얼마 전의 데이터 스냅샷을 볼 수 있음을 의미한다. 데이터를 쿼리 하기 위해 트랜잭션을 만들었다면 해당 Transaction은 데이터의 스냅샷을 보고 있는 것이다.

동일한 행에 서로 다른 트랜잭션이 동시에 업데이트를 시도할 때, 일관성 없는 데이터가 조회되지 않도록 트랜잭션을 보호하여 각 데이터베이스 세션에 대한 트랜잭션 격리를 제공한다. Multiversion과 Lock model의 주요 차이점은 MVCC에서 데이터 read를 위해 획득한 lock과 데이터 쓰기를 위해 획득한 lock이 충돌하지 않는다는 것이다. (따라서 read와 write는 서로 block 하지 않는다.) 이러한 방식을 통해서 Reading 하는 작업에 대해서 Lock을 걸지 않기에 높은 성능을 얻을 수 있게 된다.

### 1-1. Postgresql Lock에 대한 상세 설명

[\[Postgresql\] Postgresql Lock이란? (조회 및 kill, Dead lock)](https://github.com/junhkang/postgresql/blob/main/Postgresql%20Lock%EC%9D%B4%EB%9E%80%3F%20(%EC%A1%B0%ED%9A%8C%20%EB%B0%8F%20kill%2C%20Dead%20lock).md)

## 2\. PostgreSQL의 MVCC

PostgreSQL에서는 record를 tuple이라고 한다. PostgreSQL에서는 멀티버전에 대한 정보를 하나의 Page ( Table )에서 관리하고 있다. 모든 테이블에는 System Columns을 가지고 있고 이들은 미리 정의된 컬럼들로 내부 동작에 사용된다. 이 컬럼 중 mvcc를 구현하게 해주는 것이 xmin, xmax 컬럼이다.

> xmin – Tuple을 insert 하거나 update 하는 시점의 Transaction ID를 갖는 메타데이터  
> xmax – Tuple을 delete 하거나 update 하는 시점의 Transaction ID를 갖는 메타데이터

신규 insert, update시 xmin에 현재 transaction id를 넣고 xmax에는 null 값을 넣는다. delete, update시 이전 tuple의 xmax에는 작업을 수행한 transaction id 값을 넣는다. 이를 통해 트랜잭션이 시작된 시점의 Transaction ID와 같거나 작은 Transacion ID를 가지는 데이터를 읽는다. (xmin과 xmax의 범위를 통해 해당 트랜잭션이 조회할 수 있는 데이터인지를 판단한다.)

```
xmin  | xmax  |  value
-------+-------+-----
  100 |  120 | A
  102 |  120 | B
  110 |  134 | C
  115 |    0 | D
  115 |  120 | E
```

> \[Transaction ID 별 조회 가능한 데이터\]  
> Transaction 101에서는 A  
> Transaction 109에서는 A, B  
> Transaction 112에서는 A, B, C  
> Transaction 117에서는 A, B, C, D, E

하나의 page에 이전 tuple들이 그대로 존재하기 때문에, row가 삭제되어도 용량은 그대로 차지하는 경우가 있다. 쿼리 성능 또한 지속적으로 떨어지게 된다. 따라서 PostgreSQL에서는 Vacuum 작업을 진행해주어야 한다. 

Vacuum에 대한 상세 개념은 해당 포스트에서 확인할 수 있다.

[2023.10.09 - \[Postgresql\] - \[PostgreSQL\] Vacuum 개념 및 적절한 사용](https://junhkang.tistory.com/17)


\[참고\]

[https://techblog.woowahan.com/9478/](https://techblog.woowahan.com/9478/)

[https://www.postgresql.org/docs/9.1/ddl-system-columns.html](https://www.postgresql.org/docs/9.1/ddl-system-columns.html)

[https://www.postgresql.org/docs/7.1/mvcc.html](https://www.postgresql.org/docs/7.1/mvcc.html)