# Overview
스트림은 Array, Collections 와 같이 연속된 형태의 객체이다. 하지만 자료구조는 아니다.
데이터를 입력받아 메소드로 처리할 뿐이다.

스트림의 구조는 크게 3가지로 나뉜다.
~~~java
// 스트림 생성 -> 중간 연산자 -> 최종 연산자
int result = list.stream()  // 스트림 생성
        .filter( ... )      // 중간 연산자
        .map( ... )         // 중간 연산자
        .count();           // 최종 연산자
~~~
각 연산에 대한 함수는 어떤 것이 있고, 어떻게 사용하는지 알아본다.

# Create Operations
## Arrays.stream()
~~~java
String[] arr = new String[] {"Hello", "World", "Hell"};
Stream<String> stream = Arrays.stream(arr); // 배열
Stream<String> streamOfArrayPart = Arrays.stream(arr, 1, 3); // 부분 배열
~~~
## Collections.stream()

컬렉션 타입(Collection, List, Set)은 메소드를 이용하여 스트림을 생성할 수 있다.
~~~java
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream = list.stream();
Stream<String> parallelStream = list.parallelStream(); // 병렬 처리 스트림
~~~
## 직접 생성하는 연산자
### empty()
null 대신 사용할 수 있다.
~~~java
Stream stream = Stream.empty();
~~~
### build()
~~~java
Stream<String> generatedStream = Stream.<String>builder()
        .add("Hello")
        .add("World")
        .build();
~~~
### generate()
크기를 지정하지 않으면 무한하기 때문에 특정 사이즈만큼 생성하려면 반드시 limit()을 통해 최대 크기를 제한해야 한다.
~~~java
Stream<String> generatedStream = Stream.generate(() -> "gen").limit(5);
~~~
### iterate()
초기 값을 시작으로 계속해서 값을 생성한다. generate()와 마찬가지로 크기를 지정하지 않으면 무한하기 때문에 
limit()을 통해 크기를 제한해야 한다.
~~~java
Stream<Integer> iteratedStream = Stream.iterate(30, n -> n + 2).limit(5);
~~~

## 기본 타입형 스트림
### IntStream, LongStream, DoubleStream
제네릭을 사용하지 않고 기본 값을 생성하는 방법이다. 제네릭을 사용하지 않기 때문에 불필요한 오토 박싱(auto-boxing)이 발생하지 않는다.
* range는 [startPosition, endPosition) 범위를 가진다.
* rangeClosed는 [startPosition, endPosition] 범위를 가진다.
~~~java
IntStream intStream = IntStream.range(1, 5); // [1, 2, 3, 4]
LongStream longStream = LongStream.rangeClosed(1, 5); // [1, 2, 3, 4, 5]
~~~
필요한 경우 boxed 메서드를 통해 Integer 형태로 박싱할 수 있다.
~~~java
Stream<Integer> boxedIntStream = IntStream.range(1, 5).boxed();
~~~
난수 스트림을 생성할 수도 있다.
~~~java
DoubleStream doubles = new Random().doubles(3); // 난수 3개 생성
~~~

# Intermediate Operations
값을 원하는 형태로 처리하기 위한 연산자이다. 각각의 중간 연산자들은 lazy하게 실행되고, 결과로 stream을 반환한다. 
그렇기 때문에 중간 연산자는 method chaining 형태로 연결하여 처리할 수 있다. 연산의 결과가 stream으로 반환되기 때문에 
stream-producing 연산자라고 부르기도 한다.

## Lazy 처리
결과가 필요하기 전까지 실행되지 않음을 의미한다. 연산의 시점을 최대한 늦춘다는 말이다.

## filter()
원하는 요소만 추출하기 위한 메소드이다. 인자로는 Predicate를 받는데, boolean 값을 반환하는 람다식을 넣으면 된다.
~~~java
Stream<T> filter(Predicate<? super T> predicate);
~~~
***e.g. 문자열 리스트에서 특정한 문자가 포함된 문자열 뽑아내기***
~~~java
List<String> names = Arrays.asList("Hello", "World", "Test", "array");
List<String> filteredNames = names.stream()
        .filter(it -> it.contains("e"))
        .collect(Collectors.toList());
~~~

## map()
스트림 내 요소를 가공한다. mapper를 간단히 설명하자면, T를 인자로 받아 변환한 값 R을 반환하는 함수이다. 
이는 람다식으로 간단히 표현할 수 있다.
~~~java
<R> Stream<R> map(Function<? super T, ? extends R> mapper);
~~~
***e.g. 정사각형 한 변의 길이를 곱하여 넓이로 가공하기***
~~~java
Arrays.asList(3, 8, 9, 10, 20, 11, 22)
        .stream()
        .map(it -> it * it)
        .forEach(System.out::println);
