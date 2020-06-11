# Overview
Spring은 기본적으로 Bean의 LifeCycle을 관리하고, 초기화 순서를 정렬한다. 그러나 필요에 따라 `@DependsOn` 어노테이션을 이용하여
Bean 초기화 순서를 관리할 수 있다.

# @DependsOn
Bean 의존성을 지정하기 위해 사용된다. 현재 Bean의 초기화 이전에 지정한 한 Bean이 먼저 초기화될 것을 보장한다.
## e.g. Configuration
`FileReader` 및 `FileWriter`에 의존하는 `FileProcessor`가 있다. 이 경우 `FileReader` 및 `FileWriter`는 
`FileProcessor` 전에 초기화되어야 한다.

~~~java
@Configuration
@ComponentScan("com.baeldung.dependson")
public class Config {
  
    @Bean
    @DependsOn({"fileReader","fileWriter"})
    public FileProcessor fileProcessor(){
        return new FileProcessor();
    }
     
    @Bean("fileReader")
    public FileReader fileReader() {
        return new FileReader();
    }
     
    @Bean("fileWriter")
    public FileWriter fileWriter() {
        return new FileWriter();
    }   
}
~~~
또한, 아래처럼 @Component 어노테이션 사용 시에도 `@DependsOn`으로 종속성을 지정할 수 있다.
~~~java
@Component
@DependsOn({"filereader", "fileWriter"})
public class FileProcessor {
    ...
}
~~~

# Usage
아래 예제의 경우 `fileReader, fileWriter` -> `fileProcessor` 순으로 빈 종속성을 갖는다.
`fileReader` 빈 생성 시 File에 `read`가 출력되고, `fileWriter`는 `write`가 텍스트 출력된다. 로
`fileProcessor`는 File에 `read`와 `write` 텍스트가 있는 경우 `processed`를 출력한다.
## e.g. 사용 예시
~~~java
@Configuration
public class TestConfig {
    @Autowired
    File file;
    
    @Bean("fileProcessor")
    @DependsOn({"fileReader","fileWriter"})
    @Lazy
    public FileProcessor fileProcessor(){
        return new FileProcessor(file);
    }
    
    @Bean("fileReader")
    public FileReader fileReader(){
        return new FileReader(file);
    }
    
    @Bean("fileWriter")
    public FileWriter fileWriter(){
        return new FileWriter(file);
    }
}
~~~
~~~java
@RunWith(SpringExtension.class)
@ContextConfiguration(classes = TestConfig.class)
class FileProcessorIntegrationTest {
    @Autowired
    ApplicationContext context;
    
    @Autowired
    File file;
    
    @Test
    public void whenAllBeansCreated_FileTextEndsWithProcessed() {
        context.getBean("fileProcessor");
        assertTrue(file.getText().endsWith("processed"));
    }
}
~~~
## e.g. Missing Dependency
종속성이 없으면 Spring은 기본 예외 `NoSuchBeanDefinitionException`과 함께 `BeanCreationException`을 발생한다.
아래 예제의 경우 `dummyfileWriter` Bean이 존재하지 않기 때문에 `BeanCreationException`이 발생한다.
~~~java
@Configuration
public class TestConfig {
    ...

    @Bean("dummyFileProcessor")
    @DependsOn({"dummyfileWriter"})
    @Lazy
    public FileProcessor dummyFileProcessor(){
        return new FileProcessor(file);
    }
}
~~~
~~~java
...

@Test(expected=BeanCreationException.class)
public void whenDependentBeanNotAvailable_ThrowsNosuchBeanDefinitionException(){
    context.getBean("dummyFileProcessor");
}
~~~
## e.g. Circular Dependency
Bean이 서로 순환 종속성을 가지고 있기 때문에 이 경우도 `BeanCreationException`이 발생된다.
~~~java
...

@Bean("dummyFileProcessorCircular")
@DependsOn({"dummyFileReaderCircular"})
@Lazy
public FileProcessor dummyFileProcessorCircular(){
    return new FileProcessor(file);
}

@Bean("dummyFileReaderCircular")
@DependsOn({"dummyFileProcessorCircular"})
@Lazy
public FileReader dummyFileReaderCircular(){
    return new FileReader(file);
}

...
~~~
~~~java
...

@Test(expected=BeanCreationException.class)
public void whenCircularDependency_ThrowsBeanCreationException(){
    context.getBean("dummyFileReaderCircular");
}

...
~~~

# 사용시 주의점
* @DependsOn을 사용할 경우 반드시 `component-scanning`이 되어야 한다.
* DependsOn-annotated Class가 XML 방식으로 선언된 경우 `@DependsOn` 어노테이션은 무시된다.
