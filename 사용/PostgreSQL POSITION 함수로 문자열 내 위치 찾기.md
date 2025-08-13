
PostgreSQL에서 문자열 처리 작업을 하다 보면, **특정 단어가 문자열 안에 포함되어 있는지** 또는 **어느 위치에 있는지**를 알아야 할 때가 있습니다. 이럴 때 유용하게 쓸 수 있는 함수가 바로 `POSITION()` 입니다.

## 1. POSITION 함수란?

`POSITION()` 함수는 **문자열 안에서 부분 문자열(substring)의 첫 번째 위치를 반환**합니다.  
MySQL의 `LOCATE()`, Oracle의 `INSTR()`와 비슷한 역할을 합니다.

```sql
`POSITION(substring IN string)`
```

- **리턴값**: 부분 문자열이 시작하는 위치 (1부터 시작)
- **찾지 못하면**: 0 반환
## 2. 기본 예제

```sql
SELECT POSITION('abc' IN '123abc456'); 
-- 4 SELECT POSITION('zzz' IN '123abc456'); -- 0
```
`'abc'`는 네 번째 문자에서 시작하므로 `4`를 반환합니다.  
`'zzz'`는 존재하지 않기 때문에 `0`이 나옵니다.

## 3. WHERE 절에서 포함 여부 확인하기

`POSITION()`은 단순히 위치값을 확인하는 것뿐 아니라, `> 0` 조건을 주면 **문자열 포함 여부**를 체크하는 조건으로도 활용할 수 있습니다.

```sql
SELECT * FROM articles WHERE POSITION('PostgreSQL' IN title) > 0;
```
→ `title` 컬럼에 `'PostgreSQL'`이 포함된 모든 행 조회

## 4. 실무 예시: 키워드 매칭

예를 들어, **키워드 테이블**(`keywords`)과 **본문 테이블**(`contents`)이 있을 때, 각 키워드가 본문 안에 포함된 데이터를 찾고 싶다면 다음과 같이 작성할 수 있습니다.

```sql
SELECT c.id, k.keyword FROM contents c JOIN keywords k   ON POSITION(k.keyword IN c.body) > 0;
```

→ 본문에 키워드가 포함된 경우만 매칭되므로 검색·태깅·추천 시스템 등에 유용

## 5. LIKE와 비교

|구분|특징|
|---|---|
|`LIKE`|와일드카드(`%`) 기반 패턴 매칭, 인덱스 활용 제한|
|`POSITION`|단순 포함·위치 검색, 숫자 비교 가능|
|정규식(`~`)|복잡한 패턴 검색 가능, 상대적으로 느림|

`POSITION`은 **정확한 부분 문자열 검색**에 특화되어 있고, `LIKE`보다 **위치 기반 조건**을 쉽게 줄 수 있습니다.

## 6. 위치 기반 필터링

특정 단어가 **본문의 초반부에만 등장하는 경우**를 찾는 것도 가능합니다.

```sql
-- 본문 첫 100자 안에 등장하는 키워드만 검색 
WHERE POSITION('PostgreSQL' IN body) BETWEEN 1 AND 100
```
→ 뉴스 기사, 블로그 글 요약, 검색 스니펫 생성 시 유용

## 7. 성능 고려 사항

`POSITION()`은 인덱스를 직접 활용하지 않기 때문에, 대량 데이터에서 검색할 경우 **Full Table Scan**이 발생할 수 있습니다.  
성능 최적화를 위해 `pg_trgm` 확장을 이용한 GIN 인덱스를 고려할 수 있습니다.

```sql

-- pg_trgm 확장 설치 
CREATE EXTENSION IF NOT EXISTS pg_trgm; 
-- GIN 인덱스 생성 
CREATE INDEX idx_body_trgm ON contents USING gin (body gin_trgm_ops);  
-- trgm 기반 검색 (LIKE, POSITION 대체 가능) 
SELECT * FROM contents WHERE body ILIKE '%PostgreSQL%';`
```
## 8. 정리

- **`POSITION(sub IN str)`** → 문자열 내 부분 문자열 위치 반환 (없으면 0)
    
- 단순 포함 여부: `POSITION(...) > 0`
    
- 위치 기반 검색 가능: `BETWEEN`, `<`, `>`
    
- 대량 데이터 검색 시 `pg_trgm` + GIN 인덱스로 성능 향상
    
- 문자열 처리, 키워드 매칭, 텍스트 분석 등 다양한 곳에서 활용 가능