# Overview
`Spring Boot`에서 @JsonComponent 어노테이션을 사용하면 `ObjectMapper`에 수동으로 추가하지 않고도 해당 어노테이션이 달린
클래스를 ***Jackson serializer 또는 deserializer***로 노출시킬 수 있다. @JsonComponent 어노테이션 사용법을 알아보자.

# Serialization
~~~java
public class User {
    private Color favoriteColor;
 
    // standard getters/constructors
}
~~~
기본 설정으로 `Jackson`을 사용하여 이 객체를 직렬화하면 다음과 같은 결과가 나온다.
~~~json
{
  "favoriteColor": {
    "red": 0.9411764740943909,
    "green": 0.9725490212440491,
    "blue": 1.0,
    "opacity": 1.0,
    "opaque": true,
    "hue": 208.00000000000003,
    "saturation": 0.05882352590560913,
    "brightness": 1.0
  }
}
~~~
CSS에서 사용하기 위해 RGB 값만을 사용한다면 아래와 같이 `JsonSerializer`를 구현하여 JSON을 훨씬 더 압축하고 읽을 수 있게 만들 수 있다.
~~~java
@JsonComponent
public class UserJsonSerializer extends JsonSerializer<User> {
 
    @Override
    public void serialize(User user, JsonGenerator jsonGenerator, 
      SerializerProvider serializerProvider) throws IOException, 
      JsonProcessingException {
 
        jsonGenerator.writeStartObject();
        jsonGenerator.writeStringField(
          "favoriteColor", 
          getColorAsWebColor(user.getFavoriteColor()));
        jsonGenerator.writeEndObject();
    }
 
    private static String getColorAsWebColor(Color color) {
        int r = (int) Math.round(color.getRed() * 255.0);
        int g = (int) Math.round(color.getGreen() * 255.0);
        int b = (int) Math.round(color.getBlue() * 255.0);
        return String.format("#%02x%02x%02x", r, g, b);
    }
}
~~~
이 `JsonSerializer`를 사용하면 결과 JSON이 다음과 같이 줄어든다.
~~~json
{"favoriteColor":"#f0f8ff"}
~~~
`@JsonComponent` 어노테이션으로 인해 Serializer 는 Spring Boot 애플리케이션의 `Jackson ObjectMapper`에 등록된다.
다음 JUnit 테스트로 확인할 수 있다.
~~~java
@JsonTest
@RunWith(SpringRunner.class)
public class UserJsonSerializerTest {
 
    @Autowired
    private ObjectMapper objectMapper;
 
    @Test
    public void testSerialization() throws JsonProcessingException {
        User user = new User(Color.ALICEBLUE);
        String json = objectMapper.writeValueAsString(user);
 
        assertEquals("{\"favoriteColor\":\"#f0f8ff\"}", json);
    }
}
~~~

# Deserialization
동일한 예제를 계속해서 웹 색상 문자열을 JavaFX 색상 객체로 변환하는 디시리얼라이저를 작성할 수 있다.
~~~java
@JsonComponent
public class UserJsonDeserializer extends JsonDeserializer<User> {
 
    @Override
    public User deserialize(JsonParser jsonParser, 
      DeserializationContext deserializationContext) throws IOException, 
      JsonProcessingException {
 
        TreeNode treeNode = jsonParser.getCodec().readTree(jsonParser);
        TextNode favoriteColor
          = (TextNode) treeNode.get("favoriteColor");
        return new User(Color.web(favoriteColor.asText()));
    }
}
~~~
JUnit 테스트로 확인해보자.
~~~java
@JsonTest
@RunWith(SpringRunner.class)
public class UserJsonDeserializerTest {
 
    @Autowired
    private ObjectMapper objectMapper;
 
    @Test
    public void testDeserialize() throws IOException {
        String json = "{\"favoriteColor\":\"#f0f8ff\"}"
        User user = objectMapper.readValue(json, User.class);
 
        assertEquals(Color.ALICEBLUE, user.getFavoriteColor());
    }
}
~~~

# Serializer and Deserializer in One Class
원하는 경우 아래와 같이 클래스 내에 두 개의 내부 클래스를 추가하여 하나의 클래스에서 시리얼라이저와 디시리얼라이저를 연결 할 수 있다.
~~~java
@JsonComponent
public class UserCombinedSerializer {
 
    public static class UserJsonSerializer 
      extends JsonSerializer<User> {
 
        @Override
        public void serialize(User user, JsonGenerator jsonGenerator, 
          SerializerProvider serializerProvider) throws IOException, 
          JsonProcessingException {
 
            jsonGenerator.writeStartObject();
            jsonGenerator.writeStringField(
              "favoriteColor", getColorAsWebColor(user.getFavoriteColor()));
            jsonGenerator.writeEndObject();
        }
 
        private static String getColorAsWebColor(Color color) {
            int r = (int) Math.round(color.getRed() * 255.0);
            int g = (int) Math.round(color.getGreen() * 255.0);
            int b = (int) Math.round(color.getBlue() * 255.0);
            return String.format("#%02x%02x%02x", r, g, b);
        }
    }
 
    public static class UserJsonDeserializer 
      extends JsonDeserializer<User> {
 
        @Override
        public User deserialize(JsonParser jsonParser, 
          DeserializationContext deserializationContext)
          throws IOException, JsonProcessingException {
 
            TreeNode treeNode = jsonParser.getCodec().readTree(jsonParser);
            TextNode favoriteColor = (TextNode) treeNode.get(
              "favoriteColor");
            return new User(Color.web(favoriteColor.asText()));
        }
    }
}
~~~

# References
* https://www.baeldung.com/spring-boot-jsoncomponent