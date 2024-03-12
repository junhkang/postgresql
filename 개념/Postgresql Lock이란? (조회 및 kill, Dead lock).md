## 1\. Lock 확인방법

```
SELECT PSAT.RELNAME,
       PL.LOCKTYPE,
       PL.PID,
       PL.MODE,
       PL.GRANTED
FROM PG_LOCKS PL,
     PG_STAT_ALL_TABLES PSAT
WHERE PL.RELATION = PSAT.RELID
```

## 2\. Lock Kill 방법

```
SELECT PG_CANCEL_BACKEND([PID])

SELECT PG_TERMINATE_BACKEND([PID])
```

Lock 리스트에서 조회된 PID를 넣고 cancel, 혹은 terminate 시켜주면 된다. cancel은 해당 프로세스만을, terminate는 상위 프로세스들까지 종료시킨다.

## 3\. Lock 이란? (Postgresql)

> Postgresql은 다양한 종류의 lock 기능을 제공한다. 애플리케이션 단에서 제어도 가능하지만, 대부분 기본적인 SQL 실행 시 적절한 락을 자동실행시켜 관련 테이블의 무결성 유지한다.

### **3-1.  테이블 단위 Lock**

> \- 다음 락 들은 모두 테이블 단위의 락이며, 명칭과 상관없이 테이블 단위로 적용된다.  
> \- 서로 다른 락이 충돌했을때의 상관관계에 의해 대기 상태로 돌입한다. (테이블 단위 락은 유형에 따라 서로 충돌여부가 다름)  
> \- 한 테이블에는 2개의 트랜잭션이 동시에 락 적용 될 수 없다. (서로 충돌되지 않는 락은 여러 트랜잭션에 동시에 적용될 수 있다.)  
> \- 특정 락은 self-conflicting 될 수 있다.  
> (ex. access exclusive 락은 중첩불가 access share 락은 여러 트랜잭션에서 다중으로 적용될 수 있다.)

<p align="center"><img src="/img/lock.png"/></p>

### **3-2. Row 단위 Lock**

> \- 데이터 검색에는 영향을 주지 않는다.  
> \- 해당 열의 writers and lockers에만 영향을 준다.  
> \- Row Lock은 해당 트랜잭션의 종료 시에 풀리거나, save point rollback 시점에 풀린다. (테이블 락과 동일)  
> \- 주로 for update 구문을 사용하여 select 동안 데이터의 무결성을 보장하기 위해 사용한다.

<p align="center"><img src="/img/lock2.png"/></p>

### **3-3. Page 단위 Lock**

> 공유된 buffer pool 내의 페이지 (데이터 블록) 단위의 read/write를 조절하기 위해 페이지 내부에서 Lock을 수행한다.

### **3-4. Dead Lock**

> 서로 다른 트랜잭션이 각각 서로 락을 대기하는 상태이다. 예를 들어, 트랜잭션 1이 테이블 A에 배타적 락을 획득하고, 테이블 B에도 배타적 락을 획득하려고 할 때, 동시에 트랜잭션 2가 이미 테이블 B에 배타적 락을 획득한 채로 테이블 A에 대한 배타적 락을 원한다면 양쪽 모두 진행할 수 없게 된다. 데드 락이 감지되면, 테이블 레벨 또는 레코드 레벨 락을 찾을 때까지 트랜잭션은 무기한으로 기다리게 된다. Postgresql에서는 데드락 상황을 자동으로 감지하고, 그중 한 개의 트랜잭션을 중단시켜 나머지를 완료되게 하지만, 어떤 트랜잭션이 중단될지 예측이 어렵고, 데드락을 즉시 발견하지 못하는 경우가 있기에 이에 의존하면 안 된다. 가장 좋은 해결책은, 일관된 순서로 객체에 대한 Lock을 획득하고, 트랜잭션에서 객체에 대한 첫 번째 Lock은 해당 객체에 대해 필요한 최소한의 lock 형태를 보장하는 것이다.

### **3-5. Advisory lock**

> 어플리케이션에서 정의된 락 유형이다. 시스템에서 정의되지 않은 락이기에, 애플리케이션에서 올바르게 사용해야 한다. Advisory lock은 일반 락들과 같은 공유 메모리 풀에 저장되며 max\_locks\_per\_transaction와 max\_connections파라미터 풀에 의해 사이즈가 정의된다.

####           **3-5-1. Session level advisory lock**

> \- Session level advisory lock은 세션이 끝나거나 명확한 해제가 있을 때까지 유지된다.  
> \- 보통의 락 요청과는 다르게, 세션레벨의 advisory lock은 트랜잭션에 종속되지 않는다.  
> \- 트랜잭션 중간에 획득된 락은 롤백 후에도 유지된다.  
> \- 그리고 이어지는 트랜잭션이 실패가 되더라도, 락은  유지된다.  
> \- 락은 한 프로세스 내에 여러 번 획득 가능하며, 완료된 각각의 락요청 이후에는 그에 대응하는 락 해제 요청이 뒤따라야 한다.

####           **3-5-2. transaction level advisory lock**

> \- 트랜잭션 레벨 락 요청은 반면에, 기존의 락요청과 비슷하게 적용된다.  
> \- 트랜잭션의 종료시점에 자동으로 해제되며 실직적인 락 해제 작업이 없다.

### **3-6. Advisory Lock 조회** 

```
SELECT pg_advisory_lock(id) FROM foo WHERE id = 12345; -- ok
SELECT pg_advisory_lock(id) FROM foo WHERE id > 12345 LIMIT 100; -- danger!
SELECT pg_advisory_lock(q.id) FROM
(
  SELECT id FROM foo WHERE id > 12345 LIMIT 100
) q; -- ok
```

> 다음 구문을 통해 Advisory Lock을 조회가능하다. 다만 2번째 쿼리처럼 Limit을 사용하게 되면 locking 함수가 실행되기 이전에 limit이 적용되는지를 보장할 수 없기에, 예상치 못한 lock을 발생시킬 수 있으며, 세션이 끝나기 전에 해제되지 않을 수 있으므로 주의해야 한다.