# Overview
***WebClient***는 웹 요청을 수행하기 위한 기본 Entry Point를 나타내는 인터페이스이다. `Spring Web Reactive`모듈의 일부로 
작성되었으며 기존 `RestTemplate`을 대체한다. HTTP/1.1 프로토콜을 통해 작동하는 ***reactive하고, non-blocking***한 솔루션이다.

# Dependencies
***Reactor 프로젝트***와 ***spring-boot-starter-webflux***에 대한 종속성을 가진다.
~~~groovy
dependencies {
    compile 'org.springframework.boot:spring-boot-starter-webflux'
    compile 'org.projectreactor:reactor-spring:1.0.1.RELEASE'
}
~~~

# Working With the WebClient
## 1. Creating a WebClient Instance
세가지 옵션 중에 선택할 수 있다. 첫번째는 기본 설정으로 ***WebClient*** 객체를 만드는 것이다.
~~~java
WebClient client1 = WebClient.create();
~~~
두 번째는 주어진 기본 URI 로 ***WebClient*** 객체를 생성하는 것이다.
~~~java
WebClient client2 = WebClient.create("http://localhost:8080");
~~~
마지막 방법(가장 진보 된 방법)은 DefaultWebClientBuilder 클래스를 사용하여 생성하는 것이다.
~~~java
WebClient client3 = WebClient
  .builder()
    .baseUrl("http://localhost:8080")
    .defaultCookie("cookieKey", "cookieValue")
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE) 
    .defaultUriVariables(Collections.singletonMap("url", "http://localhost:8080"))
  .build();
~~~

## 2. Creating a WebClient Instance with Timeouts
종종 30초의 기본 HTTP 타임아웃은 우리의 요구사항에 비해 너무 느리다. ***WebClient***는 `ChannelOption.CONNECT_TIMEOUT_MILLIS` 값을 
통해 `연결 시간 초과`를 설정할 수 있고, `ReadTimeoutHandler 및 WriteTimeoutHandler`를 각각 사용하여 `읽기 및 쓰기 시간 초과`를
설정할 수 있다.
~~~java
TcpClient tcpClient = TcpClient
  .create()
  .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
  .doOnConnected(connection -> {
      connection.addHandlerLast(new ReadTimeoutHandler(5000, TimeUnit.MILLISECONDS));
      connection.addHandlerLast(new WriteTimeoutHandler(5000, TimeUnit.MILLISECONDS));
  });
 
WebClient client = WebClient.builder()
  .clientConnector(new ReactorClientHttpConnector(HttpClient.from(tcpClient)))
  .build();
~~~

## 3. Preparing a Request
메소드에 ***HttpMethod***를 지정하거나, ***get, post, delete***와 같은 단축 메소드를 호출하여 요청해야 한다.
~~~java
WebClient.UriSpec<WebClient.RequestBodySpec> request1 = client3.method(HttpMethod.POST);
WebClient.UriSpec<WebClient.RequestBodySpec> request2 = client3.post();
~~~
다음으로 URL을 제공하는 것이다. ***uri API***에 ***String*** 또는 ***java.net.URL*** 객체로 전달할 수 있다.
~~~java
WebClient.RequestBodySpec uri1 = client3
  .method(HttpMethod.POST)
  .uri("/resource");
 
WebClient.RequestBodySpec uri2 = client3
  .post()
  .uri(URI.create("/resource"));
~~~
필요한 경우 요청 본문, 콘텐츠 유형, 길이, 쿠키 또는 헤더를 설정할 수 있다.
예를 들어 요청 본문을 설정하려면 ***BodyInserter***로 요청을 채우거나 이 작업을 ***Publisher***에게 위임하는 두 가지 방법이 있다.
~~~java
WebClient.RequestHeadersSpec<?> requestSpec1 = WebClient
  .create("http://localhost:8080")
  .post()
  .uri(URI.create("/resource"))
  .body(BodyInserters.fromObject("data"));

WebClient.RequestHeadersSpec requestSpec2 = WebClient
  .create()
  .method(HttpMethod.POST)
  .uri("/resource")
  .body(BodyInserters.fromPublisher(Mono.just("data")), String.class);
~~~
***BodyInserter***에 데이터를 채우는 프로세스에 유용한 유틸리티 메소드를 제공하는 ***BodyInserters*** 클래스가 있다.
~~~java
BodyInserter<Publisher<String>, ReactiveHttpOutputMessage> inserter1 = BodyInserters
  .fromPublisher(Subscriber::onComplete, String.class);
~~~
***MultiValueMap***도 사용 가능하다.
~~~java
LinkedMultiValueMap map = new LinkedMultiValueMap();
 
map.add("key1", "value1");
map.add("key2", "value2");
 
