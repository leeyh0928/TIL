# Overview
Spring에서 REST API에 대한 예외처리를 구현하는 방법에 대해 알아본다. Spring 3.2 이전에는 Spring MVC 애플리케이션에서 예외를 
처리하는 두 가지 주요 접근 방식은 HandlerExceptionResolver 또는 @ExceptionHandler 어노테이션이었다. 이 두가지 방식은 단점이
존재하였고, 3.2 이후에 전체 애플리케이션에서 통일된 예외 처리를 위해서 @ControllerAdvice 어노테이션을 사용했다.
`Spring 5에서는 ResponseStatusException` 클래스가 도입되었다. REST API에서 기본적인 오류를 빠르게 처리할 수 있는 방식이다.

# @ExceptionHandler
@Controller 레벨에서 동작한다. 예외를 처리하는 메소드를 정의하고, @ExceptionHandler 어노테이션을 단다.
~~~java
public class FooController{
     
    //...
    @ExceptionHandler({ CustomException1.class, CustomException2.class })
    public void handleException() {
        //
    }
}
~~~
이 접근 방식에는 큰 단점이 있다. @ExceptionHandler 어노테이션이 있는 메소드는 전체 애플리케이션이 아닌 특정 Controller이기 때문에
전체 애플리케이션 내에서 균일 한 예외 처리를 적용하기 어렵다. 물론 모든 Controller가 기본 Controller 클래스를 확장하도록 하여
문제는 해결할 수 있지만, 어떤 이유로 이 것이 불가능한 애플리케이션은 문제가 될 수 있다.

# HandlerExceptionResolver
애플리케이션에서 발생되는 모든 예외를 처리할 수 있기 때문에 REST API에서 균일 한 예외 처리 메커니즘을 구현할 수 있다.

## 1. ExceptionHandlerExceptionResolver
Spring 3.1에서 도입되었으며 기본적으로 DispatcherServlet에서 활성화된다. 이전에 설명한 @ExceptionHandler 메커니즘이 동작하는
핵심 구성 요소이다.

