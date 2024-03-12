## 1\. 문제상황

1.  긴 텍스트에서 단순 like 조합 외 방법으로 유사 문자열 검색  
    (Ex. Susan loves hiking 을 “love hike” 이라는 키워드로 검색하고자 함)
2.  RDBMS에서 수천만 건의 데이터 처리 시 긴 문자열 검색 속도 향상

## 2\. Full Text Search(전문검색)란?

게시물의 내용/제목 등 문장, 문서 전체에서 키워드를 검색하는 기능이다. 단순한 like, 비교연산자와 달리 각 단어의 Token화 및 정규화를 통해 긴 문장내에서의 유사 검색을 가능하게 한다. Postgresql 기본 인덱스인 b-tree인덱스로는 Like 와 같은 패턴 매칭 검색시 양쪽에 %%를 거는 경우는 인덱스를 타지 않지만, gin 인덱스를 사용하여 빠른 검색이 가능하다.

#### 2-1. Gin Index

GIN stands for "Generalized Inverted index". "Inverted" refers to the way that the index structure is set up, building a table-encompassing tree of all column values, where a single row can be represented in many places within the tree. By comparison, a B-tree index generally has one location where an index entry points to a specific row.

인덱스를 적용하는 column의 값을 일정한 규칙에 따라 split 후 사용한다. 데이터 내의 각 token들을 indexing하여 등장 위치와 상관없이 해당 찾고자 하는 값이 데이터의 중간에 등장하더라도 인덱스가 가능하다.

<p align="center"><img src="/img/ginindex.png"/></p>

##### 이미지 출처: [https://pganalyze.com/blog/gin-index](https://pganalyze.com/blog/gin-index)

## 3\. 원리

document를 형태소 단위로 해체하여 각 토큰들을 숫자, 단어, 이메일 등으로 분류한다. 분류된 토큰들을 정규화(대문자를 소문자로 바꾸거나, 복수형들을 단수형으로 바꾸는 등) 를 통해 lexem 형태로 분류한다. 이 과정을 통해 전처리 된 document를 lexem 배열 형태로 저장 후 검색에 사용한다. Postgresql 의 경우 to\_tsvector 함수를 통해 document 를 tsvector 타입으로 변환이 가능하다. 변환 후에는 tsquery를 사용하여 매칭 여부를 확인가능하다

```
SELECT 'a fat cat sat on a mat and ate a fat rat'::tsvector @@ 'cat & rat'::tsquery;
```

#### 3-1. 참고 (한국어 처리)

각 lexem이 정규화 되어있기에 문장 내에 과거형, 복수형 등 유사언어에 대한 검색이 가능하지만 기본적으로 영단어만 제공되며, 한글의 경우 hunspell과 같은 한글 언어팩을 별도로 설치해야 사용 가능하다.

-   RDS 현재 설치된 extension 확인
    
    ```
    SHOW rds.extensions
    ```
    
    해당 언어팩을 사용하려면 plpythonu3u 와 같은 extension을 설치해야하나, AWS RDS의 경우 보안상의 이유로 특정 extension들의 별도 설치를 제한하고 있어 한글팩 설치가 불가하다. 다만 한글 버전의 유사도 검색이 꼭 필요하다면 AWS Lamda 함수를 통해 우회하여 한글 단어팩을 사용하는 방법은 가능하다.

## 4.데이터 타입

#### 4-1. Tsvector

Document를 token으로 분리한 후 정형화(복수형, 과거형,대소문자 등의 데이터를 정리 후 ) 중복제거된 lexemes list 유형

```
SELECT 'a:1A fat:2B,3C cat:4D'::tsvector;
          **tsvector
----------------------------**
'a':1A 'cat':4 'fat':2B,3C
```

#### 4-2. Tsquery

lexemes 내에 조건에 맞는 결과가 있는지 여부를 나타내는 유형

```
SELECT 'fat & rat & ! cat'::tsquery;
        **tsquery
------------------------**
 'fat' & 'rat' & !'cat'
```

#### 4-3. 테이블에 적용

##### \- Tsvector 컬럼 추가

```
ALTER TABLE
    table
ADD COLUMN
    tsvec_words tsvector
```

##### \- Tsvector 데이터 추가

