# Overview
`Java 8 Concurrency API` 개선으로 도입 된 ***CompletableFuture*** 클래스의 기능에 대해 알아본다.

# Using CompletableFuture as a Simple Future
***CompletableFuture*** 클래스는 ***Future*** 인터페이스 구현체로 추가 구현 로직과 함께 ***Future*** 구현체를 사용할 수 있다.
예를 들어, 기본 생성자를 통해 인스턴스를 생성하여 미래의 결과를 나타내고, 소비자에게 전달된 후 나중에 ***Complete*** 메소드를
사용하여 완료할 수 있다. 소비자는 이 메소드가 제공될 때까지 ***get*** 메소드를 사용하여 현재 스레드를 차단할 수 있다.

아래 예제는 ***CompletableFuture*** 인스턴스를 생성한 다음 다른 스레드에서 일부 계산을 스핀하고 ***Future***를 즉시 반환하는
메소드가 있다. 계산이 완료되면 메소드는 결과를 ***complete*** 메소드에 제공하여 ***Future***를 완료한다.
~~~java
public Future<String> calculateAsync() throws InterruptedException {
    CompletableFuture<String> completableFuture 
      = new CompletableFuture<>();
 
    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completableFuture.complete("Hello");
        return null;
    });
 
    return completableFuture;
}
~~~
Future 인스턴스를 수신하고 결과를 차단할 준비가 되면 ***get*** 메소드를 호출한다. 또한 `checked exceptions`, 즉 ***ExecutionException*** 및
***InterruptedException***를 발생시킨다.
~~~java
Future<String> completableFuture = calculateAsync();
 
// ... 
 
String result = completableFuture.get();
assertEquals("Hello", result);
~~~

계산 결과를 이미 알고 있는 경우 다음과 같이 결과를 나타내는 인수와 함께 ***static completedFuture*** 메소드를 사용할 수 있다. 
그러면 ***get*** 메소드로 인한 차단이 발생되지 않고 즉시 결과를 리턴한다.
~~~java
Future<String> completableFuture = 
  CompletableFuture.completedFuture("Hello");
 
// ...
 
String result = completableFuture.get();
assertEquals("Hello", result);
~~~

***Future***의 ***cancel*** 메소드로 비동기 실행을 모두 취소할 수 있다.
~~~java
public Future<String> calculateAsyncWithCancellation() throws InterruptedException {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();
 
    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completableFuture.cancel(false);
        return null;
    });
 
    return completableFuture;
}
~~~
***Future.get()*** 메소드를 사용하여 결과를 차단하면 ***Future***가 취소되고 ***CancellationException***이 발생한다.
~~~java
Future<String> future = calculateAsyncWithCancellation();
future.get(); // CancellationException
~~~

# CompletableFuture With Encapsulated Computation Logic
일부 코드를 비동기적으로 실행하려는 경우 ***runAsync*** 및 ***supplyAsync***를 사용하면 ***Runnable*** 및 ***Supplier*** 기능 유형으로
***CompletableFuture*** 인스턴스를 작성할 수 있다.
~~~java
CompletableFuture<String> future
  = CompletableFuture.supplyAsync(() -> "Hello");
 
// ...
 
assertEquals("Hello", future.get());
~~~

# Processing Results of Asynchronous Computations
계산 결과를 처리하는 가장 일반적인 방법은 계산 결과를 함수에 제공하는 것이다. ***thenApply*** 메소드는 ***Function*** 인스턴스를 받아
결과를 처리하고, ***Function***에 의해 리턴되는 값을 가지고 있는 ***Future***를 리턴한다.
~~~java
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<String> future = completableFuture
  .thenApply(s -> s + " World");
 
assertEquals("Hello World", future.get());
~~~
***Future***에서 값을 반환하지 않아도 되는 경우 ***Consumer*** 기능 인터페이스를 사용할 수 있다. ***thenAccept*** 메소드는
***Consumer***를 받아서 계산 결과를 전달하고, ***future.get()*** 호출 시에는 Void 형을 반환한다.
~~~java
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<Void> future = completableFuture
  .thenAccept(s -> System.out.println("Computation returned: " + s));
 
future.get();
~~~
계산 값이 필요하지 않거나 일부 값을 반환하려는 경우 다음과 같이 ***Runnable*** 람다를 ***thenRun*** 메소드에 전달할 수 있다.
***future.get()*** 호출 시 콘솔에 단순히 설정된 문자열 라인을 출력한다.
~~~java
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<Void> future = completableFuture
  .thenRun(() -> System.out.println("Computation finished."));
 
future.get();
~~~

# Combining Futures
`CompletableFuture API`의 가장 중요한 부분은 ***CompletableFuture*** 인스턴스를 일련의 계산 단계로 결합 하는 기능이다.
다음 예제는 ***thenCompose*** 메소드를 이용하여 두 ***Futures***를 순차적으로 연결한다.
~~~java
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " World"));
 
assertEquals("Hello World", completableFuture.get());
~~~

