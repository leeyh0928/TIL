# Overview
***Spring***에서 캐싱 추상화를 사용하는 방법을 알아본다.

# Dependency
~~~groovy
compile group: 'org.springframework.boot', name: 'spring-boot-starter-cache'
~~~

# Enable Caching
캐싱을 활성화하기 위해 구성 클래스 하나에 `@EnableCaching` 어노테이션을 추가하여 선언적으로 활성화 할 수 있다.
~~~java
@Configuration
@EnableCaching
public class CachingConfig {
 
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("addresses");
    }
}
~~~
최소 설정을 위해 캐싱을 활성화 한 후 `cacheManager`를 등록 해야 한다.

`Spring Boot`를 사용할 때 `@EnableCaching` 어노테이션과 함께 클래스 경로에 스타터 패키지만 있으면 동일한 `ConcurrentMapCacheManager`가
등록된다. 또한 하나 이상의 `CacheManagerCustomizer<T> Bean`을 사용하여 자동 구성된  `CacheManager`를  사용자 정의 할 수 있다. 
~~~java
@Component
public class SimpleCacheCustomizer 
  implements CacheManagerCustomizer<ConcurrentMapCacheManager> {
 
    @Override
    public void customize(ConcurrentMapCacheManager cacheManager) {
        cacheManager.setCacheNames(asList("users", "transactions"));
    }
}
~~~

# Use Caching With Annotations
캐싱을 활성화 한 후 다음 단계는 선언적 주석이 있는 메서드에 캐싱 동작을 바인딩하는 것이다.
## 1. @Cacheable
메소드에 대해 캐싱 동작을 사용하는 가장 간단한 방법은 `@Cacheable`로 메소드를 구분하고, 결과가 저장 될 캐시 이름으로 
매개 변수화 하는 것이다.
~~~java
@Cacheable("addresses")
public String getAddress(Customer customer) {...}
~~~
## 2. @CacheEvict
캐시는 상당히 커지고 빠르게 커질 수 있으며 많은 오래된 데이터나 사용하지 않은 데이터를 유지할 수 있다. 캐시에 자주 필요하지 
않은 값을 채우고 싶지 않을 경우 사용할 수 있다.
~~~java
@CacheEvict(value="addresses", allEntries=true)
public String getAddress(Customer customer) {...}
~~~
## 3. @CachePut
항목이 변경될 때 마다 선택적으로 지능적으로 업데이트하려고 한다. @CachePut 어노테이션을 사용하면 메소드 실행을 방해하지 않고 
캐시의 컨텐츠를 업데이트 할 수 있다. 
~~~java
@CachePut(value="addresses")
public String getAddress(Customer customer) {...}
~~~

# References
* https://www.baeldung.com/spring-cache-tutorial