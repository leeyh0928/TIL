# Overview
프로토콜 버퍼는 구조화 된 데이터의 직렬화 및 역 직렬화를 위한 언어이고, 플랫폼 중립적인 메커니즘으로
XML 및 JSON과 같은 다른 유형의 페이로드보다 훨씬 빠르고 작고 단순한 데이터 구조이다.

# Protocol Buffers
## 1. Introduction to Protocol Buffers
프로토콜 버퍼를 사용하려면 .proto 파일에 메세지 구조를 정의해야 한다. 각 파일은 한 노드에서 다른 노드로
전송되거나 데이터 소스에 저장될 수 있는 데이터에 대한 설명이다. 
아래는 src/main/resource의 `ProtobufTraining.proto` 파일 예시이다.
* java_package : Java 클래스로 생성 시 패키지명이다.
* java_outer_classname : Java 클래스로 생성 시 클래스명이다.
~~~proto
syntax = "proto3";
package baeldung;
option java_package = "com.example.protobuf";
option java_outer_classname = "ProtobufTraining";
 
message Course {
    int32 id = 1;
    string course_name = 2;
    repeated Student student = 3;
}

message Student {
    int32 id = 1;
    string first_name = 2;
    string last_name = 3;
    string email = 4;
    repeated PhoneNumber phone = 5;
    message PhoneNumber {
        string number = 1;
        PhoneType type = 2;
    }
    enum PhoneType {
        MOBILE = 0;
        LANDLINE = 1;
    }
}
~~~
## 2. Protocol Buffers With Java
Java 코드로 변환하기 위해 별도의 컴파일러 `protoc`가 필요하며 설치 방법은 [여기](https://medium.com/@erika_dike/installing-the-protobuf-compiler-on-a-mac-a0d397af46b8)
를 참고하기 바란다.

혹시 Can't exec "glibtoolize": No such file or directory 오류가 발생할 경우 아래 커맨드를 통해 `libtool`을 추가 설치한다.
~~~bash
brew install libtool
~~~
컴파일을 통해 Java 클래스 파일로 변환한다.
~~~bash
protoc --java_out=java src/main/resources/ProtobufTraining.proto
~~~
`protobuf` 런타임 Dependency를 추가한다.
~~~groovy
implementation 'com.google.protobuf:protobuf-java:3.12.2'
implementation 'com.googlecode.protobuf-java-format:protobuf-java-format:1.4'
~~~
## 3. Compiling a Message Description
컴파일러를 사용하면 `.proto` 파일의 `Course` 및 `Student` 메시지가 Java 클래스로 변환된다.
동시에 메시지의 필드는 JavaBeans 스타일의 Getter 및 Setter로 컴파일된다.

# Spring REST API의 프로토 타입
## 1. Bean Declaration
`ProtobufHttpMessageConverter` Bean은 @RequestMapping에 의한 응답 객체를 프로토콜 버퍼 메세지로 변환한다. 
~~~java
@SpringBootApplication
public class Application {
    @Bean
    ProtobufHttpMessageConverter protobufHttpMessageConverter() {
        return new ProtobufHttpMessageConverter();
    }
 
    @Bean
    public CourseRepository createTestCourses() {
        Map<Integer, Course> courses = new HashMap<>();
        Course course1 = Course.newBuilder()
          .setId(1)
          .setCourseName("REST with Spring")
          .addAllStudent(createTestStudents())
          .build();
        Course course2 = Course.newBuilder()
          .setId(2)
          .setCourseName("Learn Spring Security")
          .addAllStudent(new ArrayList<Student>())
          .build();
        courses.put(course1.getId(), course1);
        courses.put(course2.getId(), course2);
        return new CourseRepository(courses);
    }
 
    ...
}
~~~
`CourseRepository` 구현
~~~java
public class CourseRepository {
    Map<Integer, Course> courses;
     
    public CourseRepository (Map<Integer, Course> courses) {
        this.courses = courses;
    }
     
    public Course getCourse(int id) {
        return courses.get(id);
    }
}
~~~
## 2. Controller Configuration
~~~java
@RestController
public class CourseController {
    @Autowired
    CourseRepository courseRepo;
 
    @RequestMapping("/courses/{id}")
    Course customer(@PathVariable Integer id) {
        return courseRepo.getCourse(id);
    }
}
~~~

# REST 클라이언트 및 테스트
~~~java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
@WebIntegrationTest
public class ApplicationTest {
    private static final String COURSE1_URL = "http://localhost:8080/courses/1";

    @Autowired
    private RestTemplate restTemplate;
    
    @Test
    public void whenUsingRestTemplate_thenSucceed() { // RestTemplate을 사용한 테스트
        ResponseEntity<Course> course = restTemplate.getForEntity(COURSE1_URL, Course.class);
        assertResponse(course.toString());
    }
    
    @Bean
    RestTemplate restTemplate(ProtobufHttpMessageConverter hmc) {
        return new RestTemplate(Arrays.asList(hmc));
    }

    private void assertResponse(String response) {
        assertThat(response, containsString("id"));
        assertThat(response, containsString("course_name"));
        assertThat(response, containsString("REST with Spring"));
        assertThat(response, containsString("student"));
        assertThat(response, containsString("first_name"));
        assertThat(response, containsString("last_name"));
        assertThat(response, containsString("email"));
        assertThat(response, containsString("john.doe@baeldung.com"));
        assertThat(response, containsString("richard.roe@baeldung.com"));
        assertThat(response, containsString("jane.doe@baeldung.com"));
        assertThat(response, containsString("phone"));
        assertThat(response, containsString("number"));
        assertThat(response, containsString("type"));
    }
}
~~~
전체 코드는 [여기](https://github.com/leeyh0928/demo-spring-protobuf)에서 확인할 수 있다.

# References
* https://www.baeldung.com/spring-rest-api-with-protocol-buffers