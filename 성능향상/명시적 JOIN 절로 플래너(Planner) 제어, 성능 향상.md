## 1\. 개요

PostgreSQL은 쿼리 Planner가 가장 효율적인 쿼리 플랜을 세워 쿼리를 실행시킨다. 이번 포스트는 쿼리 Planner가 플랜을 검색하는 과정을 의도적으로 제한하여 플랜 검색 시간을 단축시키는 방법에 대한 내용이다. 쿼리 선택지를 제한함으로써 시간을 줄이지만, 그만큼 모든 경우를 비교하는 것이기 아니라서 최고의 플랜을 찾을 수 없기에, 테이블 scan 방식 및 인덱스 등 쿼리의 작동방식을 명확히 이해한 후 설정이 필요하며, 설정전 성능비교, 설정 후의 데이터 증감에 따른 지속적인 모니터링이 필요하다.

## 2\. 플래너의 작동

### 2-1. JOIN

Planner의 작동방식을 보기 위해 간단한 조인 쿼리를 확인해 보자

```
SELECT * FROM a, b, c WHERE a.id = b.id AND b.ref = c.id;
```

PostgreSQL 플래너는 조인 순서를 자유롭게 정할 수 있다.

-   a.id = b.id 조건으로 A, B테이블을 먼저 조인 후에 C테이블을 조인
-   b.ref = c.id 조건으로 B, C테이블을 먼저 조인 후에 A테이블을 조인
-   A, C를 조인 후에 B테이블 조인 (A, C의 조인을 최적화하는 조건이 없기에 비효율적)

중요한 점은, 모든 방식이 동일한 결과를 가져오지만, 실행 cost에는 엄청난 차이가 난다는 것이다. 그래서 planner는 가장 효율적인 쿼리 플랜을 찾는다. 사실 쿼리가 2~3개의 테이블만 참조한다면, 고려할 조인방식이 그리 많지 않다. 그러나 테이블 수가 증가할수록 조인 순서의 선택지는 확연히 증가한다. 10개 정도의 테이블이 조인될 경우 모든 경우의 수를 철저히 검색하는 것은 실용적이지 않고, 테이블 수가 6~7개만 되어도 plan을 선택하는데 굉장히 오랜 시간이 걸릴 수 있다.

너무 많은 테이블이 참조될 때, PostgreSQL planner는 검색하는 플랜의 경우의 수를 제한하는 genetic 확률적 검색으로 전환한다.

이 경우, 검색하는 플랜의 수가 줄어들기에 검색시간은 줄어들지만, 최고의 플랜을 찾지 못할 수도 있다.

### 2-2. OUTER-JOIN

다음과 같은 outer 조인에서 planner의 조인 순서 선택지는 확연히 줄어든다.

```
SELECT * FROM a LEFT JOIN (b JOIN c ON (b.ref = c.id)) ON (a.id = b.id);
```

이전 일반 join과 조건은 똑같이 적용되었지만, 작동방식은 현저히 다르다. 기존 일반 조인은 B와 C를 조인한 결과와 일치하지 않는 A의 각 로우들은 생략되어야 하고 outer join은 A의 각 로우들이 생략되지 않는다.

그래서 planner의 선택지는 다음으로 줄어든다.

-   ~a.id = b.id 조건으로 A, B테이블을 먼저 조인 후에 C테이블을 조인~
-   b.ref = c.id 조건으로 B, C테이블을 먼저 조인 후에 A테이블을 조인
-   ~A, C를 조인 후에 B테이블 조인 (A, C의 조인을 최적화하는 조건이 없기에 비효율적)~

Planner는 해당 쿼리의 유효한 플랜이 1개로 인식하고, 선택지가 1개이기에 plan을 세우는데 적은 시간이 든다.

반면에 다음과 같은 경우에 planner는 2개 이상의 plan이 유효하다고 판단할 수도 있다.

```
SELECT * FROM a LEFT JOIN b ON (a.bid = b.id) LEFT JOIN c ON (a.cid = c.id);
```

-   a.id = b.id 조건으로 A, B테이블을 먼저 조인 후에 C테이블을 조인
-   ~b.ref = c.id 조건으로 B, C테이블을 먼저 조인 후에 A테이블을 조인~
-   a.cid = c.id 조건으로 A, C테이블을 먼저 조인 후에 B테이블을 조인

