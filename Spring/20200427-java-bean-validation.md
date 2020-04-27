#Intro
>표준 프레임워크 인 Bean Validation 2.0으로 알려진 JSR380을 사용하여 Java Bean 유효성 검사의 기본사항을 살펴본다.

#JSR380
>Jakarta EE 및 JavaSE의 일부인 Bean 유효 검증을 위한 Java API 스펙으로 @NotNull, @Min, @Max 등의 어노테이을 사용하여 Bean이 특정 기준을 충존하는지 확인한다.

#Dependencies
##Validation Api
>JSR389 기반의 표준 유효성 검사 API
~~~
implementation('javax.validation:validation-api:2.0.0.Final')
~~~
##Hibernate Validator
>Validation Api의 참조 구현물로 spring-boot-starter-data-jpa 모듈에 포함되어 있다.
~~~
implementation('org.hibernate.validator:hibernate-validator:6.0.2.Final')
annotationProcessor('org.hibernate.validator:hibernate-validator-annotation-processor:6.0.2.Final')
~~~
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
- @NotNull - 값이 Null이 아닌지 검증
- @AssertTrue - 값이 true인지 검증
- @Size – 값이 min과 max 사이의 크기인지 검증. 문자열, 컬렉션, Map 및 배열에 사용 가능
- @Min - 값이 @Min의 value 속성 값보다 작지 않은지 검증
- @Max - 값이 @Max의 value 속성 값보다 크지 않은지 검증
- @Email - 값이 유효한 이메일 주소인지 검증
>message 속성은 해당 값이 유효하지 않을 경우 렌더링되는 메세지
## 그 밖에 Annotations
- @NotEmpty – 값이 null이거나 비어 있지 않은지 검증. 문자열, 컬렉션, Map 또는 배열에 사용 가능
- @NotBlank – String에만 적용 할 수 있으며 값이 null 또는 공백이 아닌지 확인
- @Positive 및 @PositiveOrZero – 숫자 값에 적용 가능. 양수인지 또는 0을 포함하여 양수인지 검증
- @Negative 및 @NegativeOrZero – 숫자 값에 적용 가능. 음수인지 또는 0을 포함하여 음수인지 검증
- @Past 및 @PastOrPresent – 날짜 값이 과거인지 또는 현재를 포함하여 과거인지 검증
- @Future 및 @FutureOrPresent – 날짜 값이 미래인지 또는 현재를 포함하여 미래인지 검증
## Collection 요소 및 Optional 지원
~~~
List<@NotBlank String> preferences; // 컬렉션의 elements에 대한 유효성 검증
~~~
~~~
private LocalDate dateOfBirth;

// Optional의 값 검증
public Optional<@Past LocalDate> getDateOfBirth() {
    return Optional.of(dateOfBirth);
}
~~~
#Programmatic validation
~~~java
public class ValidationTest {
    private static ValidatorFactory validatorFactory;
    private static Validator validator;
    
    @BeforeClass
    public static void createValidator() {
        validatorFactory = Validation.buildDefaultValidatorFactory();
        validator = validatorFactory.getValidator();
    }
    
    @Test
    public void validUserTest() {
        User user = new User();
        user.setWorking(true);
        user.setAboutMe("Its all about me!");
        user.setAge(50);

        // 유효하지 않은 결과 집합
        Set<ConstraintViolation<User>> violations = validator.validate(user);
        
        // 유효하지 않은 속성의 정의된 message 출력
        for (ConstraintViolation<User> violation : violations) {
            log.error(violation.getMessage()); 
        }
    }

    @AfterClass
    public static void close() {
        validatorFactory.close();
    }
}
~~~
#Reference
- https://www.baeldung.com/javax-validation