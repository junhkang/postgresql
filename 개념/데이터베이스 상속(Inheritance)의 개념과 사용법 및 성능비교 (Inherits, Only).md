## 1\. 데이터베이스 상속(Inheritance)이란?

상속은 객체지향 데이터베이스의 개념 중 하나이다. PostgreSQL은 테이블 생성 시 하나 이상의 다른 테이블로부터의 상속 기능을 제공하며, 이를 잘 활용하면 데이터베이스 설계에 새로운 가능성들을 열어준다. 데이터뿐만 아니라 부모 테이블의 컬럼 속성 및 인덱스 등의 특징들도 자식 테이블로 상속되기에 상황에 따라 효율적인 설계가 가능하다.

## 2. 데이터베이스 상속(Inherits) 방법

다음 예제는 PostgreSQL 공식 문서의 예제이다.

> **Capitals -** 이름, 인구, 고도, 요약어를 포함한 수도의 정보가 포함된 테이블  
> **Cities -** 이름, 인구, 고도를 포함한 도시 정보가 포함된 테이블

수도는 도시에 포함되기에, 전체 도시 리스트를 보여줄 때, 수도 리스트를 동시에 보여주는 상황이 있을 것이다. 이 경우 상속을 사용하지 않고는 보통 이런 식으로 스키마를 작성한다.

```
CREATE TABLE capitals (
  name       text,
  population real,
  elevation  int,    -- (in ft)
  state      char(2)
);

CREATE TABLE non_capitals (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE VIEW cities AS
  SELECT name, population, elevation FROM capitals
    UNION
  SELECT name, population, elevation FROM non_capitals;
```

수도(capitals)와 도시(cities) 리스트를 중복 없이 보여주는 기능은 잘 작동하겠지만, 양쪽에 중복된 데이터가 있을 경우, 데이터 변경 시 각각 테이블의 행을 업데이트해야 하기에 효율적이지 못하다. 이 같은 경우 다음과 같이 상속을 사용하면 더 효율적이다.

```
CREATE TABLE cities (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);
```

수도(capitals) 테이블의 모든 row는 부모테이블인 도시(cities)의 모든 컬럼들 (name, population, elevation)을 상속받는다.

예를 들어 다음 쿼리는 500피트 이상의 고도에 존재하는 수도를 포함한 모든 도시의 이름을 찾을 때 Cities 테이블로만 조회가 가능하다.

```
SELECT name, elevation
  FROM cities
  WHERE elevation > 500;
```

결과는 

```
  name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
 Madison   |       845
(3 rows)
```

## 3\. 상속 제외 (Only)

상속받은 테이블에서 특정 테이블의 결과를 제외하고 싶을 때 어떻게 하면 될까? 다음 쿼리는 수도가 아닌 도시 중에서 500 피드 이상의 고도를 가진 도시를 추출하는 쿼리이다.

```
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;
```

```
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
(2 rows)
```

테이블 명 앞에 ONLY를 추가하면, 상속계층 상의 다른 테이블이 아닌 해당 테이블에서만 조회를 한다.

SELECT, UPDATE, DELETE 모두 ONLY 구문을 지원한다.

## 4\. 테이블 상속의 특징

-   부모테이블에 대한 조회/수정/삭제는 자식 테이블을 포함하여 동작된다.
-   상속받은 테이블을 상속받는 또 다른 테이블이 생성될 때, 부모 테이블의 수정은 전계층 상속테이블에 영향을 준다.
-   부모테이블의 인덱스를 생성 시 자식 테이블에도 동일하게 생성된다. 
-   create index cities\_test\_name\_index on cities\_test (name);
-   자식테이블을 타깃으로 부모테이블 컬럼에 인덱스도 생성 가능하다.
-   create index cities\_test\_population\_index on capitals\_test (population);

## 5\. 적용 후 성능비교

상속을 받아 테이블을 생성 시, 테이블 자체의 속성을 상속받을 뿐 조회를 위해서는 양쪽 테이블을 다 스캔해야 하기에 성능 자체가 급격하게 증가하진 않는 것으로 예상하였다. 실제로 대량 데이터(500만 건 이상의 테이블) 2개를 union 하여 사용하고 있는 경우가 있어, 상속을 통해 성능비교를 해보았다.

### 5-1. 조건 없이 조회, 카운트

**a. Union (table\_a, table\_b)**

<p align="center"><img src="/img/inherit.png"/></p>

**b. inherit 테이블 (table\_parent, table\_child1)**

<p align="center"><img src="/img/inherit2.png"/></p>

기존 Union은 각 테이블에 seq\_scan 후 정렬하고 상속 테이블은 부모, 자식 각 테이블 seq\_scan 후 바로 완료한다.

\-> 상속 테이블을 사용 시 정렬 및 중복여부 판단 등에 대한 cost가 생략되기에 효과적이다.

### 5-2. 조건을 넣고 조회 시

<p align="center"><img src="/img/inherit3.png"/></p>
<p align="center"><img src="/img/inherit4.png"/></p>

둘 다 인덱스가 적용/미적용된 상태의 검색을 할 경우, 2개의 테이블을 각각 조회하는 코스트가 소모되며 현재 테스트 데이터에는 중복되는 값들이 극소량이라 차이가 미비하지만, 중복되는 값이 많거나 정렬이 고려되어야 하는 쿼리를 사용 시 상속 테이블 사용 시의 성능이 더 나을 것으로 보인다.

데이터의 분포 및 구조에 따라 성능은 달라질 수 있고, 단순한 조회의 성능이 아니라 실제 데이터의 유지 보수 및 운영상황에 따라 상황에 맞게 상속 테이블을 적용하면 효율적인 데이터베이스 설계가 가능할 것으로 보인다.

\* 상속 테이블 적용 후 테이블의 성능 지표를 테스트하려면 아래 포스트의 쿼리 플랜 분석법을 확인하면 된다.


[\[PostgreSQL\] 쿼리 성능향상 (실행계획 보는 법, 상세 확인방법, Explain의 어떤 지표를 봐야 할까?)](https://github.com/junhkang/postgresql/blob/main/%EC%BF%BC%EB%A6%AC%20%EC%84%B1%EB%8A%A5%ED%96%A5%EC%83%81%20(%EC%8B%A4%ED%96%89%EA%B3%84%ED%9A%8D%20%EB%B3%B4%EB%8A%94%20%EB%B2%95%2C%20%EC%83%81%EC%84%B8%20%ED%99%95%EC%9D%B8%EB%B0%A9%EB%B2%95%2C%20Explain%EC%9D%98%20%EC%96%B4%EB%96%A4%20%EC%A7%80%ED%91%9C%EB%A5%BC%20%EB%B4%90%EC%95%BC%ED%95%A0%EA%B9%8C%3F).md)

참고

[https://www.postgresql.org/docs/current/tutorial-inheritance.html](https://www.postgresql.org/docs/current/tutorial-inheritance.html)