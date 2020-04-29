# Intro
> ***Functional interface***는 함수를 ***1급 객체***로 사용할 수 없는 Java 언어의 단점을
> 보완하고자 도입됨. 이 덕분에표 전보다 간결한 표현이 가능해졌으며 가독성이 높아졌다. 

# Functional interface
> (Object 클래스의 메서드를 제외하고) 단 하나의 추상 메소드만을 가진 인터페이를 의미하며,
> 그런 이유로 단 하나의 기능적 제약을 표상하게 된다.

## 1. Functions
> 람다의 가장 단순하고 일반적인 경우인 하나의 값을 받고, 다른 값을 반환하는 메서드를 가진 ***Functional interface***
~~~java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }
    
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    static <T> Function<T, T> identity() {
      return t -> t;
    }
}
~~~
> 예시>
~~~
// as is
Map<String, Integer> nameMap = new HashMap<>();
Integer value = nameMap.computeIfAbsent("John", s -> s.length());

// to be
// 파라메터에 대해 암시적 추론을 통해 생략되고, Functional interface로 캐스트될 수 있다.
Integer value = nameMap.computeIfAbsent("John", String::length);
~~~
## 2. Primitive Function Specializations
- IntFunction, LongFunction, DoubleFunction : 파라메터 타입이 지정되어 있고, 리턴 타입은 지정 가능한 인터페이스
- ToIntFunction, ToLongFunction, ToDoubleFunction : 반대로 리턴 타입이 지정되어 있고, 파라메터 타입을 지정 가능한 인터페이스
- DoubleToIntFunction, DoubleToLongFunction, IntToDoubleFunction, IntToLongFunction, LongToIntFunction, LongToDoubleFunction : 이름처럼 파라메터와 리턴 타입이 지정된 인터페이스
> short 타입을 파라메터로 받고, byte로 리턴하는 Functional interface를 만들어보자.
~~~java
@FunctionalInterface
public interface ShortToByteFunction {
    byte apply(short s);
}

public class TransformArray {

    public byte[] transformArray(short[] array, ShortToByteFunction function) {
        byte[] transformedArray = new byte[array.length];
        for (int i = 0; i < array.length; i++) {
            transformedArray[i] = function.apply(array[i]);
        }
        return transformedArray;
    }
    
    @Test
    public 바이트_배열_변환() {
        short[] array = {(short) 1, (short) 2, (short) 3};
        byte[] transformedArray = transformArray(array, s -> (byte) (s * 2));
         
        byte[] expectedArray = {(byte) 2, (byte) 4, (byte) 6};
        assertArrayEquals(expectedArray, transformedArray);
    }
}
~~~
## 3. Suppliers
> 파라메터가 없고, 리턴 값을 가짐
* 원형
~~~java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
~~~
* 예시
~~~java
public class SupplierTest {
    @Test
    public void test(){
        Supplier<Integer> intSupplier = () -> (int) (Math.random() * 6) + 1;

        int num = intSupplier.get();
        System.out.println(num);
    }
}
~~~
## 4. Consumers
> 공급자와 반대로 파라메터만 있고, 리턴 값이 존재하지 않음
* 원형
~~~java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
    
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
~~~
* 예시
~~~java
public class ConsumerTest {
    @Test
    public void test() {
        andThen();
        accept();
    }

    private void andThen() {
        final Consumer<String> consumer1 = (i) -> System.out.println("consumer1 "+i);
        final Consumer<String> consumer2 = (i) -> System.out.println("consumer2 "+i);

        consumer1.andThen(consumer2).accept("is Consume!!");
    }

    private void accept() {
        final Consumer<String> greetings = value -> System.out.println("Hello " + value);
        greetings.accept("World");  // Hello World
    }
}
~~~
* andThen
> Consumer 인터페이스는 결과를 리턴하지 않기 때문에 함수적 인터페이스의 호출 순서만 정한다.
## 5. Predicates
> 파라메터를 받고, 부울 값을 리턴
* 원형
~~~java
@FunctionalInterface
public interface Predicate<T> {

    boolean test(T t);
    
    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }
    
    default Predicate<T> negate() {
        return (t) -> !test(t);
    }
    
    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }
    
    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef) ? Objects::isNull : object -> targetRef.equals(object);
    }
}
~~~
* 예시
~~~java
public class PredicateTest {
    @Test
    public void test() {
        tests();
        and_or_negate();
    }

    private void tests() {
        Predicate<Integer> isZero = (i) -> i == 0;
        System.out.println(isZero.test(0));
    }

    private void and_or_negate() {
        Predicate<Integer> predicateA = a -> a % 2 == 0;
        Predicate<Integer> predicateB = b -> b % 3 == 0;

        boolean result = predicateA.and(predicateB).test(9);
        System.out.println("9는 2와 3의 배수입니까? " + result);

        result = predicateA.or(predicateB).test(9);
        System.out.println("9는 2또는 3의 배수입니까? " + result);

        result = predicateA.negate().test(9);
        System.out.println("9는 홀수입니까? " + result);
    }
}
~~~
* and
> 파라메터로 받은 다른 Predicate 람다 조건식과 함께 && 연산
* or
> 파라메터로 받은 다른 Predicate 람다 조건식과 함께 || 연산
* negate
> 람다 조건식 결과애 !(not) 연산
# References
* https://www.baeldung.com/java-8-functional-interfaces