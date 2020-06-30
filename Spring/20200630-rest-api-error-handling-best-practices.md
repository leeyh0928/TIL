# Overview
`REST`는 클라이언트가 서버의 리소스에 엑세스하고, 조작할 수 있는 상태 비 저장 아키텍처이다.
일반적으로 `REST 서비스`는 HTTP를 사용하여 관리하는 자원 세트를 알리고, 클라이언트가 이러한 자원의 상태를 얻거나 변경할 수 있는 
API를 제공한다.
이때 사용자에게 관련 정보를 제공하는 유용한 방법, 대규모 웹 사이트의 예제, Spring REST 애플리케이션을 사용한 구체적 구현을
포함하여 REST API 오류 처리를 위한 `Best Practices`를 알아본다.

# HTTP 상태 코드
클라이언트가 HTTP 서버에 요청하고, 서버가 요청을 성공적으로 수신하면 요청이 성공적으로 처리되었는지 여부를 클라이언트에 알려야 한다.
HTTP는 아래 5가지 범주의 상태 코드로 이를 수행한다.
* 100 수준(정보) - 서버가 요청을 승인
* 200 수준(성공) - 서버가 예상대로 요청을 완료
* 300 수준(리디렉션) - 클라이언트는 요청을 완료하기 위해 추가 조치를 수행해야 함
* 400 수준(클라이언트 오류) - 클라이언트가 잘못된 요청을 함
* 500 수준(서버 오류) - 서버 오류로 인해 서버가 유효한 요청을 이행하지 못함

응답 코드를 기반으로 클라이언트는 특정 요청의 결과를 추측할 수 있다.

# 오류 처리
오류 처리의 첫번째는 클라이언트에 적절한 상태 코드를 제공하는 것이다. 또한 응답 본문에 더 많은 정보를 제공해야 할 수도 있다.
## 1. 기본 응답
오류를 처리하는 가장 간단한 방법은 적절한 상태 코드로 응답하는 것이다. 일반적인 응답 코드는 다음과 같다.
- 400 Bad Request - 클라이언트가 잘못된 요청을 보냄 (예: 필요한 요청 본문 또는 매개 변수 부족)
- 401 Unauthorized — 클라이언트가 서버를 인증하지 못함
- 403 Forbidden — 클라이언트가 인증되었지만 요청 된 리소스에 액세스 할 수 있는 권한이 없음
- 404 Not Found - 요청한 리소스가 없음
- 412 Precondition Failed — 요청 헤더 필드의 하나 이상의 조건이 FALSE로 평가됨
- 500 Internal Server Error — 서버에서 일반 오류가 발생함
- 503 Service Unavailable — 요청한 서비스를 사용할 수 없음

이러한 코드를 사용하면 클라이언트는 발생한 오류의 광범위한 특성을 이해할 수 있음. 그러나 많은 경우에 우리는 응답에 보충적인
세부 사항을 제공해야한다.

500 에러는 요청을 처리하는 서버에서 일부 문제 또는 예외가 발생했음을 나타내지만, 일반적으로 내부 오류는 고객의 비즈니스가 아니다.
따라서 이러한 종류의 응답을 최소화하려면 ***내부 오류를 부지런히 처리하거나 잡으려고 노력하고, 가능한 경우 다른 적절한 상태 코드로 응답***
해야한다. 예를 들어 요청된 리소스가 존재하지 않아 예외가 발생한 경우 500 에러가 아닌 404 에러로 표시해야한다.

## 2. 기본 Spring 오류 응답
`Spring`은 기본 오류 처리 메커니즘에 따라 원칙을 체계화했다. 예시를 위해 책을 관리하는 간단한 Spring REST 애플리케이션이 있고,
ID로 책을 검색하는 엔드 포인트가 있다고 가정해보자.
~~~bash
curl -X GET -H "Accept: application/json" http://localhost:8082/spring-rest/api/book/1
~~~

ID가 1인 책이 없다면 컨트롤러가 `BookNotFoundException`을 던질 것이다. 이 엔드 포인트에서 이 예외가 발생했으며 응답 본문은 다음과 같다.
~~~json
{
    "timestamp":"2019-09-16T22:14:45.624+0000",
    "status":500,
    "error":"Internal Server Error",
    "message":"No message available",
    "path":"/api/book/1"
}
~~~
이 기본 오류 처리기는 오류 발생시간, HTTP 상태 코드, 오류(제목), 메시지(기본적으로 비어 있음) 및 오류가 발생한 URL 경로를 포함한다.
또한, BookNotFoundException이 발생하면 Spring은 HTTP 상태 코드 500을 자동으로 반환한다.

일부 API들은 500 상태 코드 또는 일반적인 코드를 반환하지만 `Facebook 및 Twitter API`에서 볼 수 있듯이 모든 오류에 대해 간단하게 하기 위해
***가능한 경우 가장 구체적인 오류 코드를 사용*** 하는 것이 가장 좋다.

