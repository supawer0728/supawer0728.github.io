---
title: Spring에서 요청에 따른 부가 응답 추가하기(3) - webflux 적용
date: 2018-03-11 19:45:34
tags: [spring,webflux,non-blocking]
categories:
  - practice
  - [spring,practice]
---

# 서론

앞서 개발한 소스는 `spring-webmvc`로 작성했었다. 이번에는 Reactive Programming을 본격적으로 사용하며, non-blocking 동작을 지원하는 `spring-webflux`를 사용해서 이전까지의 소스를 포팅해보려고 한다.

<!-- more -->

# spring-webflux

| ![image1](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/images/webflux-overview.png) |
| - |
| *출처 : https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/images/webflux-overview.png* |

`spring-webflux`를 두가지 개발 모델을 지원한다. 하나는 기존의 `spring-webmvc`에서 사용했던 것과 마찬가지로 `@Controller` 등의 어노테이션을 이용한 개발 모델이다. 또 하나는 Java8의 함수형 람다 방식으로 rounter와 handler를 사용하는 하는 방식이다. 새로운 모듈인 `spring-webflux`을 사용하는 만큼 기존의 방식보다는 `Functional Endpoints` 기반으로 작성을 해보려고 한다.

`spring-webflux`에 대해서 간단히 알아보자.

왜 spring에서는 이 모듈을 만들었을까? Spring Reference의 [Motivation](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-new-framework)챕터를 읽어보자

>그 이유 중 하나는 더 적은 하드웨어 자원으로 스케일링하고, 더 적은 thread로 동시성을 처리할 수 있는 non-blocking web stack이 필요했기 때문이다. Servlet 3.1은 non-blocking I/O에 대한 API를 제공했다. 하지만 non-blocking API를 사용하므로써, Filter, Servlet 등의 동기화나 getParameter, getPart 등의 blocking을 API을 사용하지 못하게 되었다. 이는 곧 non-blocking이 기반이 되는 새로운 공통 API가 필요한 이유가 되었다.
>
>또 다른 이유로 함수형 프로그래밍을 들 수 있다. 마치 Java5에서 애노테이션이 등장해서 많은 기회를 창출했던 것처럼 - 예컨대 REST controller나, unit test를 애노테이션을 선언해서 쓸 수 있게 된 것처럼 Java8에서 추가된 람다표현식은 Java에 함수형 API를 사용할 수 있는 기회를 가져왔다. 비동기 로직을 선언적으로 구성할 수 있도록 해주는 continuation style API(`CompletableFuture, ReactiveX`)와 non-blocking application을 사용함에 있어서 요긴하게 쓸 수 있다. 프로그래밍 모델의 관점에서 Java8은 Spring Webflux로 하여금 기존과 같은 `@Controller` 애노테이션을 통한 컨트롤러 등록과 함수형 웹 엔드포인트를 동시에 제공할 수 있게끔 해주었다.
>
> **Spring Reference** - [Web Reactive 1.1.1. Motivation](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-new-framework)

위 글에서 `spring-webflux` 모듈이 나온 이유를 정리하자면 다음과 같다.

- non-blocking I/O를 지원하는 공통 API가 필요하기 때문
- 함수형 프로그래밍 스타일로 소스에서 reactive programming의 pipeline을 선언적으로 작성할 수 있게 됨

Spring5부터 `spring-webflux` 모듈이 나와서, 앞으로는 Web Application을 개발할 때 기존의 `spring-webmvc`와 어느 것을 사용해야할 지 고민하게 될 것 같다. 이에 대해서 Spring Reference에서 이야기하는 내용을 잠깐 살펴보자.

> 여러분이 사용하는 Spring MVC 어플리케이션이 잘 동작한다면 굳이 바꿀 필요가 없다. 명령형 프로그래밍은 코드를 작성, 이해, 디버깅하는 데에 있어서 가장 쉬운 방법이다. 대부분의 라이브러리는 명령형이며, 그 중에서 우리가 필요한 것을 고르면 된다.
>
> framework를 고를 기준 중 하나는, 의존성을 확인하는 것이다. 만약 blocking 방식의 persistence API(JPA, JDBC)나 network API를 사용한다면 `spring-mvc`가 적합한 선택이 될 것이다. `Reactor`나 `RxJava`를 사용해서 분리된 thread에서 blocking api를 호출할 수는 있겠으나, non-blocking web stack을 최대한으로 활용하기는 어렵다.
> 
> Spring MVC application에 원격 서비스를 호출하는 일이 있다면, Reactive WebClient를 사용해볼 것을 권장한다. Spring MVC 컨트롤러 메서드에서 reative 타입(Reactor, RxJava, etc.)를 직접 반환할 수 있다. 호출 당 지연이 길수록, 호출 간 상호 의존이 클수록, 더 큰 이득을 볼 수 있다.
> 
> 팀의 규모가 클 경우, non-blocking, 함수형, 선언적 프로그래밍의 학습 곡선이 가파르다는 것을 염두에 두어야 한다. 현실적으로 Reactive WebClient를 적용해보는 것부터 시작하는 것을 권장한다. 그 후에 작은 프로젝트에 적용해보고, 이점을 확인해 나가기 바란다. 
> 
> 어떠한 이점이 있는지 찾을 수 없을 때에는, non-blocking I/O가 어떻게 동작하며, 어떠한 효과를 볼 수 있는지 배우길 바란다.
>
> **Spring Reference** - [Web Reactive 1.1.5. Applicability](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-framework-choice)