현재는 FULL JOIN만이 테이블 간의 조인 순서 제한하지만, 대부분의 LEFT JOIN, RIGHT JOIN은 조인 순서가 재배열될 수 있다.

명시적인 INNER JOIN (INNER JOIN, CROSS JOIN 등)은 구조적으로 FROM 절에 테이블 입력 순서와 의미적으로 동일하게 실행되므로 조인순서를 제약하지 않는다. (영향을 받지 않는다.)

대부분의 JOIN이 순서를 완전히 제약하지 않지만, PostgreSQL 플래너가 모든 JOIN절을 조인 순서를 제약하도록 별도 지시할 수 있다. 예를 들어 다음 세 쿼리는 논리적으로 동일하다.

```
SELECT * FROM a, b, c WHERE a.id = b.id AND b.ref = c.id;
SELECT * FROM a CROSS JOIN b CROSS JOIN c WHERE a.id = b.id AND b.ref = c.id;
SELECT * FROM a JOIN (b JOIN c ON (b.ref = c.id)) ON (a.id = b.id);
```

그러나 planner에게 조인 순서를 정하도록 하면, 2,3번 쿼리가 첫 번째 쿼리보다 plan을 세우는데 더 적은 시간이 걸리며 실제 플랜도 다르게 나온다. 이 차이는 테이블이 3개 있을 때는 그리 크지 않지만, 많은 테이블을 대상으로 할 때는 굉장히 효율적이다.

planner가 명시적 join에 제시된 테이블 조인 순서를 따르게 하려면 join\_collapse\_limit 매개변수를 1로 설정하면 된다.

아니면 일반 FROM절 리스트에서 JOIN 구문을 추가해도 되기 때문에, plan 검색시간을 줄이기 위해 파라미터를 조정하여 조인 순서를 완벽하게 제한시킬 필요는 없다.

예를 들어

```
SELECT * FROM a CROSS JOIN b, c, d, e WHERE ...;
```

join\_collapse\_limit=1 옵션을 주면, 해당 쿼리는 다른 선택지에 대한 고려 없이 A와 B를 우선적으로 조인하도록 강제한다. 그렇기 때문에 

이 예제에서는 가능한 조인 순서가 5배로 줄어들게 된다. (5! -> 4!)

이런 방식으로 planner의 계획 검색을 제한하는 것은 planning 시간을 줄여주고 planner를 좋은 쿼리플랜으로 유도하는데 도움이 된다.

만약 planner가 안 좋은 조인 순서를 기본으로 선택하였다면, JOIN 문법을 통해 더 좋은 join 순서로 유도할 수 있다. 다만 작성 후 성능비교 및 플랜확인은 필수이다.

### 2-3. Subquery

planning시간이 영향을 주는 유사한 예로는, 서브쿼리를 상위 쿼리에 포함시키는 경우이다. 다음 간단한 서브쿼리를 확인해 보자.

```
SELECT *
FROM x, y,
    (SELECT * FROM a, b, c WHERE something) AS ss
WHERE somethingelse;
```

일반적으로 Planner는 서브쿼리를 부모쿼리에 포함시키기에 다음과 같다.

```
SELECT * FROM x, y, a, b, c WHERE something AND somethingelse;
```

이러한 쿼리가 보통 서브쿼리를 따로 planning 하는 것보다 효율적이다. (예를 들어, 부모쿼리의 where 절은 x를 A에 먼저 조인시켜 A의 많은 row를 제거함으로써 서브쿼리의 결과와 전체 조회를 실행하는 것을 피할 수 있다.) 그러나 동시에, planning 시간이 증가한다.

> 2가지의 3개의 테이블을 조인하는 경우의 수\[2 x (3!)\] -> 5개의 테이블을 조인하는 경우의 수 \[5!\]

더 많은 테이블을 참조할 경우, 가능한 조인 방식의 수가 기하급수적으로 늘어남으로써 큰 차이가 된다. planner는 subquery를 상위 쿼리에 포함시킴으로써 조인 방식의 경우의 수가 너무 커지는 문제를 방지하기 위해  from\_collapse\_limit의 파라미터 값보다 최종 조인될 테이블의 수가 많을 경우 서브쿼리를 상위쿼리에 합치지 않는다. 이 그렇기에 run-time 파라미터를 수정함으로써 planning 시간과 plan의 퀄리티를 조절하여 사용할 수 있다.