## 2. DefaultHandlerExceptionResolver
Spring 3.0에서 도입되었으며 기본적으로 DispatcherServlet에서 활성화된다. 표준 Spring 예외를 해당 HTTP 상태 코드, 
즉 클라이언트 오류 – 4xx 및 서버 오류 – 5xx 상태 코드로 처리하는데 사용된다. 
[여기에](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/mvc.html#mvc-ann-rest-spring-mvc-exceptions) 
처리되는 스프링 예외의 전체목록과 상태 코드에 매핑되는 방법이 있다.

응답의 상태 코드를 올바르게 설정하지만 한가지 `제한 사항은 응답 본문에 아무것도 설정하지 않는 것`이다. REST API의 경우 상태 코드는
실제로 클라이언트에 제공할 정보로 충분하지 않다. 응답에 본문이 있어야 클라이언트에 실패에 대한 추가 정보를 제공할 수 있다.

`ModelAndView`를 통해 뷰를 구성하고, 오류 내용을 렌더링하면 이 문제를 해결할 수 있지만 최적의 솔루션은 아니다. 그래서 Spring 3.2 이후
이보다 더 나은 옵션을 도입한 이유이다.

## 3. ResponseStatusExceptionResolver
Spring 3.0에서도 도입되었으며 기본적으로 DispatcherServlet에서 활성화된다. 주된 책임은 사용자 지정 예외에서 @ResponseStatus 어노테이션을 
사용하여 예외를 HTTP 상태 코드에 매핑하는 것`이다. 이러한 사용자 정의 예외는 다음과 같다.
~~~java
@ResponseStatus(value = HttpStatus.NOT_FOUND)
public class MyResourceNotFoundException extends RuntimeException {
    public MyResourceNotFoundException() {
        super();
    }
    public MyResourceNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
    public MyResourceNotFoundException(String message) {
        super(message);
    }
    public MyResourceNotFoundException(Throwable cause) {
        super(cause);
    }
}
~~~
`DefaultHandlerExceptionResolver`와 동일하게 응답의 본문을 다루는 방식에 제한이 있다. 응답에 상태 코드를 매핑하지만 본문은 여전히 NULL이다.

## 4. Custom HandlerExceptionResolver
`DefaultHandlerExceptionResolver와 ResponseStatusExceptionResolver`은 앞서 언급했듯이 응답 본문을 제어하지 않는다. 이상적으로는
클라이언트가 요청한 형식(Accept 헤더)에 따라 JSON 또는 XML을 출력하기를 원한다. 아래 `사용자 정의 HandlerExceptionResolver`를
통해 이 문제를 해결할 수 있다.
~~~java
@Component
public class RestResponseStatusExceptionResolver extends AbstractHandlerExceptionResolver {
    @Override
    protected ModelAndView doResolveException(
      HttpServletRequest request, 
      HttpServletResponse response, 
      Object handler, 
      Exception ex) {
        try {
            if (ex instanceof IllegalArgumentException) {
                return handleIllegalArgument(
                  (IllegalArgumentException) ex, response, handler);
            }
            ...
        } catch (Exception handlerException) {
            logger.warn("Handling of [" + ex.getClass().getName() + "] 
              resulted in Exception", handlerException);
        }
        return null;
    }
 
    private ModelAndView 
      handleIllegalArgument(IllegalArgumentException ex, HttpServletResponse response) 
      throws IOException {
        response.sendError(HttpServletResponse.SC_CONFLICT);
        String accept = request.getHeader(HttpHeaders.ACCEPT);
        ...
        return new ModelAndView();
    }
}
~~~
여기서 주목할 점은 Request 자체에 엑세스 할 수 있으므로 클라이언트가 보낸 Accept 헤더의 값을 고려할 수 있다는 것이다.
예를 들어, 클라이언트가 `application/json`을 요청하면 오류인 경우 `application/json`으로 인코딩된 응답 본문을 반환해야 한다.
다른 구현 세부 사항은 `ModelAndView`를 응답 본문으로 반환하는 것이다.

이 접근법은 `Spring REST Service`의 오류 처리를 위한 일관되고 쉽게 구성 가능한 메커니즘이다. 그러나 저수준의 `HtttpServletResponse`와
상호 작용하고, `ModelAndView`를 사용하는 이전 MVC 모델에 적합하므로 여전히 개선의 여지가 존재한다.

# @ControllerAdvice
Spring 3.2 이후 `@ControllerAdvice 어노테이션으로 전역 @ExceptionHandler`를 지원한다. 이를 통해 이전 MVC 모델에서 탈피하고 
`@ExceptionHandler`의 안전성 및 유연성과 함께 `ResponseEntity`를 사용하는 메커니즘을 사용할 수 있다.
~~~java
@ControllerAdvice
public class RestResponseEntityExceptionHandler 
  extends ResponseEntityExceptionHandler {
 
    @ExceptionHandler(value 
      = { IllegalArgumentException.class, IllegalStateException.class })
    protected ResponseEntity<Object> handleConflict(
      RuntimeException ex, WebRequest request) {
        String bodyOfResponse = "This should be application specific";
        return handleExceptionInternal(ex, bodyOfResponse, 
          new HttpHeaders(), HttpStatus.CONFLICT, request);
    }
}
~~~
실제 메커니즘은 매우 간단하지만 다음과 같이 매우 유연하다.
* 응답 본문 및 상태 코드에 대한 모든 권한
* 여러 가지 예외를 동일한 방법으로 매핑하여 함께 처리
* 최신 RESTful ResposeEntity 응답을 활용

# ResponseStatusException (Spring 5 and Above)
Spring 5 `ResponseStatusException` 클래스를 도입하여 ***HttpStatus***를 제공하고 선택적으로 이유와 원인을 제공하는 인스턴스를
만들 수 있다.
~~~java
@GetMapping(value = "/{id}")
public Foo findById(@PathVariable("id") Long id, HttpServletResponse response) {
    try {
        Foo resourceById = RestPreconditions.checkFound(service.findOne(id));
 
        eventPublisher.publishEvent(new SingleResourceRetrievedEvent(this, response));
        return resourceById;
     }
    catch (MyResourceNotFoundException exc) {
         throw new ResponseStatusException(
           HttpStatus.NOT_FOUND, "Foo Not Found", exc);
    }
}
~~~
`ResponseStatusException`을 사용하면 어떤 장점이 있는가?
* 프로토 타이핑에 탁월 : 기본 솔루션을 매우 빠르게 구현할 수 있다.
* 하나의 유형, 여러 상태 코드 : 하나의 예외 유형으로 인해 여러가지 다른 응답이 발생할 수 있다. 이것은 `@ExceptionHandler에 비해
tight coupling(강한 결합)을 줄인다.`
* 많은 사용자 정의 예외 클래스를 만들 필요가 없다.
* 프로그래밍 방식으로 예외를 생성 할 수 있으므로 `예외 처리를 보다 효과적으로 제어`할 수 있다.

`트레이드 오프`는 어떤 것이 있는가?
* 통일된 예외 처리 방법이 없다. 전역 접근 방식을 제공하는 @ControllerAdvice와 달리 애플리케이션 내 균일 한 규칙을 적용하기 어렵다.
* 중복 코드 : 여러 컨트롤러에서 중복 코드가 발생할 수 있다.

# Spring Boot Support
Spring Boot는 합리적인 방법으로 `오류를 처리하기 위해 ErrorController`를 제공한다. 간단히 말해서 브라우저에 대한 대체 오류 페이지
(일명 Whitelabel 오류 페이지)와 HTML이 아닌 RESTful 요청에 대한 JSON 응답을 제공한다.
~~~json
{
    "timestamp": "2019-01-17T16:12:45.977+0000",
    "status": 500,
    "error": "Internal Server Error",
    "message": "Error processing the request!",
    "path": "/my-endpoint-with-exceptions"
}
~~~
Spring Boot는 다음 기능을 속성으로 구성 할 수 있다.
* server.error.whitelabel.enabled : Whitelabel Error Page를 비활성화하고, 서블릿 컨테이너를 사용하여 HTML 오류 메시지를 
제공하는 데 사용할 수 있다.
* server.error.include-stacktrace: HTML과 JSON의 기본 응답 모두 `stacktrace`를 포함한다. 이러한 속성 외에도 
`화이트 라벨 페이지를 재정의 하여 / 오류에 대한 자체 뷰 리졸버 매핑을 제공` 할 수 있다.

Spring Boot에서 제공하는 `DefaultErrorAttributes` 클래스를 확장하여 작업을 쉽게 할 수 있다.
~~~java
@Component
public class MyCustomErrorAttributes extends DefaultErrorAttributes {
    @Override
    public Map<String, Object> getErrorAttributes(
      WebRequest webRequest, boolean includeStackTrace) {
        Map<String, Object> errorAttributes = 
          super.getErrorAttributes(webRequest, includeStackTrace);
        errorAttributes.put("locale", webRequest.getLocale()
            .toString());
        errorAttributes.remove("error");
 
        //...
 
        return errorAttributes;
    }
}
~~~
더 나아가서 특정 컨텐츠 유형에 대한 오류를 처리하는 방법을 정의(또는 대체)하려는 경우 `ErrorController` Bean을 등록 할 수 있다.
XML 엔드 포인트에서 트리거 된 오류를 처리하는 방법을 사용자 정의하려고 하는 경우 다음과 같이 ***BasicErrorController***를 사용하고,
@RequestMapping을 사용하여 public 메소드를 정의하고 `application/xml` 매체 유형을 생성한다고 명시하면 된다.
~~~java
@Component
public class MyErrorController extends BasicErrorController {
    public MyErrorController(ErrorAttributes errorAttributes) {
        super(errorAttributes, new ErrorProperties());
    }
 
    @RequestMapping(produces = MediaType.APPLICATION_XML_VALUE)
    public ResponseEntity<Map<String, Object>> xmlError(HttpServletRequest request) {
         
    // ...

    }
}
~~~

# References
* https://www.baeldung.com/exception-handling-for-rest-with-spring