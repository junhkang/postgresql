## 1\. Index-Only Scans

PostgreSQL의 모든 인덱스는 **"보조(Secondary)"** 인덱스이다. 각 인덱스는 테이블의 메인 데이터 영역(테이블의 **heap** 영역)과 분리되어서 저장된다. 그렇기 때문에 일반적인 인덱스 스캔에서 각 ROW를 찾기 위해서는, index와 heap 영역 모두에 접근하여 데이터를 탐색해야 한다. 보통 WHERE 절 조건에 부합하는 데이터들은

-   **인덱스 영역 -** 서로 가까이 존재하여 정렬된 순서로 빠르게 접근할 수 있다. (인덱스 테이블은 정렬된 상태로 생성)
-   **heap 영역 -**  특별한 규칙 없이 어디에서든 분포할 수 있기에 heap 영역을 스캔할 때는 무작위로 접근하게 되어 속도가 느리다.

이 퍼포먼스 문제를 해결하기 위해 PostgreSQL은 힙 영역에 대한 접근 없이 인덱스 내에서만 데이터를 조회하는 **Index-only** 스캔을 지원한다. 기본 개념은 말 그대로 heap 영역의 참조 없이 index 항목에서 바로 값을 반환하는 것으로 매우 효율적으로 보이지만 몇 가지 제한사항이 있다.

-   칼럼에 적용된 인덱스 유형이 **Index-only** 스캔을 지원해야 한다. (B-tree인덱스는 언제나 지원하고, GiST, SP-GiST는 특정 연산자에 한해서만 지원, 나머지 3가지 인덱스는 지원하지 않는다.)
-   인덱스가 각 항목에 대해 원래 데이터 값을 **물리적으로 온전히 저장하거나 재구성**할 수 있어야 한다. 예를 들어 GIN인덱스가 **Index-only** 스캔을 지원하지 않는 이유는 각 인덱스가 실제 데이터의 물리적인 값이 아닌 일부 특징 (ex. 최대, 최솟값)만을 가지고 있는 유형의 인덱스 이기 때문이다.
-   실행되는 쿼리가 인덱스로 설정된 칼럼만을 조건절에 참조해야 한다. 예를 들어 x, y칼럼에 인덱스가 설정되어 있고, z 칼럼에 인덱스가 설정되어 있지 않다면, 다음 쿼리는 index-only 스캔을 사용할 수 있다.

```
SELECT x, y FROM tab WHERE x = 'key';
SELECT x FROM tab WHERE x = 'key' AND y < 42;
```

반면에 다음 쿼리는 인덱스가 적용되지 않은 z 칼럼이 조건절, 혹은 target에 포함되어 있기에 index-only 스캔 사용이 불가능하다.

```
SELECT x, z FROM tab WHERE x = 'key';
SELECT x FROM tab WHERE x = 'key' AND z < 42;
```

이 조건들에 부합하면, 쿼리의 결과에 해당하는 데이터가 인덱스 영역 내에 모두 존재한다는 것이기에 **Index-only** 스캔이 가능하다. 

## 2\. Visiblity