# 기본 api 작성

## 의존성

```gradle
dependencies {
  compile('org.springframework.boot:spring-boot-starter-webflux')
}
```

## BoardHandler 작성

`spring-mvc`의 Controller와 대응되는 Handler를 작성해보자

```java
@Component
public class BoardHandler {

    private final BoardRepository boardRepository;
    private final BoardDtoConverter boardDtoConverter;

    @Autowired
    public BoardHandler(@NonNull BoardRepository boardRepository,
                        @NonNull BoardDtoConverter boardDtoConverter) {
        this.boardRepository = boardRepository;
        this.boardDtoConverter = boardDtoConverter;
    }

    public Mono<ServerResponse> getBoard(ServerRequest request) {
        long boardId = Long.valueOf(request.pathVariable("id"));
        Mono<ServerResponse> notFound = ServerResponse.notFound().build();
        Mono<Board> boardMono = boardRepository.findOne(boardId);
        Mono<BoardDto> boardDtoMono = boardMono.map(boardDtoConverter::convert);
        return boardDtoMono.flatMap(boardDto -> ServerResponse.ok()
                                                             .contentType(MediaType.APPLICATION_JSON)
                                                             .body(fromObject(boardDto))
                                                             .switchIfEmpty(notFound));
    }
}
```

## BoardRepository

JPA, JDBC의 API는 blocking으로 동작한다. 우선은 임시로 아래와 같이 repository를 작성했다.

```java
@Repository
public class BoardRepository {

    private long generatedId = 0L;
    private Map<Long, Board> boardMap = new ConcurrentHashMap<>();

    public void saveAll(List<Board> boards) {
        boards.forEach(board -> {
            board.setId(++generatedId);
            boardMap.put(board.getId(), board);
        });
    }

    public Mono<Board> findOne(Long id) {
        return Mono.just(boardMap.get(id));
    }
}
```

## WebConfig

요청을 handler로 routing해주는 routerFunction을 등록한다.

```java
@EnableWebFlux
@Configuration
public class WebConfig extends DelegatingWebFluxConfiguration {

    @Autowired
    private BoardHandler boardHandler;

    @Bean
    public RouterFunction<?> boardRouter() {
        return route(GET("/boards/{id}").and(accept(APPLICATION_JSON)), boardHandler::getBoard);
    }
}
```

# Interceptor, AOP

기존에는 Interceptor에서 request의 `attachment`를 해석하고, AOP에서 응답에 추가를 해줬었는데 지금은 그럴 수가 없다.

**Interceptor**

- `spring-webflux`에는 `HandlerInterceptor`가 없다
- `RouterFunction`에 `HandlerFilterFunction`을 붙여서 처리하자

**AOP**

Handler에서는 `Mono<ServerResponse>`를 반환하고 있다. 문제는 이 `ServerResponse`라는 객체 내부의 body를 바꿀 수가 없다. 때문에 Handler 내에서 `boardDto`에 모델을 추가하든지, 아니면 `BoardService`와 같이 Service 계층의 클래스를 두고 거기에 AOP를 걸든지 해야할 것 같다.

우선 부가 정보 추가를 Handler 내부에서 처리해보자. 핵심 로직은 추후에 따로 옮겨도 될 것이다.

# AttachmentHandlerFilter

```java
@Component
public class AttachmentHandlerFilter {

    public static final String TARGET_ATTRIBUTE_NAME = "attachment";
    private static final String TARGET_QUERY_PARAM_NAME = "attachment";
    private static final String TARGET_DELIMITER = ",";

    public HandlerFilterFunction<ServerResponse, ServerResponse> filter() {
        return (request, next) -> {
            Set<AttachmentType> attachmentTypes = resolveAttachmentType(request);
            request.attributes().put(TARGET_ATTRIBUTE_NAME, new AttachmentTypeHolder(attachmentTypes));
            return next.handle(request);
        };
    }

    private Set<AttachmentType> resolveAttachmentType(ServerRequest request) {
        return request.queryParam(TARGET_QUERY_PARAM_NAME)
                      .map(attachments -> Stream.of(attachments.split(TARGET_DELIMITER))
                                                .map(AttachmentType::fromCaseIgnoredName)
                                                .filter(Objects::nonNull)
                                                .collect(Collectors.toSet()))
                      .orElse(Collections.emptySet());
    }
}
```

처음에는 `AttachmentHandlerFilter`가 `HandlerFilterFunction`을 구현하도록 했었으나, 형안전성을 잡는게 어려워 Closure처럼 반환하도록 하였다. 해당 필터를 거쳐 request에 `AttachmentTypeHolder`가 존재하도록 attribute를 추가했다.

# HandlerFilterFunction 등록

