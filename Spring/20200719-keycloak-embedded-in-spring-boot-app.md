# Overview
`Keycloak`은 RedHat에서 관리하고 JBoss에서 Java로 개발 한 ***오픈 소스 ID 및 액세스 관리 솔루션***이다.
`Spring Boot 애플리케이션`에 내장 된 Keycloak 서버를 설정하는 방법을 알아본다.

Keycloak은 독립형 서버로 실행할 수도 있지만, ***관리 콘솔을 통해 다운로드하고 설정***해야 한다.

# Keycloak Pre-Configuration
서버에는 일련의 영역이 있으며 각 영역은 사용자 관리를 위한 격리 된 단위로 작동한다. 사전 구성하려면 영역 정의 파일을 JSON 형식으로
지정해야 한다.

[Keycloak 관리 콘솔](https://www.keycloak.org/docs/6.0/server_admin/#admin-console)을 사용하면 JSON 형식 파일로 저장 및 관리 된다.
파일에서 몇 가지 관련 구성을 살펴보자.
* users : 기본 사용자는 john@test.com 및 mike@other.com이다. 여기에도 자격증명이 있다.
* clients : id를 newClient로 클라이언트를 정의할 것이다.
* standardFlowEnabled : newClient에 대한 인증 코드 흐름을 활성화하려면 true로 설정한다.
* redirectUris : 인증에 성공한 후 서버가 리디렉션 할 newClient의 URL을 여기에 나열한다.
* webOrigins : redirectUris로 나열된 모든 URL에 대한 `CORS 지원`을 허용 하려면 "+" 로 설정한다.

Keycloak 서버는 기본적으로 JWT 토큰을 발행하므로 별도의 추가적인 구성이 필요하지 않다.

# Maven Configuration
~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>        
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
 
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jackson2-provider</artifactId>
    <version>3.11.2.Final</version>
</dependency>
 
<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-dependencies-server-all</artifactId>
    <version>10.0.1</version>
    <type>pom</type>
</dependency> 
~~~

# Embedded Keycloak Configuration
> 권한 서버의 스프링 구성을 정의해보자.
~~~java
@Configuration
public class EmbeddedKeycloakConfig {
 
    @Bean
    ServletRegistrationBean keycloakJaxRsApplication(
      KeycloakServerProperties keycloakServerProperties, DataSource dataSource) throws Exception {
        
        mockJndiEnvironment(dataSource);
        EmbeddedKeycloakApplication.keycloakServerProperties = keycloakServerProperties;
        ServletRegistrationBean servlet = new ServletRegistrationBean<>(
          new HttpServlet30Dispatcher());
        servlet.addInitParameter("javax.ws.rs.Application", 
          EmbeddedKeycloakApplication.class.getName());
        servlet.addInitParameter(ResteasyContextParameters.RESTEASY_SERVLET_MAPPING_PREFIX,
          keycloakServerProperties.getContextPath());
        servlet.addInitParameter(ResteasyContextParameters.RESTEASY_USE_CONTAINER_FORM_PARAMS, 
          "true");
        servlet.addUrlMappings(keycloakServerProperties.getContextPath() + "/*");
        servlet.setLoadOnStartup(1);
        servlet.setAsyncSupported(true);
        return servlet;
    }
 
    @Bean
    FilterRegistrationBean keycloakSessionManagement(
      KeycloakServerProperties keycloakServerProperties) { 
        FilterRegistrationBean filter = new FilterRegistrationBean<>();
        filter.setName("Keycloak Session Management");
        filter.setFilter(new KeycloakSessionServletFilter());
        filter.addUrlPatterns(keycloakServerProperties.getContextPath() + "/*");
        return filter;
    }
 
    private void mockJndiEnvironment(DataSource dataSource) throws NamingException {		 
        NamingManager.setInitialContextFactoryBuilder(
          (env) -> (environment) -> new InitialContext() {
            @Override
            public Object lookup(Name name) {
                return lookup(name.toString());
            }
	
            @Override
            public Object lookup(String name) {
                if ("spring/datasource".equals(name)) {
                    return dataSource;
                }
                return null;
            }
 
            @Override
            public NameParser getNameParser(String name) {
                return CompositeName::new;
            }
 
            @Override
            public void close() {
            }
        });
    }
}
~~~
먼저 영역 정의 파일에 지정된대로 `Keycloak` 특성을 지속적으로 저장하기 위해 `KeycloakServerProperties`를 사용하여 `Keycloak`을
***JAX-RS 애플리케이션으로 구성***했다. 그런 다음 세션 관리 필터를 추가하고 JNDI 환경을 Mocking 하여 메모리 내 H2 데이터베이스 인 
`spring/datasource`를 사용하였다.

# KeycloakServerProperties
> `KeycloakServerProperties`를 살펴보자.
~~~java
@Setter
@Getter
@ConfigurationProperties(prefix = "keycloak.server")
public class KeycloakServerProperties {
    String contextPath = "/auth";
    String realmImportFile = "baeldung-realm.json";
    AdminUser adminUser = new AdminUser();
 
    @Setter
    @Getter
    public static class AdminUser {
        String username = "admin";
        String password = "admin";
    }
}
~~~
이 것은 ***contextPath, adminUser 및 영역 정의 파일을 설정***하는 간단한 POJO 이다.

# EmbeddedKeycloakApplication
> 설정 한 구성을 사용하여 영역을 생성하는 클래스를 살펴보자.
~~~java
public class EmbeddedKeycloakApplication extends KeycloakApplication {
    private static final Logger LOG = LoggerFactory.getLogger(EmbeddedKeycloakApplication.class);
    static KeycloakServerProperties keycloakServerProperties;
 
    protected void loadConfig() {
        JsonConfigProviderFactory factory = new RegularJsonConfigProviderFactory();
        Config.init(factory.create()
          .orElseThrow(() -> new NoSuchElementException("No value present")));
    }
    public EmbeddedKeycloakApplication() {
        createMasterRealmAdminUser();
        createBaeldungRealm();
    }
 
    private void createMasterRealmAdminUser() {
        KeycloakSession session = getSessionFactory().create();
        ApplianceBootstrap applianceBootstrap = new ApplianceBootstrap(session);
        AdminUser admin = keycloakServerProperties.getAdminUser();
        try {
            session.getTransactionManager().begin();
            applianceBootstrap.createMasterRealmUser(admin.getUsername(), admin.getPassword());
            session.getTransactionManager().commit();
        } catch (Exception ex) {
            LOG.warn("Couldn't create keycloak master admin user: {}", ex.getMessage());
            session.getTransactionManager().rollback();
        }
        session.close();
    }
 
    private void createBaeldungRealm() {
        KeycloakSession session = getSessionFactory().create();
        try {
            session.getTransactionManager().begin();
            RealmManager manager = new RealmManager(session);
            Resource lessonRealmImportFile = new ClassPathResource(
              keycloakServerProperties.getRealmImportFile());
            manager.importRealm(JsonSerialization.readValue(lessonRealmImportFile.getInputStream(),
              RealmRepresentation.class));
            session.getTransactionManager().commit();
        } catch (Exception ex) {
            LOG.warn("Failed to import Realm json file: {}", ex.getMessage());
            session.getTransactionManager().rollback();
        }
        session.close();
    }
}
~~~
먼저 `JsonConfigProviderFactory`의 Bean 서브 클래스를 사용하여 `Keycloak`의 서버 구성 `keycloak-server.json`을 로드했다.
> public class RegularJsonConfigProviderFactory extends JsonConfigProviderFactory { }

그런 다음 `KeycloakApplication`을 확장하여 master 및 baeldung 이라는 두 가지 영역을 만들었다. 이들은 영역 정의 파일
`baeldung-realm.json`에 지정된 특성에 따라 작성된다.

또한 `org.keycloak.common.util.ResteasyProvider` 및 `org.keycloak.platform.PlatformProvider`를 자체 구현 하고 
외부 종속성에 의존하지 않도록 ***몇 가지 사용자 지정 공급자***가 필요하다.
이러한 사용자 정의 제공자에 대한 정보는 `프로젝트의 META-INF/services` 폴더에 포함되어 런타임 시 선택되도록 해야한다.

# Bringing it All Together
`Keycloak`은 애플리케이션 측에서 필요한 구성을 훨씬 단순화 했다. 프로그래밍 방식으로 데이터 소스 또는 보안 구성을 정의할 필요가 없다.
이를 모두 합치려면 `Spring과 Spring Boot Application`에 대한 구성을 정의해야 한다.
## 1. application.yml
~~~yaml
server:
  port: 8083
 
spring:
  datasource:
    username: sa
    url: jdbc:h2:mem:testdb
 
keycloak:
  server:
    contextPath: /auth
    adminUser:
      username: bael-admin
      password: ********
    realmImportFile: baeldung-realm.json
~~~
## 2. Spring Boot Application
~~~java
@SpringBootApplication(exclude = LiquibaseAutoConfiguration.class)
@EnableConfigurationProperties(KeycloakServerProperties.class)
public class AuthorizationServerApp {
    private static final Logger LOG = LoggerFactory.getLogger(AuthorizationServerApp.class);
    
    public static void main(String[] args) throws Exception {
        SpringApplication.run(AuthorizationServerApp.class, args);
    }
 
    @Bean
    ApplicationListener<ApplicationReadyEvent> onApplicationReadyEventListener(
      ServerProperties serverProperties, KeycloakServerProperties keycloakServerProperties) {
        return (evt) -> {
            Integer port = serverProperties.getPort();
            String keycloakContextPath = keycloakServerProperties.getContextPath();
            LOG.info("Embedded Keycloak started: http://localhost:{}{} to use keycloak", 
              port, keycloakContextPath);
        };
    }
}
~~~
`KeycloakServerProperties 구성`을 사용하여 `ApplicationListener Bean`에 삽입 할 수 있다. 이 클래스를 실행 한 후 
***인증 서버의 시작 페이지(http://localhost:8083/auth/)에 액세스*** 할 수 있다.

# References
* https://www.baeldung.com/keycloak-embedded-in-spring-boot-app
* https://github.com/Baeldung/spring-security-oauth/tree/master/oauth-rest/oauth-authorization-server