위의 조건에 부합하여 Index-only 스캔이 가능하더라도, 효율적인 스캔을 위해 PostgreSQL에서는 테이블 검색에 대한 추가 요구 사항이 있다. 바로 검색 결과의 각 ROW가 쿼리의 MVCC 스냅숏에 "보이는지(visible)"이다. MVCC는 PostgreSQL에서 동시성을 지원하는 원칙이며 (상세 내용은 다음 링크 참고 - [\[PostgreSQL\] MVCC (Multi-Version Concurrency Control)](https://github.com/junhkang/postgresql/blob/main/%EA%B0%9C%EB%85%90/MVCC%20(Multi-Version%20Concurrency%20Control).md))

> **Visiblity -** 해당 쿼리가 실행되는 시점에서 데이터베이스의 상태, 혹은 스냅숏에 따라 해당 쿼리에 의해 결과 데이터가 조회, 혹은 조작될 수 있는지의 의미이다. (다른 트랜잭션에 의해 삭제되거나 수정되지 않은 유효한 데이터가 반환될 수 있는지를 의미한다.) PostgreSQL은 테이블 heap의 각 페이지에 대해 모든 ROWS가 현재, 미래의 모든 트랜잭션에서 볼 수 있을 만큼 오래되었는지 여부를 추적한하여 이 정보를 테이블의 Visiblity map에 bit 형태로 저장한다.

하지만 설명을 보면 Visiblity 정보는 인덱스 영역이 아닌 heap영역에만 저장된다. 그렇다면 Visiblity를 조회하기 위해서는 heap영역을 무조건 조회해야 하는 것이 아닌가?라는 의문이 들것이다.

테이블 ROWS가 최근에 변경된 경우에는 인덱스 테이블이 업데이트되기 전일테니 당연히 매번 heap 영역을 조회해야 하지만, 변경이 잦지 않은 데이터의 경우 이 문제를 해결할 수 있는 방법이 있다.

**Index-only** 스캔은 인덱스 내에서 데이터를 찾은 후 그에 해당하는 Visiblity map bit만을 확인하여 heap영역의 해당 page를 찾는다.

-   데이터가 확인되면 - 물리적인 데이터를 찾을 수 있기에 추가적인 작업은 필요 없다.
-   데이터가 확인되지 않는다면 -  데이터의 Visiblity를 확인을 위해 heap 영역을 접근해야 하기에 표준 인덱스 스캔보다 성능이 향상되지 않는다.

성공적으로 visiblity map bit를 찾더라도 이 방식은 Visiblity map을 조회해야 한다. 하지만 visiblity map 은 heap 보다 4배 적기 때문에 물리적 I/O가 훨씬 적게 든다. 그리고 대부분의 상황에 Visiblity map은 메모리에 캐시 된 상태로 유지되기에 효율적인 조회가 가능하다.

요약하자면, **Index-only** 스캔은 기본적인 요구조건을 만족하면 사용은 가능하지만, heap 테이블의 각 pages의 대부분이 visiblity map을 가지고 있을 때 효율을 볼 수 있다.

## 3\. Covering index

**Index-only** 스캔을 효율적으로 사용하기 위해서는, 자주 사용하는 특정 쿼리의 ROW들을 포함하도록 특별히 설계된 인덱스인 커버링 인덱스(**COVERING INDEX**)를 생성하면 된다.

일반적으로 쿼리는 검색의 키로 사용되는 칼럼(WHERE 절의 칼럼) 외에도 훨씬 더 많은 칼럼을 조회(SELECT 하는 칼럼)하기에, PostgreSQL은 검색의 key로 사용되는 인덱스가 아닌 단순히 payload(부가 데이터) 일뿐인 인덱스를 생성할 수 있게 해 준다. (실제 검색되는 결과 칼럼을 조회하는 것이 아니기에 조회되는 칼럼에만 검색 키를 추가하고, 결과 칼럼은 부가적인 정보로만 가볍게 추가)

다음과 같이 INCLUDE와 검색되는 칼럼의 같이 부여하면 사용 가능하다. 

```
SELECT y FROM tab WHERE x = 'key';
```

일반적으로 위와 같은 쿼리를 실행시킨다고 했을 때, 기본적인 속도 개선의 방법은 x 칼럼에만 인덱스를 실행시키는 것이다. 

하지만 다음과 같은 인덱스를 생성한다면, 해당 쿼리를 index-only 스캔방식으로 사용 가능하다.

```
CREATE INDEX tab_x_y ON tab(x) INCLUDE (y);
```

y의 결괏값들을 heap 영역에 접근하지 않고 가져올 수 있기 때문이다.

**Covering Index**인 y는 검색 키는 아니기에, 인덱스가 처리할 수 있는 데이터 유형일 필요는 없다. 인덱스 테이블에 저장되어 있을 뿐이며 검색을 하는 데 사용되거나 내부 메커니즘에서 인덱스로 해석되지 않는다. 

```
CREATE UNIQUE INDEX tab_x_y ON tab(x) INCLUDE (y);
```

그리고 다음과 같이 유니크 인덱스로 설정한다면, 해당 유니크 특성은 x, y가 아닌 칼럼 x에만 부여된다.  (INCLUDE 절은 UK, PK 제약조건에도 작성할 수 있다.)

인덱스영역을 차지하지만, 인덱스로 여겨지지 않고, 키에도 영향을 안주며 payload 형태로 저장만 한다면, 모든 쿼리를 등록하면 어떻게 되는 건가라는 의문이 드는데, Covering Index 사용 시 주의사항을 확인해 보자.

키가 아닌 칼럼(payload)을 인덱스, 특히 사이즈가 큰 칼럼에 대한 인덱스를 설정하는 것은 최대한 보수적으로 작업하는 것이 좋다. 인덱스 tuple이 인덱스 유형이 허락하는 최대 사이즈를 초과하면, 데이터 insert가 실패할 수 있다. 키가 아닌 칼럼(Payload)이 인덱스 테이블의 데이터를 복제하여 인덱스 크기를 증가시킬 수 있고 이는 검색 속도를 늦추는 현상은 언제든지 발생할 수 있다. 그리고 **Index-only** 스캔 시, 테이블 변경속도가 충분히 느려서 heap에 접근할 필요 없이 인덱스만으로 스캔할 가능성이 있지 않다면, 인덱스에 payload를 포함시키는 것은 의미가 없다는 것을 기억해야 한다.

## 4\. Covering Index VS 일반 Index

PostgreSQL이 INCLUDE를 지원하기 전에, 종종 페이로드 칼럼을 일반 인덱스 칼럼으로 작성하여 COVERING 인덱스를 생성하였다.

```
CREATE INDEX tab_x_y ON tab(x, y);
```

y칼럼을 조건절에 사용할 의도가 없었음에도 불구하고, 타깃 칼럼(y)이 실제 조회 키 (x) 뒤에 존재한다면 index-only 스캔은 잘 작동하지만 타깃 칼럼(y)을 선행 인덱스로 생성하는 것을 효율적이지 않다. (비효율적이지만 Covering 인덱스와 같은 효과를 볼 수 있다.)

COVERING 인덱스가 실제로 B-Tree 인덱스 구조에서 어떻게 작동하는 과정을 살펴보자

> **Suffix Truncation(접미사 축소) -** B-tree 인덱스 상위 레벨에서 non-key 칼럼(payload)을 제거하는 최적화 과정. 실제 검색에 필요한 키 칼럼 외에는 제거한다.  
> **Non-key 칼럼(payload) -** 인덱스 스캔을 직접적으로 가이드하지 않고, 검색용으로 사용되지 않음 (결과와 무관)  
> **축소 과정 -** 상위 레벨에서 payload를 제거한 후, 남아있는 키칼럼의 prefix로도 최하 B-tree 튜플이 설명이 가능하다면 (조회할 수 있다면) 뒤따르는 키 칼럼을 조회할 필요가 없기에 후속 키 칼럼을 제거. 필수 정보만을 유지하며 인덱스를 통한 검색을 최적화**Covering Index, Include -** Include 없이 Covering 인덱스를 사용하는 경우, B-tree 인덱스의 최상위 레벨에서 Payload 칼럼을 저장하지 않는다. 상위 레벨에선 키 칼럼만 저장하고 있다. 쿼리가 인덱스에 포함되지 않은 칼럼 데이터를 요구하는 경우 추가적인 테이블 접근이 필요하다.

커버링 인덱스와 payload가 B-tree 인덱스에서 어떻게 작동하는지를 확인한 후 Include 사용 여부에 따른 차이점을 확인해 보면 다음과 같다.

-   **일반 Index -** Include 없이 Covering 인덱스를 사용하는 경우, B-tree 인덱스의 최상위 레벨에서 Payload 칼럼을 저장하지 않는다. 상위 레벨에선 키 칼럼만 저장하고 있다. 쿼리가 인덱스에 포함되지 않은 칼럼 데이터를 요구하는 경우 추가적인 테이블 접근이 필요하다.
-   **Covering Index(Include) -** include의 경우 B-tree 인덱스의 최하위 leaf 레벨에 Payload 칼럼 정보를 저장하여, 키칼럼 외의 칼럼 데이터를 요구하는 경우에도 추가적인 테이블 접근 없이 데이터 조회가 가능하다.

## 5\. 표현식(expresions)과 Index-only 스캔

원칙적으로 index-only 스캔은 expression 인덱스와 함께 사용할 수 있다. 예를 들어 x 칼럼에 인덱스가 설정되어 있다면 다음 쿼리도 index-only scan이 가능하다.

```
SELECT f(x) FROM tab WHERE f(x) < 1;
```

만약 f() 함수가 고 코스트의 함수라면 매우 효과적인 쿼리가 될 것이다. 그러나 PostgreSQL 플래너는 이 경우에 그렇게까지 똑똑하게 Index-only 스캔을 적용하진 않는다.

플래너는 조건절의 모든 칼럼이 인덱스 테이블에서 사용이 가능할 때만 index-only 스캔을 고려하는데, 이 경우 f(x)라는 함수가 x의 칼럼만을 참조하고 있음에도, 직접적으로 x를 사용하지 않기에 플래너는 이를 알지 못하고 index scan을 할 수 없다는 결론만을 내린다.

이 경우 다음과 같이 x 자체를 include 칼럼에 추가하여 index-only스캔을 적용할 수 있다.

```
CREATE INDEX tab_f_x ON tab (f(x)) INCLUDE (x);
```

추가적인 주의사항으로, 해당 인덱스를 적용 시 f(x) 함수의 재연산을 피하는 것이 목적이라면, 함수의 구성이 where절에 그대로 있지 않은 경우 주의해야 한다. f(x) 함수를 통해 데이터를 필터링하거나 계산하는 로직이 있을 경우, 그 로직이 where 절에 그대로 포함되지 않는다면 PostgreSQL 플래너는 인덱스를 적절하게 사용하지 않는다. 보통 위와 같은 단순 테이블 조회는 인덱스 설정으로 간단하게 해결할 수 있지만, join 절 등으로 다른 테이블을 참조하게 될 경우 해결할 수 없다. (PostgreSQL 이후 버전에서 해결 가능할 수도 있다고 함)

## 6\. 부분 인덱스 (Partial Index)와 Index-only 스캔

부분인덱스는 index-only 스캔과 흥미로운 상호작용을 한다. 다음과 같이 조건절에 success를 추가한 채로 인덱스를 설정하면

```
CREATE UNIQUE INDEX tests_success_constraint ON tests (subject, target)
    WHERE success;
```

다음 쿼리에 index-only 스캔을 적용할 수 있다.

```
SELECT target FROM tests WHERE subject = 'some-subject' AND success;
```

하지만 where = success라는 조건은 인덱스의 결과 칼럼으로 그대로 사용할 수 없는데 어떻게 적용될 수 있을까?

결과 칼럼으로 그대로 사용할 수 없지만, 인덱스 자체에 조건을 걸었기에 인덱스에 있는 모든 항목은 success=true를 만족하게 된다.  이 경우 Planner는 런타임에서 해당 조건을 명시적으로 다시 확인할 필요가 없기에 **Index-only** 스캔이 가능한 것이다. (다만 9.6 이상 버전에서는 자동으로 해당 상황을 인지하고 **Index-only**스캔하지만, 이전 버전에서는 생성되지 않는다.)

## 7\. 결론 및 성능 테스트

이전 포스트의 ORDER\_BY\_INDEX\_TEST 테이블을 다시 사용하여 성능 테스트를 진행하였다.

> 테스트 테이블(**order\_by\_index\_test**) \[1857637 ROWS\]  
> **id -** PK  
> **name -** VARCHAR not null  
> **non\_null\_varchar -** NOT NULL VARCHAR(8)  
> **null\_varchar -** NULLABLE VARCHAR(8)

현재 **non\_null\_varchar** 칼럼에만 인덱스를 설정한 상황이다.

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR ASC NULLS LAST LIMIT 1000;
```

먼저 다음과 같이, 정렬 순서는 인덱스 효과를 볼 수 있게 설정하고, 결과 칼럼은 인덱스 외 전체 칼럼을 조회를 하면

<p align="center"><img src="/img/covering.png"/></p>

일반적인 INDEX SCAN을 타게 된다.

하지만 결과 칼럼 또한 인덱스 칼럼만을 조회하게 되면, 

```
SELECT NOT_NULL_VARCHAR FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR ASC NULLS LAST LIMIT 1000;
```

<p align="center"><img src="/img/covering2.png"/></p>

INDEX ONLY SCAN을 타게 되며 TOTAL COST가 약 1/4 정도 줄어든다. (위에서 언급한 것처럼 Visiblity table이 약 1/4 크기)

그렇다면 인덱스 외 칼럼을 조회하면서 인덱스 효과를 보기 위해 INCLUDE를 사용한 COVERING 인덱스를 사용해 보자

```
CREATE INDEX ORDER_BY_INDEX_TEST_NULL_VARCHAR_INDEX ON ORDER_BY_INDEX_TEST (NULL_VARCHAR) INCLUDE (NOT_NULL_VARCHAR)
```

다음과 같이 COVERING 인덱스를 추가하고

```
SELECT NOT_NULL_VARCHAR, NULL_VARCHAR FROM ORDER_BY_INDEX_TEST ORDER BY NULL_VARCHAR ASC NULLS LAST LIMIT 1000;
```

<p align="center"><img src="/img/covering3.png"/></p>

조회하게 되면, 키 칼럼에 해당하는 NULL\_VARCHAR 칼럼 외의 다른 칼럼도 SELECT 항목에 존재하지만 INDEX ONLY SCAN을 타면서 성능 효과를 볼 수 있다.

복잡하지 않은 테이블 조합에서, 조회되는 칼럼의 크기가 작고 명확하게 정의된 상태에서는 COVERING INDEX를 사용하는 것으로 엄청난 성능 향상을 볼 수 있는 것을 확인하였다. 다만 공식 문서의 주의사항 대로 남용할 경우 데이터 인서트시 실패하거나, 인덱스 사이즈가 커져 검색 성능저하가 언제든지 일어날 수 있기에, 인덱스 테이블의 사이즈 확인 및 쿼리 플랜 등을 정확히 확인하여 최대한 보수적으로 적용해야 한다.