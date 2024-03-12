<p align="center"><img src="/img/brin.jpeg"/></p>

## **1\. BRIN 인덱스란?**

> ▪ Block range index의 약자  
> ▪ Page 검색에 도움 되는 메타 데이터를 뽑아서 인덱스를 구성 (ex, 특정컬럼의 최대/최솟값)  
> ▪ 특정 컬럼이 물리 주소의 일정한 상관관계를 가지는 매우 큰 테이블을 다루기 위해 설계 (타임시쿼스한 대용량 데이터 조회에 유용)

Block range는 테이블 내에서 근접한 물리주소를 가진 page 그룹을 의미한다. 각 Block range 에 대해 일부 요약 정보가 인덱스로 저장된다. 예를 들어 상점의 판매 주문을 저장하는 테이블에는 각 주문이 배치된 날짜 열이 있을 수 있으며 대부분의 경우 이전 주문시점에 맞게 순차적으로 주문정보가 들어갈 것이고, ZIP 코드 열을 저장하는 테이블에는 도시에 대한 모든 코드가 자연스럽게 그룹화되어 있을 것이다.

BRIN 인덱스는 정기적인 비트맵 인덱스 검색을 통해 쿼리를 결과를 확인하고, 인덱스에 의해 저장된 요약 정보가 쿼리 조건과 일치하면, 범위 내 모든 페이지의 모든 튜플을 반환한다. 쿼리 실행기는 반환된 튜플을 다시 검사하고, 쿼리 조건과 일치하지 않는 튜플을 폐기한다. (결과가 일치하지 않아 폐기된 인덱스는 손실된다.) BRIN 인덱스는 매우 작기 때문에 인덱스를 스캔하면 순차적 스캔에 비해 오버헤드가 거의 발생하지 않지만, 일치하는 튜플이 없는 것으로 알려진 테이블의 많은 부분을 스캔하는 것은 피할 수 있다.

BRIN 인덱스가 저장할 특정 데이터는 인덱스의 각 열에 대해 선택된 연산자 유형에 따라서도 달라진다. 예를 들어 선형 정렬 순서를 갖는 데이터 유형은 각 블록 범위 내에서 최솟값과 최댓값을 저장할 수 있고, 기하학적 유형은 블록 범위의 모든 객체에 대한 경계 정보를 저장할 수도 있다.

## **2\. BRIN 인덱스 관리**

Brin 인덱스가 생성될시, 모든 존재하는 heap page를 스캔하고, 각 block range마다 요약 인덱스 tuple을 생성하고 마지막으로 불완전한 block range를 생성한다. 새로운 page가 데이터로 가득 차면, 이미 요약된 block range가 새 튜플의 데이터로 요약 정보가 업데이트된다. 마지막 요약 범위에 속하지 않는 새 페이지가 생성되면 요약 튜플을 자동으로 획득하지 않고 나중에 요약 실행이 호출될 때까지 해당 튜플은 요약되지 않은 상태로 남아 해당 범위에 대한 초기 요약을 만든다. 이 과정을 직접 실행하는 몇 가지 방법이 있다. 테이블을 auto vacuum 하여 요약되지 않은 page ranges를 요약한다. 만약 auto summarize 파라미터가 on이라면(default 아님), autovacuum이 데이터베이스에 실행될 때마다 summarization이 실행된다.

```
--요약 안된 전체 범위 요약
brin_summarize_new_values(regclass) 

--주어진 page만 요약 (요약안됐을 경우에만)
brin_summarize_range(regclass, bigint)
```

을 통해 ranges에 summarization 실행 가능하다. 반대로  다음을 통해 요약을 해제 하는것도 가능하다. 

```
 brin_desummarize_range(regclass, bigint)
```

tuple의 기존값이 변경되어 인덱스 tuple이 더 이상 좋은 결과를 나타내지 못할 때 유용하다.

## **3\. BRIN VS B-TREE**

> ▪ BRIN 인덱스는 B-TREE 인덱스보다 쿼리 성능이 좋다.  
> ▪ BRIN 인덱스는, B-TREE에서 사용하는 용량의 1%만 사용한다.  
> ▪ BRIN이 특정 블록 범위만 다루다 보니, 검색 범위를 이탈할 경우 해당하는 블록 범위 전체를 검사한다.  
> ▪ BRIN은 lossy index이므로, 데이터의 hash 값을 저장하는 컬럼에 BRIN을 써도 데이터가 포함된 블록을 정확히 반환하지 못한다.  
> ▪ 인덱스 생성 속도가 BRIN이 더 빠르다.  

## **4\. 연산자** 

| 이름 | 인덱싱 된 데이터 유형 | 인덱싱 가능한 연산자 |
| --- | --- | --- |
| abstime\_minmax\_ops | abstime | < <= \= \>= \> |
| int8\_minmax\_ops | bigint | < <= \= \>= \> |
| bit\_minmax\_ops | bit | < <= \= \>= \> |
| varbit\_minmax\_ops | bit varying | < <= \= \>= \> |
| box\_inclusion\_ops | box | << &< && &> \>> ~= @> <@ &<\| <<\| \|>> \|&> |
| bytea\_minmax\_ops | bytea | < <= \= \>= \> |
| bpchar\_minmax\_ops | character | < <= \= \>= \> |
| char\_minmax\_ops | "char" | < <= \= \>= \> |
| date\_minmax\_ops | date | < <= \= \>= \> |
| float8\_minmax\_ops | double precision | < <= \= \>= \> |
| inet\_minmax\_ops | inet | < <= \= \>= \> |
| network\_inclusion\_ops | inet | && \>>= <<= \= \>> << |
| int4\_minmax\_ops | integer | < <= \= \>= \> |
| interval\_minmax\_ops | interval | < <= \= \>= \> |
| macaddr\_minmax\_ops | macaddr | < <= \= \>= \> |
| name\_minmax\_ops | name | < <= \= \>= \> |
| numeric\_minmax\_ops | numeric | < <= \= \>= \> |
| pg\_lsn\_minmax\_ops | pg\_lsn | < <= \= \>= \> |
| oid\_minmax\_ops | oid | < <= \= \>= \> |
| range\_inclusion\_ops | any range type | << &< && &> \>> @> <@ \-\|- \= < <= \= \> \>= |
| float4\_minmax\_ops | real | < <= \= \>= \> |
| reltime\_minmax\_ops | reltime | < <= \= \>= \> |
| int2\_minmax\_ops | smallint | < <= \= \>= \> |
| text\_minmax\_ops | text | < <= \= \>= \> |
| tid\_minmax\_ops | tid | < <= \= \>= \> |
| timestamp\_minmax\_ops | timestamp without time zone | < <= \= \>= \> |
| timestamptz\_minmax\_ops | timestamp with time zone | < <= \= \>= \> |
| time\_minmax\_ops | time without time zone | < <= \= \>= \> |
| timetz\_minmax\_ops | time with time zone | < <= \= \>= \> |
| uuid\_minmax\_ops | uuid | < <= \= \>= \> |

참고 :

[https://bajratech.github.io/2016/09/16/Postgres-BRIN-Index/](https://bajratech.github.io/2016/09/16/Postgres-BRIN-Index/)

[https://www.postgresql.kr/docs/13/brin-intro.html](https://www.postgresql.kr/docs/13/brin-intro.html)