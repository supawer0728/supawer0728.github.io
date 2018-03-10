---
title: (Spring Cloud) Feign
date: 2018-03-11 01:00:53
tags: [spring-cloud, feign, spring, netflix]
categories:
  - spring
  - cloud
  - netflix
---

# Feign

* REST 기반 서비스 호출을 추상화해주는 Spring Cloud Netflix 라이브러리
* 선언적 방식
* 인터페이스를 통해 클라이언트 측 프로그램 작성
* Spring이 런타임에 구현체를 제공(RestTemplate을 사용)
<!-- more -->
# Dependencies

```gradle
dependencyManagement {
  imports {
    mavenBom 'spring-cloud-dependencies:Finchley.M7'
  }
}

dependencies {
    compile 'spring.framework.cloud:spring-cloud-starter-openfeign'
}
```

# Example

**@EnableFeignClients**

```java
@EnableFeignClients
@SpringBootApplication
public class ApiGatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

**interface 정의**

```java
@FeignClient(name = "post-api", url = "${feign.post-api.url}")
public interface PostClient {

    @GetMapping("/posts/{id}")
    Post get(@PathVariable("id") Long id);
}
```

**사용**

```java
@RestController
@RequestMapping("/posts")
public class PostController {

    private final PostClient postClient;

    @Autowired
    public PostController(PostClient postClient) {
        this.postClient = postClient;
    }

    @GetMapping("/{id}")
    public Post get(@PathVariable Long id) {
        return postClient.get(id);
    }
}
```

# Feign 설정

## `@FeignClient`

* name : 서비스ID 혹은 논리적인 이름, spring-cloud의 eureka, ribbon에 사용
* url : 실제 호출할 서비스의 URL, eureka, ribbon을 사용하지 않고서도 동작
* decode404 : 404응답이 올 때 `FeignExeption`을 발생시킬지, 아니면 응답을 decode할 지 여부
* configuration : feign configuration class 지정
* fallback : hystrix fallback class 지정
* fallbackFactory : hystrix fallbak factory 지정

> Hystrix란?
> `spring-cloud`의 서비스 중 하나. Circuit Breaker Pattern을 사용. 뒷단 API 서버가 장애 발생 등의 이유로 일정 시간(Time window) 내에 여러번 오류 응답을 주는 경우(timeout, bad gateway 등), 해당 API 서버로 요청을 보내지 않고 잠시 동안 대체(fallback) method를 실행. 일정 시간이 지나서 다시 뒷단 API 서버를 호출하는 등의, 일련의 작업을 제공해준다. [Circuit Breaker Pattern\(마틴 파울러\)](https://martinfowler.com/bliki/CircuitBreaker.html)

### application.yml 설정

`.properties`로 하면 장황해질 것 같아, `yml`로 설명
아래 설정은 `@FeignClient(configuration = FooConfiguration.class)`으로 선언할 수 있음

```yml
feign:
  client:
    config: 
      feignName: # @FeignClient에서 name 값, 전역으로 설정하려면 default
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract
```
- connectionTimeout, readTimeout : hystrix의 timeout 설저이 더 짧은 경우 hystirx 옵션을 따라감
- loggerLevel : NONE, BASIC, HEADER, FULL을 지정할 수 있음
    - NONE : default, 로그를 남기지 않음
    - BASIC : Request Method, URL과 응답 코드, 실행 시간을 남김
    - HEADERS : `BASIC`의 정보를 포함하여, 요청, 응답의 헤더를 남김
    - FULL : 요청, 응답의 header, body, metadata를 남김
    - `logging.level.com.example.demo.PostClient: debug` 등의 debug logger 설정이 되어 있어야함
- encoder, decode : body의 내용을 Object로 변경하는 class 지정, 각각 
- retryer : 요청이 실패했을 때 재시도에 대한 정책

### java설정

```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

