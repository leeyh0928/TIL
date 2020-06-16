# Overview
ClassPath에 존재하는 종속성을 기반으로 스프링 애플리케이션을 자동 구성하는 방법이다.

# Creating a Custom Auto-Configuration
사용자 지정 자동 구성을 만들려면 @Configuration 어노테이션을 이용해 class를 구현한다.
~~~java
@Configuration
public class MySQLAutoConfiguration { ... }
~~~
`resources/META-INF/spring.factories`파일의 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 항목에
정의한 class 이름을 추가하여 자동 구성 후보로 등록한다.
~~~yaml
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfiguration.MySQLAutoconfiguration
~~~
다른 자동 구성 후보보다 우선하려면 class에 `@AutoConfigureOrder (Ordered.HIGHEST_PRECEDENCE)` 어노테이션을 추가할 수 있다.

## 1. Class Conditions
* @ConditionalOnClass : 지정된 클래스가 ***존재하는 경우*** 해당 Bean이 포함되도록 구성
* @ConditionalOnMissingClass : 지정된 클래스가 ***존재하지 않는 경우*** 해당 Bean이 포함되도록 구성
~~~java
@Configuration
@ConditionalOnClass(DataSource.class)
public class MySQLAutoConfiguration { ... }
~~~

## 2. Bean Conditions
* @ConditionalOnBean : 지정된 Bean이 존재하는 경우
* @ConditionalOnMissingBean : 지정된 Bean이 존재하지 않는 경우

entityManagerFactory 빈이 아직 정의되지 않았고, dataSource 빈이 존재하는 경우 
~~~java
@Bean
@ConditionalOnBean(name = "dataSource")
@ConditionalOnMissingBean
public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    LocalContainerEntityManagerFactoryBean em
      = new LocalContainerEntityManagerFactoryBean();
    em.setDataSource(dataSource());
    em.setPackagesToScan("com.baeldung.autoconfiguration.example");
    em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
    if (additionalProperties() != null) {
        em.setJpaProperties(additionalProperties());
    }
    return em;
}
~~~
JpaTransactionManager 유형의 Bean이 아직 정의되지 않은 경우 transactionManager Bean 구성
~~~java
@Bean
@ConditionalOnMissingBean(type = "JpaTransactionManager")
JpaTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
    JpaTransactionManager transactionManager = new JpaTransactionManager();
    transactionManager.setEntityManagerFactory(entityManagerFactory);
    return transactionManager;
}
~~~

## 3. Property Conditions
Properties의 속성에 따른 Bean 구성
~~~java
@Configuration
@PropertySource("classpath:mysql.properties")
public class MySQLAutoConfiguration {...}
~~~
usemysql 속성의 값이 local인 경우 Bean 구성
~~~java
@Bean
@ConditionalOnProperty(
  name = "usemysql", 
  havingValue = "local")
@ConditionalOnMissingBean
public DataSource dataSource() {
    DriverManagerDataSource dataSource = new DriverManagerDataSource();
  
    dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
    dataSource.setUrl("jdbc:mysql://localhost:3306/myDb?createDatabaseIfNotExist=true");
    dataSource.setUsername("mysqluser");
    dataSource.setPassword("mysqlpass");
 
    return dataSource;
}
~~~

## 4. Resource Conditions
지정된 리소스가 있는 경우만 구성이 로드되게 할 수 있다. `mysql.properties`가 존재하는 경우에만
해당 빈이 로드된다.
~~~java
@ConditionalOnResource(
  resources = "classpath:mysql.properties")
@Conditional(HibernateCondition.class)
Properties additionalProperties() {
    Properties hibernateProperties = new Properties();
 
    hibernateProperties.setProperty("hibernate.hbm2ddl.auto", 
      env.getProperty("mysql-hibernate.hbm2ddl.auto"));
    hibernateProperties.setProperty("hibernate.dialect", 
      env.getProperty("mysql-hibernate.dialect"));
    hibernateProperties.setProperty("hibernate.show_sql", 
      env.getProperty("mysql-hibernate.show_sql") != null
      ? env.getProperty("mysql-hibernate.show_sql") : "false");
    return hibernateProperties;
}
~~~ 
## 5. Custom Conditions
사용자 정의 조건을 직접 구현할 수 있다.
~~~java
public class HibernateCondition extends SpringBootCondition {
    private static String[] CLASS_NAMES
      = { "org.hibernate.ejb.HibernateEntityManager", 
          "org.hibernate.jpa.HibernateEntityManager" };
 
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, 
      AnnotatedTypeMetadata metadata) {
  
        ConditionMessage.Builder message
          = ConditionMessage.forCondition("Hibernate");
        return Arrays.stream(CLASS_NAMES)
          .filter(className -> ClassUtils.isPresent(className, context.getClassLoader()))
          .map(className -> ConditionOutcome
            .match(message.found("class")
            .items(Style.NORMAL, className)))
          .findAny()
          .orElseGet(() -> ConditionOutcome
            .noMatch(message.didNotFind("class", "classes")
            .items(Style.NORMAL, Arrays.asList(CLASS_NAMES))));
    }
}
~~~
# Disabling Auto-Configuration Classes
정의된 자동 구성 클래스를 비활성화하는 방법이다. `@EnableAutoConfiguration` 어노테이션의 exclude 속성에
class를 지정하여 비활성화 할 수 있다.
~~~java
@Configuration
@EnableAutoConfiguration(
  exclude={MySQLAutoconfiguration.class})
public class AutoConfigurationApplication { ... }
~~~

# References
* https://www.baeldung.com/spring-boot-custom-auto-configuration