---
title: Spring에서 요청에 따른 부가 응답 추가하기(2) - reactor 적용
date: 2018-03-11 14:59:03
tags:
---
# 개요

앞 번에 개발한 소스에는 2가지 문제점이 있었다

1. Network I/O를 순차실행
    - O(n) 시간이 걸림 : timeout * attachment 갯수
    - Async로 O(1)만에 끝내도록 튜닝 필요
2. Failover
    - attachment는 단순 부가 정보임에도 불구하고 attachmentService에서 exception이 발생하면, 아무 정보도 내려줄 수 없음
    - attach는 실패해도 Board 정보와 나머지 성공한 attachment는 보여줘야함

이번에는 위 이슈들을 reactor를 사용해서 해결해보려 한다
<!-- more -->
# Reactor

우선 왜 reactor를 사용하는 가에 대해서 간략하게 정리해보자

- Rx(Reactive Extension)를 구현하여 쉽게 비동기 프로그래밍 가능
- 또 다른 Rx 구현체인 RxJava와 비교했을 때, 다음의 장점이 있음
    - Spring5에 통합하기 쉬움
    - Java8에 대한 지원
        - rxJava는 1.6버전부터 쓸 수 있으며 자체적으로 `Function`을 구현해서 사용
        - Reactor는 Java8부터 쓸 수 있으며 Java8 Api와 Optional 등을 지원

여기서 Reactor API에 대한 기초적인 사항들은 다루지 않으려한다.
참고로 reactor는 Java버전에 영향을 받기 때문에 아래 소스를 Spring 4에 적용해도 문제없이 동작해야 한다.

# AttachmentWrapperItem

본격적으로 이슈를 해결하기 전에 먼저 한가지 리팩토링을 해야한다
`AttachmentWrapper`에서 `put()`에 `AttachmentType`과 `Attachment`를 따로 받고 있다.

```java
@ToString
@EqualsAndHashCode
public class AttachmentWrapper {

    interface AttachmentMap {
        void put(AttachmentType type, Attachment attachment);

        void putAll(Map<? extends AttachmentType, ? extends Attachment> attachmentMap);

        boolean isEmpty();

        Set<Map.Entry<AttachmentType, Attachment>> entrySet();
    }

    @Delegate(types = AttachmentMap.class)
    private Map<AttachmentType, Attachment> value = new EnumMap<>(AttachmentType.class);
}
```

reactor를 사용하게 되면 `Mono<T>`, `Flux<T>`와 같이 Generic에 적합한 타입으로 변경을 해야하기 때문에
`AttachmentType`과 `Attachment`를 하나로 묶는 `AttachmentWrapperItem` 클래스를 작성하고
이를 `AttachmentWrapper`에 반영해야한다.

**AttachmentWrapperItem**

```java
@Value
public class AttachmentWrapperItem {
    // 예외 발생시 반환할 인스턴스
    public static final AttachmentWrapperItem ON_ERROR = new AttachmentWrapperItem(null, null);
    private AttachmentType type;
    private Attachment attachment;
}
```

**AttachmentWrapper 적용**

```java
@ToString
@EqualsAndHashCode
public class AttachmentWrapper {

    interface AttachmentMap {
        boolean isEmpty();

        Set<Map.Entry<AttachmentType, Attachment>> entrySet();
    }

    @Delegate(types = AttachmentMap.class)
    private Map<AttachmentType, Attachment> value = new EnumMap<>(AttachmentType.class);

    public void put(AttachmentWrapperItem item) {
        this.value.put(item.getType(), item.getAttachment());
    }

    public void putAll(Collection<AttachmentWrapperItem> items) {
        this.value.putAll(items.stream().collect(Collectors.toMap(AttachmentWrapperItem::getType, AttachmentWrapperItem::getAttachment)));
    }
}
```

**Attachable interface 변경**

