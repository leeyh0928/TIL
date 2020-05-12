# Overview
종합적인 트랜잭션 관리는 스프링 프레임워크를 사용하는 가장 중요한 이유 중 하나이다. 스프링 프레임워크는 다음과 같은
장점을 지닌 트랜잭션 관리를 위한 추상화를 제공한다.
* 자바 트랜잭션 API(JTA), JDBC, 하이버네이트, JPA, JDO와 같은 서로 다른 트랜잭션을 아우르는 일관된 프로그래밍 모델
* 선언적 트랜잭션 관리
* JTA와 같은 복잡한 트랜잭션 API보다 간편한 프로그래밍 방식의 트랜잭션 관리 API
* 스프링 데이터 접근 추상화를 통한 훌륭한 통합

# 스프링 프레임워크 트랜잭션 추상화의 이해
스프링의 트랜잭션 추상화를 위한 핵심 인터페이스이다. 모든 스프링의 트랜잭션 기능과 코드는 이 인터페이스를 통해서
로우레벨의 트랜잭션 서비스를 이용할 수 있다.
~~~java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
    
    void commit(TransactionStatus status) throws TransactionException;
    
    void rollback(TransactionStatus status) throws TransactionException;
}
~~~
이 인터페이스는 애플리케이션 코드에 프로그래밍 방식으로도 사용 가능하지만, 기본적으로 서비스 제공 인터페이스(SPI)이다.
PlatformTransactionManager는 하나의 인터페이스이기 때문에 필요에 따라 Mock 또는 Stub 될 수 있다.
JNDI와 같은 룩업 전략에 구애받지 않고, PlatformTransactionManager의 구현체는 스프링 loC 컨테이너 안의 다른 객체(빈)들과 같은
방식으로 정의된다. 이런 장점은 JTA를 사용할 때조차도 유효하다.

PlatformTransactionManager의 모든 메소드에서 발생할 수 있는 TransactionException은 unchecked(런타임) 예외이다.
트랜잭션 인프라의 에러는 대부분 치명적이지만, 애플리케이션 코드가 트랜잭션 실패를 복구할 수 있는 경우에는 TransactionException을
캐치할 것인지 선택할 수 있다. 여기서 중요한 점은 개발자에게 트랜잭션 예외 처리를 강제하지 않고 선택권을 준다는 것이다.

***TransactionDefinition*** 인터페이스는 다음 사항들을 명시한다.
* Isolation: 이 트랜잭션이 다른 트랜잭션으로 부터 격리되는 정도. 
예) 이 트랜잭션이 다른 트랜잭션의 커밋되지 않은 쓰기 작업을 볼수 있는가? (READ UNCOMMITTED)
* Propagation: 이미 트랜잭션이 존재할 때 트랜잭션 관련 메서드가 실행될 경우 트랜잭션을 어떻게 묶을 것인지 선택할 수 있다.
* Timeout: 트랜잭션 인프라에 의하여 트랜잭션 수행이 자동 타임아웃-롤백 될 때까지 얼마만큼의 시간이 주어지는가?
* ReadOnly: 읽기 전용 트랜잭션은 트랜잭션이 데이터를 읽기는 하지만 수정할 수 없도록 할 때 사용한다.

***TransactionStatus*** 인터페이스는 트랜잭션 코드가 트랜잭션 실행과 쿼리 트랜잭션 상태를 제어하기 위한 쉬운 방법을 제공한다.
~~~java
public interface TransactionStatus extends SavepointManager {
    boolean isNewTransaction();

    boolean hasSavepoint();

    void setRollbackOnly();

    boolean isRollbackOnly();

    void flush();

    boolean isCompleted();
}
~~~

# 트랜잭션 매니저의 종류
***DataSourceTransactionManager***

DataSource Connection의 트랜잭션 API를 이용해서 트랜잭션을 관리해주는 트랜잭션 매니저이다. 

***JtaTransactionManager***

분산 트랜잭션 처리를 지원하기 위해 제공한다. JtaTransactionManager는 트랜잭션 관리자 구현체를
주입받아 공통의 트랜잭션 처리 흐름을 구성할 수 있게 한다.

![Image of transaction](https://github.com/leeyh0928/TIL/blob/master/Spring/raws/환경별_transaction_처리를_위한_구성_요소.png)

# References
* https://parkcheolu.tistory.com/35#recentEntries
* https://d2.naver.com/helloworld/5812258

