# Overview
***Spring Data MongoDB***에 대해 알아본다.

# Dependency
~~~groovy
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
~~~

# MongoTemplate 구성
AbstractMongoConfiguration의 기본 클래스를 확장하여 MongoDB를 구성할 수 있다.
~~~java
@Configuration
public class MongoConfig extends AbstractMongoConfiguration {
    @Override
    protected String getDatabaseName() {
        return "test";
    }
  
    @Override
    public MongoClient mongoClient() {
        return new MongoClient("127.0.0.1", 27017);
    }
  
    @Override
    protected String getMappingBasePackage() {
        return "org.baeldung";
    }
}
~~~
AbstractMongoConfiguration 클래스를 사용하지 않고, 처음부터 구성하는 경우
~~~java
@Configuration
public class SimpleMongoConfig {
    @Bean
    public MongoClient mongo() {
        return new MongoClient("localhost");
    }
 
    @Bean
    public MongoTemplate mongoTemplate() throws Exception {
        return new MongoTemplate(mongo(), "test");
    }
}
~~~

# MongoTemplate 사용
## 1. Insert
새 사용자를 삽입하면
~~~java
User user = new User();
user.setName("Jon");
mongoTemplate.insert(user, "user");
~~~
데이터베이스는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b4fda5830b550a8c2ca25a"),
    "_class" : "org.baeldung.model.User",
    "name" : "Jon"
}
~~~

## 2. Save – Insert
ID가 있는 경우 업데이트 아니면 삽입한다.
~~~java
User user = new User();
user.setName("Albert"); 
mongoTemplate.save(user, "user");
~~~
데이터베이스에 엔티티가 삽입된다.
~~~json
{
    "_id" : ObjectId("55b52bb7830b8c9b544b6ad5"),
    "_class" : "org.baeldung.model.User",
    "name" : "Albert"
}
~~~

## 3. Save – Update
기존 엔티티가 아래와 같이 존재할 경우
~~~json
{
    "_id" : ObjectId("55b52bb7830b8c9b544b6ad5"),
    "_class" : "org.baeldung.model.User",
    "name" : "Jack"
}
~~~
기존 사용자를 저장하면 업데이트가 된다.
~~~java
user = mongoTemplate.findOne(
  Query.query(Criteria.where("name").is("Jack")), User.class);
user.setName("Jim");
mongoTemplate.save(user, "user");
~~~
데이터베이스는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b52bb7830b8c9b544b6ad5"),
    "_class" : "org.baeldung.model.User",
    "name" : "Jim"
}
~~~

## 4. UpdateFirst
쿼리와 일치하는 첫 번째 문서를 업데이트한다. 데이터베이스가 다음과 같을 경우
~~~json
[
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Alex"
    },
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614c"),
        "_class" : "org.baeldung.model.User",
        "name" : "Alex"
    }
]
~~~
updateFirst를 실행해보자.
~~~java
Query query = new Query();
query.addCriteria(Criteria.where("name").is("Alex"));
Update update = new Update();
update.set("name", "James");
mongoTemplate.updateFirst(query, update, User.class);
~~~
다음과 같이 첫 번째 항목만 업데이트되었다.
~~~json
[
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "James"
    },
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614c"),
        "_class" : "org.baeldung.model.User",
        "name" : "Alex"
    }
]
~~~

## 5. UpdateMulti
주어진 쿼리와 일치하는 모든 문서를 업데이트한다. 현재 데이터베이스가 다음과 같을 때
~~~json
[
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Eugen"
    },
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614c"),
        "_class" : "org.baeldung.model.User",
        "name" : "Eugen"
    }
]
~~~
updateMulti 작업을 실행해보자.
~~~java
Query query = new Query();
query.addCriteria(Criteria.where("name").is("Eugen"));
Update update = new Update();
update.set("name", "Victor");
mongoTemplate.updateMulti(query, update, User.class);
~~~
기존 객체 모두 데이터베이스에서 업데이트가 되었다.
~~~json
[
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Victor"
    },
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614c"),
        "_class" : "org.baeldung.model.User",
        "name" : "Victor"
    }
]
~~~

## 6. FindAndModify
updateMulti와 유사하게 작동 하지만 수정하기 전에 객체를 반환한다. 먼저 findAndModify를 호출하기 전의 데이터베이스 상태:
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Markus"
}
~~~
실제 작업 코드를 보자.
~~~java
Query query = new Query();
query.addCriteria(Criteria.where("name").is("Markus"));
Update update = new Update();
update.set("name", "Nick");
User user = mongoTemplate.findAndModify(query, update, User.class);
~~~
리턴 된 사용자 오브젝트 는 데이터베이스의 초기 상태와 동일한 값을 갖는다. 그러나 데이터베이스의 새로운 상태는 다음과 같습니다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Nick"
}
~~~

## 7. Upsert
문서가 일치하는 경우 다른 쿼리 및 업데이트 객체를 결합하여 새 문서를 생성, 업데이트한다.
데이터베이스의 초기 상태는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Markus"
}
~~~
upsert를 실행 해보자.
~~~java
Query query = new Query();
query.addCriteria(Criteria.where("name").is("Markus"));
Update update = new Update();
update.set("name", "Nick");
mongoTemplate.upsert(query, update, User.class);
~~~
작업 후 데이터베이스의 상태는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Nick"
}
~~~

## 8. Remove
현재 데이터베이스의 상태는 다음과 같다.
~~~java
mongoTemplate.remove(user, "user");
~~~

# References
* https://www.baeldung.com/spring-data-mongodb-tutorial
