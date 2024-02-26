## 1\. WAL (Write-Ahead Logging) / 아카이브 모드 백업(Archive mode backup)이란?

아카이브 모드 백업을 이해하기 위해 WAL에 대한 개념을 먼저 살펴보자. WAL은 PostgreSQL에서 데이터의 무결성을 보장하는 표준 방법으로, 기본 콘셉트는 모든 데이터의 변경을 로깅 완료 후에 실행하는 것이다. WAL 기록을 영구적인 저장소에 먼저 기록한 후에 데이터의 변경 내용을 실행하는 것으로, 이 과정을 거치면 충돌 혹은 데이터에 문제가 있을 때 WAL 로깅 내용을 바탕으로 특정 시점으로 복구가 가능하여 데이터 무결성을 보장할 수 있다.

충돌이 발생할 때마다 Log를 통해 데이터베이스를 복구할 수 있기 때문에, 모든 트랜잭션 커밋 시마다 디스크의 데이터를 flush 할 필요 없다.  또한 WAL을 사용하면 디스크 쓰기 횟수가 현저히 줄어든다. 트랜잭션에 의해 변경된 모든 데이터 파일을 write 하는 것이 아니라 WAL 파일만 디스크에 flush 하면 트랜잭션이 커밋되기 때문이다. WAL은 순차적으로 작성되기 때문에 WAL 동기화 비용은 데이터 페이지를 flush 하는 것보다 코스트가 훨씬 적다. (특히 데이터베이스가 여러 군데의 작은 트랜잭션을 처리할 때 효율적이다.) 

## 2\. 장점

-   WAL은 온라인 백업 및 특정 시점 복구(PIT, point-in-time)를 지원 (WAL 데이터를 보관함으로써 사용가능한 WAL 데이터에 포함된 어떠한 순간으로도 롤백이 가능)
-   복원에 필요한 WAL 파일의 수량 제한이 없기에 백업을 시작한 시점 이후의 WAL 로그파일만 존재한다면 백업기간이 아무리 길더라도 복원이 가능
-   운영 서버의 WAL 파일을 주기적으로 백업해 놓는다면 운영 서버 장애 발생 시 빠르게 복구가 가능

## 3\. 단점

-   특정 데이터베이스만을 대상 불가능, 전체 데이터베이스를 대상으로 진행
-   WAL 로그를 충분히 저장할 디스크 여유 공간 필요
-   대량의 데이터를 처리할때 WAL write 코스트 증가

## 4\. 설정

$PGDATA의 postgresql.conf 파일 내의 파라미터를 통해 상세 설정 가능

#### 4-1. wal\_level (Enum)

WAL 레코드 정보의 양을 결정한다. 디폴트 값은 minimal이다.

-   **minimal -** 충돌 또는 즉시 셧다운으로부터 복구하기 위한 최소한의 정보 
-   **archive -** WAL 아카이브에 필요한 로깅 추가
-   **hot\_standby -** 대기 서버에서 읽기전용 쿼리에 필요한 정보 추가

#### 4-2. archive\_mode (Boolean)

archive\_mode를 선택하면 완료된 WAL 세그먼트가 아카이브 저장소에 저장 (wal\_level이 minimum인 경우 사용 불가)

#### 4-3. archive\_command (String)

완료된 WAL 파일 세그먼트를 아카이빙 할 때 실행하는 쉘 명령어

[https://www.postgresql.org/docs/16/wal-intro.html](https://www.postgresql.org/docs/16/wal-intro.html)

[https://postgresql.kr/docs/13/continuous-archiving.html#BACKUP-PITR-RECOVERY](https://postgresql.kr/docs/13/continuous-archiving.html#BACKUP-PITR-RECOVERY)