~~~

## flatMap()
중첩 구조를 한 단계 제거하고 단일 컬렉션으로 만들어 주는 역할을 한다. 이러한 작업을 flattening 이라고 한다.

map 과 가장 큰 차이는 함수의 반환 값이 stream 형태라는 것이다. 이는 map 만으로 처리하면 복잡해지는 코드를 간결하게 만들어준다.
~~~java
<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
~~~
***e.g. 2차원 배열과 같이 중첩된 형태의 값을 처리할 때 map을 이용하면 2중 for문의 형태를 사용해야 한다. 
반면에 flatMap을 이용하면 배열의 각 행에 있는 요소를 stream으로 만들어 처리할 수 있다.***
~~~java
String arr[][] = {
    {"minus one", "zero", "one"}, 
    {"two", "Three"}, 
    {"Four", "Five", "Six"}, 
    {"eight", "ten"}
};

Stream.of(arr)
        .flatMap(Stream::of)
        .forEach(System.out::println);

int arr[][] = {{1, 2, 3}, {4, 8}, {9, 10, 20}, {11, 22}};
Stream.of(arr)
        .flatMapToInt(IntStream::of)
        .forEach(System.out::println);
~~~

## sorted()
stream 요소를 정렬한다. 어떠한 인자도 넣지 않는다면 오름차순으로 정렬되고, Comparator 를 인자로 넣으면 Comparator 의 기준에 따라 정렬된다.
~~~java
Stream<T> sorted();
Stream<T> sorted(Comparator<? super T> comparator);
~~~
***e.g. 오름차순으로 정렬하기***
~~~java
List<String> names = Arrays.asList("Hello", "World", "stream", "API");
names.stream()
        .sorted()
        .forEach(System.out::println);
~~~
***e.g. 내림차순으로 정렬하기***
~~~java
List<String> names = Arrays.asList("Hello", "World", "stream", "API")
        .stream()
        .sorted(Comparator.reverseOrder())
        .forEach(System.out::println);
~~~
### Comparable VS Comparator
* Comparable 은 객체의 기본 정렬기준이 되는 메서드를 정의하는 인터페이스
* Comparator 는 새로운 정렬 기준을 적용하고 싶은 경우 사용하는 인터페이스

## distinct()
중복 값을 제거한다.
~~~java
List<String> names = Arrays.asList("312", "123", "123", "123", "1234");
names.stream()
        .distinct()
        .forEach(System.out::println);
~~~

## peek()
각 요소에 특정한 연산을 수행하는 메소드이다. '살짝 들여다본다'라는 단어의 뜻에서 알 수 있듯이 이 메소드가 결과에 영향을 주지는 않는다. 
중간에 값을 출력해볼 때 이용할 수 있다.
~~~java
Stream<T> peek(Consumer<? super T> action);
~~~
***e.g. 사용 예시***
~~~java
Arrays.asList("312", "123", "123", "123", "1234")
        .stream()
        .peek(System.out::println)
        .map(it -> "#" + it)
        .forEach(System.out::println);
~~~

## substream 추출 및 stream 결합
### limit()
앞선 n개의 요소만 취한다.
~~~java
Arrays.asList("Hello", "World", "Test", "array", "Hell")
        .stream()
        .limit(2)
        .forEach(System.out::println);
~~~

### Skip()
앞선 n개의 요소를 건너뛰고 다음에 오는 요소를 취한다.
~~~java
Arrays.asList("Hello", "World", "Test", "array", "Hell")
        .stream()
        .skip(2)
        .forEach(System.out::println);
~~~

### concat()
두 stream을 연결한다.
~~~java
Stream<String> streamOne = Arrays.asList("Hello", "World", "Test", "array", "What").stream();
Stream<String> streamTwo = Arrays.asList("The", "Hell").stream();
Stream.concat(streamOne, streamTwo)
        .forEach(System.out::println);
~~~

# Terminal Operations
## count(), sum(), min(), max(), average()
min, max, average의 경우에는 stream 이 비어있는 경우에 값을 표현할 수 없기 때문에 Optional 타입의 값을 반환한다.
~~~java
long count = IntStream.of(1, 2, 3, 4, 5).count();
System.out.println("Count is " + count);
    