기존의 두 개의 파라미터를 받던 것을 하나의 파라미터를 받도록 변경한다.
```java
// 변경 전
default void attach(AttachmentType type, Attachment attachment) {
    getAttachmentWrapper().put(type, attachment);
}
default void attach(Map<? extends AttachmentType, ? extends Attachment> attachment) {
    getAttachmentWrapper().putAll(attachment);
}

// 변경 후
default void attach(AttachmentWrapperItem item) {
    getAttachmentWrapper().put(item);
}
default void attach(Collection<AttachmentWrapperItem> items) {
    getAttachmentWrapper().putAll(items);
}
```

**AttachService 변경**

`getAttachment`의 반환값을 `AttachmentWrapperItem`으로 바꾸자.

```java
AttachmentWrapperItem getAttachment(Attachable attachable);
```

**AttachWriterToBoardService 변경**

`AttachService`의 변경된 로직을 반영한다.

```java
// 변경 전
@Override
public Attachment getAttachment(Attachable attachment) {
    BoardDto boardDto = supportType.cast(attachment);
    return writerClient.getWriter(boardDto.getWriterId());
}
// 변경 후
@Override
    public AttachmentWrapperItem getAttachment(Attachable attachable) {
    BoardDto boardDto = supportType.cast(attachable);
    Attachment attachment = writerClient.getWriter(boardDto.getWriterId());
    return new AttachmentWrapperItem(supportAttachmentType, attachment);
}
```

**AttachmentAspect 변경**

```java
// 변경 전
private void executeAttach(Attachable attachable) {

    Set<AttachmentType> types = attachmentTypeHolder.getTypes();

    Map<AttachmentType, Attachment> attachmentMap =
            types.stream()
                 .flatMap(type -> typeToServiceMap.get(type).stream())
                 .filter(service -> service.getSupportType().isAssignableFrom(attachable.getClass()))
                 .collect(Collectors.toMap(AttachService::getSupportAttachmentType, service -> service.getAttachment(attachable)));

    attachable.attach(attachmentMap);
}

// 변경 후
private void executeAttach(Attachable attachable) {

    Set<AttachmentType> types = attachmentTypeHolder.getTypes();

    List<AttachmentWrapperItem> items =
            types.stream()
                 .flatMap(type -> typeToServiceMap.get(type).stream())
                 .filter(service -> service.getSupportType().isAssignableFrom(attachable.getClass()))
                 .map(service -> service.getAttachment(attachable))
                 .collect(Collectors.toList());

    attachable.attach(items);
}
```

# reactor로 비동기 프로그래밍 적용

`attachService.getAttachment()`를 호출할 때 Network I/O가 발생하고 있다.
문제는 이 메서드가 attachment해야할 갯수만큼 실행이 된다는 점이다.
이를 비동기 프로그래밍을 적용해서 해결해보자.

## 의존성 설정

```gradle
compile('io.projectreactor:reactor-core:3.1.5.RELEASE')
```

## AttachService 수정

`getAttachment`의 반환형을 `Mono<AttachmentWrapperInfo>`로 수정한다.

```java
public interface AttachService<T extends Attachable> {
    AttachmentType getSupportAttachmentType();

    Class<T> getSupportType();

    Mono<AttachmentWrapperItem> getAttachment(Attachable attachable);
}
```

## AttachWriterToBoardService 수정

수정한 `AttachService`의 구현체인 `AttachWriterToBoardService`에 변경된 내용을 반영하자.

```java
@Override
public Mono<AttachmentWrapperItem> getAttachment(Attachable attachable) {
    return Mono.defer(() -> executeGetAttachment(attachable))
                // Network I/O를 사용하므로 elastic()으로 생성된 thread에서 실행되도록 선언
               .subscribeOn(Schedulers.elastic()); 
}

// 원래 getAttachment의 실행하던 부분을 가져왔습니다
// 반환 값에 Mono.just()를 씌웠습니다
private Mono<AttachmentWrapperItem> executeGetAttachment(Attachable attachable) {
    BoardDto boardDto = supportType.cast(attachable);
    Attachment attachment = writerClient.getWriter(boardDto.getWriterId());
    return Mono.just(new AttachmentWrapperItem(supportAttachmentType, attachment));
}
```

