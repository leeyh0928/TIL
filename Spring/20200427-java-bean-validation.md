#Intro
>표준 프레임워크 인 Bean Validation 2.0으로 알려진 JSR380을 사용하여 Java Bean 유효성 검사의 기본사항을 살펴본다.

#JSR380
>Jakarta EE 및 JavaSE의 일부인 Bean 유효 검증을 위한 Java API 스펙으로 @NotNull, @Min, @Max 등의 어노테이을 사용하여 Bean이 특정 기준을 충존하는지 확인한다.

#Dependencies
##Validation Api
>JSR389 기반의 표준 유효성 검사 API
````
implementation('javax.validation:validation-api:2.0.0.Final')
````
##Hibernate Validator
>Validation Api의 참조 구현물로 spring-boot-starter-data-jpa 모듈에 포함되어 있다.
````
implementation('org.hibernate.validator:hibernate-validator:6.0.2.Final')
annotationProcessor('org.hibernate.validator:hibernate-validator-annotation-processor:6.0.2.Final')
````
#Using Validation Annotations
~~~java
public class User {
    @NotNull(message = "cannot be null")
    private String name;
    
    @AssertTrue
    private boolean working;

    @Size(min = 10, max = 200, message 
          = "About Me must be between 10 and 200 characters")
    private String aboutMe;

    @Min(value = 18, message = "Age should not be less than 18")
    @Max(value = 150, message = "Age should not be greater than 150")
    private int age;

    @Email(message = "should be valid")
    private String email;
}
~~~
- @NotNull - 값이 Null이 아닌지 확인
- @AssertTrue - 값이 true인지 확인
- @Size – 값이 min과 max 사이의 크기인지 검증합니다. 문자열, 컬렉션, Map 및 배열에 사용 가