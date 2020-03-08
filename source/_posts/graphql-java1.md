---
title: GraphQL Vol.1 - GraphQL 체험해보기
date: 2020-03-07 20:04:13
modified_at: '2020-03-08'
tags: [graphql]
categories:
  - graphql
---
# 머리말

Github에서 제공하는 V3 API들을 보면서, 이거야말로 RESTful이구나 했었는데, V4로 넘어오면서 GraphQL을 사용하게 되었다.
Github은 자신들이 V4 API를 RESTful이 아닌 GraphQL로 작성한 이유를 다음과 같이 2가지로 이야기했다.

- 확장성(Scalability)
  - 클라이언트는 자신이 필요한 정보를 취득하기 위해,
   필연적으로 REST API의 하이퍼미디어를 사용하여 경로 탐색을 위한 반복적인 호출을 해야 한다.
  - 클라이언트가 응답의 일부분만 필요로하더라도 Github은 API 명세에 정의한 모든 값을 채워서 응답했다.
  - 때로는 어떠한 정보들을 조합하기 위해서, 2~3번의 별도의 API를 호출해야 한다. <!-- more -->
- API 통합의 요구
  - 각 엔드포인트에서 필요로하는 OAuth 식별할 수 있어야 한다.
  - 사용자가 보낸 파라미터에 대한 형안전성(type-safety) 보장해야 한다.
  - 소스 코드로 문서를 생산할 수 있어야 한다.
  - [수동으로 제작/관리하는 클라이언트들](https://developer.github.com/v3/libraries/)을 대체할 수 있는 무엇이 필요하다.

[GraphQL 페이지](https://graphql.org/)보다는  Github Api V4의 [About GraphQL](https://developer.github.com/v4/)를 보면 GraphQL의 장점이 잘 소개되어 있다.
대략 다음과 같다.

GraphQL은:

- 그 자체로 명세(Specification)를 나타낸다: 명세는 곧 API 서버의 스키마(Schema)를 결정한다.
또한 스키마는 클라이언트의 호출이 유효한지를 판단할 수 있다.
- 타입 기반(Type Based)이다: 스키마는 API에서 사용되는 타입과 각 타입 간의 관례를 정의한다.
- 자기성찰적이다(Introspective): 클라이언트는 스키마의 상세를 알기 위해 스키마를 질의(query)한다.
RESTful API들이 하위 도메인에 대한 경로들의 url을 제공한 것과 비슷하게, 스키마 구조를 파악하기 위한 스키마 호출을 할 수 있다.
- 계층적이다: JSON 응답의 구조와 GraphQL의 요청 구조가 동일하다.
- 애플리케이션 레이어에서 동작한다: GraphQL은 실제 저장될 모델이나, DB 질의 언어가 아니다.

마지막 부분에 대한 부연 설명을 더하려고 한다.
spring framework에서 웹 애플리케이션을 작성할 떄,
개발자가 해야 하는 일은 요청처리 흐름에 따라 셋으로 나누자면
`컨트롤러까지의 처리`/`서비스 로직처리`/`리파지토리&외부 API 처리`정도로 볼 수 있을 것 같다.
컨트롤러까지가 담당해야 할 역할은 다음과 같다.

- Http Request의 인증/권한 검증
- 파라미터 유효성 검증
- 비즈니스 로직에서 처리할 수 있는 객체로 변환
- 비즈니스 로직의 결과(혹은 예외)를 적절한 Http Response(JSON/XML/HTML)로 응답

이 중에서 GraphQL은 `파라미터 유효성 검증`과 `비즈니스 로직의 결과를 JSON으로 응답`해주는 역할을 해준다.

서론이 장황했다.
GraphQL의 내용이 상당하기에 여러 포스트로 나누어 소개하려고 한다.
우선 이번 문서에서는 GraphQL에 대해서 간단하게 설명하고 체험하는 것을 가이드하려 한다.

# GraphQL 기본

## 스키마

GraphQL에서는 API에서 다를 타입을 정의할 수 있다.

```
type Query {
    todo(id: Int!): Todo
}

type Todo {
    id: Int!
    title: String!
    completed: Boolean!
}
```

이러한 스키마 정의를 기반으로
- 클라이언트는 자신이 필요한 정보들을 선별하여 요청을 보낼 수 있다.
- 서버는 요청의 파라미터들이 스키마에 부합하는지 유효성을 검증할 수 있다.
- 서버는 클라이언트에서 요청한 정보들만 내보낼 수 있게 된다.

클라이언트가 필요한 정보를 선별해서 요청을 보낸다는 것은 잠시 후 테스트에서 살펴보도록 하자.

## 요청 경로(Path)

RESTful API에서는 각 API의 경로가 고유한 도메인의 정보를 나타낸다.
하지만 GraphQL에서는 아래와 같이 `/graphql`에서 모든 쿼리를 처리한다.

| ![rest-graphql-comparison](https://miro.medium.com/max/1600/1*qpyJSVVPkd5c6ItMmivnYg.png) |
| - |
| 출처 : https://blog.apollographql.com/graphql-vs-rest-5d425123e34b |

앞선 스키마 예제에서 id가 1인 todo의 완료 여부(`completed`)를 가져오려면 다음과 같은 요청을 한다.

```
POST /graphql
HOST: localhost
User-Agent: test
{
  "query": "{ todo(id: 1) { completed } }"
}
```

위 예제는 단순화한 예제이며, 실제로는 이것보다는 조금 복잡한 요청을 보내는 경우가 다수이다.
GET 방식으로도 요청을 보낼 수 있으며, Subscription 질의의 경우에는 웹소켓(Websocket)을 사용하기도 한다.

# GraphQL 체험해보기

## Post Schema

테스트로 쿼리해서 가져올 대상은 [JSONPlaceholder](https://jsonplaceholder.typicode.com/)에서 제공하는 todo, user 모델이다.
두 모델은 아래와 같은 구조로 되어 있으며, 서로 연관 관계를 맺고 있다.

- Todo(연관 관계 주인) -> User = N:1

**todo**
```json
{
  "id": "number",
  "userId": "number",
  "title": "string",
  "completed": "boolean"
}
```

**user**
```json
{
  "id": "number",
  "name": "string",
  "username": "string",
  "email": "string",
  "address": {
    "street": "string",
    "suite": "string",
    "city": "string",
    "zipcode": "string",
    "geo": {
      "lat": "string",
      "lng": "string"
    }
  },
  "phone": "string",
  "website": "string",
  "company": {
    "name": "string",
    "catchPhrase": "string",
    "bs": "string"
  }
}
```

## 테스트 해보기

### 첫번째 테스트

아래 테스트는 Heroku에 연결되어 있다.

**Heroku는 무료로 사용할 때, 요청이 없는 동안 어플리케이션이 sleep 모드로 전환되는 경우가 있어, 로딩이 조금 오래 걸릴 수 있다.**

아래의 테스트(iframe)이 로딩되었다면, ► 버튼을 눌러보자.

<iframe style="width: 100%" height="500px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20todos%20%7B%0A%20%20%20%20id%0A%20%20%20%20title%0A%20%20%20%20completed%0A%20%20%7D%0A%7D"></iframe>

왼쪽 박스가 요청을 나타내고 오른쪽 박스의 부분이 응답을 나타낸다.
요청한 내용의 구조 그대로 JSON의 응답이 오는 것을 확인할 수 있다.

```
{
  todos {
    id
    title
    completed
  }
}
```

위 테스트 예제에서 id를 지우거나, title을 지우거나 한 후 다시 실행시켜보자.

### 연관 타입 추가 호출

위 예제에서는 응답이 장황해지는 것을 피하고자 User를 포함하지 않았다.
아래 예제를 실행시켜보자.

<iframe style="width: 100%" height="500px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20todos(limit%3A%202)%20%7B%0A%20%20%20%20id%0A%20%20%20%20title%0A%20%20%20%20completed%0A%20%20%20%20user%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%20%20phone%0A%20%20%20%20%20%20address%20%7B%0A%20%20%20%20%20%20%20%20city%0A%20%20%20%20%20%20%20%20zipcode%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D"></iframe>

우선 아래와 같이 `todos` 조회할 때 limit라는 파라미터를 넘기고 있다.
이 부분은 질의에 파라미터를 포함할 수 있다는 정도로 이해하고 넘어가자.

```diff
- todos {
+ todos(limit: 2) {
```

여기서 설명하고 싶은 점은, 클라이언트가 자신이 필요로한느 정보들을 서버에게 요청할 수 있다는 점이다.
또한 앞에서 스키마의 구조가 자기성찰적(Introspective)라고 했던 것도, 결국 스키마는 스키마를 포함한다는 내용으로 이해할 수 있다.
todos를 조회하면서 파라미터를 넣었듯이 하위에 포함되는 스키마에도 파라미터를 넣을 수 있다.

<iframe style="width: 100%" height="500px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20posts(limit%3A%202)%20%7B%0A%20%20%20%20id%0A%20%20%20%20title%0A%20%20%20%20comments(limit%3A%202)%20%7B%0A%20%20%20%20%20%20id%0A%20%20%20%20%20%20email%0A%20%20%20%20%20%20name%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D"></iframe>

### 여러 모델 받기

앞서 Github API를 사용하는 클라이언트들에서 자신들이 원하는 화면을 구성하기 위해서는 여러 API를 호출해야 한다고 했다.

아래 예제를 실행시켜보자.

<iframe style="width: 100%" height="500px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20getAll(%24id%3A%20Int!)%20%7B%0A%20%20thisIsPost%3A%20post(id%3A%20%24id)%20%7B%0A%20%20%20%20title%0A%20%20%20%20body%0A%20%20%7D%0A%20%20thisIsTodo%3A%20todo(id%3A%20%24id)%20%7B%0A%20%20%20%20title%0A%20%20%20%20completed%0A%20%20%7D%0A%20%20albumList%3A%20album(id%3A%20%24id)%20%7B%0A%20%20%20%20title%0A%20%20%20%20photos(limit%3A%202)%20%7B%0A%20%20%20%20%20%20title%0A%20%20%20%20%20%20thumbnailUrl%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&variables=%7B%0A%20%20%22id%22%3A%201%0A%7D&operationName=getAll"></iframe>

`$id`에 대한 정의는 테스트 툴 하단의 `QUERY VARIABLES`를 열어보면 확인할 수 있다.
클라이언트에서 `thisIsPost`에 Post 질의 내용을, `thisIsTodo`에 Todo의 질의 내용을 각각 응답값으로 받도록 요청했다.

### Introspection

이미 확인한 사람도 있겠지만, 상단 메뉴 우측에 `< Docs`를 열어서 질의문들이나 각 타입에 대한 설명을 확인할 수 있다.

<iframe style="width: 100%" height="500px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20__schema%20%7B%0A%20%20%20%20types%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%20%20description%0A%20%20%20%20%20%20fields(includeDeprecated%3A%20false)%20%7B%0A%20%20%20%20%20%20%20%20name%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D"></iframe>

앞에서 `GraphQL은 자기성찰적이다(Introspective): 클라이언트는 스키마의 상세를 알기 위해 스키마를 질의(query)한다`라고 설명한 것과 같이 `__schema`를 질의해보면, 내부에서 어떤 스키마가 정의되어 있는지 확인할 수 있다.

이를 기반으로 [Voyager 툴](https://supawer-graphql-study.herokuapp.com/voyager)은 다이어그램으로 표현해주기까지 한다.

# 맺는말

조금 심심하긴 하지만 이 체험하기 포스트는 이 정도로 마무리하려고 한다.

필자는 백엔드 서버 개발자이지만, 체험하기를 주제로 다루다 보니,
아무래도 요청을 보내고 응답을 받는 클라이언트의 입장이 부각된 것 같다.

여기까지만 봤을 때의 장점을 추리자면 다음과 같이 이야기할 수 있을 것 같다.

- 새로운 API를 작성/호출하지 않는다.
  - 서버는 매번 새로운 API를 작성하지 않아도 되고,
  - 클라이언트는 새로운 API를 또 비동기로 호출해서 조합해내고, 각각의 API의 Timeout은 어쩌니 고민하지 않아도 된다.
- 네트워크 비용을 줄일 수 있다.
  - 클라이언트에서 필요한 값만 요청하고 응답받을 수 있다.

한 번 체험해본 것만으로는 REST API에 비해 어떠한 장점이 있는지, 피부에 와닿지 않을 것 같다.
위에서 설명한 장점 외에도 GraphQL을 더 깊게 공부한다면 더 많은 장점을 발견할 수 있다.

전체적인 구성을 체험해보기/GraphQL 이론/Spring Boot와 접목으로 생각하고 있는데, 갈 길이 멀다.

처음 GraphQL에 대해 배워보자 하는 분들께 조금이나마 도움이 되었으면 한다.
