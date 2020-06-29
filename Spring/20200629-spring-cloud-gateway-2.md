# Overview
지난번에 이어서 `Spring Cloud Gateway`의 `WebFilter Factories`에 대해 알아본다.

# WebFilter Factories
`Route filters`를 사용하면 들어오는 HTTP 요청 또는 나가는 HTTP 응답을 수정할 수 있다.

## 1. AddRequestHeader WebFilter Factory
HTTP Request 헤더에 원하는 내용을 아래처럼 추가한다.
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: addrequestheader_route
        uri: http://example.com
        predicates:
        - Path=/articles
        filters:
        - AddRequestHeader=X-SomeHeader, bael
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.addRequestHeader("X-TestHeader", "rewrite_request"))
  .uri("http://example.com")
  .id("addrequestheader_route")
~~~
## 2. AddRequestParameter WebFilter Factory
HTTP Request 파라메터에 원하는 내용을 아래처럼 추가한다.
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: addrequestparameter_route
        uri: http://example.com
        predicates:
        - Path=/articles
        filters:
        - AddRequestParameter=foo, bar
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.addRequestParameter("foo", "bar"))
  .uri("http://example.com")
  .id("addrequestparameter_route")
~~~
## 3. AddResponseHeader WebFilter Factory
HTTP Response 헤더에 원하는 내용을 아래처럼 추가한다.
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: addrequestheader_route
        uri: http://example.com
        predicates:
        - Path=/articles
        filters:
        - AddResponseHeader=X-SomeHeader, Bar
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.addResponseHeader("X-SomeHeader", "Bar"))
  .uri("http://example.com")
  .id("addresponseheader_route")
~~~
## 4. Circuit Breaker WebFilter Factory
`Hystrix`가 `Circuit-Breaker WebFilter Factory`로 사용된다.
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: hystrix_route
        uri: http://example.com
        predicates:
        - Path=/articles
        filters:
        - Hystrix=someCommand
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.hystrix("some-command"))
  .uri("http://example.com")
  .id("hystrix_route")
~~~
## 5. RedirectTo WebFilter Factory
리디렉션 HTTP Status인 300번대 코드 및 유효한 URL을 설정한다.
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: redirectto_route
        uri: http://example.com
        predicates:
        - Path=/articles
        filters:
        - RedirectTo=302, http://foo.bar
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.redirect("302","http://foo.bar"))
  .uri("http://example.com")
  .id("redirectto_route")
~~~
## 6. RewritePath WebFilter Factory
정규식 파라메터 Path 및 이를 대체할 파라메터 정보를 전달하여 요청 Path를 재작성한다.
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: http://example.com
        predicates: 
        - Path=/articles/**
        filters:
        - RewritePath=/articles/(?<articleId>.*), /$\{articleId}
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.rewritePath("(?<articleId>.*)", articleId))
  .uri("http://example.com")
  .id("rewritepath_route")
~~~
## 7. RequestRateLimiter WebFilter Factory
replenishRate, capacity 및 keyResolverName의 세 가지 매개 변수를 사용
* replenishRate – 사용자가 초당 허용 할 요청 수
* capacity – 최대 허용량을 정의합니다
* keyResolverName – KeyResolver 인터페이스를 구현하는 Bean 이름
~~~yaml
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: http://example.com
        predicates:
        - Path=/articles
        filters:
        - RequestRateLimiter=10, 50, userKeyResolver
~~~
자바로 구성
~~~java
//...route definition
.route(r -> r.path("/articles")
  .filters(f -> f.requestRateLimiter().configure(c -> c.setRateLimiter(myRateLimiter)))
  .uri("http://example.com")
  .id("requestratelimiter_route")
~~~

# Spring Cloud DiscoveryClient Support
`Spring Cloud Gateway`는 `Eureka Server 및 Consul`과 같은 Service Discovery 및 Registry 라이브러리와 쉽게 통합될 수 있다.
~~~java
@Configuration
@EnableDiscoveryClient
public class GatewayDiscoveryConfiguration {
    @Bean
    public DiscoveryClientRouteDefinitionLocator discoveryClientRouteLocator(DiscoveryClient discoveryClient) {
        return new DiscoveryClientRouteDefinitionLocator(discoveryClient);
    }
}
~~~

# Monitoring
`Spring Boot Actuator` API를 사용하여 모니터링할 수있다. Actuator 구성이 완료되면 `/gateway/endpoint`에 접속하여
게이트웨이 모니터링 기능을 시각화 할 수 있다.

# References
* https://www.baeldung.com/spring-cloud-gateway