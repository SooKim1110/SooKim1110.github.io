---
title: Real My SQL 5장 - 트랜잭션과 잠금
date: 2022-12-15 10:00:00 +0900
categories: [Archive]
tags: [Database]
---

# 트랜잭션과 잠금

## 트랜잭션

여러 개의 작업을 하나로 묶은 실행 유닛을 의미하고, 작업의 완전성을 보장한다

논리적인 작업 셋이 모두 처리되거나, 모두 처리되지 않도록 원 상태로 복구하는 기능



#### 트랙잭션 특성 (ACID)

| Atomicity(원자성)         | 한 트랜잭션의 연산들이 모두 성공하거나, 전부 실패            |
| ------------------------- | ------------------------------------------------------------ |
| Consistency(일관성)       | 트랜잭션 이전과 이후의 DB 상태가 일관. DB가 가진 제약과 규칙이 트랜잭션 이후에도 만족됨 |
| Isolation(격리성, 고립성) | 동시에 여러 트랜잭션이 수행될 때 각 트랜잭션은 격리되어 있어서 연속으로 실행한 것과 동일한 결과 |
| Durability(지속성)        | 트랜잭션이 성공적으로 수행되었다면 해당 트랜잭션이 DB에 영구적으로 보존. 트랜잭션 완료를 로그로 기록하고 이상 발생시 로그를 사용해 복원 |



#### MySQL에서의 트랜잭션

```mysql
-- fdpk가 PRIMARY KEY

> INSERT INTO tbl_myisam ( fdpk ) VALUES (3);
> INSERT INTO tbl_innodb ( fdpk ) VALUES (3);

-- If autocommit mode is enabled, each SQL statement forms a single transaction on its own
> SET autocommit=ON;

> INSERT INTO tbl_myisam ( c1 ) VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'

> INSERT INTO tbl_innodb ( c1 ) VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'

> SELECT c1 FROM tbl_myisam;
+----+
| c1 |
+----+
|   1|
|   2|
|   3|
+----+


> SELECT c1 FROM tbl_innodb;
+----+
| c1 |
+----+
|   3|
+----+
```

MyISAM 엔진은 트랜잭션을 지원하지 않아서 부분 업데이트(Partial Update) 발생

InnoDB 엔진은 트랜잭션이 적용되어서, 롤백됨



#### 주의사항

트랜잭션은 최소한의 코드에만 적용하는 것이 좋다!

- 실제로 DBMS를 사용하는 부분에만 적용
- 파일 전송, 원격 서버와 통신하는 작업 등은 트랜잭션에서 제거
- 묶어야하는 작업들은 하나의 트랜잭션으로 처리하고, 성격이 다른 작업은 별도 트랜잭션으로 분리

```
// 게시물 저장 예시
1. 처리 시작
2. 사용자의 로그인 여부 확인
3. 게시글 내용 오류 여부 확인
4. 첨부파일 확인 및 저장
    => DB Connection START
    => Transaction START
5. 사용자의 게시글 내용을 DB 에 저장
6. 첨부파일을 DB 에 저장
    => Transaction COMMIT
7. 저장된 내용 또는 기타 정보를 DBMS 에서 조회
8. 게시글 등록 알림 메일 발송
    => Transaction START
9. 알림 메일 발송 이력을 DB 에 저장
    => Transaction COMMIT
    => DB Connection END
10 처리 완료
```



## 잠금

`MySQL 엔진 잠금`: 모든 스토리지 엔진에 영향 미침

`스토리지 엔진 잠금`: 해당 스토리지 엔진에만 영향



### MySQL엔진 잠금

#### 글로벌 락

MySQL 서버의 모든 테이블에 잠금을 건다

`FLUSH TABLES WITH READ LOCK` 명령으로 획득

한 세션에서 글로벌 락을 획득하면, 다른 세션의 SELECT 를 제외한 거의 대부분의 DDL, DML 은 대기 상태가 된다

(반대로, 장시간 SELECT 쿼리가 실행되고 있으면 `FLUSH TABLES WITH READ LOCK` 은 해당 쿼리가 종료될 때까지 기다려야한다)



InnoDB는 트랜잭션을 지원해서 일관된 데이터 상태를 위해 모든 데이터 변경 작업을 멈출 필요가 없다

그래서, MySQL 8.0 부터는 `백업 락`이라는 더 가벼운 글로벌 락이 도입되었다.

```mysql
> LOCK INSTANCE FOR BACKUP
-- 백업 작업 실행
> UNLOCK INSTANCE
```

