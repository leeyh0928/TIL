# Overview
스프링 표현 언어 (SpEL)는 런타임에 객체 그래프를 쿼리하고 조작하는 것을 지원하는 강력한 표현 언어이다. 
XML 또는 Annotation 기반 스프링 구성에서 함께 사용할 수 있다.

| 유형        |   연산자 |
|-------      |---------------------------------------------|
|Arithmetic   |+, -, *, /, %, ^, div, mod                   |
|Relational   |<, >, ==, !=, <=, >=, lt, gt, eq, ne, le, ge |
|Logical      |and, or, not, &&, ||, !                      |
|Conditional  |?:                                           |
|Regex        |matches                                      |

# 연산자
## 1. 산술 연산자
모든 기본 산술 연산자를 지원한다.
~~~java
@Value("#{19 + 1}") // 20
private double add; 
 
@Value("#{'String1 ' + 'string2'}") // "String1 string2"
private String addString; 
 
@Value("#{20 - 1}") // 19
private double subtract;
 
@Value("#{10 * 2}") // 20
private double multiply;
 
@Value("#{36 / 2}") // 19
private double divide;
 
@Value("#{36 div 2}") // 18, the same as for / operator
private double divideAlphabetic; 
 
@Value("#{37 % 10}") // 7
private double modulo;
 
@Value("#{37 mod 10}") // 7, the same as for % operator
private double moduloAlphabetic; 
 
@Value("#{2 ^ 9}") // 512
private double powerOf;
 
@Value("#{(2 + 2) * 2 + 9}") // 17
private double brackets;
~~~
## 2. 관계형 및 논리 연산자
~~~java
@Value("#{1 == 1}") // true
private boolean equal;
 
@Value("#{1 eq 1}") // true
private boolean equalAlphabetic;
 
@Value("#{1 != 1}") // false
private boolean notEqual;
 
@Value("#{1 ne 1}") // false
private boolean notEqualAlphabetic;
 
@Value("#{1 < 1}") // false
private boolean lessThan;
 
@Value("#{1 lt 1}") // false
private boolean lessThanAlphabetic;
 
@Value("#{1 <= 1}") // true
private boolean lessThanOrEqual;
 
@Value("#{1 le 1}") // true
private boolean lessThanOrEqualAlphabetic;
 
@Value("#{1 > 1}") // false
private boolean greaterThan;
 
@Value("#{1 gt 1}") // false
private boolean greaterThanAlphabetic;
 
@Value("#{1 >= 1}") // true
private boolean greaterThanOrEqual;
 
@Value("#{1 ge 1}") // true
private boolean greaterThanOrEqualAlphabetic;
~~~
## 3. 조건부 연산자
~~~java
@Value("#{2 > 1 ? 'a' : 'b'}") // "a"
private String ternary;

@Value("#{someBean.someProperty != null ? someBean.someProperty : 'default'}")
private String ternary;
~~~
## 4. 정규식
~~~java
@Value("#{'100' matches '\\d+' }") // true
private boolean validNumericStringResult;
 
@Value("#{'100fghdjf' matches '\\d+' }") // false
private boolean invalidNumericStringResult;
 
@Value("#{'valid alphabetic string' matches '[a-zA-Z\\s]+' }") // true
private boolean validAlphabeticStringResult;
 
@Value("#{'invalid alphabetic string #$1' matches '[a-zA-Z\\s]+' }") // false
private boolean invalidAlphabeticStringResult;
 
@Value("#{someBean.someValue matches '\d+'}") // true if someValue contains only digits
private boolean validNumericValue;
~~~
## 5. List 및 Map 객체 액세스
List와 Map 형태의 Bean을 만든다.
~~~java
@Component("workersHolder")
public class WorkersHolder {
    private List<String> workers = new LinkedList<>();
    private Map<String, Integer> salaryByWorkers = new HashMap<>();
 
    public WorkersHolder() {
        workers.add("John");
        workers.add("Susie");
        workers.add("Alex");
        workers.add("George");
 
        salaryByWorkers.put("John", 35000);
        salaryByWorkers.put("Susie", 47000);
        salaryByWorkers.put("Alex", 12000);
        salaryByWorkers.put("George", 14000);
    }
    ...
}
~~~
SpEL을 사용하여 컬렉션의 값에 액세스한다.
~~~java
@Value("#{workersHolder.salaryByWorkers['John']}") // 35000
private Integer johnSalary;
 
@Value("#{workersHolder.salaryByWorkers['George']}") // 14000
private Integer georgeSalary;
 
@Value("#{workersHolder.salaryByWorkers['Susie']}") // 47000
private Integer susieSalary;
 
@Value("#{workersHolder.workers[0]}") // John
private String firstWorker;
 
@Value("#{workersHolder.workers[3]}") // George
private String lastWorker;
 
@Value("#{workersHolder.workers.size()}") // 4
private Integer numberOfWorkers;
~~~

# References
* https://www.baeldung.com/spring-expression-language