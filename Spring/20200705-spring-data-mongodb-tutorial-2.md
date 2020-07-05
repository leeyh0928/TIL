# MongoRepository 구성
MongoTemplate 구성을 기반으로 아래 어노테이션을 추가한다.
~~~java
@EnableMongoRepositories(basePackages = "xxx.xxx.xxx")
...
~~~
# Repository 생성
MongoRepository 인터페이스를 확장하여 저장소를 만든다.
~~~java
public interface UserRepository extends MongoRepository<User, String> {
    // 
}
~~~

# MongoRepository 사용
## 1. Insert
새로운 사용자를 삽입한다.
~~~json
User user = new User();
user.setName("Jon");
userRepository.insert(user);
~~~
데이터베이스 상태는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b4fda5830b550a8c2ca25a"),
    "_class" : "org.baeldung.model.User",
    "name" : "Jon"
}
~~~

## 2. Save – Insert
save는 MongoTemplate API의 저장 작업과 동일하게 작동한다.
~~~java
User user = new User();
user.setName("Aaron");
userRepository.save(user);
~~~
결과적으로 데이터베이스에 해당 사용자가 추가된다.
~~~json
{
    "_id" : ObjectId("55b52bb7830b8c9b544b6ad5"),
    "_class" : "org.baeldung.model.User",
    "name" : "Aaron"
}
~~~

## 3. Save – Update
변경 전 데이터베이스의 상태는 아래와 같다.
~~~json
{
    "_id" : ObjectId("55b52bb7830b8c9b544b6ad5"),
    "_class" : "org.baeldung.model.User",
    "name" : "Jack"81*6
}
~~~
데이터 변경을 위해 아래와 같은 작업을 실행한다.
~~~java
user = mongoTemplate.findOne(
  Query.query(Criteria.where("name").is("Jack")), User.class);
user.setName("Jim");
userRepository.save(user);
~~~
데이터베이스 상는 아래와 같다.
~~~json
{
    "_id" : ObjectId("55b52bb7830b8c9b544b6ad5"),
    "_class" : "org.baeldung.model.User",
    "name" : "Jim"
}
~~~

## 4. Delete
삭제 전 데이터베이스 상태는 아래와 같다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Benn"
}
~~~
***delete***를 실행
~~~java
userRepository.delete(user);
~~~
결과는 다음과 같다.
~~~json
{
}
~~~

## 5. FindOne
현재 데이터베이스 상태는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Chris"
}
~~~
findOne을 실행해보자.
~~~java
userRepository.findOne(user.getId())
~~~
결과는 다음과 같다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Chris"
}
~~~

## 6. findAll 정렬하기 
호출 전 데이터베이스의 상태는 다음과 같다.
~~~json
[
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Brendan"
    },
    {
       "_id" : ObjectId("67b5ffa5511fee0e45ed614b"),
       "_class" : "org.baeldung.model.User",
       "name" : "Adam"
    }
]
~~~
sort로 findAll을 실행해본다.
~~~java
List<User> users = userRepository.findAll(new Sort(Sort.Direction.ASC, "name"));
~~~
결과는 이름별 오름차순으로 정렬된다.
~~~json
[
    {
        "_id" : ObjectId("67b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Adam"
    },
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Brendan"
    }
]
~~~

## 7. findAll 페이징 적용하기
호출 전 데이터베이스의 상태는 다음과 같다.
~~~json
[
    {
        "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Brendan"
    },
    {
        "_id" : ObjectId("67b5ffa5511fee0e45ed614b"),
        "_class" : "org.baeldung.model.User",
        "name" : "Adam"
    }
]
~~~
`Pageable`을 이용해 ***findAll***을 실행해본다.
~~~java
Pageable pageableRequest = PageRequest.of(0, 1);
Page<User> page = userRepository.findAll(pageableRequest);
List<User> users = pages.getContent();
~~~
결과 사용자 목록은 한 명의 사용자만 된다.
~~~json
{
    "_id" : ObjectId("55b5ffa5511fee0e45ed614b"),
    "_class" : "org.baeldung.model.User",
    "name" : "Brendan"
}
~~~ 

# References
* https://www.baeldung.com/spring-data-mongodb-tutorial
