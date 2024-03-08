  
  

  

2단계 트랜잭션은 외부 트랜잭션 관리 시스템에서 사용하기 위해 존재하며 X/Open XA 표준에서 제안된 특징과 모델을 따른다. (사용빈도가 낮은 일부 기능은 구현되지 않았다.) 2단계 트랜잭션은 다음 스탭에 따라 작동된다.

- PREPARE를 통해 데이터베이스 각 노드에 커밋을 준비 요청
- 각 데이터베이스는 필요 리소스에 LOCK 설정, 로그파일 저장 등 커밋 준비 작업 실행
- 준비 과정의 실패/성공 여부 알림 
- 모든 데이터베이스 노드로 부터 완료 메세지를 받을때까지 대기
- 한 데이터베이스라도 PREPARE OK를 받지 못하면, 모든 데이터베이스 노드에 롤백 메세지를 보내 해당 작업 롤백
- 모든 데이터베이스에서 PREPARE OK를 받으면 모든 데이터베이스 노드에 커밋 메세지를 보내고 모든 작업 커밋

짧은 기간의 PREPARED 트랜잭션은 공유 메모리와 WAL에 저장되며 체크포인트의 트랜잭션은 pg_twophase 디렉토리에 기록된다.

### 5-1. PREPARE TRANSACTION

PREPARE TRANSACTION 'foobar';

  

PREPARED TRANSACTION 구문은 2단계 커밋을 위해 현재 트랜잭션을 준비 상태로 변경한다. 이 명령어 후에 해당 트랜잭션은 현재 세션과 완전히 분리되며, 해당 트랜잭션의 상태가 디스크에 완전히 저장된다. 그리고 해당 트랜잭션은 데이터베이스가 커밋요청 전에 충돌하더라도 성공적으로 커밋될 확률이 높다.

  

PREPARED TRANSACTION이 성공하면 트랜잭션은 COMMIT PREPARED, ROLLBACK PREPARED로만 커밋/롤백된다. 해당 커맨드는 PREPARED를 실행한 세션외에 어떤 세션에서도 실행 가능하다.

  

구문을 실행한 세션의 시점에서 봤을때 PREPARED TRANSACTION 과 ROLLBACK 명령어는 큰 차이가 없다. 

현재 진행중인 트랜잭션은 없어지고, PREPARE TRANSACTION의 실제 결과는 보이지 않기 때문이다. (PREPARED COMMIT 후에나 차이를 알수 있다.) PREPARE TRANSACTION 명령어가 실패한다면, 현재 트랜잭션이 취소되며 ROLLBACK과 동일한 결과를 나타낸다.

  

PREPARE TRANSACTION은 어플리케이션, 혹은 상호 통신하는 세션을 위한 기능이 아니다. 외부 트랜잭션 관리자가 다양한 트랜잭션이나 기타 트랜잭션 리소스에 걸쳐 원자단위의 글로벌 트랜잭션을 수행할수 있도록 하는것이 목적 -> 만약 transaction manager를 쓰는것이 아니라면 PREPARE TRANSACTION의 사용을 중단해야한다.

  

트랜잭션 내부에서 사용해야만한다.

  

임시 테이블이나 세션의 임시 네임스페이스를 포함한 명령어나 WITH HOLD, LITEN, UNLISTEN, NOTIFY를 PREPARE에 사용하는 것은 불가능하다.  해당 기능들은 현재세션과 너무 타이트하기 연결되어있어 트랜잭션 사용에 유용하지 않다.

  

런타임 파라미터를 변경한다면, PREPARE TRANSACTION 이후에도 적용되며 COMMIT PREPARED, ROLLBACK PREPARED의 영향을 ㅂ다지 않는다. 이러한 측면에서 PREPARED TRANSACTION은 롤백보다는 커밋에 더 가깝다. 

  

지금 실행중인 prepared 트랜잭션은 pg_prepared_xacts에서 확인 가능하다