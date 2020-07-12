# Overview
Mockito 라이브러리는 BDD 호환 API를 도입 한 BDDMockito 클래스와 함께 제공된다. given()을 사용하여 테스트를 정렬하고 then()을 사용하여 
어설션을 작성하는 더 BDD 친화적인 접근 방식을 취할 수 있다.

BDD 기반 Mockito 테스트를 설정하는 방법과 Mockito 와 BDDMockito API의 차이점에 대해 알아본다.

# Setup
## 1. Dependency
~~~groovy
testCompile group: 'org.mockito', name: 'mockito-core', version: '2.16.0'
~~~
## 2. Imports
~~~java
import static org.mockito.BDDMockito.*;
~~~

# Mockito vs. BDDMockito
> 전통적인 Mockito를 사용하는 테스트 본문의 예를 살펴보자.
~~~java
when(phoneBookRepository.contains(momContactName))
  .thenReturn(false);
 
phoneBookService.register(momContactName, momPhoneNumber);
 
verify(phoneBookRepository)
  .insert(momContactName, momPhoneNumber);
~~~
> BDDMockito와 비교 해보자.
~~~java
given(phoneBookRepository.contains(momContactName))
  .willReturn(false);
 
phoneBookService.register(momContactName, momPhoneNumber);
 
then(phoneBookRepository)
  .should()
  .insert(momContactName, momPhoneNumber);
~~~

# Mocking With BDDMockito
`PhoneBookRepository`를 Mocking 해야하는 `PhoneBookService`를 테스트해보자.
~~~java
public class PhoneBookService {
    private PhoneBookRepository phoneBookRepository;
 
    public void register(String name, String phone) {
        if(!name.isEmpty() && !phone.isEmpty()
          && !phoneBookRepository.contains(name)) {
            phoneBookRepository.insert(name, phone);
        }
    }
 
    public String search(String name) {
        if(!name.isEmpty() && phoneBookRepository.contains(name)) {
            return phoneBookRepository.getPhoneNumberByContactName(name);
        }
        return null;
    }
}
~~~
`BDDMockito`를 사용하면 고정되거나 동적일 수 있는 값을 반환 할 수 있고, 예외 또한 던질 수 있다.
## 1. 고정 값 반환
`BDDMockito`를 사용하여 모의 객체 대상 메소드가 호출 될 때마다 고정 결과를 반환하도록 Mockito를 쉽게 구성할 수 있다.
~~~java
given(phoneBookRepository.contains(momContactName))
  .willReturn(false);
 
phoneBookService.register(xContactName, "");
 
then(phoneBookRepository)
  .should(never())
  .insert(momContactName, momPhoneNumber);
~~~
## 2. 동적 값 반환
`BDDMockito`를 사용하면 보다 정교하게 값을 반환 할 수 있다. 입력에 따라 동적 결과를 반환 할 수도 있다.
~~~java
given(phoneBookRepository.contains(momContactName))
  .willReturn(true);
given(phoneBookRepository.getPhoneNumberByContactName(momContactName))
  .will((InvocationOnMock invocation) ->
    invocation.getArgument(0).equals(momContactName) 
      ? momPhoneNumber 
      : null);
phoneBookService.search(momContactName);
then(phoneBookRepository)
  .should()
  .getPhoneNumberByContactName(momContactName);
~~~
## 3. 예외 던지기
`BDDMockito`를 사용하면 쉽게 예외를 던지도록 할 수 있다.
~~~java
given(phoneBookRepository.contains(xContactName))
  .willReturn(false);

willThrow(new RuntimeException())
  .given(phoneBookRepository)
  .insert(any(String.class), eq(tooLongPhoneNumber));
 
try {
    phoneBookService.register(xContactName, tooLongPhoneNumber);
    fail("Should throw exception");
} catch (RuntimeException ex) { }
 
then(phoneBookRepository)
  .should(never())
  .insert(momContactName, tooLongPhoneNumber);
~~~

# References
* https://www.baeldung.com/bdd-mockito