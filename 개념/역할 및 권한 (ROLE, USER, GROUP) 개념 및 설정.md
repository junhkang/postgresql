# 역할 및 권한 (ROLE, USER, GROUP) 개념 및 설정

> 전체 글은 블로그에서 계속 업데이트한다 → **[https://junhkang.com/posts/22/](https://junhkang.com/posts/22/)**

PostgreSQL 8.1부터 USER와 GROUP은 ROLE로 통합되었고, 지금은 USER가 기본으로 LOGIN 권한을 가진다는 점만 다르다. CREATE ROLE, USER, GROUP 문법과 SUPERUSER, CREATEDB 등 권한 옵션을 정리한다.

## 다루는 내용
- 1. ROLE
  - 1-1. ROLE 생성
  - 1-2. ROLE 이란?
  - 1-3. ROLE 권한별 특징
- 2. USER
  - 2-1. USER 생성
  - 2-2. USER란?
- 3. GROUP
  - 3-1. GROUP 생성
  - 3-2. GROUP이란?
- 4. ROLE, USER, GROUP 차이

전체 본문, 예제 SQL, 그림은 [블로그 원문](https://junhkang.com/posts/22/)에 있다. 이 저장소의 이전 버전 본문은 git 히스토리에 남아 있다.
