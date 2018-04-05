---
title: Spring5에서 HTTP Streaming
date: 2018-03-15 15:30:19
tags: [spring, streaming]
categories:
  - [spring, practice]
  - practice
---

# 서론

Spring의 Stream이라고 하면, 가장 먼저 떠오르는 것은 websocket일 것 같다.
websocket은 full-duplex communication(전 이중 통신)을 표방하고 있으며, 별도의 프로토콜을 공부해야하는 등, web상의 양방향 통신을 복잡하다고 느낄 수 있을 것 같다.

때문에 다음의 조건들을 충족시킬 Http Streaming 기법에 대해, `spring-webmvc`와 `spring-webflux` 별로 하나씩 공유하려한다.

- Spring5에서 지원
- 구현 난이도가 낮음
- 기존의 Http 통신을 기반

<!-- more -->

# spring-webmvc : SSE

`spring-webmvc`에서 HTTP Streaming을 하는 가장 간단한 방법은 [SSE(Server-Sent Event)](https://www.w3.org/TR/eventsource/)를 사용하는 것이다.([예제](https://www.w3schools.com/html/tryit.asp?filename=tryhtml5_sse))

## 장점

- spring에서 서버측 구현이 간편하다
- 표준 기술이므로 javascript에서 구현이 간편하다
- Android, iOS에서도 관련 라이브러리가 있다
- 자동으로 재접속을 시도한다

## 단점

- Native Javascript로는 Header를 추가하는 등의 세밀한 조정을 할 수 없다
  - Polyfill로 대체 가능
- 표준 기술이지만 IE(edge 포함)에서 지원하지 않는다
  - Polyfill로 대체 가능
- client 측에서 연결을 끊은 경우를 감지하지 못한다

## 구현

**spring reference에 나와있는 간단한 소스**

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

`SseEmitter`를 반환하고, 별도의 thread에서 `emmiter.send()`를 호출하면 된다.

### 구현할 예제

`GET /users`로 요청시, 1초 간격으로
`https://jsonplaceholder.typicode.com/users/{id}`의 1,2,3을 차례로 호출해서 응답

### 의존성

```gradle
dependencies {
    compile('org.springframework.boot:spring-boot-starter-web')
}
```

### Controller

```java
@Slf4j
@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserEmitService service;

    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public ResponseBodyEmitter users() {
        SseEmitter emitter = new SseEmitter();
        service.add(emitter);
        return emitter;
    }
}
```

### Service

```java
@Slf4j
@Component
public class UserEmitService {

    private static final int REPEAT = 3;
    private final Map<ResponseBodyEmitter, AtomicInteger> emitterCountMap = new HashMap<>();

    public void add(ResponseBodyEmitter emitter) {
        emitterCountMap.put(emitter, new AtomicInteger(0));
    }

    @Scheduled(fixedRate = 1000L)
    public void emit() {

        List<ResponseBodyEmitter> toBeRemoved = new ArrayList<>(emitterCountMap.size());

        for (Map.Entry<ResponseBodyEmitter, AtomicInteger> entry : emitterCountMap.entrySet()) {

            Integer count = entry.getValue().incrementAndGet();
            User user = new RestTemplate().getForObject("https://jsonplaceholder.typicode.com/users/{id}", User.class, count);

            ResponseBodyEmitter emitter = entry.getKey();
            try {
                emitter.send(user);
            } catch (IOException e) {
                log.error(e.getMessage(), e);
                toBeRemoved.add(emitter);
            }

            if (count >= REPEAT) {
                toBeRemoved.add(emitter);
            }
        }

        for (ResponseBodyEmitter emitter : toBeRemoved) {
            emitterCountMap.remove(emitter);
        }
    }
}
```

### 실행 결과

브라우저에서 `GET /users`를 호출 해보면, 아래 결과가 한 줄마다 1초 간격으로 표시된다.
`data`라고 하는 key에 json value가 붙어있는 형태다.

```json
data:{"id":1,"name":"Leanne Graham","email":"Sincere@april.biz","phone":"1-770-736-8031 x56442","website":"hildegard.org"}

data:{"id":2,"name":"Ervin Howell","email":"Shanna@melissa.tv","phone":"010-692-6593 x09125","website":"anastasia.net"}

data:{"id":3,"name":"Clementine Bauch","email":"Nathan@yesenia.net","phone":"1-463-123-4447","website":"ramiro.info"}
```

## 동작 방식

Spring에서 지원하는 Async 응답 방식인 `DeferredResult`, `Callable`과 같다

**Spring Async 응답 방식 개요**

- 반환형이 Async 응답을 필요로하는 경우, Spring MVC는 `request.startAsync()`를 호출하여 `AsyncContext`를 획득하여 저장해둔다.
  - `AsyncContext`는 비동기 처리를 제어할 수 있도록 해준다.
  - ex) Sevrlet container thread에서 요청 처리를 재개시키며, 기존의 `forward`와 같은 동작을 하는 `dispatch` 메서드를 제공한다.
  - 이로 인해 응답을 별도의 thread에서 비동기로 처리할 수 있다.