> `@FeignClient(configuration = FooConfiguration.class)`와 `application.yml`이 같이 있을 시에는, `yml` 설정이 우선. 우선 순위를 변경하고 싶으면 `feign.client.default-to-properties: false`를 `yml`에 설정

### java설정(spring 연동 없는 순수 open feign 설정)

```java
public interface PostClient {
    @RequestLine("GET /posts/{id}")
    Post get(@Param("id") Long id);
}
...
Feign.builder()
             .contract(new Contract.Default())
             .retryer(new Retryer.Default())
             .options(new Request.Options(1000, 1000))
             .encoder(new Encoder.Default())
             .decoder(new Decoder.Default())
             .decode404()
             .logLevel(Logger.Level.BASIC)
             .target(new Target.HardCodedTarget<>(PostClient.class, "post-api", "https://jsonplaceholder.typicode.com"));
```

# Feign Hystrix Support

classpath 내에 hystrix가 있으며, `feign.hystrix.enabled: true` 설정이 되었다면 Hystrix를 사용 가능

```gradle
compile('org.springframework.cloud:spring-cloud-starter-netflix-hystrix')
```

> Hystrix를 사용하는 경우 기본적으로 thread time out이 1초
> 때문에 기본 설정으로는 feign의 connection, read timeout이 1초 이상인 경우라도
> 1초 안에 응답이 오지 않으면 fallback이 실행되므로 주의

```yml
feign:
  hystrix:
    enabled: true
  client:
    config:
      default:
        loggerLevel: BASIC
        # feign의 전역 timeout 설정 : 5초
        connectTimeout: 5000
        readTimeout: 5000
  post-api.url: https://jsonplaceholder.typicode.com
  http-bin-api.url: https://httpbin.org

# hystrix 명령의 기본 timeout을 10초로 변경
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 10000
```

## Fallback

```java
// Feign 선언
@FeignClient(name = "http-bin-api", url = "${feign.http-bin-api.url}", fallback = HttpBinClientFallback.class)
public interface HttpBinClient {

    @GetMapping("/delay/{seconds}")
    DelayResponse delay(@PathVariable("seconds") int seconds);
}

// Fallback 설정
@Slf4j
@Component
public class HttpBinClientFallback implements HttpBinClient {
    @Override
    public DelayResponse delay(int seconds) {
        log.debug("fallback called");
        return DelayResponse.EMPTY;
    }
}

// 응답 모델
@Value
public class DelayResponse {
    public static DelayResponse EMPTY = new DelayResponse(null, null);

    private String origin;
    private String url;

    @JsonCreator
    public DelayResponse(@JsonProperty("origin") String origin, @JsonProperty("url") String url) {
        this.origin = origin;
        this.url = url;
    }
}
```

**`http://localhost:8080/delay/1` 성공**
```json
{
"origin": "106.249.33.22",
"url": "https://httpbin.org/delay/1"
}
```
```
DEBUG 60528 --- [-http-bin-api-1] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/1 HTTP/1.1
DEBUG 60528 --- [-http-bin-api-1] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- HTTP/1.1 200 OK (2343ms)
```

**`http://localhost:8080/delay/5` timeout!!!**
```json
{"origin":null, "url":null}
```
```
DEBUG 60528 --- [-http-bin-api-5] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/5 HTTP/1.1
DEBUG 60528 --- [-http-bin-api-5] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- ERROR SocketTimeoutException: Read timed out (5883ms)
```

### 회로 열림 상태(Circuit Open) - Hystrix에 대해서

위 예제에 대해서 Hystrix의 동작을 아래와 같이 튜닝

```yml
hystrix:
  command:
    "HttpBinClient#delay(int)":
      execution.isolation.thread.timeoutInMilliseconds: 3000
      metrics.rollingStats.timeInMilliseconds: 60000
      circuitBreaker.requestVolumeThreshold: 5
      circuitBreaker.errorThresholdPercentage: 50
```

