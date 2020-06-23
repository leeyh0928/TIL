# Overview
선언적 REST 클라이언트인 `Spring Cloud OpenFeign`에 대해 알아본다.
Feign은 Feign 및 JAX-RS 어노테이션들을 지원하여 웹 서비스 클라이언트를 쉽게 작성하게 해준다.

# Dependency
~~~groovy
compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-openfeign'
~~~

# Feign 클라이언트
메인 클래스에 `@EnableFeignClients`를 추가
~~~java
@SpringBootApplication
@EnableFeignClients
public class ExampleApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(ExampleApplication.class, args);
    }
}
~~~
Feign 클라이언트 인터페이스에 `@FeignClient` 어노테이션을 사용하여 Feign 클라이언트를 선언
~~~java
@FeignClient(value = "jplaceholder", url = "https://jsonplaceholder.typicode.com/")
public interface JSONPlaceHolderClient {
    @RequestMapping(method = RequestMethod.GET, value = "/posts")
    List<Post> getPosts();
 
    @RequestMapping(method = RequestMethod.GET, value = "/posts/{postId}", produces = "application/json")
    Post getPostById(@PathVariable("postId") Long postId);
~~~

# Configuration
`Spring Cloud`는 사용자 정의 할 수 있는 `FeignClientsConfiguration` 클래스를 사용하여 각 명명된 클라이언트에 대해
기본 설정을 생성하고, 다음과 같은 Bean이 있다.
* Decoder - `SpringDecoder`를 감싸고 있는 `ResponseEntityDecoder`는 응답을 디코드 하는데 사용된다.
* Encoder - `RequestBody`를 인코드 하고, `SpringEncoder`를 사용한다.
* Logger - 디폴트 Logger로 Slf4jLogger를 사용한다.
* Contract - `SpringMvcContract`
* Feign-Builder - 구성 요소를 구성하는 데 사용되는 HystrixFeign.Builder
* Client - `LoadBalancerFeignClient` 또는 기본 `Feign client`
## 1. Custom Beans Configuration
이러한 Bean 중 하나 이상을 커스터마이즈하려면 @Configuration 클래스를 사용하여 재정의 한 다음, FeignClient
어노테이션에 추가할 수 있다.

이 예제에서는 HTTP/2 지원을 위해 기본 대신 `OkHttpClient`을 사용하도록 함.
~~~groovy
compile group: 'io.github.openfeign', name: 'feign-okhttp'
~~~
~~~java
@FeignClient(value = "jplaceholder",
  url = "https://jsonplaceholder.typicode.com/",
  configuration = MyClientConfiguration.class)

...

@Configuration
public class MyClientConfiguration {
 
    @Bean
    public OkHttpClient client() {
        return new OkHttpClient();
    }
}
~~~

## 2. Configuration Using Properties
`@Configuration` 클래스 대신에 `application.yaml`에 속성을 사용하여 Feign 클라이언트를 구성할 수 있다.
~~~yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
~~~
기본적으로 클라이언트 이름으로 구성을 만들거나 가상 클라이언트 이름을 선언할 수 있다.
~~~yaml
feign:
  client:
    config:
      jplaceholder:
...
~~~

# Interceptors
Feign에서 제공하는 또 다른 유형의 기능으로 모든 HTTP 요청/응답에 대해 인증에서 로깅에 이르기까지 
다양한 암시적 태스크를 수행한다.

각 요청에 기본 인증을 추가하는 요청 인터셉터 선언
~~~java
@Bean
public RequestInterceptor requestInterceptor() {
  return requestTemplate -> {
      requestTemplate.header("user", username);
      requestTemplate.header("password", password);
      requestTemplate.header("Accept", ContentType.APPLICATION_JSON.getMimeType());
  };
}
~~~

인터셉터를 요청 체인에 추가하려면 Bean을 @Configuration 클래스에 추가하거나 이전에 본 것처럼 
특성 파일에 선언한다.
~~~yaml
feign:
  client:
    config:
      default:
        requestInterceptors:
          com.example.cloud.openfeign.JSONPlaceHolderInterceptor
~~~

# Hystrix Support
[Hystrix](https://www.baeldung.com/spring-cloud-netflix-hystrix)를 지원 하므로 이를 활성화 하면 
fallback 패턴을 구현 할 수 있다.

fallback 구현
~~~java
@Component
public class JSONPlaceHolderFallback implements JSONPlaceHolderClient {
 
    @Override
    public List<Post> getPosts() {
        return Collections.emptyList();
    }
 
    @Override
    public Post getPostById(Long postId) {
        return null;
    }
}
~~~
Feign에 알리려면 @FeignClient 어노테이션에 대체 클래스를 설정한다.
~~~java
@FeignClient(value = "jplaceholder",
  url = "https://jsonplaceholder.typicode.com/",
  fallback = JSONPlaceHolderFallback.class)
public interface JSONPlaceHolderClient {
    // APIs
}
~~~

# Error Handling
Feign의 기본 오류 처리기 인 `ErrorDecoder.default`는 항상 `FeignException`을 발생한다.
해당 동작 방식을 변경하기 위해 `CustomErrorDecoder`를 사용할 수 있다.
~~~java
public class CustomErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
 
        switch (response.status()){
            case 400:
                return new BadRequestException();
            case 404:
                return new NotFoundException();
            default:
                return new Exception("Generic error");
        }
    }
}
~~~

이전에 한 것처럼 `@Configuration` 클래스에 Bean을 추가 하여 기본 `ErrorDecoder`를 교체한다.
~~~java
@Configuration
public class ClientConfiguration {
    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomErrorDecoder();
    }
}
~~~

# References
* https://www.baeldung.com/spring-cloud-openfeign