# Intro
> 2018년 9월 25일 final release된 jdk11의 신규 Features를 살펴본다.
# Big features
## 1. (JEP 323) Local-Variable Syntax for Lambda Parameters
> 람다 표현식을 사용할 경우 파라메터의 타입을 제공하는 var 추가되었다.
> 물론, jdk8에서도 람다의 파라메터에 대해 암묵적인 타입 추론을 제공하고 있지만
> [JEP323](http://openjdk.java.net/jeps/323)의 Goals를 보면,
> jdk10에서 로컬변수에 대한 타입 추론으로 var을 제공한 것을 람다에도 적용하여 표현식을 통일하였다.
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
## 3. (JEP 333) ZGC: A Scalable Low-Latency Garbage Collector (Experimental)
> jdk11에서 새롭게 등장한 가비지 콜렉터
> 아래의 목료를 가지고 개발됨
* GC 일시 중지 시간은 10ms를 초과하지 않는다.
* 작은 크기(수백 MB) ~ 매우 큰 크기(수백 TB) 범위의 힙을 처리
* G1 보다 애플리케이션 처리량이 15% 이상 감소하지 않는다.
* ***Colored pointers***, ***Load barriers***를 사용하여 향후 GC 최적화를 위한 기반을 마련
* 최초 Linux/x64을 지원(향후 추가 플랫폼 지원 가능)
> JVM 기반의 애플리케이션은 GC가 동작할 때 ***Stop-The-World*** 현상으로 성능에 큰 영향을 끼쳤는데
> 쓰레드의 정지시간을 줄이거나 없앰으로써 성능을 향상시킴
> ZGC의 주요 원리는 ***Load barrier***와 ***Colored object pointer***을 사용하여
> 애플리케이션 스레드가 동작하는 중간에 ZGC가 객체 재배치 같은 작업을 수행할 수 있게 해준다.
# References
* https://meetup.toast.com/posts/171
* https://johngrib.github.io/wiki/java-gc-zgc