## AttachmetAspect 수정

`Attachable`의 구현체의 타입에 맞추어 service를 실행하고
얻은 `Mono`의 List를 각각 비동기로 실행시키고, `block()`을 호출해 동기화합니다

```java
private void executeAttach(Attachable attachable) {

    List<Mono<AttachmentWrapperItem>> monoItems = createMonoList(attachable);

    List<AttachmentWrapperItem> items = executeMonoAndCollectList(monoItems);

    attachable.attach(items);
}

// Attachable의 타입에 맞추어 서비스를 실행 후 List<Mono>를 생성함
private List<Mono<AttachmentWrapperItem>> createMonoList(Attachable attachable) {
    Set<AttachmentType> types = attachmentTypeHolder.getTypes();
    return types.stream()
                .flatMap(type -> typeToServiceMap.get(type).stream())
                .filter(service -> service.getSupportType().isAssignableFrom(attachable.getClass()))
                .map(service -> service.getAttachment(attachable))
                .collect(Collectors.toList());
}

// List<Mono>를 zip()으로 각각 실행하면서 List<attachmentWrapperItem>으로 만들어 반환
// 각각의 Mono는 내부에서 elastic()에 의해 비동기로 실행되며
// block()을 통해 최종적으로 동기화됨
private List<AttachmentWrapperItem> executeMonoAndCollectList(List<Mono<AttachmentWrapperItem>> monoItems) {
    return Mono.zip(monoItems, this::filterItems)
               .block();
}

private List<AttachmentWrapperItem> filterItems(Object[] itemArray) {
    return Stream.of(itemArray)
                 .map(AttachmentWrapperItem.class::cast)
                 .collect(Collectors.toList());
}
```

## 실행

이제 비동기로 돌아가는 것을 확인해볼 차례다.
테스트 코드를 짜서 확인하는 것이 가장 좋겠으나...
간편히 `Thread.sleep(3000)`을 줘서 확인해보도록 한다.

```java
// 댓글 서비스에 3초 슬립
private Mono<AttachmentWrapperItem> executeGetAttachment(Attachable attachable) {
    try { Thread.sleep(3000); } catch (InterruptedException e) { }
    BoardDto boardDto = supportType.cast(attachable);
    Attachment attachment = new SimpleAttachmentCollection<>(commentClient.getComments(boardDto.getId()));
    return Mono.just(new AttachmentWrapperItem(supportAttachmentType, attachment));
}

// 작성자 정보 서비스에 3초 슬립
private Mono<AttachmentWrapperItem> executeGetAttachment(Attachable attachable) {
    try { Thread.sleep(3000); } catch (InterruptedException e) { }
    BoardDto boardDto = supportType.cast(attachable);
    Attachment attachment = writerClient.getWriter(boardDto.getWriterId());
    return Mono.just(new AttachmentWrapperItem(supportAttachmentType, attachment));
}
```

3초 이상, 6초 이내에 요청이 오면 성공입니다

![스크린샷 2018-03-08 오후 7.42.16.png](/images/Spring에서_요청에_따른_부가_응답_추가하기/success.png)

# reactor로 실패 극복

reactor로 실패를 극복하는 방법은 간단하다.
오류가 발생하면 앞서 작성했던 `AttachmentWrapperItem.ON_ERROR`를 반환하도록 하면 된다.
Rx는 이러한 상황을 위한 API들이 모두 정의하고 있다.

## AttachService에서 예외 발생시 처리

`AttachWriterToBoardService`에서 `Writer`를 가져오는 중에 Exception이 발생하면 `AttachmentWrapperItem.ON_ERROR`를 보내도록 변경한다.