두 개의 독립적인 ***Futures***를 실행하고 그 결과로 무언가를 수행하려면 ***thenCombine*** 메소드를 사용하여 처리할 수 있다.
~~~java
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCombine(CompletableFuture.supplyAsync(
      () -> " World"), (s1, s2) -> s1 + s2));
 
assertEquals("Hello World", completableFuture.get());
~~~
더 간단하게 두 개의 ***Futures*** 결과로 무언가를 원하지만 ***Future chain***으로 전달할 필요가 없는 경우 ***thenAcceptBoth***
메소드를 사용할 수 있다.
~~~java
CompletableFuture future = CompletableFuture.supplyAsync(() -> "Hello")
  .thenAcceptBoth(CompletableFuture.supplyAsync(() -> " World"),
    (s1, s2) -> System.out.println(s1 + s2));
~~~

# thenApply() 와 thenCompose()의 차이점
두 API 모두 서로 다른 ***CompletableFuture*** 호출을 연결하는데 도움이 되지만 두 기능의 사용법은 다르다.
## 1. thenApply()
이 방법은 ***이전 호출의 결과로 작업하는 데 사용된다.*** 그리고 ***CompletableFuture*** 호출의 결과를 변환하려고 할 때 유용하다.
~~~java
CompletableFuture<Integer> finalResult = compute().thenApply(s-> s + 1);
~~~
## 2. thenCompose()
***thenCompose()*** 메소드는 새로운 완료 단계를 리턴한다는 점에서 ***thenApply() 메소드와 유사하다. 하지만 ***thenCompose()***
메소드는 이전 단계를 인수로 사용한다. 두 메소드는 [map()과 flatMap()의 차이점](https://www.baeldung.com/java-difference-map-and-flatmap)과 
유사하다.
~~~java
CompletableFuture<Integer> computeAnother(Integer i){
    return CompletableFuture.supplyAsync(() -> 10 + i);
}
CompletableFuture<Integer> finalResult = compute().thenCompose(this::computeAnother);
~~~

# Running Multiple Futures in Parallel
여러 ***Futures***를 병렬로 실행하려는 경우 모든 ***Futures***가 실행될 때까지 기다렸다가 결합된 결과를 처리하려고 한다.
이 때 ***CompletableFuture.allOf*** 메소드를 사용하여 처리할 수 있다.
~~~java
CompletableFuture<String> future1  
  = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2  
  = CompletableFuture.supplyAsync(() -> "Beautiful");
CompletableFuture<String> future3  
  = CompletableFuture.supplyAsync(() -> "World");
 
CompletableFuture<Void> combinedFuture 
  = CompletableFuture.allOf(future1, future2, future3);
 
// ...
 
combinedFuture.get();
 
assertTrue(future1.isDone());
assertTrue(future2.isDone());
assertTrue(future3.isDone());
~~~
모든 ***Futures***의 결합된 결과를 얻기 위해서 다음과 같이 ***CompletableFuture.join()*** 메소드와 Java8 Stream API를 활용할 수 있다.
~~~java
String combined = Stream.of(future1, future2, future3)
  .map(CompletableFuture::join)
  .collect(Collectors.joining(" "));
 
assertEquals("Hello Beautiful World", combined);
~~~

# Handling Errors
***CompletableFuture*** 클래스는 ***handle()*** 메소드를 통해 예외를 처리할 수 있다.
~~~java
String name = null;
 
// ...
 
CompletableFuture<String> completableFuture  
  =  CompletableFuture.supplyAsync(() -> {
      if (name == null) {
          throw new RuntimeException("Computation error!");
      }
      return "Hello, " + name;
  })}).handle((s, t) -> s != null ? s : "Hello, Stranger!");
 
assertEquals("Hello, Stranger!", completableFuture.get());
~~~
***handle*** 메소드를 사용하여 비동기식으로 예외를 처리할 수 있지만, ***get*** 메소드를 사용하면 일반적인 동기식 예외 처리 방식을
사용할 수 있다.
~~~java
CompletableFuture<String> completableFuture = new CompletableFuture<>();
 
// ...
 
completableFuture.completeExceptionally(
  new RuntimeException("Calculation failed!"));
 
// ...
 
completableFuture.get(); // ExecutionException
~~~

# Async Methods
***CompletableFuture*** 클래스의 대부분 메소드에는 비동기 접미사가 있는 추가 기능이 있다. 이러한 메소드는 일반적으로 
***다른 쓰레드에서 해당 실행 단계를 실행***하기 위한 것이다.(비동기 postfix가 없는 메소드는 호출 스레드를 사용하여 다음
실행 단계를 호출)
다음과 같이 ***thenApplyAsync()*** 메소드는 ***ForkJoinTask*** 인스턴스에 래핑되어 ***계산을 더욱 병렬화하고 시스템 리소스를
보다 효율적으로 사용***할 수 있다.
~~~java
CompletableFuture<String> completableFuture  
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<String> future = completableFuture
  .thenApplyAsync(s -> s + " World");
 
assertEquals("Hello World", future.get());
~~~

# References
* https://www.baeldung.com/java-completablefuture