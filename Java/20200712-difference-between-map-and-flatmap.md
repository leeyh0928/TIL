# Overview
Java8의 `.map()`과 `.flatMap()`의 사용법을 간단히 살펴보고, `.map()`이 해결하지 못하는 문제를 `.flatMap()`을 사용하여
어떻게 해결하는지 알아보자.

# .map()
`.map()`은 단일 스트림의 원소를 매핑시킨 후 매핑시킨 값을 다시 스트림으로 반환하는 중간 연산을 담당한다.
> Person.java
~~~java
class Person {
    int age;
  	String name;
}
~~~
> MapTest1.java
~~~java
List<Person> sample = Arrays.asList(
  new Person(20, "park");
  new Person(35, "kyung");
  new Person(67, "seok");
  new Person(10, "test man");
  new Person(45, "test woman");
);

//List -> Stream<Person> -> map -> Stream<String>
Stream<String> mapStream = sample.stream().map(person -> person.getName());

mapStream.forEach(System.out::println);
~~~
> output
~~~
park
kyung
seok
test man
test woman
~~~
> MapTest2.java
~~~java
List<Person> sample = Arrays.asList(
  new Person(20, "park");
  new Person(35, "kyung");
  new Person(67, "seok");
  new Person(10, "test man");
  new Person(45, "test woman");
);

//without .map()
Stream<Person> stream = sample.stream()
  .filter(person -> "park".equals(person.getName()));

stream.forEach(person -> System.out.println("without Map: "+person.getName()));

//with .map()
Stream<String> mapStream = sample.stream()
  .map(person -> person.getName())
  .filter(person -> "park".equals(person);

stream2.forEach(System.out::println);
~~~
> output
~~~
park
park
~~~

# .flatMap()
`.flatMap()`은 Array나 Object로 감싸져 있는 모든 원소를 단일 원소 스트림으로 반환해준다.
> flatMapTest1.java

`.flatMap()`을 사용하지 않고, 2차원 배열의 원소 중 a를 찾는 코드
~~~java
String[][] sample = new String[][]{
  {"a", "b"}, {"c", "d"}, {"e", "a"}, {"a", "h"}, {"i", "j"}
};

//without .flatMap()
Stream<String> stream = sample.stream()
  .filter(alpha -> "a".equals(alpha[0].toString() || "a".equals(alpha[1].toString())))
stream.forEach(alpha -> System.out.println("{"+x[0]+", "+x[1]+"}"));
~~~
> output
~~~
{a, b}
{e, a}
{a, h}
~~~
> flatMapTest2.java

`.flatMap()`을 사용하면 단일 원소 스트림으로 리턴 받을 수 있다.
~~~java
String[][] sample = new String[][]{
  {"a", "b"}, {"c", "d"}, {"e", "a"}, {"a", "h"}, {"i", "j"}
};

//without .flatMap()
Stream<String> stream = sample.stream()
  .flatMap(array -> Arrays.stream(array))
  .filter(x-> "a".equals(x));

stream.forEach(System.out::println);
~~~
> output
~~~
a
a
a
~~~