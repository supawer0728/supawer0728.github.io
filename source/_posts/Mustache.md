---
title: Mustache 공유
date: 2018-03-14 11:49:21
tags: 
  - view template engine
  - view
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
`spring-boot`에서 `starter`를 지원하기 때문에 설정도 간단하게 할 수 있다.

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

[Online Fake Rest Api](https://jsonplaceholder.typicode.com/users)를 사용했다


```java
@Data
@NoArgsConstructor
public class User {

    public static final User EMPTY = new User();

    private Integer id;
    private String name;
    private String email;
    private String phone;
    private String website;
}
```

## controller

| API | URI | viewName |
| - | - | - |
| 목록 | `/users` | `/users/list` |
- 목록 : `GET /users`
- 상세 : `GET /users/{id}`
- 
`GET /users/{id}`를 할 경우, `/users/detail` 템플릿을 보여준다.
`spring.mustache.prefix=classpath:/templates/`가 기본설정으로 되어 있다.

```java
@Controller
@RequestMapping("/users")
public class UserController {

    private static final ParameterizedTypeReference<List<User>> postsListType = new ParameterizedTypeReference<List<User>>() {
    };

    @GetMapping
    public String getList(Model model) {
        Mono<List<User>> posts = WebClient.create("https://jsonplaceholder.typicode.com")
                                          .get()
                                          .uri("/users")
                                          .retrieve()
                                          .bodyToMono(postsListType)
                                          .onErrorReturn(Collections.emptyList());

        model.addAttribute("user", posts);
        return "/users/list";
    }

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {

        Mono<User> post = WebClient.create("https://jsonplaceholder.typicode.com")
                                   .get()
                                   .uri("/users/{id}", id)
                                   .retrieve()
                                   .bodyToMono(User.class)
                                   .onErrorReturn(User.EMPTY);

        model.addAttribute("user", post);
        return "/users/detail";
    }
}
```

## view

`src/main/resources/templates/users/detail.html`

User 객체의 속성을 보여준다

```html
<div class="container">
    {{#user}}
    <p>
        id : {{id}}<br/>
        name : {{name}}<br/>
        phone : {{phone}}<br/>
        website : {{website}}<br/>
    </p>
    {{/user}}
</div>
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
- 괄호를 3번 중첩 시키거나 : `{% raw %}{{{ compnay }}}{% endraw %}`
- `&`를 사용한다 : `{% raw %}{{& company }}{% endraw %}`

```html
- {{{company}}}
```

```
- <b>GitHub</b>
```

### 섹션

섹션은 콧수염과 샵으로 표현한다(`{% raw %}{{ #post }}{% endraw %}`)

섹션에 들어가는 key 타입에 따라서 표현이 달라진다

#### False, Empty List

key의 값이 false이거나 empty list인 경우, 해당 섹션은 표시되지 않는다

```html
Shown.
{{#post}}
Never shown!
{{/person}}
```

#### Non-Empty Lists

key가 비어있지 않은 list인 경우 섹션 내의 내용의 반복한다

##### `GET /users` 요청

**list.html**

```html
<div class="container">
    <table>
        <tr>
            <td>id</td>
            <td>name</td>
            <td>phone</td>
            <td>website</td>
        </tr>
        {{#user}}
        <tr>
            <td>{{id}}</td>
            <td>{{name}}</td>
            <td>{{phone}}</td>
            <td>{{website}}</td>
        </tr>
        {{/user}}
    </table>
</div>
```

**결과**

```html
<div class="container">
  <table>
    <tr>
      <td>id</td>
      <td>name</td>
      <td>phone</td>
      <td>website</td>
    </tr>
    <tr>
      <td>1</td>
      <td>Leanne Graham</td>
      <td>1-770-736-8031 x56442</td>
      <td>hildegard.org</td>
    </tr>
    <tr>
      <td>2</td>
      <td>Ervin Howell</td>
      <td>010-692-6593 x09125</td>
      <td>anastasia.net</td>
    </tr>
    <!-- ... -->
  </table>
</div>
```

#### Lambda

lambda식을 실행할 수 있다

```html
{{#user}}
  {{#doubled}}
    {{name}}
  {{/doubled}}
{{/user}}
```

```java
@ControllerAdvice
public class LayoutAdvice {
    @ModelAttribute("doubled")
    public Mustache.Lambda doubled() {
        return (frag, out) -> {
            String bodyString = frag.execute();
            out.append(bodyString)
               .append(bodyString);
        };
    }
}
```

**결과**

```html
<!-- 이름이 Leanne Graham인 경우 -->
Leanne GrahamLeanne Graham
```

lambda식은 아래 interface를 구현해서 만들 수 있다.


```java
public interface Lambda {
  void execute (Template.Fragment frag, Writer out) throws IOException;
}
```

- `frag.execute()` : 내부의 표현식을 실행한 후 String으로 반환한다.
- `frag.context()` : 상위 오브젝트(여기서는 User)를 가져온다.
- `frag.context(int n)` : n번째 상위 오브젝트를 가져온다.

> 이러한 기능을 제공하나, 행여나 view에서 serivce logic을 작성하지는 말자.

#### False값이 아닌 경우

list도 아니고, false도 아니라면 이를 block을 가진 context(즉 object)로 간주한다

```java
model.addAttribute("user", new User("abc"));
```

```html
{{#user}}
  Hello, {{name}}!
{{/user}}
```
**결과**

```html
Hello, abc!
```

#### Inverted Section

섹션에 논리 부정 연산자(`!`)가 붙은 격이다

```java
model.addAttribute("users", Collections.emptyList());
```

```html
{{#users}}
  {{name}}
{{/users}}
{{^users}}
  No users...
{{/users}
```

**결과**

```html
No users...
```

#### 주석

`{% raw %}{{! ignore me }}{% endraw %}`

#### Partials

`{% raw %}{{>users/list-item}}{% endraw %}`과 같이, 외부 문서를 불러올 수 있다.

`GET /users`

**src/main/resources/templates/users/list.html**

```html
<div class="container">
    <table>
        <tr>
            <td>id</td>
            <td>name</td>
            <td>phone</td>
            <td>website</td>
            <td>doubled name</td>
        </tr>
        {{#user}}
        {{>users/list-item}}
        {{/user}}
    </table>
</div>
```

**src/main/resources/templates/users/list-item.html**

```html
<tr>
    <td>{{id}}</td>
    <td>{{name}}</td>
    <td>{{phone}}</td>
    <td>{{website}}</td>
    <td>{{#doubled}}{{name}}{{/doubled}}</td>
</tr>
```

#### Delimter

콧수염(`{% raw %}{{ }}{% endraw %}`) 을 다른 기호로 바꿀 때 사용한다

**콧수염을 `<% %>`로**

```html
{{ name }}
{{=<% %>=}} <!-- 변환 -->
<% name %>
```

## Spring Boot Property

기본으로 설정되어 있는 Property와 기본값은 다음과 같다

```
# HttpServletRequest의 attribute를 Controller에서 생성한 동일한 이름의 model attribute로 덮어 쓸 수 있는지 여부
spring.mustache.allow-request-override=false 

# HttpSession의 attribute를 Controller에서 생성한 동일한 이름의 model attribute로 덮어 쓸 수 있는지 여부
spring.mustache.allow-session-override=false 

spring.mustache.cache=false 
spring.mustache.charset=UTF-8
spring.mustache.check-template-location=true
spring.mustache.content-type=text/html
spring.mustache.enabled=true

# 모든 request attribute를 template에 병합하기 전 포함시켜야하는 지 여부
spring.mustache.expose-request-attributes=false
# 모든 session attribute를 template에 병합하기 전 포함시켜야하는 지 여부
spring.mustache.expose-session-attributes=false
# RequestContext를 노출할지 여부("springMacroRequsetContext"라는 이름의 spring macro library에 의해 설정됨)
# <link rel="stylesheet" href="{{springMacroRequestContext.request.contextPath}}/assets/app.css">
spring.mustache.expose-spring-macro-helpers=true

spring.mustache.prefix=classpath:/templates/ 
# 전체 view에서 사용할 RequestContext의 이름(키)
spring.mustache.request-context-attribute=
# viewname의 접미사
spring.mustache.suffix=.mustache
# 처리할 view name의 white list
spring.mustache.view-names=
```

# 마무리

script 기반의 view template engine인 mustache에 대해서 알아보았다.

여타 다른 엔진에 비해서 제공하는 기능은 훨씬 적다.
하지만 간단하고 배우기 쉬우며, spring boot와 통합도 제공한다.

[2015년 기준](https://github.com/jreijn/spring-comparing-template-engines)으로 Freemarker나 Velocity에 견줄 수 있을 정도의 속도가 나왔다고 한다.
물론 이런 종류의 테스트는 최적화를 위해서는 더 다양한 케이스를 테스트해 봐야겠지만, 생각보다 훨씬 빠르다는 것에 다시 놀랐다.

logic-less의 view를 구현하고, client의 logic을 node나 전담 front-server에서 구현할 수 있다면, 꽤 괜찮은 솔루션이라 생각된다.

소스코드 : https://github.com/supawer0728/simple-mustache