ㅊ
## 1\. 인덱스 컨트롤

### 1-1. 인덱스 조회 

```
SELECT * FROM pg_indexes WHERE tablename = '{테이블명}'; -- 테이블명에 '' 필요
```
### 1-2. 인덱스 생성 

```
-- 단일 인덱스
CREATE INDEX {인덱스명} ON {테이블명} USING btree({컬럼명});

-- 결합 인덱스
CREATE INDEX {인덱스명} ON {테이블명} USING btree({컬럼명1}, {컬럼명2});

-- 유니크 인덱스
CREATE UNIQUE INDEX {인덱스명} ON table_name ({컬럼명});
```
### 1-3. 인덱스 삭제

```
DROP INDEX {인덱스명};
```
### 1-4. 인덱스 사용 빈도 확인

```
SELECT schemaname, relname, indexrelname, idx_scan as idx_scan_cnt FROM pg_stat_user_indexes ORDER BY idx_scan;
```

### 1-5. 인덱스 손상 시 재인덱싱

```
REINDEX INDEX {인덱스명}

REINDEX TABLE {테이블명}

REINDEX DATABASE {데이터베이스명}
```

## 2\. 인덱스 란?

데이터에 색인을 통해 데이터의 위치를 빠르게 찾아주는 역할을 한다. 인덱스 설정 없이는 Seq 스캔을 통해 테이블 전체를 조회하기에 검색 성능이 저하된다.

### 2-1. 테이블 스캔 방식 

Postgresql은 seq scan, index scan, bitmap index scan, index only scan, tid scan 5가지 스캔 방식을 사용한다. 그중 2가지를 확인해 보면,

#### ▪  2-1-1. Seq Scan 방식

> \- Seq Scan은 테이블을 Full Scan 하여 전체 데이터를 조회한다.  
> \- 인덱스가 존재하지 않거나, 인덱스가 존재하더라도 읽어야 할 범위가 넓은 경우에 선택된다.

#### ▪  2-1-2. Index Scan 방식

> \- Index Scan은 인덱스 Leaf 블록에 저장된 키를 이용해서 테이블 레코드를 액세스 하는 방식이다.  
> \- 레코드 정렬 상태에 따라서 테이블 블록 액세스 횟수가 크게 차이 난다.

다음과 같이, 인덱스를 사용할 경우 테이블 레코드에 효과적인 접근이 가능하다. 하지만 select 성능은 올라가지만, update, insert, delete시 index 색인정보 갱신을 하기에 시간이 더 소모된다.

### 2-2. 인덱스가 적용되지 않는 경우

▪  order by {인덱스칼럼 1}, {칼럼 2} : 복수 키에 대해 order by 하는 경우  (order by col1, col2 자체를 인덱스 설정하면 적용가능)  
▪   where {칼럼 1} = ‘x’ order by {인덱스칼럼 2} : 조건절에 의하여 연속적이지 않게 된 컬럼에 대한 order by  
▪   order by {인덱스컬럼1} desc, {인덱스컬럼2} asc : desc와 asc의 결합사용  
▪   group by {인덱스컬럼1} order by {인덱스컬럼2} : group by, order by 컬럼이 서로 다른 경우  
▪   order by abs({인덱스컬럼1}) : 형 변환 이후의 order by, group by 인 경우

### 2-3. 인덱스 설계 방법

