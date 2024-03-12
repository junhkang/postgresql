<p align="center"><img src="/img/role.jpeg"/></p>

## 1\. ROLE

###         1-1. ROLE 생성

```
-- 기본
CREATE ROLE jonathan LOGIN;
-- 비밀번호 포함
CREATE USER davide WITH PASSWORD 'jw8s0F4';
-- 권한 포함
CREATE ROLE admin WITH CREATEDB CREATEROLE;
-- 사용 기한 포함
CREATE ROLE miriam WITH LOGIN PASSWORD 'jw8s0F4' VALID UNTIL '2005-01-01';
-- 삭제
DELETE ROLE miriam;


-- Synopsis

CREATE ROLE name [ [ WITH ] option [ ... ] ]

where option can be:

      SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | REPLICATION | NOREPLICATION
    | BYPASSRLS | NOBYPASSRLS
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL
    | VALID UNTIL 'timestamp'
    | IN ROLE role_name [, ...]
    | IN GROUP role_name [, ...]
    | ROLE role_name [, ...]
    | ADMIN role_name [, ...]
    | USER role_name [, ...]
    | SYSID uid
```

###         1-2. ROLE 이란?

-   CREATE ROLE은 PostgreSQL database cluster에 새로운 ROLE을 추가한다. 
-   ROLE은 데이터베이스 object, 권한을 가질 수 있는 엔티티이다.
-   ROLE은 사용방법에 따라 USER, GROUP 혹은 둘다로 간주될 수 있다.
-   CREATEROLE 권한이 있어야지만 사용 가능하다.
-   ALTER ROLE, DELETE ROLE을 통해 권한을 수정, 삭제 가능하다.

###         1-3. ROLE 권한별 특징

-   SUPERUSER - 로그인을 제외한 모든 권한 포함 (ex. Role 생성 및 권한 부여)
-   LOGIN - 데이터베이스에 로그인하기 위한 권한
-   PASSWORD - 로그인 비밀번호 설정
-   CREATEDB - 데이터베이스 생성
-   CREATEROLE - ROLE 생성/삭제/수정
-   REPLICATION - REPLICATION 권한
-   CONNECTIONLIMIT - 데이터베이스 접속 카운트
-   INHERIT - ROLE 권한들 상속

## 2\. USER

###         2-1. USER 생성

```
-- 기본
CREATE USER jonathan;
-- 비밀번호 추가
CREATE USER davide WITH PASSWORD 'jw8s0F4';
-- 만료기한 추가
CREATE USER miriam WITH PASSWORD 'jw8s0F4' VALID UNTIL '2005-01-01';
-- 권한 추가
CREATE USER manuel WITH PASSWORD 'jw8s0F4' CREATEDB;


--Synopsis
CREATE USER name [ [ WITH ] option [ ... ] ]

where option can be:
    
      SYSID uid 
    | CREATEDB | NOCREATEDB
    | CREATEUSER | NOCREATEUSER
    | IN GROUP groupname [, ...]
    | [ ENCRYPTED | UNENCRYPTED ] PASSWORD 'password'
    | VALID UNTIL 'abstime'
```

###         2-2. USER란?

-   CREATE ROLE은 PostgreSQL database cluster에 새로운 User을 추가한다.
-   CREATEUSER 권한이 있어야지만 사용 가능하다.

## 3\. GROUP

###         3-1. GROUP 생성

```
--기본
CREATE GROUP staff;
--유저 추가
CREATE GROUP marketing WITH USER jonathan, david;
--그룹 삭제
DROP GROUP staff;

-- Synopsis
CREATE GROUP name [ [ WITH ] option [ ... ] ]

where option can be:

     SYSID gid
   | USER  username [, ...]
```

###         3-2. GROUP이란?

-   CREATE GROUP은 USER 그룹을 생성한다. 
-   SUPERUSER 권한이 있어야지만 생성가능하다.
-   데이터베이스의 cluster 레벨에 접근 가능하기 위해 GROUP, USER, ROLE은 모두 cluster단에서 정의되어 있다.

## 4\. ROLE, USER, GROUP 차이

-   ROLE은 Postgresql Database 관련 권한들을 모아 놓은 것으로, 8.1버전부터 USER와 GROUP의 개념이 ROLE로 통합되었다.
-   현재 버전에서는 USER와 ROLE의 기능은 동일하며, USER는 login 권한이 default, ROLE은 login 권한을 별도로 부여해야 하는 차이점만 있다. 
-   CREATE GROUP의 경우 PostgreSQL의 SQL 표준에는 존재하지 않으며, ROLE과 비슷한 개념을 가지고 있다.

참고

[https://www.postgresql.org/docs/8.0/sql-creategroup.html](https://www.postgresql.org/docs/8.0/sql-creategroup.html)

[https://www.postgresql.org/docs/8.0/sql-createuser.html](https://www.postgresql.org/docs/8.0/sql-createuser.html)

[https://www.postgresql.org/docs/current/sql-createrole.html#:~:text=A%20role%20is%20an%20entity,on%20how%20it%20is%20used.](https://www.postgresql.org/docs/current/sql-createrole.html#:~:text=A%20role%20is%20an%20entity,on%20how%20it%20is%20used.)

[https://docs.aws.amazon.com/ko\_kr/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.Roles.html](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.Roles.html)

[https://blog.ex-em.com/1655](https://blog.ex-em.com/1655)