# Overview
데이터 파이프라인 에서 `exactly-once`처리는 매우 중요하다. 카프카는 `0.11.0.0` 이후 버전에 대해서 `exactly-once`를 위한 
`transaction 처리`를 지원한다.

여기서 말하는 `exactly-once delivery`는 프로듀서부터 컨슈머까지 연결되는 파이프라인의 처리를 뜻한다. 기존 프로듀서의 경우
트랜잭션 처리를 하지 않으면 카프카 클러스터에 두번 이상 데이터가 저장 될 수 있다. ***데이터가 클러스터에 저장되었으나 ack가
유실되어 프로듀서가 재처리하는 경우가 대표적***이다.

결과적으로 카프카 트랜잭션 처리를 하더라도 ***컨슈머가 중복해서 데이터 처리하는 것에 대해 보장하지 않으므로, 컨슈머의 중복처리는
따로 로직을 작성***해야 한다.

# 카프카 트랜잭션 적용
카프카 트랜잭션은 프로듀서와 컨슈머 단에서 옵션을 설정해야 한다. 상세 코드는 아래와 같다.

> 프로듀서
~~~java
public class ProducerWithTransaction {
    private final static String TOPIC_NAME = "test";
    private final static String BOOTSTRAP_SERVERS = "kafka:9092";
    private final static String TRANSACTION_ID_CONFIG = UUID.randomUUID().toString(); // 트랜잭션 처리하는 프로듀서 구분자

    public static void main(String[] args) throws Exception{
        Properties configs = new Properties();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        
        configs.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, TRANSACTION_ID_CONFIG); // 트랜잭션 구분자 설정
        configs.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true"); // 트랜잭션 처리 설정

        KafkaProducer<String, String> producer = new KafkaProducer<>(configs);
        ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "data");

        producer.initTransactions(); // 트랜잭션 준비
        producer.beginTransaction(); // 트랜잭션 시작

        producer.send(record);
        
        producer.flush();

        producer.commitTransaction(); // 트랜잭션 커밋
        producer.close();
    }
}
~~~
프로듀서는 브로커로 데이터를 전송할 때 트랜잭션을 커밋하는 주체이다. 프로듀서는 트랜잭션 처리를 하기 위해 반드시
`TRANSACTIONAL_ID_CONFIG`와 `ENABLE_IDEMPOTENCE_CONFIG` 설정을 해야 한다.
- TRANSACTIONAL_ID_CONFIG : 프로듀서 트랜잭션을 구분하기 위한 ID(중복방지를 위해 UUID 사용)
- ENABLE_IDEMPOTENCE_CONFIG : 트랜잭션 처리를 한다고 명시적으로 설정

최초에 ***initTransactions()*** 메소드로 트랜잭션 처리를 위한 준비를 하고 ***send()*** 메서드 앞뒤로 ***beginTransaction()***,
***commitTransaction()*** 메서드를 선언함으로서 트랜잭션을 마친다. 만약 트랜잭션을 commit 하지 않고 프로듀서를 close 하면
***close()*** 구문에서 자동으로 commit 하고 종료한다. 또한 프로듀서가 close 하지 않고, 애플리케이션이 종료되면 
일정 시간(transaction.timeout.ms) 이후에 abort 처리된다. abort 처리된 데이터는 트랜젝션 옵션이 활성화된 컨슈머가 가져가지 못한다.

프로듀서가 데이터를 보내면 아래와 같이 데이터가 브로커에 적재되는 것을 스크립트를 통해 확인할 수 있다.
~~~bash
$ echo "exclude.internal.topics=false" > consumer.config
$ ./kafka-console-consumer.sh --consumer.config consumer.config --formatter "kafka.coordinator.transaction.TransactionLog\$TransactionLogMessageFormatter" --bootstrap-server localhost:9092 --topic __transaction_state --from-beginning

13c2df10-1a3c-4024-b28d-d155e24b941a::TransactionMetadata(transactionalId=13c2df10-1a3c-4024-b28d-d155e24b941a, producerId=1000, producerEpoch=28, txnTimeoutMs=60000, state=Ongoing, pendingState=None, topicPartitions=Set(test-1), txnStartTimestamp=1594967312702, txnLastUpdateTimestamp=1594967312702)
13c2df10-1a3c-4024-b28d-d155e24b941a::TransactionMetadata(transactionalId=13c2df10-1a3c-4024-b28d-d155e24b941a, producerId=1000, producerEpoch=28, txnTimeoutMs=60000, state=PrepareCommit, pendingState=None, topicPartitions=Set(test-1), txnStartTimestamp=1594967312702, txnLastUpdateTimestamp=1594967312712)
13c2df10-1a3c-4024-b28d-d155e24b941a::TransactionMetadata(transactionalId=13c2df10-1a3c-4024-b28d-d155e24b941a, producerId=1000, producerEpoch=28, txnTimeoutMs=60000, state=CompleteCommit, pendingState=None, topicPartitions=Set(), txnStartTimestamp=1594967312702, txnLastUpdateTimestamp=1594967312713)
~~~
***__transaction_state***는 프로듀서가 보내는 데이터의 트랜젝션을 기록하는데 사용한다. 이 데이터는 트랜잭션 옵션이 활성화 된 컨슈머가 데이터를 가져가는데 사용한다.
위 데이터를 보면 `Ongoing -> PrepareCommit -> CompleteCommit` 순서대로 트랜잭션이 순차적으로 진행된 모습을 확인할 수 있다. 이렇게 트랜잭션이 완료된 데이터만 컨슈머가 가져갈 수 있다.

