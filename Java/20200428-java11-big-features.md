# Intro
> 2018년 9월 25일 final release된 jdk11의 신규 Features를 살펴본다.
# Big features
## 1. (JEP 323) Local-Variable Syntax for Lambda Parameters
> 람다 표현식을 사용할 경우 파라메터의 타입을 제공하는 var 추가되었다.
> 물론, jdk8에서도 람다의 파라메터에 대해 암묵적인 타입 추론을 제공하고 있지만
> [JEP323](http://openjdk.java.net/jeps/323)의 Goals를 보면,
> jdk10에서 로컬변수에 타입 추론으 var을 제공한 것을 람다에도 적용하여 표현식을 통일하였다.
~~~
//jdk8
list.stream()
    .map(s -> s.toLowerCase())
    .collect(Collectors.toList());

//jdk11
list.stream()
    .map((var s) -> s.toLowerCase())
    .collect(Collectors.toList());

//jdk11 - 어노테이션 사용도 가능
list.stream()
    .map((@NotNull var s) -> s.toLowerCase())
    .collect(Collectors.toList());
~~~
## 2. (JEP 321) HTTP Client (Standard)
> 인큐베이터였던 ***java.incubator.http*** 패키지가 ***java.net.http*** 패키지로 표준화되었다.
* Non-Blocking request and response 지원(with CompletableFuture)
* Backpressure 지원
* HTTP/2 지원
* Factory method 형태로 제공
### HttpClient
> factory method 사용 방법
~~~
public void get(String uri) {
    HttpClient = HttpClient.newHttpClient();
    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(uri))
        .build();

    HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
    Systen.out.println(response.body());
}
~~~
> builder 사용 방법
~~~
public void get(String uri) {
    HttpClient client = HttpClient.newBuilder()
        .version(Version.HTTP_2)
        .followRedirects(Redirect.SAME_PROTOCOL)
        .proxy(ProxySelector.of(new InetSocketAddress(uri, 8080)))
        .authenticator(Authenticator.getDefault())
        .build();

    HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
    Systen.out.println(response.body());
}
~~~
### HttpRequest
> 위의 factory 메소드 뿐만 아니라, builder 방식도 제공
> 이렇게 생성한 인스턴스는 Immutable하고, 여러번 재사용 가능
~~~
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("http://openjdk.java.net/"))
    .timeout(Duration.ofMinutes(1))
    .header("Content-Type", "application/json")
    .POST(BodyPublishers.ofFile(Paths.get("file.json")))
    .build()
~~~
### Synchronous or Asynchronous
~~~
//Synchronous
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
System.out.println(response.statusCode());
System.out.println(response.body());

//Asynchronous
client.sendAsync(request, BodyHandlers.ofString())
    .thenApply(response -> { 
        System.out.println(response.statusCode());
        return response; 
    })
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
~~~

# References
* https://meetup.toast.com/posts/171