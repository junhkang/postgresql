## 1\. SETS, CUBE, ROLLUP의 개념 및 사용법

고급 "GROUP BY"의 기능들로 PostgreSQL에서는 **SETS, CUBE, ROLLUP** 기능을 제공한다. 기본적인 콘셉트는 일반 GROUP BY와 동일하게 FROM / WHERE 절에서 선택된 데이터는 각각 지정된 그룹으로 GROUP BY 되고, 각 그룹에 대해 집계가 계산된 후, 결과가 반환된다.

\* 다음은 테스트로 사용할 테이블 정보이다. (마지막 장의 4. 테이블 & 데이터 생성 참고)

<p align="center"><img src="/img/group1.png"/></p>

### 1-1. GROUP BY SETS의 개념 및 사용법

**GROUPING SETS**의 각 하위 요소(subsets)들은 하나 이상의 열 혹은 표현식을 지정할 수 있으며 조합에 맞게 집계 결과를 별도로 계산한다.

```
SELECT 
	((brand), (size)), brand, size, sum(sales) 
FROM test_complex_group 
GROUP BY 
GROUPING SETS ((brand), (size), ());
```

<p align="center"><img src="/img/group2.png"/></p>

예시의 쿼리에서는 brand와 size를 독립적으로 설정했기에 

-   brand에 대한 집계
-   size에 대한 집계
-   전체 합계 (빈 괄호는 모든 행을 단일 그룹으로 취급하여 집계)

가 이루어졌지만 (brand, size)를 결합하여 GROUPING SETS로 사용하였다면 다음과 같은 기준으로 집계가 된다.

-   brand, size를 둘 다 그룹으로 취급하여 집계

```
SELECT
	(brand, size), brand, size, sum(sales)
FROM test_complex_group
GROUP BY
GROUPING SETS ((brand, size));
```

<p align="center"><img src="/img/group3.png"/></p>

GROUPING SETS가 비어있으면 모든 ROW를 1개의 그룹으로 그룹화시키는 것과 같다. (GROUP BY가 없는 집계함수의 경우와 동일) 

```
SELECT sum(sales) FROM test_complex_group GROUP BY GROUPING SETS (());
```

<p align="center"><img src="/img/group4.png"/></p>

### 1-2. ROLLUP의 개념 및 사용법

ROLLUP은 지정된 열(또는 표현식)에 대해 계층적인 집계를 생성한다. 여기서 계층적이라 함은 열을 기준으로 각 열의 조합 및 그 조합의 모든 가능한 접두사 부분 집합에 대해 데이터를 그룹화하고 집계한다는 뜻이다. 공식문서에 ROLLUP을 GROUPING SETS로 변환한 내용을 확인해 보면 이해에 도움이 된다.

```
ROLLUP ( e1, e2, e3, ... )

-- 동일

GROUPING SETS (
    ( e1, e2, e3, ... ),
    ...
    ( e1, e2 ),
    ( e1 ),
    ( )
)
```

이제 테스트 테이블에서 결과를 확인해 보자.

```
SELECT BRAND, SIZE, COLOR, SUM(SALES)
FROM TEST_COMPLEX_GROUP
GROUP BY ROLLUP (BRAND, SIZE, COLOR);
```

<p align="center"><img src="/img/group5.png"/></p>

결과에서 확인할 수 있듯이, (brand, size, color) 뿐 아니라 (brand, size), brand도 GROUPING 대상이 된다. 일반적으로 계층 구조의 데이터 분석에서 많이 사용된다. (전체, 부서, 파트별 총 월급 등을 한 번에 조회할 때)

### 1-3. CUBE의 개념 및 사용법

CUBE는 모든 가능한 부분집합을 GROUPING 대상으로 사용한다. ROLLUP 이 해당 요소를 접두사로 사용하는 요소만을 대상으로 하는 반면에 전체 경우의 수를 모두 조회한다. 공식 문서에 CUBE를 GROUPING SETS로 변환한 예제를 보면 이해에 도움이 된다.

```
CUBE ( a, b, c )
-- 동일
GROUPING SETS (
    ( a, b, c ),
    ( a, b    ),
    ( a,    c ),
    ( a       ),
    (    b, c ),
    (    b    ),
    (       c ),
    (         )
)
```

이제 테스트 테이블에서 결과를 확인해 보자.

```
SELECT BRAND, SIZE, COLOR, SUM(SALES)
FROM TEST_COMPLEX_GROUP
GROUP BY CUBE (BRAND, SIZE, COLOR);
```

<p align="center"><img src="/img/group6.png"/></p>

## 2\. 고급 GROUPING의 응용

### 2-1. SUBLISTS의 사용

CUBE, ROLLUP은 개별 표현 식나 괄호를 포함한 sublist를 포함할 수 있다. sublist를 사용할 경우 개별 GROUPING SETS를 생성하기 위한 단일 단위로만 취급된다. (sublist 내부의 각 항목별로 구분되진 않는다.)

```
CUBE ( (a, b), (c, d) )

--동일

GROUPING SETS (
    ( a, b, c, d ),
    ( a, b       ),
    (       c, d ),
    (            )
)
```

```
ROLLUP ( a, (b, c), d )

-- 동일

GROUPING SETS (
    ( a, b, c, d ),
    ( a, b, c    ),
    ( a          ),
    (            )
)
```

### 2-2. GROUPING의 중첩 사용

CUBE, ROLLUP은 GROUP BY 절에 직접 사용되거나, GROUPING SETS 절 내부에 중첩되어서 사용 가능하다. 만약 하나의 GROUPING SETS 절이 다른 절 내에 중첩되는 경우, 내부 절의 모든 요소는 외부 절에 직접 작성된 것과 동일한 결과를 출력한다.