특정 세션에서 백업 락을 획득하면 모든 세션에서 테이블 스키마나 사용자의 인증 관련 정보를 변경할 수 없게 된다

일반적인 테이블의 데이터 변경은 허용된다



#### 테이블 락

개별 테이블 단위로 설정되는 잠금

명시적: `LOCK TABLES table_name [READ | WRITE ]` 로 획득하고, `UNLOCK TABLES`로 해제

특별한 상황 아니면 사용할 필요가 없다



묵시적으로는 MyISAM이나 Memory 테이블에 데이터 변경 쿼리를 실행하면 발생한다

쿼리가 실행되는 동안 자동으로 획득했다가 완료 후 해제된다

InnoDB 테이블은 레코드 기반 잠금을 제공해서 데이터 변경 쿼리로 인해 묵시적인 테이블락이 설정되지는 않는다 (테이블 락이 설정되지만 DDL에만 영향)



#### 네임드 락

`GET_LOCK()` 함수를 이용해 임의의 문자열에 대해 잠금을 설정할 수 있다

```mysql
-- 'mylock' 이라는 문자열에 대해 잠금을 획득
-- 이미 잠금을 사용중이면 2초 후 자동 잠금 해제
> SELECT GET_LOCK('mylock', 2);

-- 'mylock' 이라는 문자열에 잠금이 걸려있는지 확인
> SELECT IS_FREE_LOCK('mylock', 2)

-- 'mylock' 이라는 문자열에 대한 잠금을 반납
> SELECT RELEASE_LOCK('mylock', 2)

-- 3개의 함수 모두 정상적으로 잠금을 획득하거나 해제한 경우 1 을
-- 아니라면 NULL 또는 0 을 반환함
```

DB 1대를 5대의 웹서버가 접속하는 상황에서, 상호 동기화를 처리해야 할 때나, 많은 레코드에 대해서 복잡한 요건으로 변경을 하는 트랜잭션에 활용할 수 있다

MySQL 8.0 부터는 네임드 락의 중첩 사용도 가능



#### 메타 데이터 락

데이터베이스 객체의 이름이나 구조를 변경하는 경우 획득하는 잠금

`RENAME TABLE tab TO tab_b`  같이 테이블 이름을 변경하는 경우 자동으로 획득된다 (원본, 변경될 이름 둘 다 잠금 설정)



### InnoDB 스토리지 엔진 잠금

레코드 기반의 잠금 기능을 제공하여 MyISAM보다 뛰어난 동시성 처리를 제공한다

`Information Schema` 조회로 트랜잭션과 잠금, 잠금 대기중인 트랜잭션 목록을 조회할 수 있고, `Performance Schema`를 이용해 내부 잠금(세마포어) 모니터링도 할 수 있다 (8.0 부터는 Performance Schema 주로 이용)



#### 레코드 락

레코드 자체만을 잠그는데 <u>인덱스의 레코드</u>를 잠근다!

- 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드 모두에 락을 걸어야 한다

보조 인덱스를 이용한 변경 작업은 넥스트 키락(Next-key lock) 또는 갭락(Gap lock)을 사용하고,
프라이머리키 혹은 유니크 인덱스에 의한 변경 작업은 레코드 락을 건다



#### 갭 락

레코드와 인접한 레코드 사이의 간격을 잠그는 것으로, 간격에 새로운 레코드가 생성(INSERT)되는 것을 제어한다



#### 넥스트 키 락

레코드 락과 갭 락을 합쳐놓은 형태

범위를 지정한 쿼리를 실행하게되면 실제로는 위에서 각각 설명했던 record lock (찾아진 인덱스 레코드에 대해)과 gap lock (해당하는 인덱스 레코드 사이사이) 이 복합적으로 사용된다.

- STATEMENT 포맷의 바이너리 로그를 사용하는 MySQL 서버에서는 REPEATABLE READ 격리 수준을 사용해야 한다

바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때, 소스 서버에서 만들어낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주 목적이다



#### 자동 증가 락

자동 증가하는 숫자 값을 채번하기 위해 AUTO_INCREMENT라는 칼럼 속성을 제공하는데, 이를 위해 InnoDB에서는 자동 증가 락이라고 하는 테이블 수준의 잠금을 사용한다

자동 증가 락은 INSERT 와 REPLACE 같이 새로운 레코드를 생성하는 작업에만 걸리고, UPDATE 혹은 DELETE 에는 걸리지 않는다

`innodb_autoinc_lock_mode` 라는 시스템 변수를 이용해 작동 방식을 변경할 수 있다



