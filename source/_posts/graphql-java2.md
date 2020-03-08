---
title: GraphQL Vol.2 - GraphQL 기초
modified_at: '2020-03-08'
date: 2020-03-09 01:55:00
tags: [graphql]
categories:
  - graphql
---
# 머리말

지난 포스트에서는 GraphQL에 대해서 간단하게 테스트를 해보았다.
이번 포스트에서는 [GraphQL의 Learn](https://graphql.org/learn/)에 나오는 내용을 따라 설명을 덧붙이려고 한다.
설명과 더불어 GraphiQL을 붙여 즉각 실행해볼 수 있게 구성하였다.

> 테스트는 Heroku를 사용하고 있다.
> Heroku를 무료로 사용할 시 유휴시간이 30분이 넘는 경우, 애플리케이션이 sleep 상태로 바뀐다.
> 그렇기 때문에 아래의 테스트 로딩이 느릴 수 있으나, 1분 이내에 로딩될 것이다.

<!-- more -->

# 질의문(Query) / 변경문(Mutation)

### 필드

이전 포스트에서 체험했다시피, 클라이언트에서는 스키마에서 필요로하는 필드를 서버에 요청하여 응답받을 수 있다.

> ► 버튼을 눌러 테스트할 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20user%20%7B%0A%20%20%20%20name%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

요청의 내용과 응답의 JSON의 각 필드는 대칭된다.

> 위 요청 내용을 수정해서 테스트할 수도 있다. title을 지우고 ► 버튼을 눌러보자.

또한 객체에 포함된 하위 타입에 대한 질의를 추가할 수 있다.
User가 재직하고 있는 Company를 조회해보자.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20user%20%7B%0A%20%20%20%20name%0A%20%20%20%20company%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%20%20catchPhrase%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

응답에 포함된 User 또한 요청에 대응하는 JSON 구조를 취하고 있다.

### 질의 인수(Arguments)

GraphQL은 조회를 위해 인수를 줄 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20user(id%3A%202)%20%7B%0A%20%20%20%20name%0A%20%20%20%20phone%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

REST에서는 URL Path, Query String 등으로 인수를 넘겼다.
GraphQL에서는 모든 필드와 중첩된 객체에 스키마로 정의한 인수를 넘길 수 있다.
심지어 인수는 스칼라 타입(scala: String이나 Int)에도 보낼 수 있다.
다음 예제를 살펴보자.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20user(id%3A%202)%20%7B%0A%20%20%20%20name(toUpperCase%3A%20true)%0A%20%20%20%20phone%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

### 별칭 할당(Aliases)

아래와 같이 질의마다 변수를 할당하듯 별칭을 줄 수 있다.
user를 조회하지만, 각각 다른 별칭으로 참조할 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20me%3A%20user(id%3A%201)%20%7B%0A%20%20%20%20name%0A%20%20%20%20phone%0A%20%20%7D%0A%20%20you%3A%20user(id%3A2)%20%7B%0A%20%20%20%20name%0A%20%20%20%20phone%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>


### Fragments

위에서 별칭을 할당할 때, `me`와 `you`가 동일한 필드를 조회하고 있었다.
GraphQL은 이러한 반복을 없앨 수 있도록, Fragments라 부르는 재사용 구문을 제공한다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20me%3A%20user(id%3A%201)%20%7B%0A%20%20%20%20...basigInfo%0A%20%20%7D%0A%20%20you%3A%20user(id%3A2)%20%7B%0A%20%20%20%20...basigInfo%0A%20%20%7D%0A%7D%0A%0Afragment%20basigInfo%20on%20User%20%7B%0A%20%20name%0A%20%20phone%0A%20%20email%0A%20%20website%0A%7D&variables=%7B%7D"></iframe>

`fragment`를 선언 후 fragment의 이름을 정의하고 `on` 뒤에 타입을 명시한다.
명시한 타입을 기반으로 fragment 내부 블럭에 넣을 수 있는 필드가 결정된다.


### 연산자명

여태까지 사용한 질의는 사실 모두 축약형이었다.
질의의 원형은 `query 연산자명 { ... }`의 형식이다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20meAndYou%20%7B%0A%20%20me%3A%20user(id%3A%201)%20%7B%0A%20%20%20%20name%0A%20%20%7D%0A%20%20you%3A%20user(id%3A2)%20%7B%0A%20%20%20%20name%0A%20%20%7D%0A%7D&operationName=meAndYou&variables=%7B%7D"></iframe>

연산 타입은 `query(질의)`, `mutation(변경)`, `subscription(구독)`의 3가지로 나뉜다.
연산자명은 모든 연산 타입에 대해서 명명할 수 있다.
연산자명은 디버깅을 하거나 서버 사이드에서 로깅을 할 때 유용하게 쓰일 수 있다.


### 변수(Variables)

앞에서 인수를 통해서 다른 id의 User를 가져올 수 있는 것을 확인했었다.
만약 클라이언트에서 동적으로 다른 질의를 할 때, 매번 다른 인수를 가진 질의 문자열을 생성한다면 큰 낭비일 것이다.
질의를 템플릿으로 고정해두고, 우리는 변수를 넘길 수 있다.
변수는 주로 JSON 형식으로 정의한다.

> 하단의 `QUERY VARIABLES`를 참조.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20findAllPostsWithLimit(%24limit%3A%20Int)%20%7B%0A%20%20posts(limit%3A%20%24limit)%20%7B%0A%20%20%20%20title%0A%20%20%09comments(limit%3A%20%24limit)%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&operationName=findAllPostsWithLimit&variables=%7B%0A%20%20%22limit%22%3A%202%0A%7D"></iframe>

Http Request의 모양은 다음과 같다.

```
POST /graphql
Host: localhost
{
  "query": "query findAllPostsWithLimit($limit: Int!) { ... }",
  "variables": {
    "limit": 2
  },
  "operationName": "findAllPostsWithLimit"
}
```

이제 클라이언트에서는 질의를 매번 수정할 필요 없이, 변수만 정의해서 동적으로 다른 질의를 할 수 있다.
fragment에서도 변수를 그대로 사용할 수 있다.

```
fragment limitedComments on Post {
  title
  body
  comments(limit: $limit) {
    title
  }
}
```

### 변수 정의

위와 같이 `$limit: Int`로 변수를 정의할 수 있다.
변수는 scala, enum, 혹은 input 타입으로 정의할 수 있다.
변수로 복잡한 객체를 넘겨야 하는 경우 서버에서 제공하는 input 타입을 사용한다.

위의 질의에서는 `Int` 타입으로 선언되었기 때문에 변수가 없어도 정상적으로 동작한다.
필수값으로 지정하려면 `Int!`로 정의해야 한다.
변수 없이 정상 동작하는지 확인해보자.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20findAllPostsWithLimit(%24limit%3A%20Int)%20%7B%0A%20%20posts(limit%3A%20%24limit)%20%7B%0A%20%20%20%20title%0A%20%20%7D%0A%7D&operationName=findAllPostsWithLimit&variables=%7B%7D"></iframe>

### 변수 기본값

변수를 정의할 때 기본값을 넣을 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20findAllPostsWithLimit(%24limit%3A%20Int%20%3D%202)%20%7B%0A%20%20posts(limit%3A%20%24limit)%20%7B%0A%20%20%20%20title%0A%20%20%7D%0A%7D&operationName=findAllPostsWithLimit&variables=%7B%7D"></iframe>

### 지시자(Directives)

변수가 인수를 동적으로 전달할 수 있었다면, 지시자는 질의를 동적으로 변경할 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20post(%24id%3A%20Int!%2C%20%24withComments%3A%20Boolean!)%20%7B%0A%20%20post(id%3A%20%24id)%20%7B%0A%20%20%20%20title%0A%20%20%20%20comments%20%40include(if%3A%20%24withComments)%20%7B%0A%20%20%20%20%20%20body%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&operationName=post&variables=%7B%0A%20%20%22id%22%3A%201%2C%0A%20%20%22withComments%22%3A%20false%0A%7D"></iframe>

기본적으로 GraphQL은 아래 2개의 지시자를 서버에서 구현하도록 정의했다.
- `@include(if: Boolean)`: 인수가 `true`일 때에만 해당 필드를 포함.
- `@skip(if: Boolean)`: 인수가 `true`이면 해당 필드를 건너뜀.

서버는 새로운 지시자를 정의하여 구현할 수도 있다.

### 변경문(Mutations)

REST API에서 컨벤션 상에서 GET 요청으로는 변경을 일으키지 않듯이, GraphQL에서도 mutation으로 데이터를 변경한다.

아래 mutation을 살펴보자
```
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}

# variables
{
  "ep": "JEDI",
  "review": {
    "stars": 5,
    "commentary": "This is a great movie!"
  }
}
```

위와 같은 mutation에서 createReview를 통해 `ep`, `review` 변수가 제공되고,
`star`와 `commentary`를 필드로 갖는 오브젝트가 반환된다.
즉 Java 서버의 경우 아래와 같은 메서드 시그니처로 정의할 수 있다.

```java
public class ReviewService implements GraphQLQueryResolver {
  public Review createReview(Integer stars, ReviewInput review) {
    //...
  }
}
```

위의 변수로 주어진 `review`가 앞에서 이야기했던 input 타입이다.

### mutation의 필드들

질의문과 변경문의 가장 큰 차이점은, 질의문은 각 필드에 대해서 병렬로 실행되는 반면에 변경문의 각 필드는 순차 실행된다는 것이다.
즉, 한 번의 요청으로 `count`를 1 증가시키는 변경문을 두 개를 보낼 때,
첫 번째 요청이 완료되고 두 번째 요청이 실행되기 때문에 레이스 컨디션(Race Condition)에 빠질 우려가 없다.

### Inline Fragments

GraphQL은 타입을 추상화할 수 있는 interface를 제공하고 있으며, 다중 상속과 비슷한 union 타입 또한 제공하고 있다.

탈것에 대해 다음과 같이 스키마가 정의되어 있다.
```
interface Vehicle {
    id: Int!
    manufacturer: String!
    name: String!
}

type Car implements Vehicle {
    id: Int!
    manufacturer: String!
    name: String!
    tire: String!
}

type Airplane implements Vehicle {
    id: Int!
    manufacturer: String!
    name: String!
    flyLimit: String!
}
```

이러한 탈것 추상 타입의 구현체인 자동차, 비행기에 따라 다르게 동작하도록 하려면 아래와 같이 inline fragment를 추가한다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20vehicle(id%3A%201)%20%7B%0A%20%20%20%20name%0A%20%20%20%20manufacturer%0A%20%20%20%20...%20on%20Car%20%7B%0A%20%20%20%20%20%20tire%0A%20%20%20%20%7D%0A%20%20%20%20...%20on%20Airplane%20%7B%0A%20%20%20%20%20%20flyLimit%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D
&variables=%7B%7D"></iframe>

### 메타 필드

위에서의 추상 타입에 대한 구현체에 따라 클라이언트의 동작이 달라져야 하는 경우가 있을 수 있다.
이럴 때에는 `__typename` 메타 필드를 추가하여 실제 어떤 구현체가 응답으로 오는지 확인할 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20vehicle(id%3A%201)%20%7B%0A%20%20%20%20__typename%0A%20%20%20%20name%0A%20%20%20%20manufacturer%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

# 스키마와 타입

### 타입

```
{
  vehicle {
    name
    manufacturer
  }
}
```

위 예제와 같이 GraphQL에서 타입은:
- root 오브젝트로 시작한다.
- root 오브젝트에 field(`vehicle`)를 선언한다.
- vehicle에서 받을 응답값을 선택적으로 기재한다.

요청 스키마를 따라 구조화된 JSON 응답이 오기 때문에, 서버 응답을 예측할 수 있다.
때문에 클라이언트는 어떤 구조로 요청을 보낼 수 있는가에 대해 알아야 하는데 이를 스키마로 정의해두었다.

GraphQL을 제공하는 애플리케이션은 우리가 어떤 데이터를 질의할 수 있는지를 정의한 타입 셋을 가지고 있다.
그리고 애플리케이션은 질의가 들어오면 이 스키마를 가지고 질의 된 값들의 유효성을 검증한다.

### 객체 타입과 필드

GraphQL의 스키마에서 가장 기초적인 것은 오브젝트 타입이다.

```
type Album {
  name: String!
  photos: [Photo!]!
}
```

- 타입의 이름을 `Album`로 지었다.
- `name`과 `photo`는 `Album` 객체의 필드이다.
- `String`은 GraphQL에서 기본으로 제공하는 스칼라 타입이다. 스칼라 타입은 다시 알아보기로 하자.
- `String!`에서 `!`는 해당 필드가 Null을 허용하지 않음을 뜻한다.(non-null)
- `[Photo!]!`는 `Photo` 객체의 배열을 뜻한다.
  - `[]!`는 즉 `photos` 변수는 Null일 수 없음을 의미한다.
  - `Photo!`는 배열의 각 요소가 Null일 수 없음을 의미한다.

### Query와 Mutation 타입

GraphQL에서 대부분은 오브젝트 타입이지만, 아래의 두 타입은 특별한 취급을 받는다.

```
schema {
  query: Query
  mutation: Mutation
}
```

모든 GraphQL 서비스는 `query` 타입을 가지며 `mutation` 타입은 있을 수도 있고 없을 수도 있다.
두 타입은 일반적인 오브젝트 타입과 같지만, 모든 GraphQL의 진입점이 된다는 점에서 특별 취급을 받는다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20%7B%0A%20%20user%20%7B%0A%20%20%20%20name%0A%20%20%20%20email%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

위 질의문을 수행하기 위해서 서비스는 아래의 스키마를 가져야 한다.

```
type Query {
  user: User
}
```

`mutation` 또한 비슷한 방식으로 동작한다.

### 스칼라 타입

GraphQL은 다음의 스칼라 타입을 기본으로 제공한다.

- Int: signed 32-bit integer.
- Float: signed double-precision floating-point.(64-bit)
- String: UTF-8 Character Sequence.
- Boolean
- ID: 고유 식별자를 나타내는 타입이다. 주로 객체를 다시 페치해오거나, 캐시키로 사용된다.
ID는 String과 같은 방식으로 직렬화되지만, ID는 난독화된다는 것에 의미를 가진다.

대부분의 GraphQL 서비스는 자체적으로 스칼라 타입을 정의할 수 있다. Java도 가능하다.

아래와 같이 `Date` 타입을 정의할 수 있다.

```
scalas Date
```

이에 대해 어떻게 역/직렬화할 것인지는 구현하기에 달렸다.

### Enumumeration

아래와 같이 enum을 정의할 수 있다.

```
enum UserProvider {
  NAVER
  LINE
  KAKAO
}
```

GraphQL 서비스는 여러 언어로 구현될 수 있으므로 각자 언어에 맞게 enum을 구현한다.
enum을 객체로 다루는 언어에서는 그 혜택을 누릴 수 있겠지만,
Javascript와 같이 enum을 지원하지 않는 언어에서는 단순히 integer로 다뤄질 수 있다.
**하지만 이러한 서비스의 세부 동작은 클라이언트에 노출되지 않으며,**
**클라이언트는 정의된 enum 값들 내에서 질의, 변경을 수행할 수 있다.**

### List와 Non-null

```
type Album {
  name: String!
  photos: [Photo!]!
}
```

`name`은 `String` 타입이며 `!`를 뒤에 붙여 `Non-Null`을 명시하였다.
이 제약으로 인해 서버는 `name`에 `null` 값을 줄 수 없으며, `null`이 오는 경우에는 오류를 발생시킨다.
`Non-Null`은 인수에도 선언할 수 있다.
아래와 같이 `Non-Null`로 선언된 인수에 `null`이 들어가면 오류가 질의를 실행할 때 오류가 발생한다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=query%20getVehicleById(%24id%3A%20Int!)%7B%0A%20%20vehicle(id%3A%20%24id)%20%7B%0A%20%20%20%20name%0A%20%20%20%20manufacturer%0A%20%20%7D%0A%7D&variables=%7B%0A%20%20%22id%22%3A%20null%0A%7D&operationName=getVehicleById"></iframe>

`Non-Null`과 `List`는 조합해서 사용할 수 있다.

| myField: [String!]| |
| - | - |
| `myField: null` | valid |
| `myField: []` | valid |
| `myField: ['a', 'b']` | valid |
| `myField: ['a', null, 'b']` | error |

| myField: [String]! | |
| - | - |
| `myField: null` | error |
| `myField: []` | valid |
| `myField: ['a', 'b']` | valid |
| `myField: ['a', null, 'b']` | valid |

### 인터페이스

GraphQL은 추상 타입으로 인터페이스를 제공한다.
인터페이스를 정의하고 구현하는 방법은 다음과 같다.

```
interface Vehicle {
  id: Int!
  manufacturer: String!
  name: String!
}

type Car implements Vehicle {
  id: Int!
  manufacturer: String!
  name: String!
  tire: String!
}

type Airplane implements Vehicle {
  id: Int!
  manufacturer: String!
  name: String!
  flyLimit: String!
}
```

### Union

Union은 각 타입을 묶어낼 수 있다는 점이 인터페이스와 비슷하나 공통 필드가 선언되지 않는다는 점에서 차이가 있다.
다음과 같이 선언한다.

```
union SearchResult = Album | Post | Todo
```

클라이언트에서는 마치 통합 검색과도 같이 사용할 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20search(title%3A%20%22quos%22)%20%7B%0A%20%20%20%20...%20on%20Album%20%7B%0A%20%20%20%20%20%20title%0A%20%20%20%20%7D%0A%20%20%20%20...%20on%20Post%20%7B%0A%20%20%20%20%20%20title%0A%20%20%20%20%7D%0A%20%20%20%20...%20on%20Todo%20%7B%0A%20%20%20%20%20%20completed%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

### Input

여태까지는 enum이나 String 등의 스칼라 타입만 인수로 사용했었으나, 더 복잡한 객체 또한 인수로 넘길 수 있다.
이 경우 아래와 같이 `Input` 타입을 정의할 수 있다.

```
input CommentInput {
  name: String!
  email: String!
  body: String!
}
```

특정 Post에 댓글을 다는 변경을 다음과 같이 표현할 수 있겠다.

```
mutation CreateComment($id: Int!, $comment: CommentInput) {
  createComment(id: $id, comment: $comment) {
    id
  }
}

# Variable
{
  "id": 1,
  "comment": {
    "name": "댓글러",
    "email": "댓글러@naver.com",
    "body": "댓글"
  }
}
```

# 검증(Validation)

타입을 사용하기 때문에 GraphQL의 질의가 유효한지 여부를 미리 알 수 있다.
덕분에 서버와 클라이언트는 런타임으로 검증되기 전에 질의 생성 시점에서 유효하지 않은 질의를 알 수 있다.

fragment는 자기 자신을 참조하지 못한다. 아래 구문은 오류가 발생한다.

```
{
  task {
    ...TitleAndSubTasks
  }
}

fragment TitleAndSubTasks on Task {
  title
  subTasks {
    ...TitleAndSubTasks
  }
}
```

질의문은 해당 타입에 존재하는 필드만 선언해야 한다. 아래 구문은 오류가 발생한다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20todo(id%3A1)%20%7B%0A%20%20%20%20undefiendField%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

또한 스칼라, enum 타입 외의 다른 타입을 질의한다면 반드시 하위 필드를 명시해야 한다.
아래 질의는 오류가 발생한다.

```
{
  todo(id: 1)
}
```

# Introspection

GraphQL은 Introspection System으로 어떤 스키마가 정의되어 있는지 확인할 수 있다.
`__schema` 필드를 사용하여, 어떤 타입들이 사용가능한 지 알 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20__schema%20%7B%0A%20%20%20%20types%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

꽤 많은 타입들을 발견할 수 있다. 다음과 같이 정리해보자.

- 사용자 정의 객체 타입: Query, Todo, Post 등 개발자가 정의한 타입.
- 스칼라 타입: String, Int, Boolean
- Introspection System Type: __Schema, __Type, __TypeKind, __Field, __InputValue, __EnumValue, __Diretive

마지막에 다룬 부분이 GraphQL의 스키마를 엿볼 수 있는 Introspection의 타입들이 되겠다.

아래와 같이 최초 진입점이었던 `Query`를 조회할 수 있다.
> GraphiQL에서 제공하는 우측 상단의 `< Docs`와 비교해보자.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20__schema%20%7B%0A%20%20%20%20queryType%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%20%20fields%20%7B%0A%20%20%20%20%20%20%20%20name%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D&variables=%7B%7D"></iframe>

특정 타입에 대해서는 `__type`으로 조회할 수 있다.

<iframe style="width: 100%" height="400px" resize="vertical" frameBorder="0" src="https://supawer-graphql-study.herokuapp.com/graphiql?query=%7B%0A%20%20__type(name%3A%20%22Post%22)%20%7B%0A%20%20%20%20name%0A%20%20%20%20kind%0A%20%20%20%20fields%20%7B%0A%20%20%20%20%20%20name%0A%20%20%20%20%20%20type%20%7B%0A%20%20%20%20%20%20%20%20ofType%20%7B%0A%20%20%20%20%20%20%20%20%20%20name%0A%20%20%20%20%20%20%20%20%20%20kind%0A%20%20%20%20%20%20%20%20%20%20description%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D%0A&variables=%7B%7D"></iframe>

이러한 Introspection의 도움으로 IDE나 브라우저에서 좋은 UX툴을 만들 수 있다.
이미 우리가 체험한 [GraphiQL](https://supawer-graphql-study.herokuapp.com/graphiql)도 마찬가지이고 [Voyager](https://supawer-graphql-study.herokuapp.com/voyager) 또한 문서화할 때 유용하다.

# 맺음말

GraphQL에 대해서 기초적인 내용들을 익혀보았다.

GraphQL은 타입 기반으로 스키마를 미리 정의한다.
클라이언트는 그 스키마를 바탕으로 질의문을 생성, 조작할 수 있다.
미리 정의된 스키마가 있으므로 질의에 사용된 인수에 대한 유효성 검사를 쉽게할 수 있다.
또한 Introspection System을 제공하고 있으므로 유용한 헬퍼 툴을 만들기 쉽고, 문서화하기도 편리하다.

GraphQL을 어떻게 사용할 수 있는지는 알게 되었으나, 슬슬 백엔드에서는 어떻게 GraphQL을 구현했을 지 궁금할 즈음이다.
처음에는 체험하기, 기초, Spring Boot 연동의 3부작으로 포스트를 작성할 예정이었으나, 생각이 바뀌어 5부작이 될 것 같다.
체험하기, 기초, Best Practice, GraphQL Java, GraphQL On Spring Boot로 진행할 예정이다.
