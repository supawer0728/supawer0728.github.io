---
title: Mustache 살펴보기
date: 2018-03-13 16:39:18
tags: [view template engine, view, template]
categories:
  - [spring,view]
---

# 개요

Java 서버를 개발하면서 View Template Engine에 대해서 매번 고민하게 된다.

spring boot를 써보면,
- [더 이상 JSP를 쓰지 말아야할 것처럼 말한다](https://docs.spring.io/spring-boot/docs/2.0.0.RELEASE/reference/htmlsingle/#boot-features-spring-mvc-template-engines)
  - boot에서 jsp를 쓸 경우 war로 만들어야하며, WAS에 따라서 지원하지 않을 수 있다 - Tomcat만 고집한다면야...
- Velocity는 boot에서 지원하지 않는다
  - 너무 오랫동안 업데이트가 없다면서 뺐는데, 2017-08-06에 Velocity Engine 2.0이 나왔다
- Thymeleaf3는 여전히 느린 모양이다.
- Freemarker가 그나마 가장 무난하다.

위와 같이 알고 있던 중에 최신 spring 문서를 뒤져보다가, `script views`라는게 있다는 걸 알게 되었다.
Handlebars, Mustache, React, Kotlin Script templating 등등의 많은 라이브러리가 있었는 데, 그 중에서 Mustache를 살펴보려 한다.

<!-- more -->

# Mustache

Mustache는 Simple하다.
얼마나 Simple하냐면, 설명도 `Logic-less templates.`가 끝이다.
별다른 로직이 존재하지 않아서 배우는 것도 간단하다.
`spring-boot`에서 `starter`를 지원하기 때문에 간단하게 설정할 수 있다.

하지만 가장 중요한 것은 `Logic-less`라는 점이다.
MVC 패턴으로 개발을 하다보면, view에 로직을 구현하려는 경우를 종종 보게된다.
web view를 작성하다보면 필연적으로 JavaScript를 사용하게 되고,
client에서 필요한 로직을 JavaScript로 구현하게 된다.
개인적으로 client의 역할과 view의 역할은 분리되어야 한다고 생각하며,
view에 로직이 구현되어 있는 경우, OOP의 5원칙 중에 SRP를 어긴것으로 생각한다.

때문에 Mustache의 `Logic-less`라고 하는 점은 매우 끌리는 장점 중 하나이다.

# Mustache 배워보기

기본적으로 Mustache의 [Manual](https://mustache.github.io/mustache.5.html)을 따르며 하나씩 spring-boot와 연계해보겠다.

## 프로젝트 의존성 설정

spring-boot 버전 : 2.0.0.RELEASE

간편하게 만들기 : https://start.spring.io/


```gradle
dependencies {
	compile('org.springframework.boot:spring-boot-starter-mustache')
	compile('org.springframework.boot:spring-boot-starter-webflux')
	runtime('org.springframework.boot:spring-boot-devtools')
	compileOnly('org.projectlombok:lombok')
	testCompile('org.springframework.boot:spring-boot-starter-test')
	testCompile('io.projectreactor:reactor-test')
}
```

## application.yml

기본설정으로 `spring.mustache.suffix=.mustache`로 되어있다.
이를 `.html`로 바꿨다. (취향)

```yml
spring.mustache.suffix: .html
```

## model

[Online Fake Rest Api](https://jsonplaceholder.typicode.com/posts/1)를 사용했다

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Post {

    public static final Post EMPTY = new Post(0, "");

    private int id;
    private String body;
}
```

## controller

`GET /posts/{id}`를 할 경우, `/posts/detail` 템플릿을 보여준다.
`spring.mustache.prefix=classpath:/templates/`가 기본설정으로 되어 있다.

```java
@Controller
@RequestMapping("/posts")
public class PostController {

  @GetMapping("/{id}")
  public String getPost(@PathVariable Long id, Model model) {

      Mono<Post> post = WebClient.create("https://jsonplaceholder.typicode.com")
                                 .get()
                                 .uri("/posts/{id}", id)
                                 .retrieve()
                                 .bodyToMono(Post.class)
                                 .onErrorReturn(Post.EMPTY);

      model.addAttribute("post", post);
      return "/posts/detail";
  }
}
```

## view

`src/main/resources/templates/posts/detail.html`

Post 객체의 속성을 보여준다

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Post</title>
</head>
<body>
<div class="container">
    <table>
        <tr>
            <td>id</td>
            <td>body</td>
        </tr>
        {{#post}}
        <tr>
            <td>{{id}}</td>
            <td>{{body}}</td>
        </tr>
        {{/post}}
    </table>
</div>
</body>
</html>
```

### 변수

콧수염 두 개(`{{id}}`)로 표현한다

#### HTML Escape

콧수염 안의 문자열은 기본으로 HTML Escape가 적용된다

```java
model.addAttribute("company", "<b>GitHub</b>")
```

```html
- {{company}}
```

```
- &lt;b&gt;GitHub&lt;/b&gt
```

unescape를 하기 위해서는
- 괄호를 3번 중첩 시키거나(`{{{ compnay }}}`)
- `&`를 사용한다(`{{& company }}`)

```html
- {{{company}}}
```

```
- <b>GitHub</b>
```

### 섹션

섹션은 콧수염과 샵으로 표현한다(`{{#post}}`)

섹션에 들어가는 key 타입에 따라서 표현이 달라진다

#### False, Empty List

key의 값이 false이거나 empty list인 경우, 해당 섹션은 표시되지 않는다

```html
Shown.
{{#post}}
  Never shown!
{{/person}}
```

```java
model.addAttribute("post", false);
// 혹은
model.addAttribute("post", Collections.emptyList());
```

**output**

```html
Shown.
```

## templating

앞서 본 `detail.html`에서 변화하는 부분만 템플릿으로 뽑아보자

`src/main/resources/templates/posts/detail.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Post</title>
</head>
<body>
<div class="container">
    <table>
        <tr>
            <td>id</td>
            <td>body</td>
        </tr>
        {{#post}}
        {{> posts/post_item}}
        {{/post}}
    </table>
</div>
</body>
</html>
```

`src/main/resources/templates/posts/post_item.html`

```html
<tr>
    <td>{{id}}</td>
    <td>{{body}}</td>
</tr>
```

## listing

리스트를 반환해보자.

```java
private static final ParameterizedTypeReference<List<Post>> postsListType = new ParameterizedTypeReference<List<Post>>() {};

@GetMapping
public String getList(Model model) {
    Mono<List<Post>> posts = WebClient.create("https://jsonplaceholder.typicode.com")
                                      .get()
                                      .uri("/posts")
                                      .retrieve()
                                      .bodyToMono(postsListType)
                                      .onErrorReturn(Collections.emptyList());

    model.addAttribute("post", posts);
    return "/posts/detail";
}
```

`/post/detail.html`은 수정할 것이 없다.
`{{#post}}`가 list면 `foreach`를 수행한다.
