---
title: Redis 동시성 처리와 트랜잭션 ① - Redis의 트랜잭션 명령어
date: 2023-01-21 10:00:00 +0900
categories: [Database]
tags: [Redis, Database]
---

최근에 코드를 짜다가 Redis의 동시성 처리를 제대로 해주지 못해서 이슈가 발생했어서, 공부해본 내용을 정리해보려고 한다.

-------

## 동시성 문제
```kotlin
val value = redisRepository.find(key) // key 조회

if (value == null) {
    redisRepository.save(key, newValue) // 해당 키에 저장된 값이 없으면 새로 저장
} else {
    redisRepoitory.delete(key) // 해당 키에 저장된 값이 있으면 해당 키 삭제
}
```

언뜻보면 문제가 없는 코드처럼 보이지만 동시에 여러 클라이언트가 해당 코드를 실행하게 되면 문제가 생길 수 있다.

만약에 key에 이미 저장된 값이 없을 경우에 예상했던 올바른 동작은 다음과 같다.  

1) T1: key 조회 (값 없음)  
2) T1: key, newValue(A) 저장  
3) T2: key 조회 (값 A 조회됨)  
4) T2: 값이 조회되었으므로 해당 키 삭제

최종적으로는 key가 삭제된다.

하지만 두 클라이언트가 동시에 접근할 때의 시나리오를 보게 되면  

1) T1: key 조회 (값 없음)  
2) T2: key 조회 (값 없음)  
3) T1: key, newValue(A) 저장  
4) T2: key, newValue(B) 저장 (위에서 저장된 값이 덮어씌워지게 된다!)

예상 동작과 달리 최종적으로 key는 삭제되지 않고, 저장된 값은 B가 된다.

사실 Redis라서 발생하는 문제는 아니고 모든 데이터베이스에서 동일한 레코드에 접근할 때 발생할 수 있는 동시성 문제이다.

현재 코드는 key를 조회하는 부분과 저장하는 부분이 분리되어 있기 때문에 저장하기 전에 다른 클라이언트가 key를 조회할 수 있다는 문제가 있다.

그러므로 조회와 저장이 하나의 단위 즉, 트랜잭션으로 묶여야 의도한대로 수행할 수 있다.

-------

## Redis의 트랜잭션
NoSql에서의 트랜잭션은 RDBMS의 트랜잭션과 무엇이 다를까?

보통 데이터베이스의 트랜잭션이 안전하게 수행된다는 것을 보장하기 위해 ACID(원자성, 일관성, 고립성, 지속성) 라는 약어로 특성을 표현한다.

MySQL같은 RDBMS는 ACID 특성을 모두 지키는 반면 NoSQL은 ACID를 보장하지 않거나, 일부만 보장한다. 

그렇다면 Redis는 ACID한가?

Redis 같은 경우에는 원자성, 일관성, 고립성을 보장하고, 지속성은 일부 지원한다고 볼 수 있다.

Redis는 메인 스레드 1개에서 사용자 명령어를 처리하기 때문에 싱글 스레드 기반이라고 보통 하고(시스템 명령 등을 처리하는 다른 스레드들이 존재하기는 한다), 싱글 스레드이기 때문에 원자성, 일관성, 고립성이 보장되어
데이터를 일관성 있게 유지할 수 있다.

지속성의 경우에는 Redis가 데이터를 디스크에 특정 주기를 기반으로 flush 하기 때문에 중간에 문제가 생긴 경우 가장 마지막 flush 이후에 생긴 변화는 잃어버리기 때문에 보장되지 않는데,

`appendonly` 옵션을 통해서 지속성을 올릴 수 있다고 한다. (대신 성능이 안좋아진다고 한다)

