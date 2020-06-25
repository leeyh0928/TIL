# Overview
`Zuul`은 Netflix의 에지 서비스(또는 API Gateway)로 동적 라우팅, 모니터링, 복원력 및 보안을 제공한다.
`Zuul` 라우트 Path에 대한 `fallbacks`을 구성하는 방법을 알아본다.

# Initial Setup
간단한 두개의 애플리케이션을 만들어본다. 첫번째는 간단한 REST API 애플리케이션이고, 두번째는 Zuul 프록시를 이용한 API Gateway 애플리케이션이다.

## 1. REST API
간단하게 REST API 날씨 서비스를 위한 컨트롤러를 만든다.
~~~java
@RestController
@RequestMapping("/weather")
public class WeatherController {
 
    @GetMapping("/today")
    public String getMessage() {
        return "It's a bright sunny day today!";
    }
}
~~~

API를 호출해보자.
~~~bash
$ curl -s localhost:8080/weather/today
It's a bright sunny day today!
~~~
## 2. API Gateway
Gradle Dependency 설정하기 
~~~groovy
compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-netflix-zuul'
~~~

@EnableZuulProxy 어노테이션으로 Zuul Proxy 활성화하기
~~~java
@SpringBootApplication
@EnableZuulProxy
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
~~~

`application.yml`에 날씨 서비스 API에 대해 `Ribbon`을 사용하여 Zuul 라우팅을 구성한다.
~~~yaml
spring:
   application:
      name: api-gateway
server:
   port: 7070
   
zuul:
   igoredServices: '*'
   routes:
      weather-service:
         path: /weather/**
         serviceId: weather-service
         strip-prefix: false
 
ribbon:
   eureka:
      enabled: false
 
weather-service:
   ribbon:
      listOfServers: localhost:8080
~~~

## 3. Testing the Zuul Route
Zuul 프록시를 사용하여 날씨 서비스 API가 노출되도록 설정하였다. Zuul을 통해 날씨 서비스 API를 호출해보자.
~~~bash
$ curl -s localhost:7070/weather/today
It's a bright sunny day today!
~~~

## 4. Testing the Zuul Route Failure Without Fallback
API 호출 시 에러가 발생한다면 아래와 비슷할 것이다.
~~~bash
$ curl -s localhost:7070/weather/today
{"timestamp":"2019-10-08T12:42:09.479+0000","status":500,
"error":"Internal Server Error","message":"GENERAL"}
~~~

에러에 대한 처리 방법으로 `Fallback`을 만드는 것이다.

# Zuul Fallback for a Route
Zuul 프록시는 로드 밸런싱을 위해 리본을 사용하며, 요청은 Hystrix 명령에서 실행된다. 결과적으로 Zuul 경로의 실패는 Hystrix 매트릭스에
나타난다. 따라서 Zuul 라우트에 대한 사용자 정의 폴백을 작성하기 위해 FallbackProvider 유형의 Bean을 작성한다.

## 1. WeatherServiceFallback
~~~java
@Component
class WeatherServiceFallback implements FallbackProvider {
 
    private static final String DEFAULT_MESSAGE = "Weather information is not available.";
 
    @Override
    public String getRoute() {
        return "weather-service";
    }
 
    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        if (cause instanceof HystrixTimeoutException) {
            return new GatewayClientResponse(HttpStatus.GATEWAY_TIMEOUT, DEFAULT_MESSAGE);
        } else {
            return new GatewayClientResponse(HttpStatus.INTERNAL_SERVER_ERROR, DEFAULT_MESSAGE);
        }
    }
}
~~~

## 2. Testing the Zuul Fallback
날씨 서비스 API가 중지되어 있을 때 Zuul Proxy 호출 시 Fallback 처리가 잘 되는지 확인한다.
~~~bash
$ curl -s localhost:7070/weather/today
Weather information is not available.
~~~

# Fallback for All Routes
경로 ID를 사용하여 Zuul 라우트에 대한 Fallback을 만드는 것을 해보았는데, 모든 라우트에 대한 일반적인 Fallback을 만드는 방법이다.
오버라이드한 getRoute() 함수에서 '*' 또는 null 리턴하도록 한다.
~~~java
@Component
class GatewayServiceFallback implements FallbackProvider {

    private static final String DEFAULT_MESSAGE = "Service not available.";

    @Override
    public String getRoute() {
        return "*"; // or return null;
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        if (cause instanceof HystrixTimeoutException) {
            return new GatewayClientResponse(HttpStatus.GATEWAY_TIMEOUT, DEFAULT_MESSAGE);
        } else {
            return new GatewayClientResponse(HttpStatus.INTERNAL_SERVER_ERROR, DEFAULT_MESSAGE);
        }
    }
}
~~~

# References
* https://www.baeldung.com/spring-zuul-fallback-route