▪   명확한 이유를 가진 인덱스만 설계 (많을수록 좋은 게 아니다.)  
▪   조회 쿼리들을 전체 확인 후 자주 사용하는 컬럼  
▪   고유 값 위주의 설계  
▪   Cardinality가 높을수록 효율적 (=컬럼별 중복도가 낮을수록 좋다)  
▪   인덱스 key의 크기는 되도록 작게 설계  
▪   PK, join의 대상이 되는 컬럼에 설계  
▪  단일 인덱스 여러 개 < 복합인덱스 고려  
▪   Update, delete가 빈번하지 않은 컬럼  
▪   join의 대상이 자주 되는 컬럼  
▪   인덱스를 생성할 때 가장 효율적인 자료형은 정수형 자료 (가변적 데이터는 비효율적)  
▪   선택도가 낮을수록 효율적 (10~15%)  
\* 선택도는 데이터에서 특정 값을 얼마나 잘 선택할 수 있는지에 대한 지표  
(특정 필드값을 지정했을 때 선택되는 레코드 수를 테이블 전체 레코드 수로 나눈 수치)  
\= 컬럼의 특정 값의 row 수 / 테이블의 총 row 수 \* 100= 컬럼의 값들의 평균 row 수 / 테이블의 총 row 수 \* 100

### 2-4. 다중 컬럼 인덱스

다중 컬럼 인덱스는 두개 이상의 필드를 조합해서 생성한 인덱스이다. 다중 컬럼 인덱스는 단일 컬럼 인덱스 보다 더 비효율적으로 INDEX/UPDATE/DELETE를 수행하기 때문에 생성에 신중해야 한다. (가급적 UPDATE가 안되는 값을 선정해야 한다). 조건절에 여러개의 조건이 있을시, 선행되는 조건과 이를 만족하는 후행되는 조건들을 차례로 함께 INDEX해서 사용한다. 

#### 2-4-1. 다중 컬럼 인덱스 설계 방법

▪  다중 컬럼 인덱스 구성시 컬럼의 순서는, IO를 적게 발생시키는 순으로 구성하여야 한다.

(선행 인덱스에서 더 많은 데이터가 필터링될수록 이후 인덱스의 I/O가 덜 소모된다.)

▪  인덱스 컬럼을 합쳐 색인하기에 선행 인덱스 컬럼이 조건에 있어야 한다.

예를 들어

```
CREATE INDEX idx_user_sample ON user USING btree(first_name, address );
```

user 테이블에 first\_name, address 두컬럼을 대상으로 다중 컬럼 인덱스를 부여한 후,

```
SELECT * from 
WHERE first_name = ‘Jun’
AND address = ‘Seoul’
```

으로 테이블을 조회할 경우 ‘JunSeoul’에 대한 인덱스를 찾고,

B-Tree 인덱스의 left to right 특성대로, address 만으로는 인덱스를 사용할 수 없다. (선행되는 조건절인 first\_name에 대한 조건에 부합하는 데이터를 기준으로 인덱싱이 되어있다.)

또한 

```
CREATE INDEX idx_user_sample2 ON user USING btree(first_name, last_name, address );
```

first name, last name, address로 다중 컬럼 인덱스를 설정할 경우 다음과 같이 인덱스가 사용된다.

```
-- 인덱스 사용
SELECT * from 
WHERE first_name = ‘Jun’
AND last_name = ‘H’
AND address = ‘Seoul’

-- 인덱스 미사용
SELECT * from 
WHERE first_name = ‘Jun’
AND address = ‘Seoul’
```

결론적으로  다중 컬럼 인덱스의 성능은 절대적인 것이 아닌, 어떤 데이터를 조회하는지, 쿼리를 어떻게 작성하는지에 따라 크게 달라질 수 있기에 확실한 쿼리 플랜 분석 및 설계가 필요하다.

-   [\[PostgreSQL\] B-tree 인덱스의 원리 및 특징](https://junhkang.tistory.com/6)
-   [\[PostgreSQL\] Hash 인덱스의 원리 및 특징](https://junhkang.tistory.com/7)
-   [\[PostgreSQL\] GiST인덱스의 원리 및 특징](https://junhkang.tistory.com/8) 
-   [\[PostgreSQL\] SP-GiST인덱스의 원리 및 특징](https://junhkang.tistory.com/9) 
-   [\[PostgreSQL\] GIN인덱스의 원리 및 특징](https://junhkang.tistory.com/10)
-   [\[PostgreSQL\] BRIN 인덱스의 원리 및 특징](https://junhkang.tistory.com/11)