- `execution.isolation.thread.timeoutInMilliseconds` : hystirx명령에 대해 3초 timeout 설정
- `metrics.rollingStats.timeInMilliseconds` : 60초의 window slice를 가짐(30초씩 감시)
- `circuitBreaker.requestVolumeThreshold` : 최소 5번 이상의 요청이 있어야함
- `circuitBreaker.errorThresholdPercentage` : 50% 이상 오류가 발생시 회로를 오픈(circuit open)
- 즉 최근 1분 내에, 5번 이상의 요청이 있었고, 그 중 50% 이상이 오류가 발생했다면, command를 실행하지 않고 fallback을 실행하겠다는 의미임
- 회로가 열린 후, 일정시간(`circuitBreaker.sleepWindowInMilliseconds`)동안 fallback 응답을 보내고 성공하면 회로를 닫음

**timeout이 5번 이상 발생시 로그**

```
2018-02-28 15:46:37.653 DEBUG 99893 --- [-http-bin-api-1] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/5 HTTP/1.1
2018-02-28 15:46:40.655 DEBUG 99893 --- [ HystrixTimer-1] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:46:42.899 DEBUG 99893 --- [-http-bin-api-2] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/5 HTTP/1.1
2018-02-28 15:46:43.844 DEBUG 99893 --- [-http-bin-api-1] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- ERROR SocketTimeoutException: Read timed out (6190ms)
2018-02-28 15:46:45.902 DEBUG 99893 --- [ HystrixTimer-1] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:46:46.704 DEBUG 99893 --- [-http-bin-api-3] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/5 HTTP/1.1
2018-02-28 15:46:48.849 DEBUG 99893 --- [-http-bin-api-2] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- ERROR SocketTimeoutException: Read timed out (5950ms)
2018-02-28 15:46:49.708 DEBUG 99893 --- [ HystrixTimer-2] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:46:50.438 DEBUG 99893 --- [-http-bin-api-4] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/5 HTTP/1.1
2018-02-28 15:46:52.477 DEBUG 99893 --- [-http-bin-api-3] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- ERROR SocketTimeoutException: Read timed out (5772ms)
2018-02-28 15:46:53.443 DEBUG 99893 --- [ HystrixTimer-1] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:46:54.765 DEBUG 99893 --- [-http-bin-api-5] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] ---> GET https://httpbin.org/delay/5 HTTP/1.1
2018-02-28 15:46:56.292 DEBUG 99893 --- [-http-bin-api-4] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- ERROR SocketTimeoutException: Read timed out (5853ms)
2018-02-28 15:46:57.769 DEBUG 99893 --- [ HystrixTimer-3] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:47:00.115 DEBUG 99893 --- [nio-8080-exec-6] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:47:00.874 DEBUG 99893 --- [-http-bin-api-5] c.p.s.httpbin.client.HttpBinClient       : [HttpBinClient#delay] <--- ERROR SocketTimeoutException: Read timed out (6108ms)
2018-02-28 15:47:01.000 DEBUG 99893 --- [nio-8080-exec-7] c.p.s.h.client.HttpBinClientFallback     : fallback called
2018-02-28 15:47:01.827 DEBUG 99893 --- [nio-8080-exec-8] c.p.s.h.client.HttpBinClientFallback     : fallback called
```

> `https://httpbin.org/delay/5`에서 5번 실패가 발생한 이후에는 바로 fallback을 호출

## Fallback Factory

앞서 살펴보았던 Fallback은 어떤 Exception이 발생했는지 알 수가 없음

```java
@FeignClient(name = "http-bin-api", url = "${feign.http-bin-api.url}", fallbackFactory = HttpBinClientFallbackFactory.class)
public interface HttpBinClient {

    @GetMapping("/delay/{seconds}")
    DelayResponse delay(@PathVariable("seconds") int seconds);
}

@Slf4j
@Component
public class HttpBinClientFallbackFactory implements FallbackFactory<HttpBinClient> {

    @Override
    public HttpBinClient create(Throwable cause) {
        log.error(cause.getMessage(), cause);
        return seconds -> DelayResponse.EMPTY;
    }
}
```