- 요청 thread를 종료시키되, response는 열어둔다
- 다른 thread에서 `AsyncContext`를 사용해서 response를 완성시킨다.
  - Spring MVC는 Servlet container로 다시 요청을 보내고(dispatch), client로 응답한다.

## 주의 사항

boot를 사용하는 경우, `spring.mvc.async.request-timeout: 1000`로 timeout 설정을 할 수 있다.
`new SseEmitter(int timeout)`으로 개별로 사용할 수 있으며, 우선 설정된다.

# spring-webflux : `Flux<T>`

`reactor`에서 `Flux<T>`는 `T`의 stream을 의미한다.
`Controller`에서 `Flux<T>`를 반환하는 경우, `application/stream+json`으로 객체를 json 형식으로 받을 수 있다.

> `Flux<ServerSentEvent>` 방식으로 SSE를 사용할 수도 있다.

## 장점

- reactive의 장점을 살릴 수 있다.
- reactor를 알고 있으면 구현하기 쉽다.

## 단점

- client 측에서 지원해주는 라이브러리는 적은 듯하다.

## 구현 예제

요구 사항은 위 예제와 동일하다.
1초마다 User를 불러온다.

### 의존성

```gradle
dependencies {
    compile('org.springframework.boot:spring-boot-starter-webflux')
}
```

### Controller

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private final UserClient userClient;

    @Autowired
    public UserController(@NonNull UserClient userClient) {
        this.userClient = userClient;
    }

    // 1초마다 User 발생
    @GetMapping(produces = "application/stream+json")
    public Flux<User> users() {

        return Flux.interval(Duration.ofSeconds(1L))
                   .take(3)
                   .flatMap(number -> userClient.get(number + 1L));
    }
}
```

### UserClient

```java
@Slf4j
@Component
public class UserClient {

    public Flux<User> get(long id) {

        return WebClient.create("https://jsonplaceholder.typicode.com")
                        .get()
                        .uri("/users/{id}", id)
                        .retrieve()
                        .bodyToFlux(User.class);
    }
}
```

### 실행 결과

브라우저에서 실행해보면 1초마다 한 줄씩 표시된다.

```
{"id":1,"name":"Leanne Graham","email":"Sincere@april.biz","phone":"1-770-736-8031 x56442","website":"hildegard.org"}
{"id":2,"name":"Ervin Howell","email":"Shanna@melissa.tv","phone":"010-692-6593 x09125","website":"anastasia.net"}
{"id":3,"name":"Clementine Bauch","email":"Nathan@yesenia.net","phone":"1-463-123-4447","website":"ramiro.info"}
```

# 마무리

Spring5에서 지원해주는 Http Streaming에 대해서 `webmvc`, `webflux` 라이브러리 별로 하나씩 알아보았다.

아무래도 `websocket`과 비교를 할 수 밖에 없는데,

`websocket`과 비교했을 때의 장점은 `구현하기 쉽다`는 것이다.
이것도 좀 애매한데 `구현하기 쉽다`는 상대적인 것이어서 이미 `websocket`을 경험해본 사람들은,
오히려 더 많은 기능을 지원해주는 `websocket`을 선호할 것 같다.
`websocket`의 경우에는 `Sse`보다 오히려 client에서 많은 지원을 해주며(`SockJs` 등), reference도 다양하다.
`Flux<T>`는 spring5로 넘어오면서 `websocket`에서도 반환할 수 있다.

다른 장점은 전통적인 Http 방식이므로 infra를 건들 일이 없다는 것이다.
`websocket`은 몇몇 구식 load balancer나 proxy에서는 지원을 하지 않거나, 미미한 경우가 있다.

어떠한 서비스를 구현해야 하느냐에 따라서 다르겠지만, 대부분의 경우에는 `websocket`을 상위호환으로 볼 수 있을 것 같다.

소스코드 : https://github.com/supawer0728/simple-streaming