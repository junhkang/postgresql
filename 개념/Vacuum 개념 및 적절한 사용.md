## 1\. Vacuum 이란?

Vacuum은 postgresql에서 dead tuple이 차지하는 저장공간을 회수한다. 일반적으로 Postgresql에서 update, delete tuple 은 물리적으로 삭제되지 않으며 vacuum이 완료될 때까지 계속 존재한다. 

(update, delete 시 tuple의 순환은 MVCC 개념에서 확인할 수 있다.)

[\[PostgreSQL\] MVCC (Multi-Version Concurrency Control)](https://github.com/junhkang/postgresql/blob/main/MVCC%20(Multi-Version%20Concurrency%20Control).md)

그렇기 때문에 특히 자주 업데이트되는 테이블의 경우 주기적인 Vacuum 수행이 필요하다. Vacuum은 특정 테이블에 한해서도 실행이 가능하고, 테이블을 지정하지 않는다면 전체 테이블 (권한을 보유한)에 대해서 실행된다.

## 2\. Vacuum 명령어

```
-- DB 전체 full vacuum
vacuum full analyze;

-- DB 전체 간단하게 실행
vacuum verbose analyze;

-- 해당 테이블만 간단하게 실행
vacuum analyze [테이블 명];

-- 특정 테이블만 full vacuum
vacuum full [테이블 명];
```

## 3\. Vacuum 상세 옵션

```
VACUUM [ ( option [, ...] ) ] [ table_and_columns [, ...] ]
VACUUM [ FULL ] [ FREEZE ] [ VERBOSE ] [ ANALYZE ] [ table_and_columns [, ...] ]

where option can be one of:

    FULL [ boolean ]
    FREEZE [ boolean ]
    VERBOSE [ boolean ]
    ANALYZE [ boolean ]
    DISABLE_PAGE_SKIPPING [ boolean ]
    SKIP_LOCKED [ boolean ]
    INDEX_CLEANUP { AUTO | ON | OFF }
    PROCESS_MAIN [ boolean ]
    PROCESS_TOAST [ boolean ]
    TRUNCATE [ boolean ]
    PARALLEL integer
    SKIP_DATABASE_STATS [ boolean ]
    ONLY_DATABASE_STATS [ boolean ]
    BUFFER_USAGE_LIMIT [ size ]

and table_and_columns is:

    table_name [ ( column_name [, ...] ) ]
```

## 4\. Vacuum analyze

Vacuum analyze 는 vacuum 후 테이블 별로 analyze를 수행(통계정보 수집)하기에 유지보수에 원활하다.

Postgresql 버전업을 수행한 후, 인덱스 및 테이블 튜닝이 완료된 쿼리 (평균 소요시간 0.8초 이내)가 5분 이상 소모되어 서비스에 문제를 일으키는 상황이 발생하였다. Vacuum analyze를 사용해 통계 정보를 조정 후 플랜이 정상적으로 작동하는 것을 확인하였다.

## 5\. Vacuum without Full

Full 옵션 없이 vacuum 은 단순히 tuple을 삭제후 공간을 확보하여 재사용 가능하게 한다. 이 작업은 일반적은 read/write와 동시에 실행될 수 있으며 배타적 lock이 발생하지 않는다. 그러나 일반적으로 추가 공간이 OS로 반환되지 않고, 동일 테이블에 재사용가능 상태로 반환된다.  인덱스를 처리하기위해 여러 개의 CPU를 사용할 수 있으며, 이를 parallel Vacuum이라고 한다.

## 6\. Vacuum Full

vacuum full은 테이블 전체 내용을 추가 공간 없이 새로운 디스크파일에 다시 작성한다. 미사용 space는 OS로 반환된다. 일반적인 vacuum 보다 훨씬 시간이 오래 걸리며, 해당 테이블에 lock 이 발생하기에 운영 중인 테이블에 실행 시 주의하여야 한다. Vacuum은 transaction 블럭 내에서는 사용이 불가능하다. Gin 인덱스의 경우 대기중인 인덱스 생성까지 완료된다.

(Gin 인덱스에 대한 개념은 다음 포스트에서 확인 가능하다.)

[\[PostgreSQL\] GIN인덱스의 원리 및 특징](https://github.com/junhkang/postgresql/blob/main/GIN%EC%9D%B8%EB%8D%B1%EC%8A%A4%EC%9D%98%20%EC%9B%90%EB%A6%AC%20%EB%B0%8F%20%ED%8A%B9%EC%A7%95.md)

## 7\. Vacuum의 적절한 사용

Vacuum의 수동 실행이아니더라도 postgresql 에는 _autovacuum launcher_가 상시 수행되고 있어 dead tuple이 임계치에 도달하거나 table, tuple의 age가 누적되어 임계치에 도달하였을 때 auto vacuum이 실행된다. Postgresql에서는 Autovacuum 활성화를 권장하고 있다.

Vacuum Full의 경우 테이블 자체의 락이 발생하기에 사용에 주의하여야 하며, 보통 사용자가 없는 시간대나 시스템 영향도가 낮은 시간대에 실행하나, 해당 지표만으로 Vacuum 주기를 설정하는 것이 정답은 아니다. Vacuum이 너무 잦은 것도 Vacuum 부하로 쿼리성능을 저하시키기에 좋지 않고, 영향도가 너무 낮은 시간을 찾기 위해 너무 드물게 수행하면 삭제할 tuple 및 작업이 많아져 오히려 큰 부하를 유발할 수 있다. 그러므로 상세 파라미터 튜닝을 통해 현재 운영 중인 시스템에 맞는 Vacuum주기를 찾는 것이 중요하다.

참고

[https://www.postgresql.org/docs/current/sql-vacuum.html](https://www.postgresql.org/docs/current/sql-vacuum.html)

[https://techblog.woowahan.com/9478/](https://techblog.woowahan.com/9478/)