## 3\. from\_collapse\_limit, join\_collapse\_limit 설정

from\_collapse\_limit와 join\_collapse\_limit  은 비슷한 역할을 하기에 네이밍이 비슷하게 되어있다. from\_collapse\_limit는 서브쿼리 사용 시 하위 쿼리를 상위쿼리에 포함시키는 조건을 제어하고, join\_collapse\_limit는 조인 테이블의 수에 따른 테이블 축소 관계의 최대 수를 제어한다. 일반적으로 두 파라미터의 값은 똑같이 설정하거나(명시적 조인과 서브쿼리가 비슷하게 행동하게 하기 위해) 아니면 join\_collapse\_limit을 1로 세팅한다.(명시적 조인의 순서를 컨트롤하고 싶을 때) 그러나 planning 시간과 run time 시간을 적절하게 조정하기 위해 다른 값을 부여할 수 있다.

### 3-1. from\_collapse\_limit (integer) 

-   planner는 FROM 절 테이블 개수가 파라미터 값보다 작을 경우 서브쿼리를 부모쿼리로 재배열한다.
-   적은 값들은 planning 시간을 줄여주지만 최적의 쿼리 plan을 찾을 수 없을 수도 있다.
-   기본 값은 8이다.

### 3-2. join\_collapse\_limit (integer)

-   planner는 FROM 절 테이블 개수가 파라미터 값보다 작을 경우 명시적인 JOIN절을 다시 구성한다. 
-   작은 값일수록 planning 시간을 줄여주지만, 최적의 쿼리 플랜을 찾을 수 없을 수도 있다.
-   기본값은 from\_collapse\_limit과 동일한 값이며 대부분의 경우에 적절한 값이다.
-   해당값을 1로 설정하는 것은 명시적 Join절의 순서 변환 없이 그대로 사용하게 한다.
-   쿼리가 항상 이상적인 쿼리 플랜을 선택하는 것이 아니기 때문에, 쿼리를 통해 변수를 일시적으로 1로 설정한 후 원하는 조인순서를 명시적으로 지정할 수 있다.

## 4\. 적용

테스트 결과 FROM절의 순서를 변경함으로써 PostgreSQL Planner의 계획대상을 변경하는 방법은 두 가지가 있다.

### 4-1. join\_collapse\_limit 파라미터의 값을 1로 설정

이 경우 from절의 순서에 맞게 플랜이 변경됨을 확인할 수 있다.

### 4-2. 서브쿼리에 OFFSET을 추가하여 서브 쿼리 축소를 방지한다.

다음 두쿼리는 결과는 같지만, 명시적 JOIN 절을 사용하여 Planner의 조인 순서를 변경한 케이스이다.

```
select a.*
   from a, b, c
  where a.id = b.id
and b.id  = c.id;
```

<p align="center"><img src="/img/imp.png"/></p>

\-> a와 b 테이블을 먼저 조인한 후 c 테이블 조인

```
select *
  from b
 cross join lateral
       (select a.*
          from a, c
         where a.id = c.id offset 0) as foo;
```

<p align="center"><img src="/img/imp2.png"/></p>

\-> a와 c 테이블을 먼저 조인한 후 b 테이블 조인

## 5\. 결론

from\_collapse\_limit, join\_collapse\_limit 파라미터를 통해 조인 및 서브쿼리 사용 시 쿼리 planner의 플랜 선택지를 조정하여 plan 선택시간을 줄일 수 있다. 다만 제한한 조건 내에 최상의 쿼리 플랜이 없을 경우 최적의 쿼리를 찾을 수 없기에, scan, index, 테이블의 크기 등에 따른 상관관계를 명확이 이해하고 추가해야 하며, 설정 전의 충분한 플랜 분석, 설정 후의 데이터 증감에 따른 지속적인 모니터링 및 튜닝이 필요하다.

참고

[https://www.postgresdba.com/bbs/board.php?bo\_table=C05&wr\_id=201](https://www.postgresdba.com/bbs/board.php?bo_table=C05&wr_id=201)

[https://www.postgresql.org/docs/current/explicit-joins.html](https://www.postgresql.org/docs/current/explicit-joins.html)