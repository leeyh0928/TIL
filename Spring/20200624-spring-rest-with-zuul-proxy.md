# Overview
프론트 엔드 애플리케이과 별도로 배치 된 REST API 간의 통신에 대해 살펴 본다. 브라우저의 CORS 및 동일한 Origin 정책 제한 사항을 해결하고, 
동일한 Origin을 공유하지 않더라도 UI가 API를 호출 할 수 있도록 해본다.

> `Zuul`은 Netflix의 JVM 기반 `라우터 및 서버 로드 밸런서`이다.

# API
간단한 API 애플리케이션을 만들어본다.
> DTO 정의
~~~java
public class Foo {
    private long id;
    private String name;
 
    ...
}
~~~
> 컨트롤러 정의
~~~java
@RequestMapping("/spring-zuul-foos-resource")
@RestController
public class FooController {

    @GetMapping("/foos/{id}")
    public Foo findById(
      @PathVariable long id, HttpServletRequest req, HttpServletResponse res) {
        return new Foo(Long.parseLong(randomNumeric(2)), randomAlphabetic(4));
    }
}
~~~

# UI Application
> 별도의 UI Web 애플리케이션에서 다른 URL로 요청이 불가능하므로 Zuul 프록시를 구성하여 라우팅하도록 한다.

# Dependency
~~~groovy
compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-netflix-zuul'
~~~

# Zuul Properties
Spring Boot를 사용하기 위해 `application.yml`에 Zuul 구성을 설정한다.
~~~yaml
zuul:
  routes:
    foos:
      path: /foos/**
      url: http://localhost:8081/spring-zuul-foos-resource/foos
~~~
~~~java
@EnableZuulProxy
@SpringBootApplication
public class UiApplication extends SpringBootServletInitializer {
 
    public static void main(String[] args) {
        SpringApplication.run(UiApplication.class, args);
    }
}
~~~
`/foos/`로 시작하는 모든 요청은 `http://localhost:8081/spring-zuul-foos-resource/foos/` 인
Foos 리소스 서버로 라우팅 된다.

# Test the Routing
UI 서버를 통한 API (Proxy)호출이 정상적으로 되는지 확인한다.
~~~java
@Test
public void whenSendRequestToFooResource_thenOK() {
    Response response = RestAssured.get("http://localhost:8080/foos/1");
  
    assertEquals(200, response.getStatusCode());
}
~~~

# A Custom Zuul Filter
[Zuul 필터](https://github.com/eugenp/tutorials/tree/master/spring-cloud/spring-cloud-zuul)는 여러가지가 있고,
자체적인 사용자 지정 필터를 만들 수 있다.
> 간단하게 Test 라는 헤더를 추가하는 e.g
~~~java
@Component
public class CustomZuulFilter extends ZuulFilter {
 
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.addZuulRequestHeader("Test", "TestSample");
        return null;
    }
 
    @Override
    public boolean shouldFilter() {
       return true;
    }
    // ...
}
~~~

# Test Custom Zuul Filter
먼저 기존 `Foos 리소스 서버`에서 `FooController`를 수정한다.
~~~java
@RequestMapping("/spring-zuul-foos-resource")
@RestController
public class FooController {
 
    @GetMapping("/foos/{id}")
    public Foo findById(
      @PathVariable long id, HttpServletRequest req, HttpServletResponse res) {
        if (req.getHeader("Test") != null) {
            res.addHeader("Test", req.getHeader("Test"));
        }
        return new Foo(Long.parseLong(randomNumeric(2)), randomAlphabetic(4));
    }
}
~~~
테스트 코드로 확인해본다.
~~~java
@Test
public void whenSendRequest_thenHeaderAdded() {
    Response response = RestAssured.get("http://localhost:8080/foos/1");
  
    assertEquals(200, response.getStatusCode());
    assertEquals("TestSample", response.getHeader("Test"));
}
~~~