## 3. More Detailed Responses
때때로 일반적인 상태 코드로는 오류의 세부 사항을 표시하기에 충분하지 않다. 필요한 경우 응답 본문을 사용하여 고객에게 추가 정보를 제공 할 수 있다.
자세한 응답을 제공 할 때는 다음을 포함해야 한다.
* Error - 오류의 고유 식별자
* Message - 사람이 읽을 수 있는 간단한 메세지
* Detail - 보다 긴 오류에 대한 설명

예를 들어, 클라이언트가 자격 증명이 잘못된 요청을 보내는 경우 401 상태 코드로 다음과 같은 본문을 보낼 수 있다.
~~~json
{
    "error": "auth-0001",
    "message": "Incorrect username and password",
    "detail": "Ensure that the username and password included in the request are correct"
}
~~~
`error` 필드는 응답(상태) 코드와 같아야 한다.(같은 유형) 대신 애플리케이션 고유의 error 코드이어야 한다. 일반적으로 이 필드에는
영숫자 및 대시(-) 또는 밑줄(_)과 같은 연결문자만 포함한다.

`message` 필드는 일반적으로 ***사용자 인터페이스***에 표시 가능해야 한다. 그러므로 국제화를 지원한다면 `Accept-Language` 헤더를 사용하여
해당 언어로 변환하여 제공되어야 한다.

`detail` 필드는 최종 사용자가 아닌 ***클라이언트 개발자***를 위한 것으로 번역이 필요하지는 않다. 또한, 추가 정보를 발견하기 위한 `help`(도움말 필드)
에 URL을 제공할 수도 있다.
~~~json
{
    "error": "auth-0001",
    "message": "Incorrect username and password",
    "detail": "Ensure that the username and password included in the request are correct",
    "help": "https://example.com/help/error/auth-0001"
}
~~~
때때로, 요청에 대해 둘 이상의 오류를 보고 할 수도 있다. 이 경우 목록으로 오류를 반환해야 한다. 그리고 가장 중대한 오류를 
첫번째 오류로 응답하면 충분하다. 만약 단일 오류가 발생하면 하나의 요소만 포함하는 목록으로 응답한다.  
~~~json
{
    "errors": [
        {
            "error": "auth-0001",
            "message": "Incorrect username and password",
            "detail": "Ensure that the username and password included in the request are correct",
            "help": "https://example.com/help/error/auth-0001"
        },
        ...
    ]
}
~~~

## 4. Standardized Response Bodies
대부분의 REST API는 유사한 규칙을 따르지만 필드 이름과 응답 본문에 포함된 세부 사항이 다르다. 이러한 차이점으로 인해 라이브러리와
프레임워크에서 오류를 균일하게 처리하기 어렵다.

REST API 오류 처리를 표준화하기 위해 `IETF는 RFC 7807을 고안하여 일반화 된 오류 처리 스키마`를 작성했다. 이 스키마는 5가지 부분으로 구성된다.
1. type - 오류를 분류하는 URI 식별자
2. title - 오류에 대한 사람이 읽을 수있는 간단한 메시지
3. status - HTTP 응답 코드 (선택 사항)
4. detail - 사람이 읽을 수있는 오류에 대한 설명
5. instance - 오류의 발생 지점을 식별하는 URI
~~~json
{
    "type": "/errors/incorrect-user-pass",
    "title": "Incorrect username or password.",
    "status": 401,
    "detail": "Authentication failed due to incorrect username or password.",
    "instance": "/login/log/abc123"
}
~~~
***URI를 사용하여 클라이언트는 이러한 경로를 따라*** HATEOAS 링크를 사용하여 REST API를 탐색하는 것과 같은 방식으로 ***오류에 대한 자세한 정보***를 
찾을 수 있다. `RFC 7807`을 준수하는 것은 선택 사항이지만 ***균일성을 원하는 경우 유리***하다.

# Examples
## 1. Twitter
Twitter API에 인증 데이터를 제공하지 않고, GET 요청을 보내보자.
~~~bash
curl -X GET https://api.twitter.com/1.1/statuses/update.json?include_entities=true
~~~
다음과 같은 본문으로 오류를 응답(status code: 400)한다.
~~~json
{
    "errors": [
        {
            "code":215,
            "message":"Bad Authentication data."
        }
    ]
}
~~~
## 2. Facebook
트위터와 마찬가지로 Facebook의 Graph REST API도 응답에 자세한 정보를 포함한다. Facebook Graph API로 인증을 위한 요청을 수행해보자.
~~~bash
curl -X GET https://graph.facebook.com/oauth/access_token?client_id=foo&client_secret=bar&grant_type=baz
~~~
다음과 같은 오류가 발생한다.
~~~json
{
  "error": {
    "message": "Missing redirect_uri parameter.",
    "type": "OAuthException",
    "code": 191,
    "fbtrace_id": "AHaZMkdWN0dMhesN8cSwA2w"
  }
}
~~~
Twitter와 마찬가지로 Facebook은 400대 수준의 오류가 아닌 일반적인 오류를 사용하여 오류를 나타낸다. Facebook은 메시지 및 숫자 코드 외에도 
오류를 분류하는 유형 필드와 내부 지원 식별자 역할을 하는 추적 ID(fbtrace_id)를 포함한다.

# References
* https://www.baeldung.com/rest-api-error-handling-best-practices
