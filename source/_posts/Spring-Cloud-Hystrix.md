---
title: (Spring Cloud) Hystrix
date: 2018-03-11 01:03:18
tags: [spring-cloud, hystrix, netflix, spring]
categories:
  - [spring,cloud,netflix]
---

# Hystrix란

Netflix에서 [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)을 구현한 라이브러리.
Micro Service Architecture에서 장애 전파 방지를 할 수 있음
<!-- more -->
## Circuit Breaker Pattern

![Inline-image-2018-02-28 17.38.25.834.png](https://martinfowler.com/bliki/images/circuitBreaker/sketch.png)

# 장애 연쇄

위 그림에서 `supplier 서버`에 장애가 생겨 항상 Timeout이 발생한다는 경우,
`supplier 서버`를 호출한 `client 서버`는 Timeout이 발생할 때까지 응답이 밀리게 되고,
응답이 밀리는 동안 요청이 계속 쌓여 결국 `client 서버`까지 요청이 과하게 밀려 장애가 발생할 수 있음.

이러한 상황이 발생하지 않도록 `circuit breaker`를 두어 장애 전파를 막을 수 있도록 함

![image2](https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/master/docs/src/main/asciidoc/images/HystrixFallback.png)

# Hystrix Flow Chart

![Inline-image-2018-03-05 10.55.00.102.png](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/hystrix-command-flow-chart.png)

1. `HystrixCommand`, `HystrixObservableCommand` 객체 생성
2. Command 실행
3. 캐시 여부 확인
4. 회로 상태 확인
5. 사용가능한 Thread Pool/Queue/Semaphore가 있는지 확인
6. `HystrixObservableCommand.construc()`, 혹은 `HystrixCommand.run()` 실행
7. 회로 상태 연산(Calculate circuit health)
8. fallback 실행
9. 응답 반환

# Hystrix Circuit Breaker 구현

![Inline-image-2018-03-05 11.04.14.221.png](https://github.com/Netflix/Hystrix/wiki/images/circuit-breaker-1280.png)

1. circuit health check를 위한 최소한의 요청이 있을 때(`HystrixCommandProperties.circuitBreakerRequestVolumeThreshold()`)
2. 그리고, 지정한 오류율을 초과했을 때(`HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()`)
3. 회로의 상태를 `CLOSED`에서 `OPEN`으로 변경
4. 회로가 열린 동안, 모든 요청에 대해서 fallback method을 바로 실행
5. 일정 시간이 지난 후(`HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()`), 하나의 요청을 원래 method로 실행(`HALF OPEN`). 이 요청이 실패한다면 `OPEN`으로 두고, 이 요청이 성공한다면 `CLOSED`로 상태를 변경. 다시 1번으로 돌아감.

# 기본 설정

- `metrics.rollingStats.timeInMilliseconds` : 오류 감시 시간, 기본값 10초
- `circuitBreaker.requestVolumeThreshold` : 감시 시간 내 요청 수, 기본값 20
- `circuitBreaker.errorThresholdPercentage` : 요청 대비 오류율, 기본값 50

기본 설정을 풀어서 설명하면 다음과 같음

> 감시시간 내(30초)에, 20번 이상의 요청이 있었고, 그 중에서 오류율이 50% 이상일 때 Circuit Breaker가 작동(circuit open)
> 감시 시간 내에 요청이 반드시 20번 이상이 있어야 회로가 열림. 30초 동안 요청이 19번이었고 모두 실패했어도 Circuit Breaker는 작동하지 않음

# 예제

**dependencies**

```gradle
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-netflix-hystrix')
}
```

**사용**
```java
@SpringBootApplication
@EnableCircuitBreaker
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}

@Component
public class StoreIntegration {

    @HystrixCommand(commandKey="getStores", fallbackMethod = "defaultStores")
    public Object getStores(Map<String, Object> parameters) {
        //do stuff that might fail
    }

    public Object defaultStores(Map<String, Object> parameters) {
        return /* something useful */;
    }
}
```

- `StoreIntegration.getStores()`가 실패하거나 회로가 열렸을 시에는 `.defaultStores`가 실행됨

# 설정

**application.yml로 설정예시**

```yml
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 10000
    # 특정 commandKey에 대한 설정
    getStores:
      execution.isolation.thread.timeoutInMilliseconds: 10000
```
**java 설정예시**

```java
class UserResource {
    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
        },
        threadPoolProperties = {
            @HystrixProperty(name = "coreSize", value = "30"),
            @HystrixProperty(name = "maxQueueSize", value = "101"),
            @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),
            @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),
            @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "12"),
            @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "1440")})
    public User getUserById(String id) {
        return userResource.getUserById(id);
    }
}
```

설정 참고:
https://github.com/Netflix/Hystrix/wiki/Configuration#execution.isolation.strategy

# 실사용례

Micro Service Architecture에서 하나의 게시판을 보여주는 예제(게시글, 댓글, 추천 게시글 API를 호출)
예제는 spring 4.3.x + reactor로 작성되었습니다

{% plantuml %}
component client
component apiGateway
component boardServer
component commentServer
component recommendServer

client -> apiGateway: /boards/1
apiGateway -up-> boardServer: /boards/1
apiGateway -> commentServer: /comments/board-id/1
apiGateway -down-> recommendServer: /recommendations/type/board/id/1
{% endplantuml %}

**서비스**

```java
@Slf4j
@Component
public class BoardService {
  @Autowired private BoardClient boardClient;
  @Autowired private CommentClient commentClient;
  @Autowired private RecommendationClient recoClient;
  @Value("service.board.detail.timeout") private Duration detailTimeout;
  
  public BoardDetailDto getDetail(long id) {
  
    // board server에서 게시물 단건 호출, 실패시 실패 응답 반환
    Board board = boardClient.getOne(id);
    if (board == Board.ERROR) {
      return BoardDetailDto.ERROR;
    }
    
    // board를 기반으로 댓글과 추천 게시물을 비동기 동시 호출
    return Mono.zip(Mono.just(board), commentClient.getCommentsByBoardId(id), recoClient.getRecommendationsByBoardId(id))
               .map(TupleUtils.function(BoardDetailDto::new))
               .timeout(detailTimeout)
               .doOnError(e.getMessage(), e)
               .onErrorReturn(BoardDetailDto.create(board)) // 실패시 board 만이라도 DTO로 만들어 응답
               .blockOptional()
               .orElse(BoardDetailDto.ERROR);
  }
}
```

**CommentClient.java**

```java
public class CommentClient {

  @HystrixCommand(fallbackMethod = "getCommentsByBoardIdFallback")
  public Mono<List<Comment>> getCommentsByBoardId(long id) {
    //...
  }
  
  public Mono<List<Comment>> getCommentsByBoardIdFallback(long id) {
    return Mono.just(Collections.emptyList());
  }
}
```

- 여러 서비스를 호출하면서도, 하위 서비스의 장애가 client로 노출되지 않고, 성공한 서비스 응답들은 모두 client로 줄 수 있음

# Isolation

bulkhead pattern를 채용하여 종속성(dependency)을 분리하며, 각각에 대한 접근을 제한함

![Inline-image-2018-03-05 11.45.23.425.png](https://github.com/Netflix/Hystrix/wiki/images/soa-5-isolation-focused-640.png)

## Threads & ThreadPool

호출 thread와 별도의 thread(ex. Tomcat thread pool)에서 동작. 

![Inline-image-2018-03-05 11.48.29.958.png](https://github.com/Netflix/Hystrix/wiki/images/request-example-with-latency-1280.png)

> ThreadPool을 사용하지 않아도 되는 경우
> 1. 네트워크 connection/read timeout, retry 옵션을 사용하여 매우 빨리 실패하거나
> 2. client가 항상 정상동작한다는 신뢰가 있는 경우
> 즉, 그냥 ThreadPool을 사용합시다^^

**Netflix에서 각각의 Thread pool을 사용하여 의존성 격리를 구성한 이유**

- 결론부터 먼저 말하자면, Thread를 나누어 다른 Thread에 접근하기 어렵도록 종속성을 원천차단하기 위함
- application은 수없이 많은 팀의, 수없이 많은 back-end service 를 수없이 많이 호출한다
- 각 service는 client library를 가지고 있다
- client library는 항상 바뀐다
- client library는 새로운 네트워크를 호출할 수도 있고, retry, parsing, caching 등의 logic을 가지며 `blackbox` 취급된다

![Inline-image-2018-03-05 11.59.03.923.png](https://github.com/Netflix/Hystrix/wiki/images/isolation-options-1280.png)

**Thread Pool 사용상 이점**

- application이 client library로부터 보호된다
- 덕분에 새 client library를 추가할 때의 risk를 낮출 수 있다, 장애는 격리된 thread에서 발생한다

**Thread Pool 사용상 단점**

- queueing, scheduling, context switching 등의 오버 헤드 발생(Netflix에서는 이를 사소한 정도로 간주)

**Thread 비용**

- Hystrix는 자식 thread에서 `construct()`, `run()`을 실행할 때, 부모 thread에서 총 종단 시간을 측정하여 overhead를 계산
- Netflix에서는 10억 건 이상의 Hystrix Command를 실행하며, 각 API 인스턴스마다 5-20개의 thread를 가지고 있는 thread pool을 40+개를 설정함(대부분의 thread pool 내의 thread 개수는 10개)

**ThreadLocal**

기본적으로 `@HytrixCommand`는 다른 Thread로 동작을 하기 때문에, ThreadLocal이나 spring에서 지원해주는 `@RequestScope`, `@SessionScope` 빈에 접근할 수 없음
필요한 경우 `execution.isolation.strategy: SEMAPHORE`로 변경하여 현재 Thread에서 연산을 실행하게 할 수 있음
Spring Security를 사용하는 경우, `hystrix.shareSecurityContext=true`로하여 `SecurityContext`를 공유할 수 있음

> THREAD 동작 방식의 경우에는 Thread-pool내의 Thread 갯수 만큼, SEMAPHORE 동작 방식의 경우에는 semaphore count 만큼 요청을 수행할 수 있음
> `execution.isolation.semaphore.maxConcurrentRequests`

## Semaphore

Thread pool을 사용하는 대신, `Semaphore(counter)`를 사용하여 종속성에 대한 동시 호출 수를 제한할 수 있음.
따라서 Thread를 사용하지 않고 부하를 분한하지만, timeout과 격리가 느슨해지는 단점이 있음
위에서 `ThreadPool을 사용하지 않아도 되는 경우`에서 설명한 것과 같이 `back-end server를 신뢰할 수 있다면 사용해도 괜찮음`

`HystrixCommand`와 `HystrixObservableCommand`는 두 곳에서 `semaphore`를 지원

- Execution: `execution.isolation.strategy=SEMAPHORE`로 설정이 되어 있으면, 해당 command를 실행할 수 있는 부모 스레스 수를 제한
- Fallback 검색
