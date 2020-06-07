# Overview
빈 후처리를 이용하면 초기화 콜백 메서드(@Bean의 initMethod 또는 @PostConstruct) 전후에 원하는 로직을
빈에 적용할 수 있다.

# 모든 빈 인스턴스를 처리하는 후처리기 만들기
> 빈 후처리기는 BeanPostProcessor Interface 구현체이다. 스프링은 이 인터페이스를 구현한 빈을 만나면
> 자신이 관장하는 모든 빈 인스턴스에 해당 빈 후처리기를 적용한다.
~~~java
@Component
public class AuditCheckBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) 
        throws BeansException {
        System.out.println("In AuditCheckBeanPostProcessor.postProcessBeforeInitialization, " 
            + "processing bean type: " + bean.getClass());
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
        throws BeansException {
        return bean;
    }
}
~~~

# @Required 로 프로퍼티 검사하기
특정 빈 프로퍼티가 설정되었는지 체크하고 싶은 경우 커스텀 후처리기를 작성하고, 해당 프로퍼티에 @Required 를 붙인다.
`RequiredAnnotationBeanPostProcessor` 는 `@Required`를 붙인 프로퍼티값이 설정됐는지 살핀다.
(단, 설정 여부만 체크할 뿐 그 값이 null 인지, 아니면 다른 값인지 신경쓰지 않는다.)
> `@Required`를 붙인 프로퍼티는 스프링이 감지해서 값의 존재 여부를 조사하고 값이 없으면 `BeanInitializationException`a을 던진다.
~~~java
public class SequenceGenerator {
    private PrefixGenerator prefixGenerator;
    private String suffix;
    ...
    
    @Required
    public void setPrefixGenerator(PrefixGenerator prefixGenerator) {
        this.prefixGenerator = prefixGenerator;
    }
    
    @Required
    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }
}
~~~