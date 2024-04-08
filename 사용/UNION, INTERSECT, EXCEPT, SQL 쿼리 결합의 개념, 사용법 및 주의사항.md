## 1\. UNION, INTERSECT, EXCEPT를 통한 쿼리 결합

UNION(결합), INTERSECT(교차), EXCEPT(차이) 구문을 통해 두 쿼리의 결과를 결합할 수 있다.

```
query1 UNION [ALL] query2
query1 INTERSECT [ALL] query2
query1 EXCEPT [ALL] query2
```

해당 구문들을 실행시키기 위해서는 query1, query2가 동일한 개수, 동일한 type의 칼럼을 리턴해야 한다.

## 2\. UNION

query2의 결과를 query1에 이어 붙인다. 그냥 사용할 경우 중복을 제거하여 distinct와 같은 효과를 볼 수 있으며, UNION ALL을 사용하면 중복을 포함하여 쿼리를 합친다.

### 2-1. UNION 단일 사용

1~5 번째 ROWS, 4~8번째 ROWS를 합친 후 중복 제거한 결과를 보면 다음과 같다.

```
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5)
UNION
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3)
ORDER BY ID;
```

<p align="center"><img src="/img/union1.png"/></p>

### 2-2. UNION ALL

1~5 번째 ROWS, 4~8번째 ROWS를 합친 결과를 보면 다음과 같다. (1000004, 1000005 중복 출력)

```
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5)
UNION ALL
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3)
ORDER BY ID;
```

<p align="center"><img src="/img/union2.png"/></p>

## 3\. INSERSET

query1과 query2에 동시에 존재하는 ROWS를 반환한다. INTERSECT ALL을 사용하면 중복을 제거해 준다. 예제에선 동일 값을 중복생성하기 위해 UNION과 결합해서 사용하였다.

### 3-1. INTERSECT 단일 사용

1~5 번째 ROWS X 2, 4~8번째 ROWS X 2중 교차되는 ROWS만 중복 제거한 후 출력(1000004, 1000005가 중복 제거된 후 출력)

```
((SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5)
 UNION ALL
 (SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5))
INTERSECT
((SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3)
 UNION ALL
 (SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3))
ORDER BY ID;
```

<p align="center"><img src="/img/union3.png"/></p>

### 3-2. INTERSECT ALL

1~5 번째 ROWS X 2, 4~8번째 ROWS X 2중 교차되는 ROWS만 중복을 포함하여 출력(1000004, 1000005가 두 번씩 중복 출력)

```
((SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5)
 UNION ALL
 (SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5))
INTERSECT ALL
((SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3)
 UNION ALL
 (SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3))
ORDER BY ID;
```

<p align="center"><img src="/img/union4.png"/></p>

## 4\. EXCEPT

query1에는 존재하고, query2에는 존재하지 않는 ROWS를 반환한다. EXCEPT ALL을 사용하면 중복을 제거해 준다. 위 예제와 동일하게 중복된 상황을 위해 UNION 쿼리와 결합하여 사용하였다.

### 4-1. EXCEPT 단일 사용

선행쿼리에 존재하고, 후행쿼리에 존재하지 않는 ROWS를 중복을 제거한 후 반환한다.

```
((SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5)
UNION ALL
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5))
EXCEPT
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3)
ORDER BY ID;
```

<p align="center"><img src="/img/union5.png"/></p>

### 4-2. EXCEPT ALL

선행쿼리에 존재하고, 후행쿼리에 존재하지 않는 ROWS를 중복 제거 없이 모두 반환한다.

```
((SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5)
 UNION ALL
     (SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5))
EXCEPT ALL
(SELECT * FROM TEST_EXPLAIN ORDER BY ID LIMIT 5 OFFSET 3)
ORDER BY ID;
```

<p align="center"><img src="/img/union6.png"/></p>

주의할 점은, EXCEPT만 사용할 경우, 선행쿼리에 동일한 ROWS가 아무리 많더라도 후행쿼리에 하나라도 존재할 경우 모두 삭제되지만, EXCEPT ALL로 중복을 모두 리턴할 경우, 후행 쿼리에 존재하는 만큼만 ROWS가 삭제된다.

(EXCEPT ALL의 경우 1000003, 1000004 ROWS가 후행 쿼리에 존재하지만, 선행쿼리에 2개씩 존재하기에 여전히 출력되고 있다.)

## 5\. 복합 사용

세 가지 구문은 복합적으로 사용할 수 있다. 예를 들어 다음 두 개의 쿼리는 같은 결과를 리턴한다.

```
query1 UNION query2 EXCEPT query3
-- 동일
(query1 UNION query2) EXCEPT query3
```

해당 예제에서 확인할 수 있듯, 괄호를 통해 구문의 실행 순서를 조절할 수 있다. 괄호가 없을 때 UNION, EXCEPT는 왼쪽에서 오른쪽으로 실행되며, **INTERSECT가 최우선**으로 적용된다. (공통부분 추출이기에 더 엄격한 필터 취급)

```
query1 UNION query2 INTERSECT query3
-- 동일
query1 UNION (query2 INTERSECT query3)
```

또한 개별 쿼리를 괄호로 지정할 수 있으며, LIMIT/OFFSET 같이 특정 쿼리에만 지정하고 싶은 구문이 있을 경우 다르게 적용할 수 있다.

(2~4 예제에서 사용 중)

```
-- UNION 결과에 LIMIT 10 적용
SELECT a FROM b UNION SELECT x FROM y LIMIT 10
```

```
-- UNION 결과에 LIMIT 10 적용 (1과 동일)
(SELECT a FROM b UNION SELECT x FROM y) LIMIT 10
```

```
-- 후행 쿼리 결과에만 LIMIT 10 적용
SELECT a FROM b UNION (SELECT x FROM y LIMIT 10)
```

참고

[https://www.postgresql.org/docs/16/queries-union.html](https://www.postgresql.org/docs/16/queries-union.html)