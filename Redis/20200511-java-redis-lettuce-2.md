# Intro
지난번에 공부한 ***Lettuce***에 대해 빠진 부분을 이어서 정리한다.
# Batching
결과가 즉지 필요하지 않거나 대량의 데이터를 업로드하는 경우 사용한다.
~~~java
// setAutoFlushCommands를 false로 설정(flushCommands를 수동 실행)하여
// 큐에 적재하고 즉시 실행하지 않는다.
commands.setAutoFlushCommands(false);
 
List<RedisFuture<?>> futures = new ArrayList<>();
for (int i = 0; i < iterations; i++) {
    futures.add(commands.set("key-" + i, "value-" + i);
}
// flushCommands를 실행하여 채널을 flush하고 명령을 수행할 수 있도록 한다. 
commands.flushCommands();

// awaitAll을 통해 RedisFuture가 완료될 때까지 기다린다. 
boolean result = LettuceFutures.awaitAll(5, TimeUnit.SECONDS,
  futures.toArray(new RedisFuture[0]));
~~~
# Publish/Subscribe
Redis 데이터에 대한 알림을 제공을 통해 클라이언트는 키 설정, 삭제, 만료 등의 이벤트를 수신할 수 있음

[전체 Spec 보기](https://redis.io/topics/notifications)

## 1. Subscriber
### Define Listener
~~~java
public class Listener implements RedisPubSubListener<String, String> {
 
    @Override
    public void message(String channel, String message) {
        log.debug("Got {} on channel {}",  message, channel);
    }
}
~~~
### Install Listener
~~~java
StatefulRedisPubSubConnection<String, String> connection = client.connectPubSub();
connection.addListener(new Listener())
 
RedisPubSubAsyncCommands<String, String> async = connection.async();
async.subscribe("channel");
~~~
## 2. Publisher
publishing을 위해 Pub/Sub 채널을 연결하고 명령을 검색하기 만하면 됨.
~~~java
StatefulRedisPubSubConnection<String, String> connection = client.connectPubSub();

RedisPubSubAsyncCommands<String, String> async = connection.async();

async.publish("channel", "Hello, Redis!");
~~~
## 3. Reactive Subscriptions
Pub/Sub 메시지의 구독을 위한 reactive interface를 제공함.
~~~java
StatefulRedisPubSubConnection<String, String> connection = client.connectPubSub();
 
RedisPubSubAsyncCommands<String, String> reactive = connection.reactive();
 
reactive.observeChannels().subscribe(message -> log.debug("Got {} on channel {}",  message, channel));
reactive.subscribe("channel").subscribe();
~~~

# 고가용성
## 1. Master/Slave
* Redis 서버는 마스터/슬레이브 구성으로 스스로를 복제. 
* 슬레이브로 마스터 캐시를 복제하는 명령 스트림을 실행
* Redis는 양방향 복제를 지원하지 않으므로, Slave는 읽기 전용
> Lettuce는 Master/Slave 시스템에 연결하여 토폴로지를 쿼리 한 다음 읽기 작업을 위해 슬레이브를 선택하여 처리량을 향상시킬 수 있다.
~~~java
RedisClient redisClient = RedisClient.create();
 
StatefulRedisMasterSlaveConnection<String, String> connection
 = MasterSlave.connect(redisClient, 
   new Utf8StringCodec(), RedisURI.create("redis://localhost"));
  
connection.setReadFrom(ReadFrom.SLAVE);
~~~
## 2. Sentinel
* 마스터 및 슬레이브 인스턴스를 모니터링하고 마스터 페일 오버시 슬레이브에 대한 페일 오버를 오케스트레이션
* Lettuce는 Sentinel에 연결하여 현재 마스터의 주소를 찾은 다음 연결을 반환할 수 있음

[전체 Spec 보기](https://redis.io/topics/sentinel)

~~~java
RedisURI redisUri = RedisURI.Builder
    .sentinel("sentinelhost1", "clustername")
    .withSentinel("sentinelhost2")
    .build();

RedisClient client = new RedisClient(redisUri);
RedisConnection<String, String> connection = client.connect();
~~~
## 3. Clusters
* 분산 구성을 사용하여 고 가용성 및 처리량을 제공하는 방식
* 최대 1000 개의 노드에 샤드 키를 클러스터하므로 트랜잭션을 사용할 수 없음
> RedisAdvancedClusterCommands는 클러스터에서 지원하는 Redis 명령 세트를 보유하고, 이에 대한 키를 보유한 인스턴스로 라우팅한다.

[전체 Spec 보기](https://redis.io/topics/cluster-spec)

~~~java
RedisURI redisUri = RedisURI.Builder
    .redis("localhost")
    .withPassword("authentication").build();

RedisClusterClient clusterClient = RedisClusterClient.create(rediUri);
StatefulRedisClusterConnection<String, String> connection = clusterClient.connect();

RedisAdvancedClusterCommands<String, String> syncCommands = connection.sync();
~~~

# References
- https://www.baeldung.com/java-redis-lettuce