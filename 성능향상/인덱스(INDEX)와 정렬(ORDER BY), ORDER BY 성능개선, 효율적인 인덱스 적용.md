## 1\. 인덱스(INDEX)와 정렬(ORDER BY)

인덱스는 쿼리의 결과로 특정 row를 찾는 것뿐만 아니라, 특정 순서로 데이터를 정렬하는데도 효율적일 수 있다. ORDER BY와 인덱스를 효율적으로 사용하면 별도의 정렬 과정 없이 ORDER BY를 수행할 수 있다. PostgreSQL에서 현재 지원하는 인덱스 타입 중에서는 B-tree 인덱스만이 정렬 결과로 인덱스를 생성할 수 있다. 다른 인덱스 유형은 특정되지 않은 순서로, 실행 때마다 다른 순서로 열을 반환한다.

\* 상세한 B-tree 인덱스의 개념은 다음 글을 참고 - [\[Postgresql\] B-tree 인덱스의 원리 및 특징](https://github.com/junhkang/postgresql/blob/main/%EA%B0%9C%EB%85%90/%EC%9D%B8%EB%8D%B1%EC%8A%A4/B-tree%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%EC%9D%98%20%EC%9B%90%EB%A6%AC%20%EB%B0%8F%20%ED%8A%B9%EC%A7%95.md)

플래너는 ORDER BY를 수행할 때 

-   해당 조건에 맞는 사용 가능한 인덱스를 스캔
-   테이블을 물리적으로 스캔하여 명시적으로 정렬을 수행한 후 ORDER BY 사양에 충족하는 row를 스캔 (실제 테이블 스캔)


중 효율적인 스캔을 실행한다. 테이블의 많은 부분을 조회하는 쿼리의 경우, 명시적 조회가 인덱스를 조회하는 것보다 빠르다. (대량 데이터를 조회할 시에는 데이터를 순차적 접근 패턴이 디스크 I/O를 덜 필요로 하기 때문이다.) 이는 기본적인 인덱스의 특징과 동일하고, 적은 수의 row를 반환하는 경우에 더 유용하다. (선택도가 낮을수록 효율적, 10~15%) 

\* 효율적인 인덱스 설계 및 작동 방식은 다음 글의 2-1, 2-3 참고 - [\[Postgresql\] 인덱스(INDEX) 개념 및 생성, 삭제, 분석, 설계 방법](https://github.com/junhkang/postgresql/blob/main/%EA%B0%9C%EB%85%90/%EC%9D%B8%EB%8D%B1%EC%8A%A4/%EC%9D%B8%EB%8D%B1%EC%8A%A4(INDEX)%EA%B0%9C%EB%85%90%20%EB%B0%8F%20%EC%83%9D%EC%84%B1%2C%20%EC%82%AD%EC%A0%9C%2C%20%EB%B6%84%EC%84%9D%2C%20%EC%84%A4%EA%B3%84%20%EB%B0%A9%EB%B2%95.md)

특히 LIMIT n과 ORDER BY를 결합하여 결과 값을 제한하는 경우가 인덱스 테이블의 사용이 효율적이다. 이 경우 명시적 조회는 첫 n개의 rows를 반환하기 위해 전체 데이터를 조회해야 하지만, 해당 ORDER BY와 일치하는 인덱스가 있다면 첫 n개의 row는 나머지 row를 조회할 것 없이 바로 출력된다.

기본적으로 B-tree인덱스는 오름차순(ASC)에 NULLS LAST로 정렬된 채로 데이터를 저장한다. (같은 순서의 경우 테이블의 TID를 기준으로 정렬) 그렇기 때문에, 칼럼 x의 인덱스를 일반적인 정방향 스캔을 할 경우, x칼럼의 오름차순(**ORDER BY x ASC NULLS LAST**)의 정렬과 동일한 결과가 출력된다.

정렬 인덱스는 정렬 설정된 방향의 역방향으로도 스캔될 수 있기 때문에 **ORDER BY x DESC NULLS FIRST**에 대한 정렬도 인덱스 스캔할 수 있다.

## 2\. ORDER BY 인덱스 생성

B-tree 인덱스를 생성할 때 ASC, DESC, NULLS FIRST, NULLS LAST 옵션을 부여하여 정렬을 조정할 수 있다.

```
CREATE INDEX test2_info_nulls_low ON test2 (info NULLS FIRST);
CREATE INDEX test3_desc_index ON test3 (id DESC NULLS LAST);
```

칼럼 x에 대해 ASC NULLS FIRST로 저장된 인덱스는 스캔 방향에 따라 x ASC NULLS FIRST 혹은 x DESC NULLS LAST의 쿼리에 효과를 볼 수 있다는 것인데, 여기까지 보면 정방향, 역방향 (역스캔 가능한) 옵션이 ORDER BY 모든 변형을 포함할 수 있다. ORDER BY의 모든 변형 4가지를 살펴보면 다음과 같은데

> 1\. ASC NULLS FIRST  
> 2\. DESC NULLS LAST  
> 3\. ASC NULLS LAST  
> 4\. DESC NULLS FIRST

1,2번과 3,4 번은 서로의 역방향에서 조회가 가능하기에 인덱스를 각각 한 개씩(1번 3번, 2개) 적용해도 네 개를 개별적용 하는 것과 같은 효과를 볼 수 있다. 그렇다면 4가지 적용 방식이 아닌 2가지 적용방식으로도 충분히 ORDER BY 인덱스를 컨트롤할 수 있을 텐데, 왜 4가지 적용 방식이 존재할까?

단일 칼럼의 인덱스를 설정하기에는 해당 옵션이 충분하지만, 복합 칼럼 (Multi-column) 인덱스를 설정할 때는 옵션의 다양성이 유리할 수 있다. 예를 들어 **x, y칼럼에** 인덱스가 있다고 가정해 보면, 정방향 스캔은 **ORDER BY x, y** 혹은 역방향은 **ORDER BY x DESC, y DESC** 가 될 것이다. 하지만 만약 **ORDER BY x ASC, y DESC**의 조회가 애플리케이션에서 자주 일어날 수 있다면 기존 일반 인덱스로는 해당 순서 조회를 할 수 없고, 인덱스가 **x ASC, y DESC** 혹은 **x DESC, y ASC**로의 인덱스 조회가 가능할 것이다.

PostgreSQL 공식문서에 의하면, 디폴트 정렬에 맞지 않는 ORDER BY 인덱스는 상당히 전문화된 기능이지만 특정 쿼리에 한해서 굉장히 큰 효율을 보일 수 있다. 특정 순서로 정렬된 상태의 조회가 얼마나 자주 일어나냐에 따라 인덱스를 사용여부를 판단하면 하면 될 것이다.

## 3\. 성능 비교 및 결론

기존에 ORDER BY 자체를 인덱싱에 포함시켜 정렬된 자체로 인덱스 테이블을 생성할 수 있는지는 알고 있었다. 하지만

-   동일 인덱스 테이블을 역방향 조회할 때도 인덱스의 효과를 볼 수 있는지는 인지하지 못하였다. (생각해 보면 정방향으로 정렬된 인덱스테이블의 마지막 부분을 역방향으로 조회하면 동일한 효과를 보는 건 인덱스의 당연한 개념이고 다른 부분에도 그렇게 적용 사용 중이 많았음에도 불구하고)

> **ASC NULLS LAST <=> DESC NULLS FIRST**

-   칼럼 자체에 인덱스 설정 시 디폴트가 ASC NULLS LAST, DESC NULLS LAST인지 알지 못하였다. 

그래서 역방향, 정방향 인덱스를 둘 다 생성해 보고, 해당 칼럼자체의 인덱스도 별도로 생성하여 테스트해 보니 다른 인덱스의 효율을 떨어 트리는 상황이 발생하여서 인덱스 튜닝 시점에 롤백을 했었었다. 정확한 영향도 파악과 성능 비교를 위해 ORDER BY 인덱스를 부여 시 정확히 어떤 효과를 보는지 테스트를 진행하였다.

> 테스트 테이블(**order\_by\_index\_test**) \[1857637 ROWS\]  
> **id -** PK  
> **name -** VARCHAR not null  
> **non\_null\_varchar -** NOT NULL VARCHAR(8)  
> **null\_varchar -** NULLABLE VARCHAR(8)

\* 쿼리 성능 분석하기에 대한 상세 내용은 다음 포스트를 참고

[\[Postgresql\] 쿼리 성능향상 (실행계획 보는 법, 상세 확인방법, Explain의 어떤 지표를 봐야 할까?)](https://github.com/junhkang/postgresql/blob/main/%EC%84%B1%EB%8A%A5%ED%96%A5%EC%83%81/%EC%BF%BC%EB%A6%AC%20%EC%84%B1%EB%8A%A5%ED%96%A5%EC%83%81%20(%EC%8B%A4%ED%96%89%EA%B3%84%ED%9A%8D%20%EB%B3%B4%EB%8A%94%20%EB%B2%95%2C%20%EC%83%81%EC%84%B8%20%ED%99%95%EC%9D%B8%EB%B0%A9%EB%B2%95%2C%20Explain%EC%9D%98%20%EC%96%B4%EB%96%A4%20%EC%A7%80%ED%91%9C%EB%A5%BC%20%EB%B4%90%EC%95%BC%ED%95%A0%EA%B9%8C%3F).md)

### 3-1. 단일 인덱스 칼럼 정방향 ORDER BY

id 칼럼에 PK만 부여된 상태 (인덱스 부여)로 ORDER BY 정렬하면 기본 정렬인 **ORDER BY id ASC NULLS LAST, ORDER BY id DESC NULLS FIRST**에서만 인덱스 효과를 볼 수 있다.

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY ID ASC NULLS LAST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY ID DESC NULLS FIRST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex.png"/></p>

하지만 반대 정렬의(**ASC NULLST FIRST, DESC NULLS LAST**) 경우 인덱스의 효과를 볼 수 없다.

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY ID ASC NULLS FIRST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY ID DESC NULLS LAST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex2.png"/></p>

### 3-2. NOT NULL 속성 칼럼에 기본 인덱스 부여

NOT NULL 인 칼럼에 기본 인덱스를 부여할 경우 **ORDER BY id ASC NULLS LAST, ORDER BY id DESC NULLS FIRST** (동일 순서가 없다면 결과는 같다.)의 효과를 동시에 볼 수 있을까?

```
CREATE INDEX ORDER_BY_INDEX_TEST_NOT_NULL_VARCHAR_INDEX ON ORDER_BY_INDEX_TEST (NOT_NULL_VARCHAR);
```

기본 정렬에 대해서는 여전히 인덱스를 타지만,

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR ASC NULLS LAST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR DESC NULLS FIRST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex3.png"/></p>

반대 정렬은 여전히 인덱스의 효과를 볼 수 없다.

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR ASC NULLS FIRST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR DESC NULLS LAST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex4.png"/></p>

\-> 칼럼 자체에 NULL 데이터가 없어서 NULLS FIRST, NULLS LAST의 결과가 같더라도, 인덱스 테이블은 별도로 생성되며 반대로 정렬 시 인덱스의 효과를 볼 수 없다.

### 3-3. 기본 정렬 인덱스와, 역방향 정렬 인덱스를 동시에 부여

기본 정렬과 역방향 정렬의 인덱스를 동시에 부여할 시 성능차이가 어떻게 될까?

```
CREATE INDEX ORDER_BY_INDEX_TEST_NOT_NULL_VARCHAR_ASC_FIRST_INDEX ON ORDER_BY_INDEX_TEST (NOT_NULL_VARCHAR ASC NULLS FIRST);
```

역방향으로 인덱스를 설정하고 다시 조회를 할 경우,

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR ASC NULLS FIRST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR DESC NULLS LAST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex5.png"/></p>

역방향은 인덱스 효과를 볼 수 있고,

```
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR ASC NULLS LAST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST ORDER BY NOT_NULL_VARCHAR DESC NULLS FIRST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex6.png"/></p>

기존 정방향도 인덱스 효과를 볼 수 있다.

하지만 지금 테이블 같은 경우는 간단한 구조의 단일 테이블 조회이고 캐시나 다른 영향도 있을 수 있다. 예전에 도입 검토 했을 당시, 복잡한 대량 데이터의 쿼리를 기준으로 적용할 시(해당 시점에 120만 건 x 500만 건 x 1200만 건 + 1:1의 데이터 조회였던 걸로 기억한다.), 양방향 인덱스를 모두 설정할 경우, 플랜이 변경되어 기존 인덱스가 적용되지 않거나, 둘 중 코스트가 낮은 한 방향 인덱스만 적용되는 현상이 있었다. 플래너가 한정적인 자원에서 최상의 인덱싱 효과를 위해 실행계획을 짜는 만큼 복잡한 구조에서는 실제로 의도한 방향으로 쿼리가 플래닝 되는지 꼭 확인해보아야 할 듯하다. 해당 테스트에는, 양방향으로도 인덱스 설정이 가능하다 정도를 확인하면 좋을 듯하다. 

### 3-4. NULLABLE 테이블에서, NOT NULL 조건과 함께 정렬

NULLABLE 테이블에서 NOT NULL 조건을 줄 경우 NULLS LAST와 NULLS FIRST의 결과가 동일할 텐데(동순위가 없을 경우), 한 방향 인덱스로 해결할 수 있지 않을까? (2번의 NOT NULL 칼럼과 동일)

보통 정렬 기능을 제공할 때 단방향만 제공하는 경우는 잘 없고 산군의 경우 ASC NULLS LAST, DESC NULLS LAST와 같이 NULL 데이터를 배제한 상태로 조회하는 경우가 많기에 성능을 비교해 보았다. 

```
CREATE INDEX ORDER_BY_INDEX_TEST_NULL_VARCHAR_INDEX ON ORDER_BY_INDEX_TEST (NULL_VARCHAR);
```

다음과 같이 정방향으로 인덱스를 설정하고 해당 칼럼에 NOT NULL 조건을 부여한 후 조회를 시도하면

```
SELECT * FROM ORDER_BY_INDEX_TEST WHERE NULL_VARCHAR IS NOT NULL ORDER BY NULL_VARCHAR ASC NULLS FIRST LIMIT 1000;
SELECT * FROM ORDER_BY_INDEX_TEST WHERE NULL_VARCHAR IS NOT NULL ORDER BY NULL_VARCHAR DESC NULLS LAST LIMIT 1000;
```

<p align="center"><img src="/img/orderbyindex7.png"/></p>

역방향은 NOT NULL 여부에 상관없이 역방향 인덱스의 효과를 볼 수 없다.

### 3-5. 결론 및 적용 검토

PostgreSQL 공식 문서의 설명대로, 단순히 인덱스 자체에 순서를 붙이는 것이 아닌 훨씬 고도화된 기능인듯하다. 위에서 설명한 것처럼 일전에 테스트 시 복잡한 쿼리의 경우 역방향 인덱스를 거는 것만으로 기존 플랜에 영향을 주는 것을 확인하였다. 단방향으로만 조회하는 것이 확실한 상황에서는 상당한 성능개선이 있겠지만, 양방향 조회나 기존 플랜에 영향을 주는 경우, ORDER BY 인덱스 튜닝을 검토 시 해당 테이블을 참조하는 쿼리들의 사용 빈도 및 영향도를 파악하여야 하고, 충분한 테스트 및 플랜 분석이 선행되어야 할 듯하다.


