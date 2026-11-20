# 2단계 커밋 프로토콜(Two-Phase Commit Protocol), Prepare transaction

> 전체 글은 블로그에서 계속 업데이트한다 → **[https://junhkang.com/posts/68/](https://junhkang.com/posts/68/)**

PostgreSQL 2단계 커밋(2PC)은 분산 트랜잭션의 모든 데이터베이스가 함께 커밋되거나 롤백되도록 보장한다. PREPARE TRANSACTION, COMMIT PREPARED, ROLLBACK PREPARED 사용법과 남은 트랜잭션의 위험을 정리한다.

## 다루는 내용
- 1. 2단계 커밋 프로토콜(two-phase commit (2PC))
- 2. PREPARE TRANSACTION
  - 2-1. PREPARE TRANSACTION란?
  - 2-2. 주의사항
- 3. COMMIT/ROLLBACK PREPARE

전체 본문, 예제 SQL, 그림은 [블로그 원문](https://junhkang.com/posts/68/)에 있다. 이 저장소의 이전 버전 본문은 git 히스토리에 남아 있다.
