## 1\. Foreign Key 외래키란?

Foreign key constraint 외래키 제약은 특정 칼럼 혹은 칼럼들의 값이 다른 테이블의 특정 row와 매칭되어야 하는 제약조건이다. 이를 두 관련 테이블 사이의 참조 무결성 (referential integrity)를 유지한다고 말한다. 그렇게 복잡한 개념은 아니니 바로 사용법을 확인해 보도록 하자

## 2\. 예제

### 2-1. 기본 외래키(Foreign Keys) 생성

products 테이블은 물품의 이름, 가격 정보 테이블이고, orders 테이블은 존재하는 물품 각각에 대한 순서 정보가 들어있는 테이블이다. orders, products 테이블의 product\_no에 외래키 제약을 적용하는 예제이다.

```
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products (product_no),
    quantity integer
);
```

orders 테이블의 제약조건을 위와 같이 주었을 때 products 테이블에 없는 product\_no로는 데이터 생성이 불가능하다. 이 경우 다음과 같이 명칭 한다.

-   **orders -** referencing(참조하는) 테이블
-   **products -** referenced(참조된) 테이블

### 2-2. 칼럼을 지정하지 않은 외래키(Foreign Keys)

```
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products,
    quantity integer
);
```

특정 칼럼을 지정하지 않는다면 reference 칼럼으로 Primary Key에 해당하는 칼럼을 자동으로 사용하기에 별도로 칼럼명을 명시하지 않아도 된다. (PK가 바뀔 일은 거의 없겠지만) 테이블 구조가 바뀔 수 있고, 명확한 제약조건을 명시하는 것이 좋기에 칼럼을 지정하는 것이 좋다. 

### 2-3. 복합 칼럼 외래키(Foreign Keys)

FK 제약조건을 여러 개의 칼럼을 대상으로도 사용할 수 있다.

```
CREATE TABLE t1 (
  a integer PRIMARY KEY,
  b integer,
  c integer,
  FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
);
```

물론 참조하는 칼럼과 참조되는 테이블의 칼럼 수는 일치하여야 한다.

### 2-4. 자기 참조 외래키(Self-referential Foreign Keys)

종종 다른 테이블이 아니라 같은 테이블 내의 칼럼의 FK로 두는 것이 효율적일 때가 있다. 이를 자기 참조 외래키 (self-referential foreign key)라고 한다. 예를 들어 트리 구조의 노드들을 테이블 row로 표현하고 싶을 때, 다음과 같이 parent\_id를 node\_id에 참조시키면 된다. 

```
CREATE TABLE tree (
    node_id integer PRIMARY KEY,
    parent_id integer REFERENCES tree,
    name text,
    ...
);
```

최상위 노드의 parent\_id는 null이 될 것이고, parent\_id가 null이 아닌 항목들은 해당 테이블의 유효한 노드를 참조하도록 제한된다.

### 2-5. 다중 외래키(Foreign Keys)

한 개의 테이블은 여러 개의 FK를 가질 수 있으며 이는 다대다 (many-to-many) 테이블 관계 구현에 사용된다. 기존 예제의 상품, 상품순서 구조에 추가로, 한 순서에 많은 상품을 포함할 수 있게 한다면 다음과 같은 테이블 구조를 사용할 수 있을 것이다. 

```
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text,
    ...
);

CREATE TABLE order_items (
    product_no integer REFERENCES products,
    order_id integer REFERENCES orders,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);​
```

## 3\. 외래키(Foreign Keys) 옵션

앞서 말한 대로, FK 제약조건이 걸려있다면 참조되지 않은 값으로는 데이터 생성이 불가능하다. 그러나 참조된 orders가 생성된 후에 product가 삭제되면 어떻게 될까? Postgresql에서는 다음 상황들 중에 선택적으로 사용이 가능하다.

-   참조하는 데이터(orders)가 있을 경우 삭제 불가
-   참조하는 데이터(orders)까지 함께 삭제
-   등등...

데이터를 처리하는 상황을 선택하는 ON DELETE, ON UPDATE 등의 옵션과, 처리하는 방식을 선택하는 RESTRICT, CASCADE 등의 옵션을 조합하여 원하는 결과를 만들면 된다. 