BodyInserter<MultiValueMap, ClientHttpRequest> inserter2
 = BodyInserters.fromMultipartData(map);
~~~
단일 객체를 사용하는 방법으로는 다음과 같다.
~~~java
BodyInserter<Object, ReactiveHttpOutputMessage> inserter3 = BodyInserters.fromObject(new Object());
~~~
본문을 설정한 후 헤더, 쿠키, 허용 가능한 미디어 유형 등을 설정할 수 있다. ***WebClient*** 인스턴스화 할 때 설정된 값에 값이 추가된다.
~~~java
WebClient.ResponseSpec response1 = uri1
  .body(inserter3)
    .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .accept(MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML)
    .acceptCharset(Charset.forName("UTF-8"))
    .ifNoneMatch("*")
    .ifModifiedSince(ZonedDateTime.now())
  .retrieve();
~~~

## 4. Getting a Response
요청을 보내고 응답을 받는 방법으로 ***exchange*** 또는 ***retrieve*** 메소드를 사용하여 수행할 수 있다.
두 방식의 반환 타입이 다른데, ***exchange***는 상태, 헤더와 함께 ***ClientResponse***를 제공하고, ***retrieve***는
본문을 직접 가져오는 가장 빠른 방법이다.
~~~java
String response2 = request1.exchange()
  .block()
  .bodyToMono(String.class)
  .block();

String response3 = request2
  .retrieve()
  .bodyToMono(String.class)
  .block();
~~~
***bodyToMono*** 메소드는 상태 코드가 4xx(클라이언트 오류) 또는 5xx(서버 오류)인 경우 ***WebClientException***이 발생한다.
***Monos***의 ***block*** 메소드를 사용하여 응답과 함께 전송 된 실제 데이터를 구독하고 검색할 수 있다.

# Working With the WebTestClient
`WebTestClient`는 ***WebFlux*** 서버 엔드 포인트를 테스트하기 위한 주 진입 점이다. ***WebClient***와 매우 유사한 API를 가지고 있으며 
대부분의 작업을 주로 테스트 컨텍스트 제공에 중점을 둔 ***내부 WebClient*** 인스턴스에 위임한다.

테스트 용 클라이언트는 실제 서버에 바인딩되거나 특정 컨트롤러 또는 기능과 작동 할 수 있다. 실행 중인 서버에 대한 실제 요청으로 
엔드 투 엔드 통합 테스트를 수행하기 위해 ***bindToServer*** 메소드를 사용할 수 있다.
~~~java
WebTestClient testClient = WebTestClient
  .bindToServer()
  .baseUrl("http://localhost:8080")
  .build();
~~~
특정 ***RouterFunction***을 ***bindToRouterFunction*** 메소드에 전달하여 테스트 할 수 있다.
~~~java
RouterFunction function = RouterFunctions.route(
  RequestPredicates.GET("/resource"),
  request -> ServerResponse.ok().build()
);
 
WebTestClient
  .bindToRouterFunction(function)
  .build().get().uri("/resource")
  .exchange()
  .expectStatus().isOk()
  .expectBody().isEmpty();
~~~
***WebHandler*** 인스턴스를 사용하는 ***bindToWebHandler*** 메소드를 사용하여 동일한 동작을 수행 할 수 있다.
~~~java
WebHandler handler = exchange -> Mono.empty();

WebTestClient.bindToWebHandler(handler).build();
~~~
***bindToApplicationContext*** 메소드에 ***applicationContext***를 전달하면 컨트롤러 Bean 및 @EnableWebFlux가 구성에 대한
컨텍스트를 분석한다.
~~~java
@Autowired
private ApplicationContext context;
 
WebTestClient testClient = WebTestClient.bindToApplicationContext(context)
  .build();
~~~

더 나아가서 ***bindToController*** 메소드로 테스트하려는 컨트롤러 배열을 제공하는 것이다. Controller 클래스가 있고 이를 필요한 클래스에 
주입 했다고 가정하면 다음과 같이 작성할 수 있다.
~~~java
@Autowired
private Controller controller;
 
WebTestClient testClient = WebTestClient.bindToController(controller).build();
~~~
***WebTestClient***는 모든 작업이 ***WebClient***와 유사하며, ***expectStatus, expectBody, expectHeader***와 같은 유용한 메소드와 함께
작동 하도록 ***WebTestClient.ResponseSpec*** 인터페이스를 제공한다.
~~~java
WebTestClient
  .bindToServer()
    .baseUrl("http://localhost:8080")
    .build()
    .post()
    .uri("/resource")
  .exchange()
    .expectStatus().isCreated()
    .expectHeader().valueEquals("Content-Type", "application/json")
    .expectBody().isEmpty();
~~~

# References
* https://www.baeldung.com/spring-5-webclient