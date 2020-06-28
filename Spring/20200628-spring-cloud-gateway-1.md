# Overview
Spring5, Spring Boot2 및 Project Reactor를 기반으로 하는 `Spring Cloud Gateway`에 대해 알아본다.
MSA 애플리케이션에서 자주 사용되는 즉시 사용 가능한 라우팅 메커니즘을 제공한다.

# Routing Handler
라우팅 요청에 중점을 둔 `Spring Cloud Gateway`는 요청을 게이트웨이 처리기 매핑으로 전달하여 특정 경로와 일치하는 요청으로
수행할 작업을 결정한다. Gateway Handler가 RouteLocator를 사용하여 라우트 구성을 해결하는 방법에 대한 간단한 예이다.
~~~java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
      .route("r1", r -> r.host("**.baeldung.com")
        .and()
        .path("/baeldung")
        .uri("http://baeldung.com"))
      .route(r -> r.host("**.baeldung.com")
        .and()
        .path("/myOtherRouting")
        .filters(f -> f.prefixPath("/myPrefix"))
        .uri("http://othersite.com")
        .id("myOtherID"))
    .build();
}
~~~
* Route : 게이트웨이의 기본 API. 지정된 식별 (ID), 목적지 (URI) 및 Predicate와 Filter 세트로 정의된다.
* Predicate : Java 8의 Predicate, 헤더, 메소드 또는 매개 변수를 사용하여 HTTP 요청을 일치시키는 데 사용
* Filter : 표준 Spring의 WebFilter

# Dynamic Routing
Zuul과 마찬가지로 Spring Cloud Gateway는 요청을 다른 서비스로 라우팅하는 수단을 제공
~~~yaml
spring:
  application:
    name: gateway-service  
  cloud:
    gateway:
      routes:
      - id: example
        uri: example.com
      - id: myOtherRouting
        uri: localhost:9999
~~~

# Routing Factories
Spring Cloud Gateway는 Spring WebFlux HandlerMapping 인프라를 사용한다. 또한 많은 내장 루트 Predicate가 포함되고,
'and'를 통해 여러 경로의 Predicate Factories 를 결합 할 수 있다.

## 1. Before Route Predicate Factory
아래 예시는 `Before Route`의 Predicate는 현재 시간 이전에 발생한 요청인 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: http://example.com
        predicates:
        - Before=2017-09-11T17:42:47.789-07:00[America/Alaska]
~~~
Java로 구성방법
~~~java
//..route definition
.route(r -> r.before(LocalDateTime.now().atZone(ZoneId.systemDefault())) 
.id("before_route")
.uri("http://test.com")
~~~

## 2. Between Route Predicate Factory
아래 예시의 `Between Route`는 두 DataTime(datetime1<포함>, datetime2<제외>) 사이에 발생한 요청인 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: between_route
        uri: http://example.com
        predicates:
        - Between=2017-09-10T17:42:47.789-07:00[America/Alaska], 2017-09-11T17:42:47.789-07:00[America/Alaska]
~~~
Java로 구성 방법
~~~java
ZonedDateTime datetime1 = LocalDateTime.now().minusDays(1).atZone(ZoneId.systemDefault());
ZonedDateTime datetime2 = LocalDateTime.now().atZone(ZoneId.systemDefault())
//..route definition
.route(r -> r.between(datetime1, datetime2))
.id("between_route")
.uri("http://example.com")
~~~

## 3. Header Route Predicate Factory
헤더 이름 및 정규식이 일치하는 요청인 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: http://example.com
        predicates:
        - Header=X-Request-Id, \d+
~~~
Java로 구성 방법
~~~java
//..route definition
.route(r -> r.header("X-Request-Id", "\\d+")
.id("header_route")
.uri("http://example.com")
~~~

## 4. Host Route Predicate Factor
Host 헤더의 패턴이 일치하는 요청인 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: http://example.com
        predicates:
        - Host=**.example.com
~~~

## 5. Method Route Predicate Factory
HttpMethod가 일치하는 요청인 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://example.com
        predicates:
        - Method=GET
~~~
Java로 구성 방법
~~~java
//..route definition
.route(r -> r.method("GET")
.id("method_route")
.uri("http://example.com")
~~~

## 6. Path Route Predicate Factory
Path 패턴이 요청과 일치되는 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: http://example.com
        predicates:
        - Path=/articles/{articleId}
~~~
Java로 구성 방법
~~~java
//..route definition
.route(r -> r.path("/articles/"+articleId)
.id("path_route")
.uri("http://example.com")
~~~

## 7. Query Route Predicate Factory
쿼리의 파라메터 및 정규식과 일치하는 요청인 경우
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://example.com
        predicates:
        - Query=articleId, \w
~~~
Java로 구성 방법
~~~java
//..route definition
.route(r -> r.query("articleId", "\w")
.id("query_route")
.uri("http://example.com")
~~~

## 8. RemoteAddr Route Predicate Factory
`RemoteAddr`와 일치하는 요청인 경우 
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: http://example.com
        predicates:
        - RemoteAddr=192.168.1.1/24
~~~
Java로 구성 방법
~~~java
//..route definition
.route(r -> r.remoteAddr("192.168.1.1/24")
.id("remoteaddr_route")
.uri("http://example.com")
~~~

# References
* https://www.baeldung.com/spring-cloud-gateway