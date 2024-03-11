## 1\. 인덱스(INDEX) 상세 개념

[\[Postgresql\] - 인덱스(INDEX)개념 및 생성, 삭제, 분석, 설계 방법](https://github.com/junhkang/postgresql/blob/main/%EC%9D%B8%EB%8D%B1%EC%8A%A4(INDEX)%EA%B0%9C%EB%85%90%20%EB%B0%8F%20%EC%83%9D%EC%84%B1%2C%20%EC%82%AD%EC%A0%9C%2C%20%EB%B6%84%EC%84%9D%2C%20%EC%84%A4%EA%B3%84%20%EB%B0%A9%EB%B2%95.md)


## 2\. 미사용 인덱스

> 간단히 말해, 인덱스는 지정 컬럼에 매핑된 정보를 별도로 저장하고 있다. 보통 플랜 확인을 통해 효율적으로 인덱스를 추가하여 쿼리 최적화를 진행하게 된다. 오래되고 변경이 잦은 어플리케이션일수록 미사용 인덱스는 늘어나고, 인덱스가 사용되지 않는 경우를 매번 모니터링하여 삭제하는 것은 힘든 일이다. 하지만 불필요 인덱스는 디비 성능저하 및 vacuum 코스트를 증가시키기에, 최적화된 인덱스 생성만큼 최적화된 인덱스 삭제도 중요하다.

###      2-1. 미사용 인덱스 검색 쿼리

```
SELECT
    schemaname AS schema_name,
    relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid::regclass)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC
```

###      2-2. 인덱스 삭제 쿼리

```
DROP INDEX [ CONCURRENTLY ] [ IF EXISTS ] name [, ...] [ CASCADE | RESTRICT ]

--simple
DROP INDEX IDX_NAME



-- 옵션
IF EXISTS
해당 인덱스가 없어도 오류를 내지 않고, 알림 메시지만 보여줌

CASCADE
해당 인덱스와 의존성 관계가 있는 모든 객체도 함께 삭제한, 삭제될 다른 객제와 관계된 또 다른 객체들도 함께 삭제

RESTRICT
해당 인덱스와 의존성 관계가 있는 객체가 있으면 작업을 중지 (기본값)
```

###      2-3. 인덱스 삭제시 주의사항

> 1\. 검색된 인덱스가 실제 미사용 인덱스인지 재검토가 필요하다.   
> 2\. 애초에 사용주기가 긴 인덱스의 경우, 사용되는 인덱스이지만 검색되는 경우도 있다.  
> 3\. 인덱스는 해당 소유자만 삭제 가능하다.