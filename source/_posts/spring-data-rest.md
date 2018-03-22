---
title: spring-data-rest 소개
date: 2018-03-20 10:26:46
tags: [spring]
categories: 
  - [spring,reference]
  - [spring,data]
---

# 개요

Spring Data REST는 도메인 모델과 repository를 분석해서, `RESTful API`를 제공해준다.
본 문서에서는 Spring Data REST에 대한 간단한 예제와 함께 사용법에 관한 소개를 하고자 한다.

<!-- more -->

# RESTful API

뜬금없지만 `RESTful`에 관해 좀 더 이야기를 해두고자 한다.
아래 그림은 마틴 파울러 아저씨의 [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html)에 나오는 `REST의 영광`을 누리기 위한 단계를 표현하고 있다.

| ![image1](hateoas.png) |
| :--: |
| *출처: https://martinfowler.com/articles/images/richardsonMaturityModel/overview.png* |

> 한국어 풀이로 해둔 블로그가 있다(http://jinson.tistory.com/190)

결국 `api가 RESTful하다`는 말을 듣기 위해서는 `level 3`를 충족시켜야 한다.
`level 3`를 충족시키기 위해서 `HATEOAS(Hypertext As The Engine Of Application State)`개념을 도입해야 한다.
`HATEOAS`가 어떤 것인지 상세히 설명하는 것은 본 문서에서 초점을 맞추는 부분이 아니기에, 여기서는 하지 않겠다.
정말 간단히 요약하자면 아래와 같다.

- Resource로 `무엇`을 할 수 있는지 알 수 있다.
- `무엇`을 `어떻게` 해야 하는 지 알려준다.

`HATEOAS`에 대해서 궁금하다면 상기한 블로그를 읽어보자

# 간단한 예제

백문이 불여일견이다.
우선 예제를 보며 어떻게 동작하는 것인지 알아보자.
여기서는 `spring-data-jpa`와 연동하여 사용한다.

## 의존성

`spring-boot` 버전은 `2.0.0.RELEASE`다.

```gradle
dependencies {
  compile('org.springframework.boot:spring-boot-starter-data-jpa')
  compile('org.springframework.boot:spring-boot-starter-data-rest')
  compileOnly('org.projectlombok:lombok')
  runtimeOnly('com.h2database:h2')
  testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

## Domain

**Member**

```java
@Entity
@Table(name = "member")
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@Getter(AccessLevel.PUBLIC)
@Setter(AccessLevel.PRIVATE)
@EqualsAndHashCode
public class Member {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private Integer age;

    @Enumerated(EnumType.STRING)
    private Grade grade;

    private Member(String name, Integer age, Grade grade) {
        this.name = name;
        this.age = age;
        this.grade = grade;
    }

    public static Member join(@NonNull String name, @NonNull Integer age) {
        return new Member(name, age, Grade.BRONZE);
    }
}
```

**Repository**

```java
@RepositoryRestResource
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

## application 설정

**application.yml**

```yml
spring.data.rest.base-path: /api
```

**SimpleDataRestApplication.java**

```java
@SpringBootApplication
public class SimpleDataRestApplication implements CommandLineRunner {

    @Autowired
    private MemberRepository memberRepository;

    public static void main(String[] args) {
        SpringApplication.run(SimpleDataRestApplication.class, args);
	}

    @Override
    public void run(String... args) throws Exception {
        memberRepository.save(Member.join("member1", 20));
        memberRepository.save(Member.join("member2", 21));
        memberRepository.save(Member.join("member3", 22));
        memberRepository.save(Member.join("member4", 23));
        memberRepository.save(Member.join("member5", 24));
    }
}
```

## 실행

미리 데이터를 넣거나 했지만, 
사실 **Domain**영역에서 `entity`과 `repository`만 정의했다.
바로 실행해서 테스트해 볼 수 있다.

**`GET /api`**

`application.yml`에서 `spring-rest-data`의 root를 `/api`로 잡았다.
root uri의 자원을 호출하면 어떻 것이 나오는 지 확인하자.

```
{  
   "_links":{  
      "members":{  
         "href":"http://localhost:8080/api/members{?page,size,sort}",
         "templated":true
      },
      "profile":{  
         "href":"http://localhost:8080/api/profile"
      }
   }
}
```

- root 경로의 하위로 어떠한 자원들을 가지고 있는지 알려준다.
  - members를 가지고 있다
  - members에 접근하려면 `/api/members`를 호출해야 한다고 알려준다(`HATEOAS`)

**`GET /api/members`**

```
{
  "_embedded" : {
    "members" : [ {
      "name" : "member1",
      "age" : 20,
      "grade" : "BRONZE",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/api/members/1"
        },
        "member" : {
          "href" : "http://localhost:8080/api/members/1"
        }
      }
    }, {
      ...
    }]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/members{?page,size,sort}",
      "templated" : true
    },
    "profile" : {
      "href" : "http://localhost:8080/api/profile/members"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 5,
    "totalPages" : 1,
    "number" : 0
  }
}
```
목록 호출을 해보면

- 페이징 처리를 해주고 있다.
  - `MemberRepository`가 `PagingAndSortingRepository`를 구현하고 있기 때문이다.
  - 현재 총 5개의 요소가 있는데, 만약 `page=1&size=2`라는 query를 추가했다면 `_links`에 다음 주소가 추가된다
    - first : http://localhost:8080/api/members?page=0&size=2
    - previous : http://localhost:8080/api/members?page=0&size=2
    - next : http://localhost:8080/api/members?page=2&size=2
    - last : http://localhost:8080/api/members?page=2&size=2
- 목록들을 보면 ID가 없다
  - `HATEOAS`를 써서 `자기 서술적(self-descriptive)`으로 동작한다.
  - 때문에 상세를 조회하기 위해 client는 `location.href = '/api/members/' + members{% raw %}[0]{% endraw %}.id`가 아니라,
`location.href = members{% raw %}[0]{% endraw %}._link.self.href`를 넣어주면 된다.

**`GET /api/members/1`**

```
{
  "name" : "member1",
  "age" : 20,
  "grade" : "BRONZE",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/members/1"
    },
    "member" : {
      "href" : "http://localhost:8080/api/members/1"
    }
  }
}
```

**`POST /api/members`**

생성 요청을 해보자

```
{
  "name" : "Test Created",
  "age" : 22,
  "grade" : "BRONZE"
}
```

아래와 같이 `201 CREATED` 응답이 온다

```
{
  "name" : "Test Created",
  "age" : 22,
  "grade" : "BRONZE",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/members/6"
    },
    "member" : {
      "href" : "http://localhost:8080/api/members/6"
    }
  }
}
```

**`PATCH /api/members/6`**

수정 요청을 해보자

```
{
  "age" : 30
}
```

아래와 같이 응답이 온다(200 OK)

```
{
  "name" : "Test Created",
  "age" : 30,
  "grade" : "BRONZE",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/members/6"
    },
    "member" : {
      "href" : "http://localhost:8080/api/members/6"
    }
  }
}
```

`PUT`이나 `DELETE`도 당연히 동작을 하기 때문에 굳이 설명하지 않겠다.

# 기본 설정

앞서 예제를 통해 기본적인 동작을 살펴보았다.
이번에는 `spring-data-rest`가 어떠한 기본 설정으로 동작하는 지를 설명하겠다.
[Spring Data REST Reference](https://docs.spring.io/spring-data/rest/docs/3.0.5.RELEASE/reference/html/#getting-started.basic-settings)에서 Java 설정의 예제가 있다.
여기서는 `spring-boot`에서 지원해주는 properties를 사용하여 설정하는 것을 예로 들겠다.

**Prefix는 `spring.data.rest.`**

| property | default | description |
| - | - | - |
| base-path | | repository resource를 노출할 기본 경로 |
| default-media-type | `application/hal+json;charset=UTF-8` | 개별 설정이 없을 때 사용할 `Content-Type` |
| default-page-size | | 기본 페이지 사이즈 |
| detection-strategy | default | api로 만들 repository를 찾는 전략<br>- `default` : 모든 public repository. `@RestResource`의 `exported`가 `false`인 경우 제외<br>- `all` : 가시성, annotation과 상관 없이 모든 repository 노출<br>- `annoation` : `exported`가 `true`이며 `@RepositoryRestResource`, `@RestResource`가 달린 자원들을 노출<br>- `visibility` : public interface만 노출 |
| limit-param-name | size | 한 번에 반환되는 결과 갯수를 받을 query string 이름<br>`/api/members?size=10` |
| max-page-size | 20 | 페이지당 항목 수 |
| page-param-name | page | 페이지 번호를 받을 query string 이름<br>`/api/members?page=2` |
| return-body-on-create | true | 요청에 의해 생성된 엔티티를 응답으로 내려줄 지 여부 |
| return-body-on-update | true | 요청에 의해 수정된 엔티티를 응답으로 내려줄 지 여부 |
| sort-param-name | sort | 정렬 요청을 받을 query string 이름<br>`/api/members?sort=name,desc` |

# 지원하는 저장소

- Spring Data JPA
- Spring Data MongoDB
- Spring Data Neo4j
- Spring Data GemFire
- Spring Data Cassandra

# Search API

repository에 [query method](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods)를 추가해보자.

```java
@RepositoryRestResource
public interface MemberRepository extends JpaRepository<Member, Long> {

    Member findByName(@Param("name") String name);
}
```

**`GET /api/members`***

```
{
  "_embedded" : { ... },
  "_links" : {
    "self" : { ... },
    "profile" : { ... },
    "search" : {
      "href" : "http://localhost:8080/api/members/search"
    }
  },
  "page" : { ... }
}
```

links에 search가 추가되었다.
search를 확인해보자.

**`GET /api/members/search`**

```
{
  "_links" : {
    "findByName" : {
      "href" : "http://localhost:8080/api/members/search/findByName{?name}",
      "templated" : true
    },
    "self" : {
      "href" : "http://localhost:8080/api/members/search"
    }
  }
}
```

사용법에 대한 설명이 나온다

**`GET /api/members/search/findByName?name=member1`

```
{
  "name" : "member1",
  "age" : 20,
  "grade" : "BRONZE",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/members/1"
    },
    "member" : {
      "href" : "http://localhost:8080/api/members/1"
    }
  }
}
```

## Paging 처리

`query method`에 `Pageable`를 파라미터로 줘서, Paging처리를 시킬 수 있다.

```java
@RepositoryRestResource
public interface MemberRepository extends JpaRepository<Member, Long> {

    Page<Member> findByName(@Param("name") String name, Pageable pageable);
}
```

# Projection

interface와 annotation의 조합으로 projection 할 수 있다.

```java
@RepositoryRestResource
public interface MemberRepository extends JpaRepository<Member, Long> {

    Member findByName(@Param("name") String name);
}

@Projection(name = "only-name", types = {Member.class})
interface OnlyName {

    String getName();
}
```

**`GET /api/members/1?projection=only-name`**

```
{
  "name" : "member1",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/members/1"
    },
    "member" : {
      "href" : "http://localhost:8080/api/members/1{?projection}",
      "templated" : true
    }
  }
}
```

projection의 목록은 `/api/profile/members`에서 조회할 수 있다.

![profile1](profile1.png)

## JsonIgnore 값 보내기

회원의 연령 정보에 `@JsonIgnore`가 걸린 상황이라도,

**Member.java**

```java
private String name;
@JsonIgnore
private Integer age;
```

`@Projection`을 사용하면 노출시킬 수 있다.

```java
@Projection(name = "how-old", types = { Member.class })
interface HowOldProjection {
  Integer getAge();
}
```

## SpEL

`@Projection` 내의 메서드에 [SpEL](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/core.html#expressions)을 적용할 수 있다.

```java
@Projection(name = "brief", types = { Member.class })
public interface BriefProjection {

  @Value("#{target.name} : #{target.age}") 
  String getBrief();
}
```

## 엔티티 조합에서 Projection

`Board` 엔티티가 `Member` 엔티티를 `writer`로서 가지고 있다고 가정하자.

```java
@Entity
@Table(name = "board")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter(AccessLevel.PUBLIC)
@Setter(AccessLevel.PRIVATE)
@EqualsAndHashCode
public class Board {

    @Id
    @GeneratedValue
    private Long id;

    private String title;
    private String content;

    @ManyToOne
    @JoinColumn(name = "writer_id")
    private Member writer;
}
```

이 경우 writer는 어떻게 참조될까?

**`GET /api/boards/1`**

```
{
  "title" : "title1",
  "content" : "content1",
  "_links" : {
    ... ,
    "writer" : {
      "href" : "http://localhost:8080/api/boards/6/writer"
    }
  }
}
```

보다시피 `writer` 정보를 얻기 위해서 `/api/boards/6/writer`를 호출해야 한다.
`writer`정보를 한 번에 얻을 수 있도록 `@Projection` 설정을 할 수 있다.

```java
@Projection(name = "with-writer", types = {Board.class})
interface WithWriterProjection {
    String getTitle();
    String getContent();
    Member getWriter();
}
```

**`GET /api/boards/6?projection=with-writer`**

```
{
  "content" : "content1",
  "title" : "title1",
  "writer" : {
    "name" : "member1",
    "age" : 20,
    "grade" : "BRONZE"
  },
  "_links" : { ... }
}
```

### 엔티티 조합 Projection 상시 설정

`@RepositoryRestResource`의 `excerptProjection`으로 상시 설정할 `Projection`을 걸어 둘 수 있다.

```java
@RepositoryRestResource(excerptProjection = WithWriterProjection.class)
public interface BoardRepository extends JpaRepository<Board, Long> {
}
```

# Header와의 연계

## ETag, If-Match, If-None-Match

`spring-data-commons`에는 `@Version`이라는 애노테이션이 있다.

> JPA에서는 이 Version으로 낙관적 락을 구현하기도 한다.

```java
public class Member {
  @Version Long version
  // ...
}
```

Spring Data REST에서는 위와 같이 `@Version` 주석이 달린 객체를 응답할 때, `ETag` 헤더를 추가한다.

**`GET /api/members/1`**

```
ETag: "0"
Content-Type: application/hal+json;charset=UTF-8
Transfer-Encoding: chunked
Date: Tue, 20 Mar 2018 05:50:15 GMT
```

`ETag` 헤더를 획득하였으니 이제 `If-Match`, `If-Non-Match` 등의 헤더를 추가해서 사용할 수 있다.

**If-Match**

```
curl -v -X PATCH -H 'If-Match: <value of previous ETag>' ...
```

**If-None-Match**

```
curl -v -H 'If-None-Match: <value of previous etag>' ...
```

## If-Modified-Since

`@LastModifiedDate` 애노테이션을 사용하여 `Last-Modified` 응답 헤더를 줄 수 있다.

```java
public class Member {
  @LastModified
  // ...
}
```

이를 이용해 `If-Modified-Since` 헤더를 추가해서 요청할 수 있다.

```
curl -H "If-Modified-Since: Wed, 24 Jun 2015 20:28:15 GMT" ...
```

**`GET /api/members/1`**

```
Last-Modified: Tue, 20 Mar 2018 05:57:01 GMT-4s
Date: Tue, 20 Mar 2018 05:57:05 GMT
ETag: "0"
Content-Type: application/hal+json;charset=UTF-8
```

# Events

엔티티를 처리하는 중에 아래의 이벤트들이 발생하며, 이를 처리할 수 있다.

- BeforeCreateEvent
- AfterCreateEvent
- BeforeSaveEvent
- AfterSaveEvent
- BeforeLinkSaveEvent
- AfterLinkSaveEvent
- BeforeDeleteEvent
- AfterDeleteEvent

## AbstractRepositoryEventListener를 상속

`AbstractRepositoryEventListener`에는 각 이벤트별로 처리하기 위한 메서드들이 `protected`로 정의되어 있다.
필요한 메서드를 재정의하여 사용한다.

```java
public class MemberEventListener extends AbstractRepositoryEventListener<Member> {

  @Override
  public void onBeforeSave(Member member) {
    // ...
  }

  @Override
  public void onAfterDelete(Member member) {
    // ...
  }
}
```

## Annotated Handler

`@RepositoryEventHandler` 등의 애노테이션으로 이벤트 핸들러를 등록할 수 있다.
이 때 핸들러는 spring에 bean으로 등록되어야 한다.

```java
@RepositoryEventHandler 
public class MemberEventHandler {

  @HandleBeforeSave
  public void handleMemberSave(Member p) {
    // ...
  }

  @HandleBeforeSave
  public void handleAddressSave(Address p) {
    // ...
  }
}
```

# Security

`spring-security`와 연동하여 권한에 따른 연산을 수행할도록 설정할 수 있다.

## @PreAuthorize

`MemberRepository`의 `findByName`은 `ADMIN` 권한의 사용자만 사용할 수 있고,
나머지는 `USER` 권한의 사용자도 사용할 수 있다.

```java
@PreAuthorize("hasRole('USER')")
@RepositoryRestResource
public interface MemberRepository extends JpaRepository<Member, Long> {

    @PreAuthorize("hasRole('ADMIN')")
    Member findByName(@Param("name") String name);
}
```

# 마무리

Spring Data REST에 대해서 알아보았다.
Repository와 Entity의 내용을 기반으로 간편히 RESTful API를 제공해주기는 하지만,
DDD에 적용할 정도의 정교함을 보여주지는 않는 것 같다.
`board` 엔티티에 `public void like()` 메서드를 `/api/boards/1/like`를 통해 실행할 수 있다면 좋을 것 같다.
생각해보면 `spring-data`의 하위 라이브러리들은 `repository`에 필요한 기능들을 추상화하는 것들이니, 일부러 굳이 서비스 로직까지 건들지는 않는 것 같다.
위에서 말한 것도, 결국 커스텀 설정으로 가능하게 만들 수 있다.

client에서 `HATEOAS`를 적용하여 개발을 한다면 충분히 사용할 만한 기술이라고 생각한다.