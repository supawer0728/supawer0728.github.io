---
title: Spring Framework 5.0.4.RELEASE Reference Web Servlet (Chapter1)
date: 2018-03-11 02:44:10
tags: [spring, reference]
categories:
  - spring
  - reference
---

# 글에 앞서서

- 본문은 Spring Framework Version 5의 습득을 위한 글이다.
- 이 글을 상업적 목적으로 쓰지 않았다
{% blockquote 원본 https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/ %}
Authors
Rod Johnson , Juergen Hoeller , Keith Donald , Colin Sampaleanu , Rob Harrop , Thomas Risberg , Alef Arendsen , Darren Davison , Dmitriy Kopylenko , Mark Pollack , Thierry Templier , Erwin Vervaet , Portia Tung , Ben Hale , Adrian Colyer , John Lewis , Costin Leau , Mark Fisher , Sam Brannen , Ramnivas Laddad , Arjen Poutsma , Chris Beams , Tareq Abedrabbo , Andy Clement , Dave Syer , Oliver Gierke , Rossen Stoyanchev , Phillip Webb , Rob Winch , Brian Clozel , Stephane Nicoll , Sebastien Deleuze

Copyright © 2004-2016

Copies of this document may be made for your own use and for distribution to others, provided that you do not charge any fee for such copies and further provided that each copy contains this Copyright Notice, whether distributed in print or electronically.
{% endblockquote %}

# 1. Spring Web MVC

# 1.2. DispatcherServlet

## 1.2. DispatcherServlet

![a](https://nesoy.github.io/assets/posts/20170217/2.PNG)
<!-- more -->
```xml
<web-app>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

![b](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/images/mvc-context-hierarchy.png)

**serlvet-context.xml**
```xml
<context:component-scan base-package="org.example">
  <cnotext:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

**application-context.xml**
```xml
<context:component-scan base-package="org.example">
  <cnotext:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

# 1.3. Filters

## 1.3. Filters

`spring-web` 모듈은 몇가지 유용한 필터를 제공한다

- `HttpPutFormContentFilter` : browser는 form의 요청을 `GET`, `POST`만 사용할 수 있으나, 해당 필터를 사용해서 `PUT`, `PATCH`요청도 해석할 수 있음
- `ShallowETagHeaderFilter` : 응답값을 버퍼링하고 ETag값을 계산

# 1.4. Annotated Controllers

## 1.4.2. RequestMapping

**RequestMapping의 shortcut**

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

### URI patterns

**wildcards**

`?`, `*`, `**` 등의 와일드 카드 사용 가능

**regx**

`/spring-web-3.0.5.jar` 요청에 대한 매핑

```java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariables String name, @PathVariable String version, @PathVariable String ext) {
    // ...
}
```

## 1.4.2. RequestMapping

### Pattern comparison

요청이 여러 URL 패턴에 매칭되었을 때, `AntPathMatcher.getPatternComparator(String path)`에 의해 더 정확한 패턴에 매칭됨

### Consumable media types

**Request Header의 `Content-Type`에 매칭**

```java
@PostMapping(path = "/pets", consumes = "application/json")
public void addPet(@RequestBody Pet pet) {
    // ...
}
```

### Producible media types

**RequestHeader의 `Accept`에 매칭**

```java
@GetMapping(path = "/pets/{petId}", produces = "application/json;charset=UTF-8")
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
```

### Parameters, headers

```java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue")
public void findPet(@PathVariable String petId) {
    // ...
}

