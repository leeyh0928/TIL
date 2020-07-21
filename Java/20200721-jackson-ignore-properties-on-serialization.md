# Overview
`Jackson 2.x`를 사용하여 ***객체를 JSON으로 직렬화*** 할 때 특정 필드를 무시 하는 방법을 알아본다.

# Ignore Fields at the Class Level
`@JsonIgnoreProperties` 어노테이션을 사용하고, 이름으로 필드를 지정 하여 클래스 레벨에서 특정 필드를 무시할 수 있다.
~~~java
@JsonIgnoreProperties(value = { "intValue" })
public class MyDto {
 
    private String stringValue;
    private int intValue;
    private boolean booleanValue;
 
    public MyDto() {
        super();
    }
 
    // standard setters and getters are not shown
}
~~~
이제 객체가 JSON에 작성된 후 필드가 실제로 출력되지 않는지 확인해본다.
~~~java
@Test
public void givenFieldIsIgnoredByName_whenDtoIsSerialized_thenCorrect()
  throws JsonParseException, IOException {
 
    ObjectMapper mapper = new ObjectMapper();
    MyDto dtoObject = new MyDto();
 
    String dtoAsString = mapper.writeValueAsString(dtoObject);
 
    assertThat(dtoAsString, not(containsString("intValue")));
}
~~~

# Ignore Field at the Field Level
필드에서 직접 `@JsonIgnore` 어노테이션을 통해 필드를 직접 무시할 수도 있다.
~~~java
public class MyDto {
 
    private String stringValue;
    @JsonIgnore
    private int intValue;
    private boolean booleanValue;
 
    public MyDto() {
        super();
    }
 
    // standard setters and getters are not shown
}
~~~
이제 intValue 필드가 직렬화 된 JSON 출력의 일부가 아닌지 테스트 할 수 있다.
~~~java
@Test
public void givenFieldIsIgnoredDirectly_whenDtoIsSerialized_thenCorrect() 
  throws JsonParseException, IOException {
 
    ObjectMapper mapper = new ObjectMapper();
    MyDto dtoObject = new MyDto();
 
    String dtoAsString = mapper.writeValueAsString(dtoObject);
 
    assertThat(dtoAsString, not(containsString("intValue")));
}
~~~

# Ignore All Fields by Type
`@JsonIgnoreType` 어노테이션을 사용하여 지정된 유형의 모든 필드를 무시 할 수 있다.
~~~java
@JsonIgnoreType
public class SomeType { ... }
~~~

# Ignore Fields Using Filters
필터를 사용 하여 Jackson의 특정 필드를 무시할 수 있다.
~~~java
@JsonFilter("myFilter")
public class MyDtoWithFilter { ... }
~~~
intValue 필드를 무시할 간단한 필터를 정의해보자.
~~~java
SimpleBeanPropertyFilter theFilter = SimpleBeanPropertyFilter
  .serializeAllExcept("intValue");
FilterProvider filters = new SimpleFilterProvider()
  .addFilter("myFilter", theFilter);
~~~
객체를 직렬화하고 intValue 필드가 JSON 출력에 없는지 확인해보자.
~~~java
@Test
public final void givenTypeHasFilterThatIgnoresFieldByName_whenDtoIsSerialized_thenCorrect() 
  throws JsonParseException, IOException {
 
    ObjectMapper mapper = new ObjectMapper();
    SimpleBeanPropertyFilter theFilter = SimpleBeanPropertyFilter
      .serializeAllExcept("intValue");
    FilterProvider filters = new SimpleFilterProvider()
      .addFilter("myFilter", theFilter);
 
    MyDtoWithFilter dtoObject = new MyDtoWithFilter();
    String dtoAsString = mapper.writer(filters).writeValueAsString(dtoObject);
 
    assertThat(dtoAsString, not(containsString("intValue")));
    assertThat(dtoAsString, containsString("booleanValue"));
    assertThat(dtoAsString, containsString("stringValue"));
    System.out.println(dtoAsString);
}
~~~