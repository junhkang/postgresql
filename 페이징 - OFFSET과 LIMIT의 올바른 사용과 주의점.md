## 1\. Limit과 Offset의 개념 및 사용법

Limit과 Offset은 쿼리 결과 중 특정 부분의 ROW만을 조회하게 해주는 기능이다.

```
SELECT select_list
    FROM table_expression
    [ ORDER BY ... ]
    [ LIMIT { number | ALL } ] [ OFFSET number ]
```

예를 들어 ID순서로 정렬된 결과 셋에서 21번째부터 30번째의 ROWS를 반환하고 싶다면 다음과 같이 사용하면 된다.

```
-- 21~30번째 열 조회
SELECT * FROM TEST_EXPLAIN 
ORDER BY ID
LIMIT 10 OFFSET 20
```

<p align="center"><img src="/img/limit.png"/></p>

-   LIMIT을 지정하면 해당 값만큼의 결과만 출력되며 총 결과 값이 더 적을 경우 있는 만큼만 출력한다.
-   LIMIT ALL, LIMIT NULL은 LIMIT을 설정하지 않은 것 (전체 ROWS 리턴)과 동일하다.
-   OFFSET은 리턴되는 ROWS의 시작점을 지정한다.
-   OFFSET 0, OFFSET NULL은 OFFSET을 설정하지 않은 것 (첫 ROW부터 리턴)과 동일하다.
-   OFFSET과 LIMIT을 둘 다 사용할 경우, LIMIT 카운트를 세기 전에 OFFSET만큼의 ROWS가  앞에서 생략된다.

## 2\. 주의점

LIMIT을 사용할 경우 결과 ROWS의 순서를 유니크하게 만들어주는 ORDER BY와 함께 쓰는 것이 중요하다. 그게 아니라면 예상하지 못한 부분 집합을 조회하게 될 수 있다. 앞의 예시 쿼리에서 ORDER BY를 뺀다면, 10~20 번째를 가져오려고 해도, 어떤 순서에서의 10~20 번째인지 알 수 없다.

```
-- 21~30번째 열 조회 (순서 없이)
SELECT * FROM TEST_EXPLAIN 
LIMIT 10 OFFSET 20
```

<p align="center"><img src="/img/limit2.png"/></p>

다음 예시에서 볼 수 있듯이, 쿼리 옵티마이저는 쿼리 PLAN을 생성할 때 LIMIT을 고려하여 생성하기 때문에, LIMIT, OFFSET이 다를 경우 다른 PLAN이 생성되며, 서로 다른 순서의 결과를 얻을 확률이 매우 높다. 그래서 쿼리 결과에 부분 집합을 선택하기 위해 ORDER BY 없이 LIMIT / OFFSET 값을 사용하는 경우, 일관성 없는 결과를 얻게 된다.

주로 LIMIT / OFFSET은 페이징 처리를 하여 특정 순서의 일정량의 데이터를 반환할 때 사용하게 되는데, 이 경우 "유니크"한 정렬에 주의하여야 한다. 다음 예제를 보면, 동일 값을 다수 보유하고 있는 address 칼럼을 기준으로 정렬 후 LIMIT/OFFSET을 설정하면, 21~30번째 ROWS와 31~40번째의 ROWS가 연속됨을 보장할 수 없다.

```
select * from TEST_EXPLAIN order by address
LIMIT 10 OFFSET 20;
```

<p align="center"><img src="/img/limit3.png"/></p>

```
select * from TEST_EXPLAIN order by address
LIMIT 10 OFFSET 30;
```

<p align="center"><img src="/img/limit4.png"/></p>

각각 31~40, 41~50번째 ROWS를 추출하였으나 동일한 값도 포함되어 있고, 순서도 명확하지 않다. 따라서 LIMIT / OFFSET을 통해 부분 데이터를 추출할 경우, 단순히 ORDER BY를 하는 것이 아닌, 유니크한 순서 정렬이 될 수 있도록 정렬해 주어야 한다.

또한 OFFSET으로 생략된 앞부분은 서버 내부에서 계산이 되어야 하기 때문에, 아주 큰 OFFSET을 설정하는 것은 효율적이지 못할 수 있다.