@GetMapping(path = "/pets", headers = "myHeader=myValue")
public void findPet(@PathVariable String petId) {
    // ...
}
```

## 1.4.3. Handler Methods

### Method Arguments

| Controller method arguments | Description |
| - | - |
| `javax.servlet.http.HttpSession` | 현재 세션 정보(Thread-safe하지 않음) |
| `java.security.Principal` | 인증된 유저 정보 |
| `java.util.Locale` | 요청의 Locale 정보, `LocaleResolver`로 인해 계산 |
| java6+: `java.util.TimeZone`<br>java8+: `java.time.ZoneId` | `LocaleContextResolver`에 의해 계산 |
| `@PathVariable` | |
| `@MatrixVariable` | |
| `@RequestParam` | | 
| `@RequestHeader` | |
| `@CookieValue` | |
| `@RequestBody` | | 
| `HttpEntity<B>` | |
| `@RequestPart` | |
| `RedirectAttributes` | | 
| `@ModelAttribute` | |
| `Errors`, `BindingResult` | |
| `@RequestAttribute` | |

## 1.4.3. Handler Methods

### Return Values

| Controller method return value | Description |
| - | - |
| `@ResponseBody` | `HttpMessageConverter`에 의해 변환되어 응답에 쓰임 |
| `HttpEntity<B>`, `ResponseEntity<B>` | header와 body를 가지며, body는 `HttpMessageConverter`에 의해 변환됨 |
| `HttpHeaders` | body가 없이 header만 응답 |
| `String` | viewName |
| `View` |  |
| `void` | `ServletResponse`, `OuputStream`을 파라미터로 받았거나, `@ResponseStatus`가 메서드에 등록된 경우에<br>모든 처리가 완료된 것으로 간주 |
| `DeferredResult<V>` | 위 반환값으로 다른 스레드(any thread)에서 비동기로 반환 |
| `Callable<V>` | 위 반환값으로 Spring MVC가 관리하는 스레드에서 비동기로 반환 |
| Reactive types<br>Reactor, RxJava, etc. | |
| 그 외 | String의 경우 viewName으로 간주, 그 외에는 `@ModelAttribute`로 간주 |

## 1.4.3. Handler Methods

### Jackson JSON

```java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```

## 1.4.5. Binder Methods

**custom editor**

```java
@Controller
public class FormController {
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }
}
```

**formatter**

```java
@Controller
public class FormController {
    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }
}
```

## 1.4.6. Exception Methods

```java
@Controller
public class SimpleController {

    // ...

    @ExceptionHandler(IOException.class)
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }

}
```

> `@ControllerAdvice`에서도 사용 가능

## 1.4.7. Controller Advice

`@ExceptionHandler`, `@InitBinder`, `@ModelAttribute` 등의 처리는 `@ControllerAdvice`에서 할 수 있음

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```
# 1.5. URI Links

## 1.5.1. UriComponents

```java
String uriTemplate = "http://example.com/hotels/{hotel}";

UriComponents uriComponents = UriComponentsBuilder.fromUriString(uriTemplate)  
        .queryParam("q", "{q}")  
        .build(); 

URI uri = uriComponents.expand("Westin", "123").encode().toUri();  // http://example.com/hotels/Westin?q=123
```
```java
URI uri = UriComponentsBuilder.fromUriString(uriTemplate)
        .queryParam("q", "{q}")
        .buildAndExpand("Westin", "123")
        .encode()
        .toUri();
```

# 1.6. Async Requests

## 1.6. Async Requests

Spring MVC는 Servlet 3.0을 확장하여 아래와 같은 기능을 제공한다

- 하나의 비동기 결과값을 담는 `DeferredResult`, `Callable`을 반환할 수 있음
- `SSE`, `raw data`를 stream할 수 있음
- reactive 동작

## 1.6.1. DeferredResult

```java
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
    DeferredResult<String> deferredResult = new DeferredResult<String>();
    // Save the deferredResult somewhere..
    return deferredResult;
}

// From some other thread...
deferredResult.setResult(data);
```
> 외부 이벤트, scheduled task 등의 다른 스레드에서 처리된 비동기 결과를 반환

## 1.6.2. Callable

```java
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {

    return new Callable<String>() {
        public String call() throws Exception {
            // ...
            return "someView";
        }
    };
}
```

> 미리 설정해둔 `TaskExecutor`에서 반환

## 1.6.3. Processing

**비동기 처리 개요**

- `ServletRequest`의 `startAsync()`를 호출하여 비동기 모드로 사용할 수 있다
이로 인해, Servlet과 Filter를 종료시키지만 response는 남아 따로 처리하고 완료시킬 수 있게 된다
- `request.startAsync()`를 호출하면 추가적인 제어를 할 수 있는 `AsyncContext`가 반환된다(`dispatch(String)`, `setTimeout(long)`)
- `ServletRequest`는 현재 상태(초기화, dispatch, forward) 등을 파악할 수 있는 `DispatcherType`에 대한 접근을 제공한다

## 1.6.3. Processing

**DefferedResult**

- Controller는 `DeferredResult`를 반환하고, 이를 in-memory queue에 저장한다
- Spring MVC가 `request.startAsync()`를 호출한다
- 그동안 `DispatcherServlet`과 `Filter`는 요청을 처리하는 스레드를 종료시킨다. response만 남겨둔다
- 어느 스레드에서 `DeferredResult`에 값을 넣으면, Spring MVC는 ServletContainer에 요청을 되돌린다(dispatch)
- `DispatcherServlet`이 다시 실행되며, 비동기 반환 값에 대한 처리가 시작된다

## 1.6.3. Processing

**Callable**

