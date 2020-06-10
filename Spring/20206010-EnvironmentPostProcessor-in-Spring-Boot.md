# Overview
`EnvironmentPostProcessor`를 이용하면 ApplicationContext가 갱신되기 전에 Application의 Environment를
커스터마이즈 할 수 있다. 이에 대해 알아본다.

# Spring Environment
Spring의 환경 추상화는 현재 동작 중인 애플리케이션의 환경을 말한다. Properties 파일, JVM 시스템 Properties, 시스템 환경변수,
서블릿 컨텍스트 파라매터 등 다양한 환경 소스에 엑세스하는 방식을 통하는 경향이 있다.
따라서 대부분의 경우 Environment를 커스터마이징하면 Bean에 노출되기 전에 다양한 특성을 조작할 수 있다.

# 예제 구현해보기
아래 환경변수를 읽어서
~~~ 
calculation_mode=GROSS 
gross_calculation_tax_rate=0.15
~~~ 
다음과 같이 환경변수를 변경한다.
~~~properties
com.custom.environmentpostprocessor.calculation.mode=GROSS
com.custom.environmentpostprocessor.gross.calculation.tax.rate=0.15
~~~

## 1. EnvironmentPostProcessor 구현
> 구현을 간단히 설명하자면 key 값을 원하는 형태로 RENAME 하고, Environment에서 `calculation_mode` 와 `gross_calculation_tax_rate` 값을 읽어서,
> key, value 형식인 Map 저장한 후 해당 Map을 PropertySource에 추가한다.  
~~~java
@Slf4j
@Order
public class PriceCalculationEnvironmentPostProcessor implements EnvironmentPostProcessor {
    ...

    private static final String PREFIX = "com.custom.environmentpostprocessor.";    
        
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        PropertySource<?> system = environment.getPropertySources()
                .get(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME);

        Map<String, Object> prefixed;
        if (!hasOurPriceProperties(Objects.requireNonNull(system))) {
            log.warn("System environment variables [calculation_mode,gross_calculation_tax_rate] not detected, fallback to default value [calcuation_mode={},gross_calcuation_tax_rate={}]",
                    CALCUATION_MODE_DEFAULT_VALUE, GROSS_CALCULATION_TAX_RATE_DEFAULT_VALUE);

            prefixed = names.stream()
                    .collect(Collectors.toMap(this::rename, this::getDefaultValue));

            environment.getPropertySources()
                    .addAfter(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, new MapPropertySource("prefixer", prefixed));
            return;
        }

        prefixed = names.stream()
                .collect(Collectors.toMap(this::rename, system::getProperty));

        environment.getPropertySources()
                .addAfter(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, new MapPropertySource("prefixer", prefixed));
    }
    
    private String rename(String key) {
        return PREFIX + key.replaceAll("\\_", ".");
    }                                                 

    private boolean hasOurPriceProperties(PropertySource<?> system) {
        if (system.containsProperty(CALCUATION_MODE) && system.containsProperty(GROSS_CALCULATION_TAX_RATE)) {
            return true;
        } 
        return false;
    }
    ...                                                      
}
~~~

## 2. spring.factories 등록
`Spring Boot bootstrap process`에서 구현을 invoke 하기 위해서는 `META-INF/spring.factories`에 class를 등록해야한다.
~~~properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.customenvironment.autoconfig.PriceCalculationAutoConfig

org.springframework.boot.env.EnvironmentPostProcessor=\
com.example.customenvironment.PriceCalculationEnvironmentPostProcessor
~~~


## 3. 테스트 및 검증
* `@Value Annotation`을 사용하여 Properties 액세스
~~~java
public class GrossPriceCalculator implements PriceCalculator {
    @Value("${com.custom.environmentpostprocessor.gross.calculation.tax.rate}")
    double taxRate;
 
    @Override
    public double calculate(double singlePrice, int quantity) {
        //calcuation implementation omitted
    }
}
~~~
* Spring Boot `Auto-configuration`을 통한 Properties 액세스
~~~java
@Configuration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
public class PriceCalculationAutoConfig {
    @Bean
    @ConditionalOnProperty(name = "com.custom.environmentpostprocessor.calculation.mode", havingValue = "NET")
    @ConditionalOnMissingBean
    public PriceCalculator getNetPriceCalculator() {
        return new NetPriceCalculator();
    }

    @Bean
    @ConditionalOnProperty(name = "com.custom.environmentpostprocessor.calculation.mode", havingValue = "GROSS")
    @ConditionalOnMissingBean
    public PriceCalculator getGrossPriceCalculator() {
        return new GrossPriceCalculator();
    }
}
~~~
* 서비스 구현
~~~java
@Service
@RequiredArgsConstructor
public class PriceCalculationService {
    private final PriceCalculator priceCalculator;

    public double productTotalPrice(double singlePrice, int quantity) {
        return priceCalculator.calculate(singlePrice, quantity);
    }
}
~~~
* 단위 테스트 구현
~~~java
@ExtendWith(SpringExtension.class)
@SpringBootTest(classes = CustomEnvironmentApplication.class)
class PriceCalculationEnvironmentPostProcessorTest {
    @Autowired
    private PriceCalculationService pcService;

    @Test
    public void whenSetNetEnvironmentVariableByDefault_thenNoTaxApplied() {
        double total = pcService.productTotalPrice(100, 4);
        assertEquals(400.0, total, 0);
    }
}
~~~

전체 소스는 [custom-environment]()에서 확인할 수 있다.

#References
* https://www.baeldung.com/spring-boot-environmentpostprocessor