## 격리 수준

여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는 것

|                 | DIRTY READ    | NON-REPEATABLE READ | PHANTOM READ      |
| --------------- | ------------- | ------------------- | ----------------- |
| READ UNCOMITTED | 발생          | 발생                | 발생              |
| READ COMITTED   | 발생하지 않음 | 발생                | 발생              |
| REPEATABLE READ | 발생하지 않음 | 발생하지 않음       | 발생 (InnoDB는 X) |
| SERIALIZABLE    | 발생하지 않음 | 발생하지 않음       | 발생하지 않음     |

DIRTY READ: 트랜잭션에서 처리한 작업이 완료되지 않았는데 다른 트랜잭션에서 볼 수 있는 현상

NON-REPEATABLE READ: SELECT 시 항상 같은 결과를 가져오는 REPEATABLE READ 이 보장되지 않는다

PHANTOM READ: 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안 보였다 하는 현상

- “DIRTY READ”라고도 하는 READ UNCOMMITTED는 일반적인 데이터베이스에서는 사용하지 않는다
- SERIALIZABLE 또한 동시성이 중요한 데이터베이스에서는 사용되지 않는다
- 아래로 갈수록 고립 정도가 높아지며, 동시성도 보통 떨어진다



#### READ UNCOMITTED

변경 내용이 커밋과 롤백 여부에 상관없이 다른 트랜잭션에 보여진다

데이터 정합성에 문제가 많기 때문에 보통 사용되지 않는다



#### READ COMITTED

커밋이 된 데이터만 다른 트랜잭션에서 조회할 수 있다

오라클에서 기본으로 사용되는 격리 수준이며, 온라인 서비스에서 가장 많이 사용된다

변경 전 데이터가 언두 로그로 복사되고, 커밋 전 데이터를 조회하면 이 언두 로그 데이터가 반환된다



#### REPEATABLE READ

동일 트랜잭션 내에서는 동일한 결과를 보장한다

InnoDB의 기본 격리 수준으로, 바이너리 로그를 가진 MySQL은 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다

(InnoDB는 갭 락과 넥스트 키 락이 있어서, 검색하고자 하는 범위 내에 갭 락을 걸어서 특정 데이터가 추가, 삭제되어 발생하는 Phantom Read가 발생하지 않는다 -> 확인 필요)



REPEATABLE READ와의 차이는 언두 영역에 백업된 레코드의 여러 버전 중 몇 번째 이전 버전까지 찾아 들어가느냐에 있다

모든 트랜잭션은 고유 번호를 가지고, 언두 영역의 레코드도 변경을 발생시킨 트랜잭션의 번호가 포함되어 있다

해당 트랜잭션 안에서 실행되는 모든 SELECT 쿼리는 트랜잭션 번호가 자신의 번호보다 작은 번호에서 변경된 것만 보게 된다



`SELECT FOR UPDATE`쿼리

- 가정 먼저 LOCK을 획득한 SESSION의 SELECT 된 ROW들이 UPDATE 쿼리후 COMMIT 되기 이전까지 다른 SESSION들은 해당 ROW들을 수정하지 못하도록 하는 기능
- 언두 레코드에는 잠금을 걸 수 없어서, 언두 영역의 변경 데이터가 아닌 현재 레코드 값을 가져오게 된다 (PHANTOM READ)



#### SERIALIZABLE

보통 SELECT는 아무런 레코드 잠금도 설정하지 않고 실행되지만 SERIALIZABLE 수준에서는 읽기 작업도 공유 잠금을 획득해야 하고, 다른 트랜잭션이 레코드를 변경하지 못한다





### 기타 참고자료

#### 트랜잭션 문법

- `START TRANSACTION` 트랜잭션 시작
- `COMMIT` 트랜잭션을 DB에 적용
- `ROLLBACK` 트랜잭션을 취소하고, START TRANSACTION 실행 전 상태로 롤백

DDL은 트랜잭션의 롤백 대상이 아니다



#### DDL, DML, DCL

`DDL(Data Definition Language)`: CREATE, ALTER, DROP, RENAME, TRUNCATE

`DML(Data Manipulation Language)`: SELECT, INSERT, UPDATE, DELETE

`DCL(Data Control Lanaguage)`: GRANT, REVOKE

`TCL(Transaction Control Language)`: COMMIT, ROLLBACK, SAVEPOINT



#### 락 & 데드락

https://www.letmecompile.com/mysql-innodb-lock-deadlock/



#### Consistent Read

https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html

https://www.letmecompile.com/mysql-innodb-transaction-model/