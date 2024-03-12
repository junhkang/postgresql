## 1\. 윈도우 함수 (Window Functions)란?

윈도우 함수는 행과 행 간의 관계를 쉽게 정의하기 위해 만든 함수이다. 이 기능은 일반 집계함수의 연산과 유사하지만, 일반 집계함수가 행 각각을 단일 그룹화해서 출력하는 반면에, 윈도우 함수는 각각의 행들이 **그룹화되지 않으며 별도의 ID**를 가진다. 그렇기에 윈도우 함수는 현재 row의 정보보다 더 많은 정보에 접근이 가능하다. 예를 들면 다음과 같다.

> **일반집계함수 :** COUNT() + GROUP BY-> 그룹별 1개의 행 출력 (그룹 개수만큼 출력, 자르기 + 집약)  
> **윈도우집계함수 :** COUNT() OVER (PARTITION BY) -> ID개수만큼 행 출력 (행의 개수가 줄어들지 않는다, 자르기)

다음의 공식문서 예제를 보며 윈도우 함수가 어떻게 작동하는지 알아보자. 임직원의 월급, 부서, 직원번호가 포함된 empsalary 테이블이 있다.

```
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;

  depname  | empno | salary |          avg
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(10 rows)
```

첫 3개의 컬럼은 테이블의 데이터를 바로 사용하는 것이고, row 당 1개의 값을 가지고 있다. 4번째 컬럼은 같은 부서명의 ROW 끼리의 평균 월급을 나타낸다. (비윈도우 함수의 avg 함수와 동일하지만, over 구문을 사용할 경우 윈도우 함수로 취급받고, window frame 상에서 연산될 수 있게 해 준다.)

윈도우 함수는 함수명, 혹은 변수 뒤에 항상 over를 바로 뒤에 붙여 사용한다. over 구문은 쿼리의 row들이 윈도우 함수에 의해 정확히 어떻게 분할되어 작동하는지에 대한 결정을 내린다. over 내의 partition by 구분은 동일한 값을 공유하는 groups 혹은 partitions으로 행을 분할한다. 이렇게 분할된 파티션 상에서 각 행과 동일한 파티션에 속하는 행끼리 연산하게 된다. over 내에 order by를 통해 윈도우 함수에 통과시킬 row의 순서를 정할 수 있다.

```
SELECT depname, empno, salary,
       rank() OVER (PARTITION BY depname ORDER BY salary DESC)
FROM empsalary;

  depname  | empno | salary | rank
-----------+-------+--------+------
 develop   |     8 |   6000 |    1
 develop   |    10 |   5200 |    2
 develop   |    11 |   5200 |    2
 develop   |     9 |   4500 |    4
 develop   |     7 |   4200 |    5
 personnel |     2 |   3900 |    1
 personnel |     5 |   3500 |    2
 sales     |     1 |   5000 |    1
 sales     |     4 |   4800 |    2
 sales     |     3 |   4800 |    2
(10 rows)
```

rank 함수는 해당 파티션 당 order by 값에 맞는 숫자 형태의 순위를 나타낸다. rank는 over 절에 의해서만 결정되기에 명시적인 매개 변수가 추가로 필요하지 않다.

윈도우 함수는 from 절의 테이블에서 where, group by 그리고 having 절로 필터링된 "가상 테이블"의 행을 대상으로 작동하기에 조건에 부합하지 않아 제거된 row는 윈도우 함수 내에서 사용되지 않는다. 쿼리에 다양한 over 절을 사용하여 데이터를 분할할 수 있지만, 이 가상 테이블에 정의된 row를 대상으로 동일하게 작동한다. 행의 순서가 중요하지 않은 경우, order by를 생략해도 되는 것처럼, 단일 파티션이 전체 row를 포함하는 경우 partition by를 생략할 수도 있다.

### **1-1. Window frame**

윈도우 함수에 관한 중요한 개념 중 하나는 window frame이다. window frame이라고 불리는 row의 집합이 파티션 내에 존재한다. 몇몇 윈도우 함수는 전체 파티션이 아닌, window frame의 row에 대해서만 동작한다. 기본적으로 ORDER BY를 사용하면 frame은 시작 행부터 현재 행까지의 정보로만 구성되며, order by 가 생략되면, 기본 frame은 파티션 내의 전체 row로 이루어진다. 다음 sum의 예제를 보면

```
SELECT salary, sum(salary) OVER () FROM empsalary;

salary |  sum
--------+-------
   5200 | 47100
   5000 | 47100
   3500 | 47100
   4800 | 47100
   3900 | 47100
   4200 | 47100
   4500 | 47100
   4800 | 47100
   6000 | 47100
   5200 | 47100
(10 rows)
```