long sum = IntStream.of(1, 2, 3, 4, 5).sum();
System.out.println("Sum is " + sum);
    
OptionalInt min = IntStream.of(1, 2, 3, 4, 5).min();
System.out.println("Sum is " + min.getAsInt());
    
OptionalInt max = IntStream.of(1, 2, 3, 4, 5).max();
System.out.println("Sum is " + max.getAsInt());
    
OptionalDouble average = IntStream.of(1, 2, 3, 4, 5).average();
System.out.println("Sum is " + average.getAsDouble());
~~~
Optional 값이 존재할 때만 특정 작업을 하고 싶다면 Optional 클래스에서 제공하는 ifPresent 메소드로 처리할 수 있다.
~~~java
IntStream.of(1, 2, 3, 4, 5)
        .average()
        .ifPresent(System.out::println);
~~~
합, 차, 평균, 최소값, 최대값 등의 정보가 모두 필요할 때에는 Collectors.summarizingInt()를 이용한다.

## reduce()
세 가지의 인자를 받아 처리할 수 있다.
~~~java
Optional<T> reduce(BinaryOperator<T> accumulator);
T reduce(T identity, BinaryOperator<T> accumulator);
<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);
~~~
* accumulator: 각 요소를 처리하는 계산 로직이다. 각 요소가 올 때마다 중간 결과를 생성한다.
* identity: 계산을 위한 초기값이다. stream이 비어서 계산할 값이 없더라도 이 값은 반환된다.
* combiner: 병렬 stream에서 나눠 계산한 결과를 하나로 합쳐 반환한다.

## Collecting
스트림 값을 모아 Map, Set, List와 같은 컬렉션 형태로 변환해준다. 가장 많이 쓰이는 최종 연산자가 아닐까 싶다.
### Collectors.toList()
리스트 형태로 결과를 반환한다.
### Collectors.joining()
스트림 작업 결과를 하나의 스트링으로 연결한다. 세 가지 인자를 입력할 수 있다.
* delimiter: 각 요소 중간에 들어가는 구분자
* prefix: 이어붙인 결과 맨 앞에 붙는 문자
* suffix: 이어붙인 결과 맨 끝에 붙는 문자
### Collectors.groupingBy()
특정 조건으로 요소들을 그룹화하여 Map 타입으로 반환한다. 예를 들어, 나이, 이름, 성별을 가진 클래스를 나이 기준으로 그룹화 할 수 있다.
### Collectors.partitioningBy()
Predicate로 특정 조건을 받아 해당 조건을 만족하면 true, 아니면 false 그룹으로 구분하여 Map 타입으로 반환한다.

## average[typeName], summing[typeName], summarizing[typeName]
collect 메소드에 사용할 수 있는 메소드이다. 각 기본 타입(int, long, double)별로 제공된다. 아래에서는 Integer를 기준으로 설명한다.
### Collectors.averageingInt()
요소들의 평균을 Integer형으로 반환한다.
### Collectors.summingInt()
요소들의 합을 Integer형으로 반환한다.
### Collectors.summarizingInt()
다양한 연산 결과를 IntSummaryStatistics 형으로 반환한다.
* 제공 메소드: getCount(), getSum(), getAverage(), getMin(), getMax()
이를 이용하면 스트림을 여러 번 생성하지 않고 원하는 결과를 한 번에 모두 만들 수 있다.

## anyMatch(), allMatch(), noneMatch()
Predicate로 특정 조건을 받아 해당 조건을 만족하는지 확인하여 결과를 boolean 값으로 반환한다. 아래 세 가지 메소드로 세분화된다.
* anyMatch: 하나라도 조건을 만족하는 요소가 있는지 확인한다.
~~~java
boolean anyMatch(Predicate<? super T> predicate);
~~~
* allMatch: 모두 조건을 만족하는지 확인한다.
~~~java
boolean allMatch(Predicate<? super T> predicate);
~~~
* noneMatch: 모두 조건을 만족하지 않는지 확인한다.
~~~java
boolean noneMatch(Predicate<? super T> predicate);
~~~

## forEach()
요소를 순회하면서 실행되는 작업이다. 인자로 넘긴 메소드에 요소를 대입하여 호출한다. 주로 System.out::println과 같은 출력 함수를 인자로 넘긴다. 
Intermediate Operator에 속하는 peek와 유사하다고 볼 수 있다.
~~~java
Arrays.asList("Hello", "World", "Hell", "World").stream()
        .forEach(System.out::println);
~~~

# References
* https://velog.io/@kskim/Java-Stream-API