## 1\. Visibility Map(가시성 맵)란?

Visibility Map은 트랜잭션에서 데이터에 접근할 때 어떤 데이터가 가시적인지(**모든 트랜잭션에서 읽을 수 있는지**), 안정적인지 (**동결된 튜플인지**) 판별하는데 도움을 준다. 데이터 접근 시 불필요한 I/O작업을 줄여주고, 데이터베이스가 어떤 페이지를 직접 접근할 수 있는지를 빠르게 판단함으로써 시스템의 효율적을 올려주는 역할을 한다.

## 2\. Visibility Map(가시성 맵)의 데이터 관리

Visibility Map은 데이터를 주요 데이터와는 별도의 파일(fork)에 \_vm 접미사를 붙여 관리한다. 예를 들어 예를 들어 employees 테이블이 있다고 하면 테이블의 Visibility Map은 별도의 포크에 저장된다. 이 포크의 이름은 파일 노드 번호에 \_vm 접미사를 붙여 구성되며, 예를 들어 파일 노드번호가 12345인 경우 VM 파일은 12345\_vm으로 저장된다. 데이터에는 해당 테이블의 page가 모든 트랜잭션에 보이는지, 동결된 튜플만을 포함하는지 등의 정보를 저장한다. 데이터베이스가 employees 테이블을 조회할 때, 가시성 맵을 먼저 확인한다. 만약 쿼리가 접근하려는 pages가 모든 트랜잭션에게 보이는 상태라고 확인되면, 시스템은 데이터에 더 빠르게 접근한다. 불필요한 버전검사나 락을 안 해도 되기에 성능이 향상된다.

## 3\. Visibility Map(가시성 맵)의 원리

Visiblity Map은 힙 pages당 2개의 비트를 별도로 저장한다. 첫 번째 비트가 설정되어 있으면, 해당 페이지가 모두 visible(가시적) 한 상태이고, 이는 vacuum이 필요한 튜플을 포함하지 않는다는 뜻이다. 이는 인덱스 영역의 tuple만을 사용하여 **index-only-scan**으로 쿼리를 조회할 때도 사용된다. **index-only-scan**은 해당 포스트에서 확인 가능하다.

[\[PostgreSQL\] Index-Only 스캔과 Covering 인덱스, Index-only스캔의 효율적인 사용](https://github.com/junhkang/postgresql/blob/main/%EC%84%B1%EB%8A%A5%ED%96%A5%EC%83%81/Index-Only%20%EC%8A%A4%EC%BA%94%EA%B3%BC%20Covering%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%2C%20Index-only%EC%8A%A4%EC%BA%94%EC%9D%98%20%ED%9A%A8%EC%9C%A8%EC%A0%81%EC%9D%B8%20%EC%82%AC%EC%9A%A9.md)

두 번째 bit가 설정되어 있다면 모든 pages의 튜플이 frozen(동결된) 상태라는 뜻이다. 이 상태에선 일반적인 vacuum은 물론 **anti-wraparound vacuum**도 동작시킬 필요가 없다.

**anti-wraparound-vacuum -** 전체 데이터베이스를 검사하여 트랜잭션 ID가 안전한 범위 내에 있는지 확인하여, 필요에 따라 조정하며 트랜잭션 ID의 오버플로우를 방지한다. **트랜잭션 ID**에 대한 상세 내용은 해당 포스트에서 확인 가능하다.

[\[PostgreSQL\] 트랜잭션(Transaction)의 작동원리](https://github.com/junhkang/postgresql/blob/main/%EA%B0%9C%EB%85%90/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98(Transaction)%EC%9D%98%20%EC%9E%91%EB%8F%99%EC%9B%90%EB%A6%AC.md)

Visiblity Map의 2가지 비트는 최대한 보수적으로 해석된다. 1, 2 번째 비트가 설정되어 있을 경우에는 무조건 참이지만, 비트가 설정되지 않을 경우에는 참 일수도 거짓일 수도 있다.

## 4\. Visibility Map(가시성 맵)의 생명주기

Visiblity Map의 비트는 vacuum에 의해서만 설정된다. 데이터베이스 내의 pages에 vacuum 작업이 수행되면 관련 Visiblity Map의 비트가 설정이 되고, 해당 pages가 모든 트랜잭션에서 완전히 가시적임을 표시하며, 더 이상 vacuum 안 해도 됨을 나타낸다. 그 후에 pages의 데이터가 하나라도 수정(update, insert, delete 등) 될 경우, VM의 비트는 초기화된다. 데이터의 상태가 변경되었기에 vacuum 작업 대상에 포함시켜야 함을 나타낸다.

## 5\. Visibility Map(가시성 맵) 정보 확인

pg\_visibility 함수를 사용해서 vm에 저장된 정보를 확인할 수 있다.

-   _**pg\_visibility\_map**(relation regclass, blkno bigint, all\_visible OUT boolean, all\_frozen OUT boolean) returns record-_ 해당 테이블, 해당 블록의 모든 VM의  visible, frozen 비트 조회

-   _**pg\_visibility**(relation regclass, blkno bigint, all\_visible OUT boolean, all\_frozen OUT boolean, pd\_all\_visible OUT boolean) returns record -_ 해당 테이블, 해당 블록의 모든 VM의  visible, frozen 비트 조회 + PD\_ALL\_VISIBLE 비트 

-   _**pg\_visibility\_map**(relation regclass, blkno OUT bigint, all\_visible OUT boolean, all\_frozen OUT boolean) returns setof record -_ 해당 테이블의 모든 블록의 VM의  visible, frozen 비트 조회

-   _**pg\_visibility**(relation regclass, blkno OUT bigint, all\_visible OUT boolean, all\_frozen OUT boolean, pd\_all\_visible OUT boolean) returns setof record -_ 해당 테이블의 모든 블록의 VM의  visible, frozen 비트 조회 + PD\_ALL\_VISIBLE 비트

-   _**pg\_visibility\_map\_summary**(relation regclass, all\_visible OUT bigint, all\_frozen OUT bigint) returns record -_ VM에 연관 있는 테이블의 visible 페이지 수량, frozen 페이지 수량 확인

-   _**pg\_check\_frozen**(relation regclass, t\_ctid OUT tid) returns setof tid -_ VM에 frozen으로 마킹되어 있는 pages 중 non-frozen 튜플의 TID, 존재해서는 안 되는 경우로, 뭔가 조회가 된다면 VM에 문제가 있는 것

-   _**pg\_check\_visible**(relation regclass, t\_ctid OUT tid) returns setof tid -_ VM에 visible으로 마킹되어 있는 pages 중 non-visible 튜플의 TID, 존재해서는 안 되는 경우로, 뭔가 조회가 된다면 VM에 문제가 있는 것

-   **pg\_truncate\_visibility\_map**(relation regclass) returns void - 해당 테이블의 VM을 truncate 한다. VM에 문제가 있는 경우 강제로 재설정이 필요할 때 사용. 해당 테이블의 첫 번째 vacuum이 실행될 때 재생성되며, 그전까지는 모든 VM이 모두 0 값으로 유지

참고

[https://www.postgresql.org/docs/16/storage-vm.html](https://www.postgresql.org/docs/16/storage-vm.html)

[https://www.postgresql.org/docs/16/pgvisibility.html](https://www.postgresql.org/docs/16/pgvisibility.html)