over 절에 order by가 없기에, window frame은 파티션 전체와 같고, 각 sum은 전체 테이블을 조회하여 일반 집계 함수와 동일한 결과를 가진다. 하지만 order by 가 들어갈 경우 결과가 달라진다. 아래 쿼리는 월급의 최저값 ROW부터 현재 ROW까지 (파티션의)의 합계이다.

```
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;

 salary |  sum
--------+-------
   3500 |  3500
   3900 |  7400
   4200 | 11600
   4500 | 16100
   4800 | 25700
   4800 | 25700
   5000 | 30700
   5200 | 41100
   5200 | 41100
   6000 | 47100
(10 rows)
```

### ****1-2.** 제약조건**

위도우 함수는 SELECT와 ORDER BY 절에서만 허용된다. group by, having, where 절 같은 곳에서는 사용이 불가능하다.

논리적으로 해당 조건들을 모두 조회한 후에 작동하기 때문이다.

그리고 윈도우 함수는 비윈도우집계함수 이후에 실행된다. 즉 윈도우 함수의 인수에 일반 집합 함수 호출을 포함하는 것은 가능하지만, 그 반대의 경우는 불가능하다. 만약 윈도우 함수의 연산 후에 filter 혹은 group by를 할 경우, 서브쿼리를 사용해야 한다. 아래와 같이 사용하면 내부 쿼리의 순위가 3 이하인 row 들만 보여준다.

```
SELECT depname, empno, salary, enroll_date
FROM
  (SELECT depname, empno, salary, enroll_date,
          rank() OVER (PARTITION BY depname ORDER BY salary DESC, empno) AS pos
     FROM empsalary
  ) AS ss
WHERE pos < 3;
```

### ****1-3.** WINDOW AS**

쿼리가 만약에 다수의 윈도우 함수를 포함한다면, 각각이 OVER문으로 작성하는 것이 가능하지만, 여러 함수에 대해 동일한 윈도우 설정 동작을 하는 경우 중복되고 에러가 발생하기 쉽다. 이럴 경우 WINDOW에 해당하는 레퍼런스를 설정하고 해당 값을 over에서 사용 이 가능하다.

```
SELECT sum(salary) OVER w, avg(salary) OVER w
  FROM empsalary
  WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);
```

### **1-4. 성능**

윈도우 함수를 사용할 경우 집계, 순위 등의 쿼리를 편하게 사용할 수 있고, 테이블의 스캔 횟수도 훨씬 줄어든다. 다만 파티션 내 다른 행과 현재행의 관계정보로 다루어지기에, 윈도우 함수를 사용할 시 기본적으로 정렬하는 과정에서 자원이 소모된다. 테이블 및 데이터 정보에 따라 달라지겠지만, 분포율이 5~7%정도 되는 1200만 건의 데이터를 기준으로 윈도우 함수와 group by 정렬을 비교해 보았다.

<p align="center"><img src="/img/window.png"/></p>
<p align="center"><img src="/img/window2.png"/></p>

실제로 윈도우 함수를 포함한 경우 sort 과정에 자원이 많이 소모되어 데이터가 많을 경우 오히려 비윈도우 함수보다 효율이 좋지 않았다. 따라서 기능의 편의성 외에도 데이터의 양이나 테이블 구조에 맞춰 윈도우 함수를 사용하고, 서브쿼리나 조건절 튜닝을 통해 스캔해야할 행의 갯수를 줄인 후 사용하는 방법을 고려해야 한다.

## 2\. 윈도우 함수의 종류 및 사용법

### **2-1. 일반집계함수**

-   **SUM -** 파티션별 윈도우의 합계

```
SELECT MGR, ENAME, SAL
     , SUM(SAL) OVER (PARTITION BY MGR ORDER BY SAL RANGE UNBOUNDED PRECEDING) as MGR_SUM 
  FROM EMP;
```

-   **MAX -** 파티션별 윈도우의 최댓값

```
SELECT MGR, ENAME, SAL
     , MAX(SAL) OVER (PARTITION BY MGR) as MGR_MAX 
  FROM EMP;
```

-   **MIN -** 파티션별 윈도우의 최솟값

```
 SELECT MGR, ENAME, HIREDATE, SAL
      , MIN(SAL) OVER(PARTITION BY MGR ORDER BY HIREDATE) as MGR_MIN 
   FROM EMP;
```

-   **AVG -** 파티션별 윈도우의 평균값

```
SELECT MGR, ENAME, HIREDATE, SAL
     , ROUND (AVG(SAL) OVER (PARTITION BY MGR ORDER BY HIREDATE ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)) as MGR_AVG 
  FROM EMP;
```

-   **COUNT -** 파티션별 윈도우의 카운트