다음 예제는 서로 다른 테이블에 참조된 칼럼 각각의 값이 삭제됐을 때 참조된 상위 값이 있을 때 각각 같이 삭제할지, 삭제를 방지할지를 설정한 테이블 생성 쿼리이다.

```
CREATE TABLE order_items (
    product_no integer REFERENCES products ON DELETE RESTRICT,
    order_id integer REFERENCES orders ON DELETE CASCADE,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```

다음과 같이 설정하면 order\_items에서 product\_no를 삭제하려 할 때 참조하는 데이터가 있을 경우 삭제가 불가능하고, order\_id의 경우 참조하는 데이터와 함께 삭제가 된다. (RESTRICT, CASCADE는 가장 기본적으로 사용되는 옵션)

### 3-1. RESTRICT

Restrict는 참조하는 열의 삭제를 방지한다. 참조하는 오브젝트가 존재할 시 실행 자체를 실패한다.

### 3-2. NO ACTION

NO ACTION은 제약조건을 선택할 때 참조 행이 존재한다면 오류가 발생하고, 지정하지 않을 시에는 기본 동작(RESTRICT / CASCADE)이 된다. NO ACTION을 선택하나, NO ACTION을 선택하지 않고 RESTRICT를 설정하나 실행이 되지 않는 건 똑같지만, 본질적으로 NO ACTION은 무결성 체크하는 시점을 트랜잭션의 후반부까지 연기할 수 있다는 차이점이 있다. (DEFERRABLE, NOT DEFERRABLE 옵션으로 제어 가능하다.)

### 3-3. CASCADE

폭포 혹은 단계적인 이라는 의미로, 연관된 데이터의 일괄적인 적용을 의미한다.  CASCADE는 참조하는 열에도 함께 변경을 가한다. 

두 가지 옵션을 보면

-   SET NULL
-   SET DEFAULT

이 옵션은 참조하는 테이블의 칼럼값(order\_id)을 null로 변경할지, default 값으로 변경할지 선택하는 옵션이다.

SET NULL, SET DEFAULT는 FK가 추가적인 정보를 나타낼 때 적절하다. 예를 들어, 위 예제에서 products 테이블에서 product manager의 정보가 참조되고 있고, product manager가 삭제될 때 product에 해당 참조 데이터를 null 혹은 default로 자동으로 바꿔준다면 별도의 추가 명령 없이 관리할 수 있을 것이다.

하지만 SET NULL, SET DEFAULT가 참조 테이블의 다른 제약조건들까지 먼저 확인하지는 않기에, 만약 SET DEFAULT로 설정했는데 DEFAULT 값이 다른 제약조건에 부합하지 않을 경우 해당 동작은 실패한다. 

```
CREATE TABLE tenants (
    tenant_id integer PRIMARY KEY
);

CREATE TABLE users (
    tenant_id integer REFERENCES tenants ON DELETE CASCADE,
    user_id integer NOT NULL,
    PRIMARY KEY (tenant_id, user_id)
);

CREATE TABLE posts (
    tenant_id integer REFERENCES tenants ON DELETE CASCADE,
    post_id integer NOT NULL,
    author_id integer,
    PRIMARY KEY (tenant_id, post_id),
    FOREIGN KEY (tenant_id, author_id) REFERENCES users ON DELETE SET NULL (author_id)
);
```

위 예제 테이블을 보면 posts의 데이터를 삭제할 때,  FK 제약조건으로 참조된 tenant\_id를 null로 변경하려 하지만, tenant\_id는 PK의 일부로 null이 될 수 없기에 실행되지 않는다.

### 3-4. ON DELETE

ON DELETE 옵션은 테이블에 연관된 오브젝트의 유형에 따라 적절하게 사용되어야 한다. 참조하는 테이블이 참조된 테이블 값의 구성요소이며 독립적으로 존재할 수 없다면 CASCADE 옵션을 적용하여 한번에 처리하는 것이 적절하고, 두 테이블의 오브젝트가 독립적인 관계라면 RESTRICT, NO ACTION이 적합하다. 