- Controller가 `Callable`을 반환한다
- Spring MVC는 `request.startAsync()`를 호출하고, `Callable`가 `TaskExecutor`에 의해 다른 스레드에서 실행되도록 한다
- 그동안 `DispatcherServlet`과 `Filter`는 요청을 처리하는 스레드를 종료시킨다. response만 남겨둔다
- `Callable`에 완료되어 값이 반환되면, Spring MVC는 Servlet Container에 값을 되돌린다(dispatch)
- `DispatcherServlet`이 다시 실행되며, 비동기 반환 값에 대한 처리가 시작된다

## 1.6.3. Processing

### Exception handling

**DeferredResult**

`DeferredResult`를 사용하면 `setResult`나 `setErrorResult`를 사용한다.
두 경우 모두 Spring MVC에서 Servlet Container로 완료 처리를 dispatch하며, 기존의 예외 처리 방식을 따르게 된다(`@ExceptionHandler` 등)

**Callable**

대부분 비슷하다. 다만 `Callable`에는 `setErrorResult` 등의 메서드가 없으므로, 스스로 exception을 throw한다.

### Interception

`AsyncHandlerInterceptor`, `CallableProcessingInterceptor`, `DeferredResultProcessingInterceptor` 등이 존재

## 1.6.3. Processing

### Compare to WebFlux

Servlet 3.0에서 비동기를 하는 방법은 `Filter-Serlvet chain`은 종료시킨채, reponse만 남겨두어 다른 스레드에서 응답을 채우는 방식
개별 요청에 대한 응답의 쓰기 작업은 여전히 `blocking I/O`으로 이루어지며,
WebFlux는 이를 `non-blocking I/O`으로 처리하는 것이 큰 차이점

또 하나는, WebFlux는 Controller의 메서드에 비동기 혹은 반응형 타입을 지원함

## 1.6.4. HTTP Streaming

`DeferredResult`나 `Callable`은 비동기 처리 후, 한 번만 반환.
`Streaming`은 여러번 반환

### Objects

```java
@GetMapping("/events")
public ResponseBodyEmitter handle() {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();

```

## 1.6.4. HTTP Streaming

### SSE(Server Sent Events)

`SseEmitter`는 `ResponseBodyEmitter`의 서브 클래스로서 `W3C SSE` 스펙을 준수하여 Controller에서 Stream을 생성하도록 함
https://www.w3schools.com/html/html5_serversentevents.asp

```java
@GetMapping(path="/events", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter handle() {
    SseEmitter emitter = new SseEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
```

하지만 SSE는 IE에서 돌아가지 않으므로(망할), WebSocket과 SockJS 등을 사용할 것을 권장

## 1.6.4. HTTP Streaming

### Raw data

```java
@GetMapping("/download")
public StreamingResponseBody handle() {
    return new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            // write...
        }
    };
}
```

## 1.6.5. Reactive Types

- 단일 값(`Mono`, `Single`)에 대해서는 `DeferredResult`와 유사한 동작
- 여러 값(`Flux`, `Observable`) 등, `application/stream+json`이나 `text/event-stream` 유형의 값은 `ResponseBodyEmitter`, `SseEmitter`와 유사한 동작

## 1.6.6. Configuration

### Servlet Container

`Filter`와 `ServletContainer`는 `asyncSupported`값이 있으며 이를 `true`로 해야한다

**Java**

`AbstractAnnotationConfigDispatcherServletInitializer`로 Servlet Container를 사용한다면 자동으로 설정되어 있다

**web.xml**

`<async-supported>true</async-supported>`를 `DispatcherServlet` 설정과 `Filter` 설정에 추가한다
`Filter Mapping`에 `<dispatcher>ASYNC</dispatcher>`를 추가한다

### Spring MVC

**Java**

`WebMvcConfigurer`의 `configureAsyncSupport`를 사용

**xml**

`<mvc:annotation-driven>` 아래에 `<async-support>` 선언

# 1.7. CORS

# 1.7.3. `@CrossOrigin`

```java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
      // ...
    }
}
```

**default**

- 모든 origin 허용
- 모든 헤더 허용
- 모든 HTTP method 허용
- `maxAge`는 30분
- `allowedCredentials`는 기본적으로 비활성화됨.

## 1.7.3. `@CrossOrigin`

```java
@CrossOrigin(origins = "http://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```

## 1.7.4. Global Config

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("http://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);
    }
}
```

## 1.7.5. CORS Filter

```java
CorsConfiguration config = new CorsConfiguration();

// Possibly...
// config.applyPermitDefaultValues()

config.setAllowCredentials(true);
config.addAllowedOrigin("http://domain1.com");
config.addAllowedHeader("");
config.addAllowedMethod("");

UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
source.registerCorsConfiguration("/**", config);

CorsFilter filter = new CorsFilter(source);
```

## 1.9. HTTP Caching

## 1.9.1. Cache-Control

```java
// Cache for an hour - "Cache-Control: max-age=3600"
CacheControl ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS);