-------
## Redis의 트랜잭션 명령어
MySQL에 트랜잭션 명령어로 START TRANSACTION, COMMIT, ROLLBACK 이 있다고 한다면, Redis에는 다음의 [트랜잭션 명령어](https://redis.io/docs/manual/transactions/)들이 있다.

- Multi: 트랜잭션 시작 

- Exec: 트랜잭션 종료, Multi 이후 모든 명령어들을 실행

- Discard: 트랜잭션 취소, Multi 이후 모든 명령어들을 삭제

- Watch: Exec나 Discard 실행 전에 모니터링 중인 키의 변경사항이 있다면 해당 Multi/Exec 블록이 수행되지 않도록 함

- Unwatch: Watch의 처리를 중단

간단하게 다음과 같이 Multi와 Exec를 사용하는 예시를 보자. foo와 bar를 increment하는 명령어들을 하나의 트랜잭션으로 묶어서 처리하고 있다.
```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC

1) (integer) 1
2) (integer) 1
```
이렇게 Multi 이후의 명령들을 queue에 들어가게 된다.
그리고, Exec 명령이 실행되면 queue의 모든 명령들이 실행되고 트랜잭션이 종료되고,
Discard 명령이 실행되면 queue의 명령들을 비우고 트랜잭션을 종료한다.

<p class="message">
주의!  
Redis에서 트랜잭션은 일부 실패하더라도 롤백이 되지 않는다는 점이다. Redis는 성능과 단순성에 초점을 맞춘 데이터베이스이므로 롤백을 지원하지 않는다.
그래서 중간에 한 명령이 실패하더라도, 큐에 있는 다른 명령들은 그대로 수행된다.
</p>

롤백을 지원하지는 않지만 대신 Watch 명령을 사용해서, 트랜잭션에서 참조하는 값에 대해 변화가 있는지 체크하고, 있다면 명령을 수행하지 않도록하는 로직을 추가할 수 있다.

다음과 같이 키를 조회하고, 값을 1 증가시킨 후 저장하는 로직을 생각해보자.
```
val = GET mykey
val = val + 1
SET mykey $val
```
위의 방식을 사용하면, 두 클라이언트가 동시에 명령을 수행하는 경우에 앞서 언급한 동시성 문제와 비슷하게 경쟁 상태에 빠지게 된다.

이런 경우에 대신, Watch를 사용해줄 수 있다.

```
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```

이렇게 Watch를 사용하면, Watch와 Exec 사이에 다른 클라이언트가 mykey의 값을 변경하면 트랜잭션이 실패한다.
트랜잭션이 실패할 경우, 경쟁 상태가 일어나지 않기를 바라며 해당 명령들을 다시 수행해주면 된다. 

-------
## Lua Script
위의 트랜잭션 명령어를 사용하면 가장 처음에 예시로 들었던 동시성 문제를 해결할 수 있을까?
```kotlin
val value = redisRepository.find(key) // key 조회

if (value == null) {
    redisRepository.save(key, newValue) // 해당 키에 저장된 값이 없으면 새로 저장
} else {
    redisRepoitory.delete(key) // 해당 키에 저장된 값이 있으면 해당 키 삭제
}
```
해당 코드에서는 value를 조회해서, 값에 따라 저장을 하거나 삭제를 하는 조건 로직이 들어가 있다.
Redis에서 트랜잭션을 지원하기는 하지만, Get을 한 결과 값에 따라서 다르게 처리하는 조건 로직까지 Multi-Exec 안에 넣을 수는 없는 것 같다. 

스택오버플로우에도 비슷한 [질문](https://stackoverflow.com/questions/50309206/redis-atomic-get-and-conditional-set)이 있었는데, 답변에서 Lua Script 사용을 권장했다.

Redis 2.6.0부터 사용할 수 있는 기능인데, Lua script에 명령들을 작성하고 Redis에 내장된 Lua 인터프리터에서 해당 script의 명령어들을 처리하여 수행하게 할 수 있다.
여러가지 명령어들을 조합하고 복잡한 로직을 추가할 수 있을 뿐만 아니라, 전체 스크립트 자체가 하나의 명령어로 해석되기 때문에 atomic하게 처리된다.

Lua Script를 작성해서 문제를 해결하지는 않았어서, 해당 방식에 대해 더 궁금하신 분들은 [Redis 문서](https://redis.io/docs/manual/programmability/eval-intro/)를 참고하시면 좋을 것 같다.

-------

다음 글에서는 Spring 어플리케이션에서 어떻게 Redis 트랜잭션을 이용할 수 있는지, 그리고 결국 동시성 문제는 어떻게 해결했는지에 대해서 더 알아보겠다.