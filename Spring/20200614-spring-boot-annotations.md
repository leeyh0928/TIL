# Overview
Spring Boot는 `auto-configuration` 기능으로 Spring을 보다 쉽게 구성할 수 있다.
`org.springframework.boot.autoconfigure` 및 `org.springframework.boot.autoconfigure.condition` 패키지의
***Annotation*** 에 대해 알아본다.

# @SpringBootApplication
Spring Boot 애플리케이션의 기본 클래스를 표시한다. `@SpringBootApplication`은 기본 속성으로 
***@Configuration , @EnableAutoConfiguration 및 @ComponentScan*** 어노테이션들을 캡슐화하고 있다.
~~~java
@SpringBootApplication
class SpringDemoApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(SpringDemoApplication.class, args);
    }
}
~~~
## @EnableAutoConfiguration
`@EnableAutoConfiguration` 자동 구성을 활성화한다. Spring Boot는 클래스 경로에서 `auto-configuration beans`을 찾아 
자동으로 적용 한다는 것을 의미한다. 이 어노테이션은 `@Configuration`과 함께 사용해야 한다.
~~~java
@Configuration
@EnableAutoConfiguration
class SpringDemoConfig {}
~~~
# Auto-Configuration Conditions
커스텀한 자동 설정을 조건부로 적용하기 위한 `annotation`들로 @Configuration 클래스 또는 @Bean 메소드에 사용할 수 있다.
## 1. @ConditionalOnClass 및 @ConditionalOnMissingClass
***특정 클래스***의 존재 여부에 따라 자동 구성을 정의하는 경우
~~~java
@Configuration
@ConditionalOnClass(DataSource.class)
class MySQLAutoconfiguration { ... }
~~~
## 2. @ConditionalOnBean 및 @ConditionalOnMissingBean
***특정 bean***의 존재 여부에 따라 자동 구성을 정의하는 경우
~~~java
@Bean
@ConditionalOnBean(name = "dataSource")
LocalContainerEntityManagerFactoryBean entityManagerFactory() { ... }
~~~
## 3. @ConditionalOnProperty
***특정 properties의 value***의 상태에 따라 자동 구성을 정의하는 경우
~~~java
@Bean
@ConditionalOnProperty(
    name = "usemysql", 
    havingValue = "local"
)
DataSource dataSource() { ... }
~~~
## 4. @ConditionalOnResource
***특정 리소스***의 존재 여부에 따라 자동 구성을 정의하는 경우
~~~java
@ConditionalOnResource(resources = "classpath:mysql.properties")
Properties additionalProperties() { ... }
~~~
## 5. @ConditionalOnWebApplication 및 @ConditionalOnNotWebApplication
웹 응용 프로그램인지 아닌지에 따른 자동 구성을 정의하는 경우
~~~java
@ConditionalOnWebApplication
HealthCheckController healthCheckController() { ... }
~~~
## 6. @ConditionalExpression
SpEL 표현식이 true인 경우에 따라 자동 구성을 정의하는 경우
~~~java
@Bean
@ConditionalOnExpression("${usemysql} && ${mysqlserver == 'local'}")
DataSource dataSource() { ... }
~~~
## 7. @Conditional
복잡한 상황에 따라 `사용자 정의 조건`을 클래스로 구현하고, @Conditional 어노테이션으로 정의한 `사용자 정의 조건`을 사용하도록
지정할 수 있다.
~~~java
public class HibernateCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // application specific condition check
        return true;
    }
}
~~~
~~~java
@Conditional(HibernateCondition.class)
Properties additionalProperties() { ... }
~~~

# References
* https://www.baeldung.com/spring-boot-annotations