```
UPDATE
    table
SET
    tsvec_words = to_tsvector(column);
```

##### \- Tsvector 인덱스 추가

```
CREATE INDEX
    example_idx
ON
    table
USING
    gin(tsvec_words);
```

## 5\. 연산자

문법의 경우 [공식문서](https://www.postgresql.org/docs/current/textsearch-controls.html)를 참고하는 것이 가장 정확하지만, 직접 적용해본 후 가장 기본적이고 유용한 문법을 정리해보자면

```
1. & : and
2. | : or
3. ! : not 
4. :* : like (sql의 %%와 동일)
5. <-> : 문자 사이의 간격 확인
```

예를들어 cat과 dog의 lexem을 모두 포함하고 있는 document를 검색하고 싶다면,

```
SELECT
  *
FROM
  Table
WHERE
  tsvec_words @@ to_tsquery(‘cat:*&dog:*’)
```

## 6\. 구문 분석 쿼리

PostgreSQL은 쿼리를 tsquery 데이터 형식으로 변환하기 위해 to\_tsquery , plainto\_tsquery , phraseto\_tsquery 및 websearch\_to\_tsquery 함수를 제공한다.

#### 6-1. plainto\_tsquery

plainto\_tsquery는 and, a, or 등의 단어를 제외한 형식화되지 않은 텍스트만을 tsquery로 변환 후 단어 사이에 & 연산자를 포함하여 리턴한다. (연산자는 인식하지 않음)

```
SELECT plainto_tsquery('english', 'the Fat Rats eat ttt a and young bean');
'fat' & 'rat' & 'eat' & 'ttt' & 'young' & 'bean'
```

#### 6-2. phraseto\_tsquery

phraseto\_tsquery는 plainto\_tsquery와 유사하지만, & 대신 <->(followed by) 연산자로 구분된다. 사이에 삭제되는단어가 있다면 으로 표현되기에 lexeme의 시퀀스를 확인할때 유용하다.  
(연산자는 인식하지 않음)

```
 SELECT phraseto_tsquery('english', 'the Fat Rats eat ttt a and young a bean');
'fat' <-> 'rat' <-> 'eat' <-> 'ttt' <-> 'young' <2> 'bean'
```

#### 6-3. websearch\_to\_tsquery

websearch\_to\_tsquery는 plainto\_tsquery와 동일 (연산자 인식)

```
SELECT websearch_to_tsquery('""" )( dummy \\ query <->');
 websearch_to_tsquery
----------------------
'dummy' <-> 'query'
```

## 7\. 검색결과 순위

보통 유사 검색을 실행할 시 가장 유사도가 높은( 검색결과에 가장 부합하는) 결과를 최우선적으로 노출시켜야한다.  
Full Text Search를 통해 조회할 경우 우선 순위를 검색 결과에 가장 부합한다는 지표를 어떻게 확인할 수 있을까?

ts\_rank\_cd 함수는 쿼리와 매칭 결과 간의 유사도를 점수화하여 보여준다.

```
SELECT 
  title, 
  ts_rank_cd(textsearch, query) AS rank
FROM apod, to_tsquery(‘test|white&’blue’) query
WHERE query @@ textsearch
ORDER BY rank DESC
```

해당 쿼리를 사용하면 ‘test|white&’blue’ 조건에 부합하면서, 관련도를 추출할수 있어  
관련성 높은 document를 최우선적으로 리스트업 할 수 있다.

## 마무리

-   full text search를 사용시 like문의 결합으로 텍스트를 검색할 시보다 효율 높음  
    a. full Text Search(gin index) 사용시  
    <p align="center"><img src="/img/ginindex2.png"/></p>
    b. 기본 like절 사용시  
    <p align="center"><img src="/img/ginindex3.png"/></p>
    150만건 데이터 기준 실행시간 9초 -> 1초 향상.  
    데이터 수량이 더 많을 경우 향상될 것으로 보임
-   정교한 유사단어 검색 가능  
    (시제, 단복수, 줄임말 등이 고려된 유사어에 대한 검색이 가능)
-   "키워드%", 짧은 단어 내 검색, 숫자형태 검색 등 b-tree 인덱스의 효율을 살릴 수 있는 검색의 경우 충분한 검토 후 도입이 필요