트랜젝션을 활성화할 경우 프로듀서는 기존에 사용하던 토픽에 추가적으로 데이터를 넣는다. 상세 내용을 살펴보자.
~~~bash
$ ./kafka-dump-log.sh --files kafka_2.12-2.5.0/data/test-1/00000000000000000000.log --deep-iteration

baseOffset: 16 lastOffset: 16 count: 1 baseSequence: 0 lastSequence: 0 producerId: 1000 producerEpoch: 21 partitionLeaderEpoch: 0 isTransactional: true isControl: false position: 1680 CreateTime: 1594965813907 size: 122 magic: 2 compresscodec: NONE crc: 3909138376 isvalid: true
| offset: 16 CreateTime: 1594965813907 keysize: -1 valuesize: 54 sequence: 0 headerKeys: []
baseOffset: 17 lastOffset: 17 count: 1 baseSequence: -1 lastSequence: -1 producerId: 1000 producerEpoch: 21 partitionLeaderEpoch: 0 isTransactional: true isControl: true position: 1802 CreateTime: 1594965813942 size: 78 magic: 2 compresscodec: NONE crc: 3102183917 isvalid: true
| offset: 17 CreateTime: 1594965813942 keysize: 4 valuesize: 6 sequence: -1 headerKeys: [] endTxnMarker: COMMIT coordinatorEpoch: 0
~~~
위 스크립트를 통해 카프카 브로커에 적재된 세그먼트 파일을 확인할 수 있다. test토픽의 1번 파티션에 저장된 데이터를 확인했는데, 16번 오프셋과 17번 오프셋이 작성된 것을 알 수 있다.
* 16번 오프셋 - isTransactional: true, isControl: false
* 17번 오프셋 - isTransactional: true, isControl: true,  endTxnMarker: COMMIT

즉, 프로듀서가 트랜젝션을 끝내면 해당 토픽에 COMMIT 메시지를 명시적으로 전달한다. 이로 인해 추가 오프셋이 소모된 것을 알 수 있다.
컨슈머는 이 토픽에서 데이터를 가져가더라도 16번 오프셋만 가져가고 17번 오프셋은 무시한다. ***17번 오프셋은 COMMIT 을 명시하는 
데이터이고 실질적인 프로듀서가 전송하는 값은 16번에만 존재***하기 때문이다.

> 컨슈머
~~~java
public class ConsumerWithSyncCommit {
    private final static Logger logger = LoggerFactory.getLogger(ConsumerWithSyncCommit.class);
    private final static String TOPIC_NAME = "test";
    private final static String BOOTSTRAP_SERVERS = "kafka:9092";
    private final static String GROUP_ID = "test-group";
    private final static String TRANSACTION_CONFIG = "read_committed"; // 트랜잭션 설정 커밋된 데이터만 읽는다.

    public static void main(String[] args) {
        Properties configs = new Properties();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID);
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        
        configs.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // 명시적 오프셋 커밋
        configs.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, TRANSACTION_CONFIG); // 트랜잭션 설정

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);
        consumer.subscribe(Arrays.asList(TOPIC_NAME));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                logger.info("{}", record.toString());
            }
            consumer.commitSync();
        }
    }
}
~~~
컨슈머에서는 오토 커밋을 false로 설정하고 ISOLATION_LEVEL_CONFIG를 read_committed로 설정해야만 한다. 이를 통해 프로듀서가 
브로커로 보낸 데이터 중 트랜잭션이 완벽하게 완료된 데이터에 대해서만 읽을 수 있는 것이다. 위 컨슈머 실행 결과는 아래와 같다.
~~~bash
[main] INFO org.apache.kafka.clients.consumer.ConsumerConfig - ConsumerConfig values: 
	allow.auto.create.topics = true
	auto.commit.interval.ms = 5000
	auto.offset.reset = latest
	bootstrap.servers = [localhost:9092]
	check.crcs = true
	client.dns.lookup = default
	client.id = 
	client.rack = 
	connections.max.idle.ms = 540000
	default.api.timeout.ms = 60000
...
[main] INFO com.example.ConsumerWithSyncCommit - ConsumerRecord(topic = test, partition = 1, leaderEpoch = 0, offset = 16, CreateTime = 1594970489649, serialized key size = -1, serialized value size = 54, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value =data)
~~~
16번 오프셋 데이터만 읽은 것을 확인할 수 있다.

# 결론
카프카 트랜잭션은 ***프로듀서부터 시작하여 컨슈머까지 데이터가 전달될 때 exactly-once를 지원하기 위한 옵션***이다. 
이 옵션은 만능이 아니다. 결과적으로 ***컨슈머 단에서 중복이 생길 수 있기 때문***이다. 다만 프로듀서에서 시작하여 컨슈머까지 
파이프라인에 대해서는 이 옵션을 적용하여 exactly-once 전달을 보장할 수 있게 되었다. 

이 옵션은 `transaction` 관련 내부 브로커 처리를 하기 때문에 옵션을 사용하지 않는 것보다 성능이 다소 떨어질 수 있다는 점은 
주의 하여야 한다.

`Kafka Stream` 의 경우 컨슈머 중복 문제에 해결을 위한 처리 부분이 추가적으로 있는 것 같은데 다음 기회에 다뤄보도록 하겠다.

# References
* https://blog.voidmainvoid.net/354#recentComments  