## 1\. 순번 부여하기 

PostgreSQL에서는 각 데이터에 의미 있는 순번을 부여하기 위해 ROW\_NUMBER(), RANK(), DENSE\_RANK() 함수를 제공한다. 

```
ROW_NUMBER() OVER(PARTITION BY * ORDER BY * )

RANK() OVER(PARTITION BY * ORDER BY * )

DENSE_RANK() OVER(PARTITION BY * ORDER BY * )
```

예제를 통해 자세한 사용법을 알아보자. (2. 테스트 테이블 & 데이터 생성 참고)

<p align="center"><img src="/img/rownum1.png"/></p>

### 1-1. ROW\_NUMBER()

####           1-1-1. 단일 그룹 순번 부여

```
SELECT ROW_NUMBER() OVER (ORDER BY BRAND) AS ROWNUM, *
FROM TEST_COMPLEX_GROUP;
```

<p align="center"><img src="/img/rownum2.png"/></p>

특정 그룹에 대한 설정 없이 순번만을 지정할 경우, 전체 데이터를 기준으로 명시된 order by 순서대로 순번을 부여한다. 예제에서는 BRAND의 순서대로 정렬된 순번을 차례로 부여하였다.

####           1-1-2. 다중 그룹 순번 부여

보통 단일 그룹으로 순번을 부여하기보다, 부서별, 사이즈별 순번 등 다중 그룹에 대한 개별 순번을 부여할 일이 많이 있다. 그럴 경우 다음과 같이 PARTITION BY를 통해 순번 그룹을 명시해 주면 그룹별 순번을 확인할 수 있다.

```
SELECT ROW_NUMBER() OVER (PARTITION BY BRAND ORDER BY SALES) AS ROWNUM, *
FROM TEST_COMPLEX_GROUP;
```

<p align="center"><img src="/img/rownum3.png"/></p>

물론 PARTITION BY에 여러 그룹을 명시함으로써 여러 그룹에 대한 순위를 얻을수도 있다.

```
SELECT ROW_NUMBER() OVER (PARTITION BY BRAND, COLOR ORDER BY SALES) AS ROWNUM, *
FROM TEST_COMPLEX_GROUP;
```

<p align="center"><img src="/img/rownum4.png"/></p>

### 1-2. RANK()

RANK()는 같은 순위의 값에 대해 같은 순위를 부여하고, 그 다음 순위는 중복된 만큼을 스킵한 다음 순번을 부여한다. (공동 3위가 두명일 경우, 다음은 5위)

```
SELECT RANK() OVER (PARTITION BY BRAND ORDER BY SALES) AS ROWNUM, *
FROM TEST_COMPLEX_GROUP;
```

<p align="center"><img src="/img/rownum5.png"/></p>

### 1-3. DENSE\_RANK()

DENSE\_RANK()는 같은 순위의 값에 대해 같은 순위를 부여하고, 그다음 순위는 중복순위에 상관없이 연속된 순번을 부여한다. (공동 3위가 두명일 경우, 다음은 4위)

<p align="center"><img src="/img/rownum6.png"/></p>

## 2\. 테스트 테이블 & 데이터 생성

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