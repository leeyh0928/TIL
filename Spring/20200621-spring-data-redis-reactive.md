# Overview
Spring Data의 `ReactiveRedisTemplate`을 사용하여 Redis 작업을 구성하고, 구현하는 방법을 알아본다.

# Dependency
~~~groovy
dependencies {
    compile('org.springframework.boot:spring-boot-starter-data-redis-reactive')
}
~~~

# Configuration
ConnectionFactory Bean 정
~~~java
@Bean
public ReactiveRedisConnectionFactory reactiveRedisConnectionFactory() {
    return new LettuceConnectionFactory(host, port);
}
~~~

# List Operations
## 1. String Template
String 직렬화 컨텍스트를 제공하는 ReactiveRedisTemplate 빈 정의
~~~java
@Bean
public ReactiveRedisTemplate<String, String> reactiveRedisTemplateString
  (ReactiveRedisConnectionFactory connectionFactory) {
    return new ReactiveRedisTemplate<>(connectionFactory, RedisSerializationContext.string());
}
~~~
ReactiveListOperations 인스턴스 생성하기
~~~java
@Autowired
private ReactiveRedisTemplate<String, String> redisTemplate;
 
private ReactiveListOperations<String, String> reactiveListOps;
 
@Before
public void setup() {
    reactiveListOps = redisTemplate.opsForList();
}
~~~
## 2. LPUSH and LPOP
ReactiveListOperations 인스턴스 활용하여 목록에 대해 LPush 또는 LPop 해본다.

Reactive 구성 요소를 테스트할 경우 `StepVerifier`를 사용하여 task가 종료되는 것을 막을 수 있다.
~~~java
@Test
public void givenListAndValues_whenLeftPushAndLeftPop_thenLeftPushAndLeftPop() {
    Mono<Long> lPush = reactiveListOps.leftPushAll(LIST_NAME, "first", "second")
      .log("Pushed");
 
    StepVerifier.create(lPush)
      .expectNext(2L)
      .verifyComplete();
 
    Mono<String> lPop = reactiveListOps.leftPop(LIST_NAME)
      .log("Popped");
 
    StepVerifier.create(lPop)
      .expectNext("second")
      .verifyComplete();
}
~~~
# Value Operations
문자열 대신 커스텀한 객체 생성하여 사용하기
~~~java
public class Employee implements Serializable {
    private String id;
    private String name;
    private String department;
 
    // ... getters and setters
 
    // ... hashCode and equals
}
~~~
## 1. Employee Template
~~~java
@Bean
public ReactiveRedisTemplate<String, Employee> reactiveRedisTemplate(
  ReactiveRedisConnectionFactory factory) {
 
    StringRedisSerializer keySerializer = new StringRedisSerializer();
    Jackson2JsonRedisSerializer<Employee> valueSerializer =
      new Jackson2JsonRedisSerializer<>(Employee.class);
    RedisSerializationContext.RedisSerializationContextBuilder<String, Employee> builder =
      RedisSerializationContext.newSerializationContext(keySerializer);
    RedisSerializationContext<String, Employee> context = 
      builder.value(valueSerializer).build();
 
    return new ReactiveRedisTemplate<>(factory, context);
}
~~~
## 2. Save and Retrieve Operations
~~~java
@Autowired
private ReactiveRedisTemplate<String, Employee> redisTemplate;
 
private ReactiveValueOperations<String, Employee> reactiveValueOps;
 
@Before
public void setup() {
    reactiveValueOps = redisTemplate.opsForValue();
}

@Test
public void givenEmployee_whenSet_thenSet() {
 
    Mono<Boolean> result = reactiveValueOps.set("123", 
      new Employee("123", "Bill", "Accounts"));
 
    StepVerifier.create(result)
      .expectNext(true)
      .verifyComplete();
}

@Test
public void givenEmployeeId_whenGet_thenReturnsEmployee() {
 
    Mono<Employee> fetchedEmployee = reactiveValueOps.get("123");
 
    StepVerifier.create(fetchedEmployee)
      .expectNext(new Employee("123", "Bill", "Accounts"))
      .verifyComplete();
}
~~~

## 3. Operations With Expiry Time
~~~java
@Test
public void givenEmployee_whenSetWithExpiry_thenSetsWithExpiryTime() 
  throws InterruptedException {
 
    Mono<Boolean> result = reactiveValueOps.set("129", 
      new Employee("129", "John", "Programming"), 
      Duration.ofSeconds(1));
 
    StepVerifier.create(result)
      .expectNext(true)
      .verifyComplete();
 
    Thread.sleep(2000L); 
 
    Mono<Employee> fetchedEmployee = reactiveValueOps.get("129");
    StepVerifier.create(fetchedEmployee)
      .expectNextCount(0L)
      .verifyComplete();
}
~~~
# Redis Commands
## 1. String and Key Commands
Redis 명령 작업을 수행하기 위해 `ReactiveKeyCommands` 및 `ReactiveStringCommands`의 Bean 정의
~~~java
@Bean
public ReactiveKeyCommands keyCommands(ReactiveRedisConnectionFactory 
  reactiveRedisConnectionFactory) {
    return reactiveRedisConnectionFactory.getReactiveConnection().keyCommands();
}
 
@Bean
public ReactiveStringCommands stringCommands(ReactiveRedisConnectionFactory 
  reactiveRedisConnectionFactory) {
    return reactiveRedisConnectionFactory.getReactiveConnection().stringCommands();
}
~~~
## 2. Set and Get Operations
~~~java
@Test
public void givenFluxOfKeys_whenPerformOperations_thenPerformOperations() {
    Flux<SetCommand> keys = Flux.just("key1", "key2", "key3", "key4");
      .map(String::getBytes)
      .map(ByteBuffer::wrap)
      .map(key -> SetCommand.set(key).value(key));
 
    StepVerifier.create(stringCommands.set(keys))
      .expectNextCount(4L)
      .verifyComplete();
 
    Mono<Long> keyCount = keyCommands.keys(ByteBuffer.wrap("key*".getBytes()))
      .flatMapMany(Flux::fromIterable)
      .count();
 
    StepVerifier.create(keyCount)
      .expectNext(4L)
      .verifyComplete();
}
~~~

# References
* https://www.baeldung.com/spring-data-redis-reactive