**오류 발생시 로그**
```
2018-02-28 16:03:26.213 ERROR 14890 --- [ HystrixTimer-1] c.p.s.h.c.HttpBinClientFallbackFactory   : null

com.netflix.hystrix.exception.HystrixTimeoutException: null
	at com.netflix.hystrix.AbstractCommand$HystrixObservableTimeoutOperator$1$1.run(AbstractCommand.java:1154) [hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.strategy.concurrency.HystrixContextRunnable$1.call(HystrixContextRunnable.java:45) [hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.strategy.concurrency.HystrixContextRunnable$1.call(HystrixContextRunnable.java:41) [hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.strategy.concurrency.HystrixContextRunnable.run(HystrixContextRunnable.java:61) [hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.AbstractCommand$HystrixObservableTimeoutOperator$1.tick(AbstractCommand.java:1159) [hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.util.HystrixTimer$1.run(HystrixTimer.java:99) [hystrix-core-1.5.12.jar:1.5.12]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [na:1.8.0_152]
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308) [na:1.8.0_152]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180) [na:1.8.0_152]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294) [na:1.8.0_152]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_152]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_152]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_152]
```

> fallbackFactory를 사용할 때에, Spring Application이 올라오면서 검증차원에서 일부러 fallbackFactory를 한 번 실행시킴.
> `Object exampleFallback = fallbackFactory.create(new RuntimeException());`
> 때문에 불필요한 log가 올라오는데, 이를 배제할 방법을 모름...

# Feign 상속 지원

Feign이 Spring MVC의 `@Controller`의 애너테이션들을 사용하는 것을 응용해서 다음과 같은 소스를 작성할 수 있음

```java
public interface UserService {

    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}

@RestController
public class UserController implements UserService {

}

@FeignClient("users")
public interface UserClient extends UserService {

}
```

이처럼 다른 곳에 API client를 납품할 일이 있을때, Feign을 사용한다면
`UserService`를 `serivce` 모듈에 넣고, 그를 구현한 `UserController`를 `api` 모듈에 넣고,
그에 대한 feign client인 `UserClient`를 `client` 모듈에 넣어
`client`모듈을 외부에 공유하는 식의 개발도 가능

**하지만 권장하지 않음. server-client 간의 커플링을 높이며, 현재 실질적으로 Spring MVC의 기능을 Feign이 모두 소화하고 있지 않음**

# 요청, 응답 압축

GZIP을 통해 요청, 응답을 압축

```yml
feign:
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json
      request.min-request-size: 2048
    response.enabled: true
```

# Pageable

`spring-data`를 쓰는 환경이라면 `Pageable`을 빼놓을 수 없음
아쉽게도 요청이나, 응답을 받을 때 `Pageable`이나 `Page<T>`를 지원하지는 않음
직접 Encoder와 Jackson 설정을 해야함

## Pageable 요청 보내기

아래와 같이 Encoder를 구현하여, Pageable로 요청을 만들 수 있음
참고 : https://github.com/spring-cloud/spring-cloud-netflix/issues/556

```java
@Configuration
@EnableFeignClients
public class FeignClientConfig {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public Encoder feignEncoder() {
        return new PageableQueryEncoder(new SpringEncoder(messageConverters));
    }
}

public class PageableQueryEncoder implements Encoder {

    private final Encoder delegate;

    public PageableQueryEncoder(Encoder delegate) {
        this.delegate = delegate;
    }

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException {
        if (object instanceof Pageable) {
            Pageable pageable = (Pageable) object;
            template.query("page", pageable.getPageNumber() + "");
            template.query("size", pageable.getPageSize() + "");

            if (pageable.getSort() != null) {
                Collection<String> existingSorts = template.queries().get("sort");
                List<String> sortQueries = existingSorts != null ? new ArrayList<>(existingSorts) : new ArrayList<>();
                for (Sort.Order order : pageable.getSort()) {
                    sortQueries.add(order.getProperty() + "," + order.getDirection());
                }
                template.query("sort", sortQueries);
            }
        } else {
            delegate.encode(object, bodyType, template);
        }
    }
}

```