```java
@Slf4j
@Component
public class AttachWriterToBoardService implements AttachService<BoardDto> {

    private static final AttachmentType supportAttachmentType = AttachmentType.WRITER;
    private static final Class<BoardDto> supportType = BoardDto.class;
    private final WriterClient writerClient;
    private final Duration timeout;

    @Autowired
    public AttachWriterToBoardService(@NonNull WriterClient writerClient,
                                      @Value("${attach.writer.timeoutMillis:5000}") long timeout) {
        this.writerClient = writerClient;
        this.timeout = Duration.ofMillis(timeout);
    }

    @Override
    public AttachmentType getSupportAttachmentType() {
        return supportAttachmentType;
    }

    @Override
    public Class<BoardDto> getSupportType() {
        return supportType;
    }

    @Override
    public Mono<AttachmentWrapperItem> getAttachment(Attachable attachable) {
        return Mono.defer(() -> executeGetAttachment(attachable))
                   .subscribeOn(Schedulers.elastic())
                   .timeout(timeout) // reactor에 timeout을 줘도 되고, client에서 자체적으로 timeout을 걸 수 있으면 믿고 쓰자
                   .doOnError(e -> log.warn(e.getMessage(), e)) // 오류가 발생하면 log를 남긴다. 대체 값을 반환하므로 warn으로 지정
                   .onErrorReturn(AttachmentWrapperItem.ON_ERROR); // 오류가 발생하면 ON_ERROR를 반환
    }

    private Mono<AttachmentWrapperItem> executeGetAttachment(Attachable attachable) {
        BoardDto boardDto = supportType.cast(attachable);
        Attachment attachment = writerClient.getWriter(boardDto.getWriterId());
        return Mono.just(new AttachmentWrapperItem(supportAttachmentType, attachment));
    }
}
```

## AttachmentAspect에서 ON_ERROR를 거르도록 로직 변경

앞서 `AttachmentAspect`에서 `List<Mono>`를 비동기로 실행시키고 결과 값들을 `List<AttachmentWrapperItem>`에 모아서 `Attachable`에 넣어줬다.
간단히 비동시 실행결과가 `ON_ERROR`인 경우를 필터링하면 성공한 결과만을 모아 `List<AttachmentWrapperItem>`을 만들 수 있다.

```java
@Slf4j
@Component
@Aspect
public class AttachmentAspect {

    private final AttachmentTypeHolder attachmentTypeHolder;
    private final Map<AttachmentType, List<AttachService<? extends Attachable>>> typeToServiceMap;

    @Autowired
    public AttachmentAspect(@NonNull AttachmentTypeHolder attachmentTypeHolder,
                            @NonNull List<AttachService<? extends Attachable>> attachService) {
        this.attachmentTypeHolder = attachmentTypeHolder;
        this.typeToServiceMap = attachService.stream()
                                             .collect(Collectors.groupingBy(AttachService::getSupportAttachmentType, Collectors.toList()));
    }

    @Pointcut("@annotation(com.parfait.study.simpleattachment.attachment.Attach)")
    private void pointcut() {
    }

    @AfterReturning(pointcut = "pointcut()", returning = "returnValue")
    public Object afterReturning(Object returnValue) {

        if (attachmentTypeHolder.isEmpty() && !(returnValue instanceof Attachable)) {
            return returnValue;
        }

        executeAttach((Attachable) returnValue);

        return returnValue;
    }

    private void executeAttach(Attachable attachable) {

        List<Mono<AttachmentWrapperItem>> monoItems = createMonoList(attachable);

        List<AttachmentWrapperItem> items = executeMonoAndCollectList(monoItems);

        attachable.attach(items);
    }

    private List<Mono<AttachmentWrapperItem>> createMonoList(Attachable attachable) {
        Set<AttachmentType> types = attachmentTypeHolder.getTypes();
        return types.stream()
                    .flatMap(type -> typeToServiceMap.get(type).stream())
                    .filter(service -> service.getSupportType().isAssignableFrom(attachable.getClass()))
                    .map(service -> service.getAttachment(attachable))
                    .collect(Collectors.toList());
    }

    private List<AttachmentWrapperItem> executeMonoAndCollectList(List<Mono<AttachmentWrapperItem>> monoItems) {
        // 여기에 timeout을 주어서 모든 Mono가 실행되는 최대 시간을 지정할 수도 있다
        return Mono.zip(monoItems, this::filterItems)
                   .doOnError(e -> log.warn(e.getMessage(), e))
                   .onErrorReturn(Collections.emptyList()) // 모든 Mono를 실행시키고 취합하는 과정에서 오류가 발생하면 emptyList()를 반환
                   .block();
    }

    private List<AttachmentWrapperItem> filterItems(Object[] itemArray) {
        return Stream.of(itemArray)
                     .map(AttachmentWrapperItem.class::cast)
                     .filter(item -> item != AttachmentWrapperItem.ON_ERROR) // 예외 발생으로 인해 실패한 요청은 걸러냄
                     .collect(Collectors.toList());
    }
}
```

