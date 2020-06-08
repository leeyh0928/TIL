# Overview
스프링 빈은 기본적으로 싱글톤이다. 그래서 다른 스코프의 Bean을 접근하려고 할때 문제가 발생된다.
이를 `the scoped bean injection problem`이라고 부른다. 싱글톤 빈에서 프로토타입 빈을 사용할 때 이런 문제점을
해결하는 방법을 알아본다.

# 프로토 타입 빈 주입 문제
> 문제 재현을 위해 싱글톤 빈에서 프로토 타입 빈을 사용하는 예제를 만들어 본다.
~~~java
public class PrototypeBean {
    public PrototypeBean() {
        log.info("Prototype instance created");
    }
}

public class SingletonBean { 
    @Autowired
    private PrototypeBean prototypeBean;
 
    public SingletonBean() {
        log.info("Singleton instance created");
    }
 
    public PrototypeBean getPrototypeBean() {
        log.info(String.valueOf(LocalTime.now()));
        return prototypeBean;
    }
}

@CompnentScan(...)
@Configuration
public class AppConfig {
 
    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public PrototypeBean prototypeBean() {
        return new PrototypeBean();
    }
 
    @Bean
    public SingletonBean singletonBean() {
        return new SingletonBean();
    }
}
~~~
> Main 에서 실행해보자.
~~~java
public class Application {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context 
          = new AnnotationConfigApplicationContext(AppConfig.class);

        PrototypeBean firstPrototype = context.getBean(SingletonBean.class).getPrototypeBean();
        // get singleton bean instance one more time
        PrototypeBean secondPrototype = context.getBean(SingletonBean.class).getPrototypeBean();
     
        isTrue(firstPrototype.equals(secondPrototype), "The same instance should be returned");
    }
} 
~~~
> 실행 콘솔을 확인해보면 `PrototypeBean` 인스턴스의 스코프가 프로토 타입으로 지정되었지만, `getBean()` 호출 시 동일한 인스턴스를 반환하고 있다.
~~~
...
02:11:27.242 [main] INFO com.example.demo.SingletonBean - SingletonBean instance[406765571] created
02:11:27.249 [main] INFO com.example.demo.PrototypeBean - Prototype instance[1640296160] created
...
02:11:27.286 [main] INFO com.example.demo.SingletonBean - 02:11:27.286
02:11:27.286 [main] INFO com.example.demo.SingletonBean - 02:11:27.286
...
~~~

# 문제 해결 방법
## 1. Method Injection
> @Lookup 어노테이션을 이용한 메소드 주입 방법이 있다.
~~~java
@Component
public class SingletonLookupBean {
    @Lookup
    public PrototypeBean getPrototypeBean() {
        return null;
    }
}
~~~
> Spring은 @Lookup 어노테이션이 달린 `getPrototypeBean()` 메서드를 재정의하여 `ApplicationContext`에 등록하고, 
> `getPrototypeBean()` 메서드를 요청할 때마다 `새로운 PrototypeBean 인스턴스`를 반환한다.

## 2. javax.inject API
> javax.inject API 의 `Provider 인터페이스`를 이용하여 프로토 타입 빈을 주입할 수 있다.
~~~java
@Component
public class SingletonProviderBean {
    @Autowired
    private Provider<PrototypeBean> myPrototypeBeanProvider;

    public PrototypeBean getPrototypeInstance() {
        return myPrototypeBeanProvider.get();
    }
}
~~~

## 3. ObjectFactory Interface
> Spring에서 제공되는 `ObjectFactory<T> interface`를 이용하는 방법이다.
~~~java
@Component
public class SingletonObjectFactoryBean {
    @Autowired
    private ObjectFactory<PrototypeBean> prototypeBeanObjectFactory;

    public PrototypeBean getPrototypeInstance() {
        return prototypeBeanObjectFactory.getObject();
    }
}
~~~

## 4. Create a Bean at Runtime Using java.util.Function
> `java.util.Function`을 이용하여 런타임에 프로토 타입 빈을 생성하는 방법이다.
~~~java
public class PrototypeFunctionBean {
    private String name;

    public PrototypeFunctionBean(String name) {
        this.name = name;
        log.info("Prototype instance[" + this.hashCode()+ "] " + name + " created");
    }
}

@Component
public class SingletonFunctionBean {
    @Autowired
    private Function<String, PrototypeFunctionBean> beanFactory;

    public PrototypeFunctionBean getPrototypeInstance(String name) {
        return beanFactory.apply(name);
    }
}

@CompnentScan(...)
@Configuration
public class AppConfig {
    ...
    
    @Bean
    public Function<String, PrototypeFunctionBean> beanFactory() {
        return this::prototypeBeanWithParam;
    }

    @Bean
    @Scope(value = "prototype")
    public PrototypeFunctionBean prototypeBeanWithParam(String name) {
        return new PrototypeFunctionBean(name);
    }
}
~~~

# 테스트
> 문제 해결 방법 중 `ObjectFactory Interface` 방법에 대해 Test 코드로 확인해보자.
~~~java
public class BeanScopeTest {
    ...

    @Test
    public void givenPrototypeInjection_WhenObjectFactory_ThenNewInstanceReturn() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        PrototypeBean firstInstance = context.getBean(SingletonObjectFactoryBean.class).getPrototypeInstance();
        PrototypeBean secondInstance = context.getBean(SingletonObjectFactoryBean.class).getPrototypeInstance();

        assertTrue("New instance expected", firstInstance != secondInstance);
    }
}
~~~
> 실행 콘솔 결과를 보면, `PrototypeBean` 인스턴스를 호출할 때마다 새로 생성된 것을 알 수 있다.
~~~
...
02:32:49.496 [Test worker] INFO com.example.demo.PrototypeBean - Prototype instance[1207332330] created
02:32:49.496 [Test worker] INFO com.example.demo.PrototypeBean - Prototype instance[1243211260] created
...
~~~

전체 코드는 [Bean Scope](https://github.com/leeyh0928/bean-scope)에서 확인할 수 있다.

# References
* https://www.baeldung.com/spring-inject-prototype-bean-into-singleton