// Prevent caching - "Cache-Control: no-store"
CacheControl ccNoStore = CacheControl.noStore();

// Cache for ten days in public and private caches,
// public caches should not transform the response
// "Cache-Control: max-age=864000, public, no-transform"
CacheControl ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS)
                                    .noTransform().cachePublic();
```

## 1.9.2. Static resources

`ResourceHttpRequestHandler`를 설정해서 `Last-Modified` 헤더와 `Cache-Control` 헤더를 적절히 변경할 수 있다

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public-resources/")
                .setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS).cachePublic());
    }

}
```

## 1.9.3. `@Controller caching`

`Controller`는 `Cache-Control`, `ETag`, `If-Modified-Since` 등의 다양한 헤더를 다룰 수 있다

```java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {

    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
                .ok()
                .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
                .eTag(version) // lastModified is also available
                .body(book);
}
```

위 코드에서 응답에 `ETag` 및 `Cache-Control` 헤더가 포함되며
클라이언트가 보낸 조건부 헤더가 컨트롤러에서 설정한 캐시 정보와 일치하면 `HTTP 304 Not Modified` 응답의 빈 body를 내려보낸다

## 1.9.3. `@Controller caching` 

```java
@RequestMapping
public String myHandleMethod(WebRequest webRequest, Model model) {

    long lastModified = // 1. application-specific calculation

    if (request.checkNotModified(lastModified)) {
        // 2. shortcut exit - no further processing necessary
        return null;
    }

    // 3. or otherwise further request processing, actually preparing content
    model.addAttribute(...);
    return "myViewName";
}
```

- `request.checkNotModified(lastModified)` : `true`를 반환하는 경우, response status와 header를 설정
- `return null` : `request.checkNotModified()`와 결합하여 Spring MVC가 요청을 처리하지 않게 만듦

## 1.9.4. ETag Filter

- `ShallowEtagHeaderFilter`가 응답의 내용을 캐싱하고 ETag 헤더로 내보낼 MD5 해시를 생성
- 클라이언트가 동일 자원에 대한 요청을 보내면 해당 해시를 `If-None-Match` 값으로 사용
- 동일한 요청일 경우 필터가 304 응답
- 네트워크 대역 트래픽을 낮추는 효과는 있지만, CPU 사용은 낮추지 못함
- 앞서 설명한 `Controller` 수준의 다른 전략들은 연산을 줄일 수 있음

# 1.10. View Technologies

# 1.10. View Technologies

- Thymeleaf
- FreeMarker
- Groovy Markup
- Script Views
    - Mustache
    - React
    - ext.
- JSP & JSTL
- Tiles
- RSS, Aom
- PDF, Excel
- Jackson
- XML
- XSLT

# 1.11. MVC Config

## 1.11.2. MVC Config API

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Implement configuration methods...
}
```
보통은 `WebMvcConfigurerAdapter`를 상속받아서 쓰지 않을까...

## 1.11.3. Type conversion

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }
}
```

## 1.11.4. Validation

**global validator**

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public Validator getValidator(); {
        // ...
    }
}
```

**local validator**

```java
@Controller
public class MyController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new FooValidator());
    }
}
```

## 1.11.5. Interceptors

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleInterceptor());
        registry.addInterceptor(new ThemeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
```

## 1.11.6. Content Types

`ContentNegotiationConfigurer`를 통해 확장자로 `MediaType`을 결정할 수 있음

예) `/a.json`의 응답을 `application/json`으로 변경

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.mediaType("json", MediaType.APPLICATION_JSON);
    }
}
```

## 1.11.7. Message Converters

아래와 같이 custom ObjectMapper를 주입할 수 있다
```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.xml().build()));
    }
}
```

classpath 내에 아래 모듈이 발견되면 자동으로 설정된다

- `jackson-datatype-jdk7` : `java.nio.file.Path` 등의 Java 7에 정의된 타입
- `jackson-datatype-joda` : Joda-Time 타입 지원
- `jackson-datatype-jsr310` : Java 8에 정의된 Date, Time 타입 지원
- `jackson-datatype-jdk8` : `Optional`등 Java 8에 정의된 타입

그 외

- `jackson-datatype-money`: `javax.money` 타입 지원(unofficial module)
- `jackson-datatupe-hibernate` : `lazy-loading`등의 하이버네이트 지원

## 1.11.8. View Controllers

특정 URL을 바로 view로 매핑

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
    }
}
```

## 1.11.10. Static Resources

`/resources`로 들어오는 요청을 `/public`혹은 classpath 내의 `/static`에서 찾도록 설정.

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCachePeriod(31556926);
    }
}
```