`Encoder`로 할 수 있는 일이 한정적이기 때문에, 실제 운영에서 사용하고자 않다면 아래와 같은 구현체를 만들어야함

```java
@Configuration
@EnableFeignClients
public class FeignClientConfig {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public Encoder feignEncoder() {
        return new CustomEncoder(new SpringEncoder(messageConverters), new PageableQueryEncoder());
    }
}

public class CustomEncoder implements Encoder {
  private final Encoder defaultEncoder;
  private final Map<Class, TypeSupportEncoder> encoders;
  
  public CustomEncoder(Encoder defaultEncoder, TypeSupportEncoder... encoders) {
    //...
  }
}

public interface TypeSupportEncoder extends Encoder {
  Class getSupportType();
}
```

## Page 응답 Json 파싱하기

Jackson 설정
참고 : https://github.com/spring-cloud/spring-cloud-netflix/issues/556

```java
@Configuration
public class ApiConfig {

    @Bean
    public Module customModule() {
        SimpleModule module = new SimpleModule("simple-feign", new Version(0, 0, 1, "SNAPSHOT", "com.parfait", "simple-feign"));
        module.setMixInAnnotation(Page.class, SimplePageImpl.class);
        return module;
    }

    @JsonDeserialize(as = SimplePageImpl.class)
    private interface PageMixIn {
    }
}

public class SimplePageImpl<T> implements Page<T> {

    private final Page<T> delegate;

    public SimplePageImpl(
            @JsonProperty("content") List<T> content,
            @JsonProperty("page") int number,
            @JsonProperty("size") int size,
            @JsonProperty("totalElements") long totalElements) {
        delegate = new PageImpl<>(content, new PageRequest(number, size), totalElements);
    }


    @JsonProperty
    @Override
    public int getTotalPages() {
        return delegate.getTotalPages();
    }

    @JsonProperty
    @Override
    public long getTotalElements() {
        return delegate.getTotalElements();
    }

    @JsonProperty("page")
    @Override
    public int getNumber() {
        return delegate.getNumber();
    }

    @JsonProperty
    @Override
    public int getSize() {
        return delegate.getSize();
    }

    @JsonProperty
    @Override
    public int getNumberOfElements() {
        return delegate.getNumberOfElements();
    }

    @JsonProperty
    @Override
    public List<T> getContent() {
        return delegate.getContent();
    }

    @JsonProperty
    @Override
    public boolean hasContent() {
        return delegate.hasContent();
    }

    @JsonIgnore
    @Override
    public Sort getSort() {
        return delegate.getSort();
    }

    @JsonProperty
    @Override
    public boolean isFirst() {
        return delegate.isFirst();
    }

    @JsonProperty
    @Override
    public boolean isLast() {
        return delegate.isLast();
    }

    @JsonIgnore
    @Override
    public boolean hasNext() {
        return delegate.hasNext();
    }

    @JsonIgnore
    @Override
    public boolean hasPrevious() {
        return delegate.hasPrevious();
    }

    @JsonIgnore
    @Override
    public Pageable nextPageable() {
        return delegate.nextPageable();
    }

    @JsonIgnore
    @Override
    public Pageable previousPageable() {
        return delegate.previousPageable();
    }

    @JsonIgnore
    @Override
    public <S> Page<S> map(Function<? super T, ? extends S> converter) {
        return delegate.map(converter);
    }

    @JsonIgnore
    @Override
    public Iterator<T> iterator() {
        return delegate.iterator();
    }
}
```