```java
@Bean
public RouterFunction<?> boardRouter() {
    return route(GET("/boards/{id}").and(accept(APPLICATION_JSON)), boardHandler::getBoard).filter(attachmentFilter.filter());
}
```

# BoardHandler 수정


```java
@Component
public class BoardHandler {
    private final BoardRepository boardRepository;
    private final BoardDtoConverter boardDtoConverter;
    private final AttachExecutor attachExecutor;
    private final Mono<ServerResponse> notFound = ServerResponse.notFound().build();

    @Autowired
    public BoardHandler(@NonNull BoardRepository boardRepository,
                        @NonNull BoardDtoConverter boardDtoConverter,
                        @NonNull AttachExecutor attachExecutor) {
        this.boardRepository = boardRepository;
        this.boardDtoConverter = boardDtoConverter;
        this.attachExecutor = attachExecutor;
    }

    public Mono<ServerResponse> getBoard(ServerRequest request) {
        long boardId = Long.valueOf(request.pathVariable("id"));
        // 좀더 NPE나 형안전성 관련해서 강하게 제약을 할 수 있으면 좋을텐데...
        AttachmentTypeHolder typeHolder = request.attribute(TARGET_ATTRIBUTE_NAME)
                                                 .map(AttachmentTypeHolder.class::cast)
                                                 .orElseGet(() -> new AttachmentTypeHolder(Collections.emptySet()));
        Mono<Attachable> attachableMono =
                boardRepository.findOne(boardId)
                               .map(boardDtoConverter::convert)
                               .flatMap(boardDto -> attachExecutor.attach(boardDto, typeHolder));

        return attachableMono.flatMap(boardDto -> ServerResponse.ok()
                                                                .contentType(MediaType.APPLICATION_JSON)
                                                                .body(fromObject(boardDto))
                                                                .switchIfEmpty(notFound));
    }
}
```

`AttachmentTypeHolder`를 꺼내어서 `AttachExecutor`로 보내, 부가정보를 추가하고 `Mono<Attachable>`을 반환한다.
`AttachExecutor`는 이전의 `AttachmentAspect`에서 하던 일을 수행한다.

# AttachService와 그 구현체들

바뀐게 없다. 그대로 사용할 수 있다.

# AttachExecutor

```java
@Slf4j
@Component
public class AttachExecutor {

    private final Map<AttachmentType, List<AttachService<? extends Attachable>>> typeToServiceMap;

    @Autowired
    public AttachExecutor(@NonNull List<AttachService<? extends Attachable>> attachService) {
        this.typeToServiceMap = attachService.stream()
                                             .collect(Collectors.groupingBy(AttachService::getSupportAttachmentType, Collectors.toList()));
    }

    public Mono<Attachable> attach(Attachable attachable, AttachmentTypeHolder holder) {

        if (holder.isEmpty()) {
            return Mono.just(attachable);
        }

        return executeAttach(attachable, holder.getTypes());
    }

    private Mono<Attachable> executeAttach(Attachable attachable, Set<AttachmentType> types) {

        List<Mono<AttachmentWrapperItem>> monoItems = createMonoList(attachable, types);
        
        // 1. block()을 호출하지 않는다
        return Mono.zip(monoItems, this::filterItems)
                   .map(attachable::attach)
                   .doOnError(e -> log.warn(e.getMessage(), e))
                   .onErrorReturn(attachable);
    }

    private List<Mono<AttachmentWrapperItem>> createMonoList(Attachable attachable, Set<AttachmentType> types) {
        return types.stream()
                    .flatMap(type -> typeToServiceMap.get(type).stream())
                    .filter(service -> service.getSupportType().isAssignableFrom(attachable.getClass()))
                    .map(service -> service.getAttachment(attachable))
                    .collect(Collectors.toList());
    }

    private List<AttachmentWrapperItem> filterItems(Object[] itemArray) {
        return Stream.of(itemArray)
                     .map(AttachmentWrapperItem.class::cast)
                     .filter(item -> item != AttachmentWrapperItem.ON_ERROR)
                     .collect(Collectors.toList());
    }
}
```

`1.` 주석을 단 곳의 내용이 핵심적인 차이점이다. 기존의 `AttachmentAspect`에서는 `block()`을 호출해서 `List<AttachmentWrapperItem>`을 `실행 후` 취합했다. 하지만 여기서는 실행의 흐름(`Data Flow`)을 pipeline으로 선언만하고 넘어간다.

# 마무리

이전까지 `spring-webmvc`로 작업한 내용을 `spring-webflux`로 포팅해보았다. blocking을 non-blocking으로 바꾸었다는 것이 가장 중요하다.

앞서 설명한 바와 같이 아직까지 JDBC, JPA 등을 non-blocking으로 호출하지 못하기에 이러한 호출이 있는 Application에는 적용하지 못한다. 그 외에, api-gateway와 같은 역할을 하거나, 원격 서비스 호출이 주를 이루는 Application을 작성할 때에는 `spring-webflux`를 유용하게 시도해볼 수 있을 것 같다.

소스코드 : https://github.com/supawer0728/simple-attachment/tree/apply-webflux