```
SELECT ENAME, SAL
     , COUNT(*) OVER (ORDER BY SAL RANGE BETWEEN 50 PRECEDING AND 150 FOLLOWING) as SIM_CNT
  FROM EMP;
```

### **2-2. 그룹 내 행 순서 함수**

-   **FIRST\_VALUE -** 파티션별 윈도우에 가장 먼저 나오는 값

```
SELECT DEPTNO, ENAME, SAL
     , FIRST_VALUE(ENAME) OVER (PARTITION BY DEPTNO ORDER BY SAL DESC ROWS UNBOUNDED PRECEDING) as DEPT_RICH 
  FROM EMP;
```

-   **LAST\_VALUE -** 파티션별 윈도우에 가장 나중에 나오는 값

```
SELECT DEPTNO, ENAME, SAL
     , LAST_VALUE(ENAME) OVER ( PARTITION BY DEPTNO ORDER BY SAL DESC ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) as DEPT_POOR 
  FROM EMP;
```

-   **LAG -** 파티션별 윈도우의 이전 몇 번째 행의 값

```
SELECT ENAME, HIREDATE, SAL
     , LAG(SAL) OVER (ORDER BY HIREDATE) as PREV_SAL 
  FROM EMP 
 WHERE JOB = 'SALESMAN';
```

-   **LEAD -** 파티션별 윈도우의 이후 몇번째 행의 값

```
SELECT ENAME, HIREDATE
     , LEAD(HIREDATE, 1) OVER (ORDER BY HIREDATE) as "NEXTHIRED" 
  FROM EMP;
```

### **2-3. 그룹 내 순위함수**

-   **RANK -** 파티션 내 전체 윈도우에 대한 순위, 동일 값에 대해서는 동일한 순위, 그 다음 값은 순위는 동일한 순위 만큼 증가된 채로 부여 (ex. 1,1,1,4,5,6,7...)

```
SELECT JOB, ENAME, SAL,
       RANK( ) OVER (ORDER BY SAL DESC) ALL_RANK, 
       RANK( ) OVER (PARTITION BY JOB ORDER BY SAL DESC) JOB_RANK
  FROM EMP;
```

-   **DENSE\_RANK -** 파티션 내 전체 윈도우에 대한 순위, 동일 값에 대해서는 동일한 순위, 그 다음 값은 순위는 동일한 순위에 상관없이 다음값 부여 (ex. 1,1,1,2,3,4,5...)

```
SELECT JOB, ENAME, SAL
     , RANK( ) OVER (ORDER BY SAL DESC) RANK
     , DENSE_RANK( ) OVER (ORDER BY SAL DESC) DENSE_RANK
  FROM EMP; 
```

-   **ROW\_NUMBER -** 파티션 내 전체 윈도우에 대한 순번, 동일한 값이어도 고유한 순위 부여

```
SELECT JOB, ENAME, SAL 
     , RANK( ) OVER (ORDER BY SAL DESC) RANK
     , ROW_NUMBER() OVER (ORDER BY SAL DESC) ROW_NUMBER
  FROM EMP; 
```

### **2-4. 그룹 내 비율 함수**

-   **RATIO\_TO\_REPORT -** 파티션 내 전체 SUM에 대한 컬럼별 백분율 소수점 값

```
SELECT ENAME, SAL
     , ROUND(RATIO_TO_REPORT(SAL) OVER (), 2) as R_R 
  FROM EMP 
 WHERE JOB = 'SALESMAN'; 
```

-   **PERCENT\_RANK -** 파티션별 윈도우에서 가장 먼저 나오는 것은 0, 제일 마지막에 나오는 것은 1로 나타낸 후 값에 상관없이 행의 순서만으로의 백분율 값

```
SELECT DEPTNO, ENAME, SAL
     , PERCENT_RANK() OVER (PARTITION BY DEPTNO ORDER BY SAL DESC) as P_R 
  FROM EMP; 
```

-   **CUME\_DIST -** 파티션별 윈도우의 전체 건수에서 현재 행보다 작거나 같은 건에 대한 누적 백분률 값

```
SELECT DEPTNO, ENAME, SAL
     , CUME_DIST() OVER (PARTITION BY DEPTNO ORDER BY SAL DESC) as CUME_DIST 
  FROM EMP; 
```

-   **NTITLE -** 파티션별 전체 건수를 Argument로 N등분한 값

```
SELECT ENAME, SAL
     , NTILE(4) OVER (ORDER BY SAL DESC) as QUAR_TILE 
  FROM EMP ;
```

**참고**
윈도우 함수별 기능 및 예제 - [http://www.gurubee.net/lecture/2382](http://www.gurubee.net/lecture/2382)

윈도우 함수 공식 문서 - [https://www.postgresql.org/docs/current/tutorial-window.html](https://www.postgresql.org/docs/current/tutorial-window.html)