GROUP BY 절에 여러 개의 GROUPING 요소가 있다면, 모든 GROUPING 요소에 대해 가능한 모든 조합이 기준으로 사용되기에 최종 GROUPING SETS 목록은 각 항목의 교차의 곱으로 생성된다.

```
GROUP BY a, CUBE (b, c), GROUPING SETS ((d), (e))

-- 동일

GROUP BY GROUPING SETS (
    (a, b, c, d), (a, b, c, e),
    (a, b, d),    (a, b, e),
    (a, c, d),    (a, c, e),
    (a, d),       (a, e)
)
```

실제 테스트 데이터에서 확인해 보자.

```
SELECT BRAND, SIZE, COLOR, SUM(SALES)
FROM TEST_COMPLEX_GROUP
group by brand,
         cube(size, color);
```

<p align="center"><img src="/img/group7.png"/></p>

결과를 보면 brand만으로 GROUPING 된 것이 아니라, 중첩된 GROUPING 절의 교차의 곱으로 생성된다.

-   brand : 2개 조합
-   cube (size, color)
    -   (size, color) : 6개 조합
    -   size : 2개 조합
    -   color : 3개 조합
    -   () : 1개 조합

단순 계산으로는 (6+2+3+1) x 2 = 24의 조합이 구성되지만, 데이터 분포상 

<p align="center"><img src="/img/group8.png"/></p>

브랜드별로 8개의 조합이 가능하기에 총 16개의 그루핑 결과가 노출된다.

### 2-3. DISTINCT

다중 그루핑된 항목을 여러 개 지정할 경우, 최종 그루핑 세트 목록이 중복될 수 있다.

```
SELECT BRAND, SIZE, COLOR, SUM(SALES)
FROM TEST_COMPLEX_GROUP
group by
         rollup(brand, color),
              rollup (brand, size);
```

이 경우,

-   rollup (brand, color) - (brand, color), brand
    -   brand, color의 모든 조합
    -   각 brand에 대한 color의 서브 토털
    -   전체 토털
-   rollup (brand, size) = (brand, size), size
    -   brand와 size의 모든 조합
    -   각 brand에 대한 size의 서브 토털
    -   전체 토털

를 교차로 GROUPING 하기에  다음 두 결과 셋을 교차로 GROUPING 하게 된다.

<p align="center"><img src="/img/group9.png"/></p>

두 조건을 교차로 GROUPING 한 후 유효한 값들만 출력된 결과는 다음과 같다.

<p align="center"><img src="/img/group10.png"/></p>

brand에 대한 서브 토털이 중복되어 나타나기에 최종적으로 전체 토털도 중복되어 출력된다.

의도하지 않은 중복이 발생한 경우, 단일 ROLLUP으로 쿼리를 튜닝해도 되지만, DISTINCT절을 사용하여 중복을 명시적으로 제거할 수도 있다.

```
SELECT BRAND, SIZE, COLOR, SUM(SALES)
FROM TEST_COMPLEX_GROUP
GROUP BY
	DISTINCT ROLLUP (BRAND, COLOR),
    ROLLUP ( BRAND, SIZE)
ORDER BY BRAND, SIZE, COLOR;
```

<p align="center"><img src="/img/group11.png"/></p>

결과를 중복 제거하는 것이 아닌, 그루핑 조건의 중복제거를 하는 것이기에 SELECT DISTINCT와는 다른 개념이다.

## 3\. 주의사항

### 3-1. 결과가 NULL인 집계

그룹에 포함되지 않은 다른 열의 값이 NULL인 경우, 특정 열이 그룹화되면서 그 결과로 해당 열의 값이 NULL이 되는 경우(ex. 집계함수를 사용하는 경우 특정 그룹에 해당하는 데이터가 없어 결과가 NULL인 경우) 사이의 구분이 되지 않아 같은 취급을 당하기에 주의해야 한다.

### 3-2. 명시적 ROW 생성

(a, b) 형태의 표현식은 일반적으로 쿼리 내에서 ROW 생성자로 인식되어 여러 개의 값을 하나의 행으로 묶는다.

```
SELECT (COLOR, SIZE)
FROM TEST_COMPLEX_GROUP;
```

<p align="center"><img src="/img/group12.png"/></p>

하지만 GROUP BY 절에서는 이러한 규칙이 최상위 수준의 표현식에는 적용되지 않으며 a, b를 별도의 표현식으로 해석하여 각각을 GROUPING 기준으로 사용한다. GROUPING 표현식 내에서 (a, b)를 하나의 단일 그루핑 기준으로 사용하고 싶다면  ROW(a, b)을 사용하여 명시적으로 행을 생성해야 한다.

> GROUP BY (a, b) = GROUP BY a, b!= GROUP BY ROW(a, b)

## 4\. 테스트 테이블&데이터 생성

```
CREATE TABLE TEST_COMPLEX_GROUP
(
    BRAND VARCHAR(10),
    SIZE  VARCHAR(1),
    COLOR VARCHAR(10),
    SALES INTEGER
)

INSERT INTO TEST_COMPLEX_GROUP (BRAND, SIZE, COLOR, SALES)
VALUES 
    ('FOO', 'L', 'BLUE', 10),
    ('FOO', 'M', 'BLUE', 20),
    ('FOO', 'L', 'RED', 30),
    ('BAR', 'M', 'RED', 15),
    ('BAR', 'L', 'GREEN', 5),
    ('BAR', 'M', 'GREEN', 25);
```

참고
[https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-GROUPING-SETS](https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-GROUPING-SETS)

블로그
https://junhkang.tistory.com/84