## 실행

이전에 100번 게시판을 불러올 때, 작성자 정보를 가져오려고 하면 `FeignClient`에서 404를 던지고 아래와 같이 API 자체가 실패했었다.

`GET /boards/100?attachment=comments,writer`

```json
{
  "timestamp": "2018-03-08T07:55:22.127+0000",
  "status": 500,
  "error": "Internal Server Error",
  "message": "status 404 reading WriterClient#getWriter(long); content: {}",
  "path": "\/boards\/100"
}
```

```
feign.FeignException: status 404 reading WriterClient#getWriter(long); content: {}
	at feign.FeignException.errorStatus(FeignException.java:62) ~[feign-core-9.5.1.jar:na]
        ...
	at java.lang.Thread.run(Thread.java:748) ~[na:1.8.0_152]
```

하지만 장애 극복을 적용한 후에는 예외는 `warn`으로 로그를 남기되, 성공한 부분까지는 응답을 할 수 있게 되었다.

```json
{
{
  "id": 100,
  "title": "title100",
  "content": "content100",
  "comments": [
    {
      "id": 496,
      "email": "Zola@lizzie.com",
      "body": "neque unde voluptatem iure\nodio excepturi ipsam ad id\nipsa sed expedita error quam\nvoluptatem tempora necessitatibus suscipit culpa veniam porro iste vel"
    }
  ]
}
```
```
2018-03-08 19:59:12.056  WARN 64890 --- [      elastic-5] c.p.s.s.a.s.w.AttachWriterToBoardService : status 404 reading WriterClient#getWriter(long); content:
{}

feign.FeignException: status 404 reading WriterClient#getWriter(long); content: {}
	at feign.FeignException.errorStatus(FeignException.java:62) ~[feign-core-9.5.1.jar:na]
        ...
	at java.lang.Thread.run(Thread.java:748) ~[na:1.8.0_152]
```

# 마무리

Reactor를 사용해서 비동기 프로그래밍을 하고, 장애에 대처해 극복할 수 있게 해봤다.
중간에 reactor에 `timeout()`을 사용했는 데,
이 부분은 client를 `FeignClient`를 사용해서 `application.yml`로 빼서 따로 관리할 수 있다.
이전에 공유했던 `Hystrix`와도 연계해서 fallback을 구현할 수도 있어서, 강력한 장애 대응을 할 수 있다.

여전히 현재의 코드는 큰 단점이 있다.
`AttachmentAspect`에서 reactor의 `block()`을 호출한다는 점이다.

이게 왜 단점이냐는 것은 [reactor learn 페이지](https://projectreactor.io/learn)에서 가져온 사진 3장으로 설명할 수 있을 것 같다.

![스크린샷 2018-03-09 오전 10.30.20.png](/images/Spring에서_요청에_따른_부가_응답_추가하기/block.PNG)

단점은 Non-Blocking을 사용해서 `자원을 효율적으로` 사용하지 않았다는 것이다.
100만원짜리 서버를 써야 하던 일을, 50만원짜리 서버로 처리할 수 있다면 그렇게 해야한다.
때문에 SpringFramework 5에서는 webflux를 사용하여 netty기반(기본설정)으로
Non-Blocking + Async를 사용할 수 있도록 했다.

추후 기회가 된다면 Spring5 webflux 모듈로 포팅해서 올리겠다.
가능한지 불가능한지는 아직 공부를 안해서 모르겠다...