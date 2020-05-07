# Intro
***Lettuce***는 Redis Java 클라이언트이다. Redis의 데이터 구조, pub/sub 메세징 및 고가용성 연결을 비롯하여 
Redis API의 동기/비동기 통신을 모두 지원한다.
# Why Lettuce?
기존 제공되던 Jedis와의 큰 차이점은 ***Java 8의 CompletionStage*** 인터페이스를 통한 비동기 지원과 Reactive Streams이다. 또한, Netty를 사용하여 서버와 통신한다.
# Dependency
> 최신 버전의 라이브러리는 아래 링크에서 확인
>
> [***Github repository***](https://github.com/lettuce-io/lettuce-core) or [***Maven Central***](https://search.maven.org/classic/#search%7Cgav%7C1%7Cg%3A%22io.lettuce%22%20AND%20a%3A%22lettuce-core%22)
## 1. Gradle
~~~groovy
compile 'io.lettuce:lettuce-core:5.3.0.RELEASE'
~~~
## 2. Maven
~~~xml
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>5.3.0.RELEASE</version>
</dependency>
~~~
# Connections
## 1. 서버 연결
Redis에 연결하는 과정은 4단계로 구성된다.
1. Redis URI 생성
2. URI를 사용하여 RedisClient에 연결
3. Opening Redis Connection
4. Redis Command Set 생성
~~~java
RedisClient redisClient = RedisClient.create("redis://password@localhost:6379/");
StatefulRedisConnection<String, String> connection = redisClient.connect();
~~~
***StatefulRedisConnection***은 Thread-safe 하게 Redis 서버에 연결하고, Command를 동기 또는 비동기로 실행할 수 있다. 

***RedisClient***는 Redis 서버와 통신하기 위해 Netty를 사용하여 상당한 시스템 리소스를 사용하므로 다중 연결이 필요한 경우는 단일 RedisClient를 사용해야 한다.
## 2. Redis URI
static factory 메소드에 URI를 전달하여 RedisClient를 생성한다.

### 스키마
~~~
# 기본 구문
redis :// [password@] host [: port] [/ database]
  [? [timeout=timeout[d|h|m|s|ms|us|ns]]
  [&_database=database_]]
~~~
- redis - standalone Redis
- rediss - SSL 연결을 통한 standalone Redis
- redis-socket - 유닉스 도메인 소켓을 통한 standalone Redis
- redis-sentinel - Redis Sentinel 서버
### Builder
~~~java
RedisURI.Builder
    .redis("localhost", 6379).auth("password")
    .database(1)
    .build();
~~~
### Constructor
~~~java
new RedisURI("localhost", 6379, 60, TimeUnit.SECONDS);
~~~
## 3. Sync commands
Connection 생성 후 command 셋트를 생성
~~~java
RedisCommands<String, String> syncCommands = connection.sync();
syncCommands.set("key", "Hello, Redis!"); // 키-값 설정
String value = syncommands.get(“key”);    // 값 얻기 by key

// Hash 사용
syncCommands.hset("recordName", "FirstName", "John");
syncCommands.hset("recordName", "LastName", "Smith");
Map<String, String> record = syncCommands.hgetall("recordName");
~~~
## 4. Async commands
~~~java
RedisAsyncCommands<String, String> asyncCommands = connection.async();

// CompletableFuture 기반의 RedisFuture를 반환한다.
RedisFuture<String> result = asyncCommands.get("key");
~~~
[***CompletableFuture 참고***](https://www.baeldung.com/java-completablefuture)

## 5. Reactive API
> Reactive Command 셋트 생성
> 해당 Command 셋트로 명령 실행 시 Mono 또는 Flux로 래핑된 결과를 반환
~~~java
RedisStringReactiveCommands<String, String> reactiveCommands = connection.reactive();
~~~
[***Project Reactor 참고***](https://github.com/reactor/reactor-core)
# Redis Data Structures
## 1. Lists
> 삽입 순서가 유지되는 리스트 타입, 값은 양쪽 끝에서 삽입되거나 검색된다.
~~~java
asyncCommands.lpush("tasks", "firstTask");
asyncCommands.lpush("tasks", "secondTask");
RedisFuture<String> redisFuture = asyncCommands.rpop("tasks");
 
String nextTask = redisFuture.get();

asyncCommands.del("tasks");
asyncCommands.lpush("tasks", "firstTask");
asyncCommands.lpush("tasks", "secondTask");
redisFuture = asyncCommands.lpop("tasks");
 
String nextTask = redisFuture.get();
~~~
## 2. Sets
> Java의 Set과 비슷한 순서가 없는 타입, 중복이 없음
~~~java
asyncCommands.sadd("pets", "dog");
asyncCommands.sadd("pets", "cat");
asyncCommands.sadd("pets", "cat");
  
RedisFuture<Set<String>> pets = asyncCommands.smembers("nicknames");
RedisFuture<Boolean> exists = asyncCommands.sismember("pets", "dog");
~~~
## 3. Hashes
> 문자열 필드(키)와 값이 있는 레코드로 각 레코드에는 기본 인덱스 키가 있다.
~~~java
asyncCommands.hset("recordName", "FirstName", "John");
asyncCommands.hset("recordName", "LastName", "Smith");
 
RedisFuture<String> lastName = syncCommands.hget("recordName", "LastName");
RedisFuture<Map<String, String>> record = syncCommands.hgetall("recordName");
~~~
## 4. Sorted Sets
> 값과 순위가 정렬되고, 순위는 64 비트 부동 소수점 값을 가진다.
> 항목이 순위와 함께 추가되고, 범위로 검색된다.
~~~java
asyncCommands.zadd("sortedset", 1, "one");
asyncCommands.zadd("sortedset", 4, "zero");
asyncCommands.zadd("sortedset", 2, "two");
 
RedisFuture<List<String>> valuesForward = asyncCommands.zrange(key, 0, 3);
RedisFuture<List<String>> valuesReverse = asyncCommands.zrevrange(key, 0, 3);
~~~
# Transactions
> 트랜잭션을 통해 일련의 명령을 Atomic 하게 실행할 수 있다.
> 명령 셋트 중 하나가 실패하더라도 Rollback은 되지 않는다.
~~~java
asyncCommands.multi();
     
RedisFuture<String> result1 = asyncCommands.set("key1", "value1");
RedisFuture<String> result2 = asyncCommands.set("key2", "value2");
RedisFuture<String> result3 = asyncCommands.set("key3", "value3");
 
RedisFuture<TransactionResult> execResult = asyncCommands.exec();
 
TransactionResult transactionResult = execResult.get();
 
String firstResult = transactionResult.get(0);
String secondResult = transactionResult.get(0);
String thirdResult = transactionResult.get(0);
~~~
# References
- https://www.baeldung.com/java-redis-lettuce