예를 들어, 위의 예제에서 order\_items는 orders의 일부분이고 독립적으로 사용될 일이 없기에 orders가 삭제될 때 자동으로 지워지는 것이 편할 것이다. orders와 products는 독립적으로 사용할 여지가 있기에 삭제 시 products를 자동으로 지우는 건 문제가 될 수 있다.

### 3-5. ON UPDATE

ON DELETE와 유사하게 참조열이 업데이트될 때 호출되는 ON UPDATE도 있다. SET NULL, SET DEFAULT의 설정이 다른 제약조건에 위배된다면 적용할 수 없다는 점은 동일하다. CASCADE와 사용 시 참조하는 칼럼의 업데이트된 값이 참조된 열로 복사된다.

## 4\. 인덱스

FK 제약조건은 PK이거나, Unique 제약조건(UK)이거나, "복합 인덱스의 일부가 아닌" 칼럼을 참조해야만 한다. 이 뜻은 참조된 칼럼은 인덱스를 항상 가지고 있다는 뜻이다. 하지만 참조하는 칼럼은 인덱스가 필수이거나 인덱스를 자동으로 생성하지 않는다. 열의 DELETE, UPDATE는 참조하는 테이블에서 이전 값을 찾아야 하기에, 참조하는 칼럼에도 인덱스를 설정하는 것이 유리한 경우도 있다.

## 5\. 장단점

### 5-1. 장점

-   데이터의 무결성
-   쉬운 데이터 구조/관계 확인
-   update, delete 등의 로직 간소화

### 5-2. 단점 

-   참조 테이블들을 스캔하는데 드는 추가 코스트
-   테이블 구조 변경 시 고려해야 할 사항 증가
-   데이터 변경이 찾을 때 특히 코스트가 증가
-   테스트 데이터 생성 및 데이터 강제 보정이 번거로움

## 6\. 적용 검토

이미 FK를 사용하는 테이블도 다수 존재한다. 중요 기준 데이터의 무결성을 보장하는 데는 효율적인 기능이라고 생각한다. 다만 "실시간으로 업데이트되는 대용량 데이터의 경우에도 무결성 보장을 위해 FK를 적용해야 하는가?"에 대한 판단을 위해 FK에 대한 내용을 좀 깊게 들여다보았다. 일단 결론은, 실시간으로 변동되는 대용량 데이터 베이스, 특히 타 시스템과의 동기화가 이루어지고 있는 산군의 데이터베이스에는 적합하지 않다고 판단된다.

"FK의 스캔 코스트가 성능에 큰 영향이 없다", 혹은 "적절한 인덱싱으로 스캔 코스트를 관리할 수 있다."라는 상황 자체가 참조 테이블의 인덱스를 전제한다. FK만을 위한 인덱스를 참조된 테이블에 추가하거나, 참조된 테이블에 업데이트나 삭제가 발생하여 참조하는 테이블에서 이전 값을 조회하기 위해 역스캔하는 경우의 코스트를 줄이기 위해 인덱스를 추가해야 하는 상황이 발생한다.

현재 운영 중인 데이터베이스는 최소 수백만 건에서 수천만건의 테이블이 서로 연결되어 있으며 read/write 간 동기화, ElasticSearch 등으로의 동기화를 고려하여 최적의 인덱스로 세팅, 되어있고 실시간 모니터링을 통한 튜닝이 지속적으로 진행 중이다. 모든 칼럼에 인덱스를 거는 것이 효율적이지 않은 것처럼, 한정된 자원으로 최적의 인덱싱을 찾아야 하고, 데이터의 변경에 민감하게 대응해야 하는 상황에서는 FK를 설정하는 것이 효율적이지 못하다는 판단이다. 

또한, 참조되는 테이블의 추가 인덱스 설정 없이 FK를 설정하는 것만으로는 코스트가 증가하지 않지만 실시간으로 몇천 건, 많게는 몇만 건의 데이터가 업데이트 및 동기화되는 상황에서 테이블의 Lock이 여러 테이블로 전파될 위험성도 무시할 수 없다.

참고 : [https://www.postgresql.org/docs/16/ddl-constraints.html#DDL-CONSTRAINTS-FK](https://www.postgresql.org/docs/16/ddl-constraints.html#DDL-CONSTRAINTS-FK)