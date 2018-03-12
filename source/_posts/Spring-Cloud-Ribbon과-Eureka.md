---
title: (Spring Cloud)Ribbon과 Eureka
date: 2018-03-11 01:07:43
tags: [spring-cloud,ribbon,eureka,netflix]
categories:
  - [spring,cloud,netflix]
---
# Ribbon(Load Balancer)

## Ribbon

* 클라이언트 측 로드 밸런서
* 여러 서버를 라운드로빈 방식의 부하 분산 기능을 제공(여러 알고리즘 사용 가능)
* Spring Cloud Config와 결합하여, 서버 목록을 제공받아 사용할 수 있음

<!-- more -->

## 전통적인 LoadBalance 구조

**상황**

- 게시글 상세 화면 : 게시글 내용, 댓글, 추천 게시물을 노출
- client에서 `http://api-gateway.com/boards/1`로 `api-gateway` 호출
- `api-gateway`에서는 `board 서버`, `comment 서버`, `recommend 서버`를 각각 호출

{% plantuml %}
[client] -down-> [LoadBalancer] : 1. http://api-gateway.com/boards/1 호출
[LoadBalancer] -down-> [api-gateway] : 2. /boards/1 호출 
[api-gateway] -up-> [LoadBalancer] : 3. http://board-server/boards/1 호출
[LoadBalancer] -down-> [board-server] : 4. /boards/1 호출

note left of [board-server]
  http://board-server:9001
  http://board-server:9002
end note
{% endplantuml %}

## Ribbon을 사용한 구조

{% plantuml %}
[client] -down-> [LoadBalancer] : 1. http://api-gateway.com/boards/1 호출
[LoadBalancer] -down-> [api-gateway] : 2. /boards/1 호출 
[api-gateway] -down-> [member-server] : load balance
[api-gateway] -down-> [board-server] : load balancing

note right of [api-gateway]
  http://localhost:8080
end note

note right of [member-server]
  http://localhost:8081
  http://localhost:8082
end note

note right of [board-server]
  http://localhost:8083
  http://localhost:8084
end note
{% endplantuml %}

## Example

**의존성**

```
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-ribbon')
}

ext {
    springCloudVersion = 'Finchley.M7'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

**application.yml**

```
board-api:
  ribbon:
    listOfServers:
      - localhost:8081
      - localhost:8082
```

**interface**

```
@FeignClient(name = "board-api")
public interface BoardClient {
    @GetMapping("/boards/{id}")
    Board get(@PathVariable("id") Long id);
}
```

**최초 호출시 로그**

```
DynamicServerListLoadBalancer for client videos-proxy initialized:
DynamicServerListLoadBalancer:{
  NFLoadBalancer: name = videos-proxy,
  current list of Servers = [10.162.3.34:8003, 10.162.3.37:8003],
  Load balancer stats = Zone stats: {
    unknown = [Zone:unknown;	Instance count:2;
      Active connections count: 0;
      Circuit breaker tripped count: 0;
      Active connections per server: 0.0;]
}
```

## Ribbon 설정

- properties 설정 : `application.yml` 내에 `<client>.ribbon.*` 프로퍼티를 설정할 수 있음
- Java 설정 : `@Configuration`, `@RibbonClient`를 선언하여 설정. `RibbonClientConfiguration`에 있는 설정을 `CustomConfiguration`으로 덮어씀

```java
@Configuration
@RibbonClient(name = "custom", configuration = CustomConfiguration.class)
public class TestConfiguration {
}
```

> `CustomConfiguration` 클래스는 반드시 `@Configuration`이 선언되어 있어야 하지만, `@ComponentScan`에 의해 등록되어서는 안된다
> 그렇지 않으면 모든 `@RibbonClients`가 공유하게 된다
> `@ComponentScan`이나 `@SpringBootApplication`을 사용하는 경우, 명시적으로 `CustomConfiguration`을 제외 시켜야한다

**Spring Cloud Netflix는 Ribbon에서 사용하기 위해 아래의 Bean을 기본으로 제공함(즉, 확장 포인트!)**

| Bean Type | Bean Name | Class Name |
| - | - | - |
| IClientConfig | ribbonClientConfig | DefaultClientConfigImpl |
| IRule | ribbonRule | ZoneAvoidanceRule |
| IPing | ribbonPing | DummyPing |
| ServerList<Server> | ribbonServerList | ConfigurationBasedServerList |
| ServerListFilter<Server> | ribbonServerListFilter | ZonePreferenceServerListFilter |
| ILoadBalancer | ribbonLoadBalancer | ZoneAwareLoadBalancer |

**커스터마이징 예제**

```java
@Configuration
protected static class FooConfiguration {
	@Bean
	public ZonePreferenceServerListFilter serverListFilter() {
		ZonePreferenceServerListFilter filter = new ZonePreferenceServerListFilter();
		filter.setZone("myTestZone");
		return filter;
	}

	@Bean
	public IPing ribbonPing() {
		return new PingUrl();
	}
}
```
> 이러한 확장 포인트로 어떻게 LoadBalancing을 할 것인지, 상세한 커스터마이징을 할 수 있음
> Load Balancing은 기본적으로 Round Robin 방식으로 동작하지만,
> spring-actuator의 metrics에 의존하여 그리 어렵지 않게, CPU 사용량, 메모리 사용양, 최근 30초 내 요청 수 등을 기준으로
> 상세한 부하분산 정책을 정의할 수 있음(https://cloud.spring.io/spring-cloud-netflix/multi/multi_netflix-metrics.html)

## Ribbon의 한계

`board-server`에 과부하가 염려되어 서버를 5개를 추가하였다
Ribbon의 설정을 변경하여, `api-gateway`도 다시 빌드해서 배포해야한다
**`board-server`가 새로 추가되고, 제거되는 것을 재빌드/배포없이 무중단으로 적용하고 싶다면 Eureka와 함께 사용하자**

여기까지의 예제 소스

https://github.com/supawer0728/simple-spring-cloud/tree/ribbon

# Eureka(동적 서비스 등록, 발견)

## 동적 서비스 등록, 발견

### 왜?

* 서버 개수 동적 조절 - elastic service에서 URL을 환경설정 파일에 정적으로 미리 지정하는 것은 적합하지 않음
* IP주소는 예측할 수 없으며, 어떤 파일에서 정적으로 관리하기가 어려움

### 등록

{% plantuml %}
component [board-api:8080\n<eureka client>]
component [board-api:8081\n<eureka client>]
component [board-api:8082\n<eureka client>]
component [eureka-server(registry)]

[board-api:8080\n<eureka client>] -up-> [eureka-server(registry)] : 등록
[board-api:8081\n<eureka client>] -up-> [eureka-server(registry)] : 등록
[board-api:8082\n<eureka client>] -up-> [eureka-server(registry)] : 등록
{% endplantuml %}

* 서비스 제공자 관점의 프로세스
* 새로운 서비스가 시작될 때 중앙의 서비스 레지스트리에 등록
* 장애가 발생시 서비스 레지스트리에서 제외
* 레지스트리는 항상 최신 정보

### 발견

{% plantuml %}
[api-gateway\n<eureka client>] -> [eureka-server(registry)] : 1.등록된 서비스 조회

[api-gateway\n<eureka client>] -down-> [board-api:8080\n<eureka client>] : 2.호출
[api-gateway\n<eureka client>] -down-> [board-api:8081\n<eureka client>] : 2.호출
[api-gateway\n<eureka client>] -down-> [board-api:8082\n<eureka client>] : 2.호출
{% endplantuml %}

* 사용자 관점의 프로세스
* 사용자가 서비스 레지스트리에서 필요한 서비스를 찾아서 호출할 수 있게 함
* URL을 정적으로 관리하는 대신 서비스 레지스트리를 통해 그때 그때 사용 가능한 URL을 발견할 수 있음

## Eureka

**Server**

* Server는 모든 micro service가 자신의 가용성을 등록하는 레지스트리
* 등록 정보는 service id와 url을 포함
* micro service가 시작되면 Eureka 서버에 접근해서 서비스 ID와 URL 등의 정보를 등록하고 자신을 알림(30초 heart-beat ping)

**Client**

* Server로부터 레지스트리 정보를 읽어와 로컬에 캐시
* 30초마다 갱신
* 레지스트리 정보의 차이를 가져오는 방식으로 갱신(delta updates)

## Server Example

**의존성**

```
dependencies {
    compile 'org.springframework.cloud:spring-cloud-starter-eureka-server'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-netflix:${springCloudNetflixVersion}"
    }
}
```

**application.properties**

```
spring.application.name=service-registry
server.port=8761

eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
eureka.client.registerWithEureka=false
eureka.client.fetchRegistry=false
```

**@EnableEurekaServer**

```
@EnableEurekaServer
@SpringBootApplication
public class ServiceRegistryApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaSampleApplication.class, args);
    }
}
```

## Client Example

**의존성**

```
compile('org.springframework.cloud:spring-cloud-starter-eureka')
```

### board-api

**application.yml**

```
spring.application.name: board-api
eureka.client.service-url.defaultZone: http://localhost:8761/eureka
```

**설정**

```
@EnableDiscoveryClient
@SpringBootApplication
public class BoardApplication {

    public static void main(String[] args) {
        SpringApplication.run(BoardApplication.class, args);
    }
}
```

### api-gateway

**application.yml**

```
spring.application.name: api-gateway
eureka.client.service-url.defaultZone: http://localhost:8761/eureka
```

**설정**

```
@EnableDiscoveryClient
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

## service-registry 실행 예제

`http://localhost:8761`

![스크린샷 2018-03-05 오후 6.00.03.png](/images/ribbon-eureka/실행예제1.png)

## board-api 실행 예제

`8081`, `8082` 로 서버를 띄움

`BOARD-API`에 두 서버가 등록됨

![스크린샷 2018-03-05 오후 6.07.27.png](/images/ribbon-eureka/실행예제2.png)

## gateway 실행 예제

`8080`으로 서버를 띄움

`API-GATEWAY`로 서버가 등록된

![스크린샷 2018-03-05 오후 6.11.08.png](/images/ribbon-eureka/실행예제3.png)

**`GET http://localhost:8080/boards/1`을 실행시 로그**

```
2018-03-05 18:12:44.865  INFO 36121 --- [nio-8080-exec-1] c.n.l.DynamicServerListLoadBalancer      : 
DynamicServerListLoadBalancer for client board-api initialized: DynamicServerListLoadBalancer:
{
  NFLoadBalancer:name=board-api,current list of Servers=[10.78.93.40:8081, 10.78.93.40:8082],
  Load balancer stats=Zone stats: {
    defaultzone=[
      Zone:defaultzone;
      Instance count:2;
      Active connections count: 0;
      Circuit breaker tripped count: 0;
      Active connections per server: 0.0;
    ]
  },
  Server stats: [
    [
      Server:10.78.93.40:8082;
      Zone:defaultZone;
      Total Requests:0;
      Successive connection failure:0;
      Total blackout seconds:0;
      Last connection made:Thu Jan 01 09:00:00 KST 1970;
      First connection made: Thu Jan 01 09:00:00 KST 1970;
      Active Connections:0;
      total failure count in last (1000) msecs:0;
      average resp time:0.0;
      90 percentile resp time:0.0;
      95 percentile resp time:0.0;
      min resp time:0.0;
      max resp time:0.0;
      stddev resp time:0.0
    ], [
      Server:10.78.93.40:8081;
      Zone:defaultZone;	Total Requests:0;
      Successive connection failure:0;
      Total blackout seconds:0;
      Last connection made:Thu Jan 01 09:00:00 KST 1970;
      First connection made: Thu Jan 01 09:00:00 KST 1970;
      Active Connections:0;
      total failure count in last (1000) msecs:0;
      average resp time:0.0;
      90 percentile resp time:0.0;
      95 percentile resp time:0.0;
      min resp time:0.0;
      max resp time:0.0;
      stddev resp time:0.0
    ]
  ]
}
```

```json
{
  "title": "title1",
  "content": "lorem ipsum1"
}
```

## Eureka 고가용성

**구조**

{% plantuml %}
[eureka-client1] -up-> [DNS/Load Balancer]
[eureka-client2] -up-> [DNS/Load Balancer]
[eureka-client3] -up-> [DNS/Load Balancer]
[DNS/Load Balancer] -up-> [eureka-server1]
[DNS/Load Balancer] -up-> [eureka-server2]

[eureka-server1] <-> [eureka-server2] : peer to peer
{% endplantuml %}

**설정**

```java
@EnableEurekaServer
@EnableDiscoveryClient
@SpringBootApplication
public class ServiceRegistryApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(AmpVideoCloudEurekaServerApplication.class, args);
    }
}
```

**application.yml**

`spring.profiles.active`을 `peer1`, `peer2` 등으로 설정하여 사용하도록 한 예제

```yml
peer1:
  address: peer1
  port: 8761
peer2:
  address: peer2
  port: 8761
---
spring:
  profiles: peer1
server:
  address: ${peer1.address}
  port: ${peer1.port}
eureka:
  instance:
    hostname: peer1
  client:
    service-url:
      defaultZone: http://${peer2.address}:${peer2.port}/eureka/
---
spring:
  profiles: peer2
server:
  address: ${peer2.address}
  port: ${peer2.port}
eureka:
  instance:
    hostname: peer2
  client:
    service-url:
      defaultZone: http://${peer1.address}:${peer1.port}/eureka/
```

**TC서버에 올린 뒤 확인된 주의사항**

1. `/etc/hosts`에 peer1, peer2에 대한 설정을 해줘야함

```
10.162.3.34     tcampapp-92c901.svr.toastmaker.net tcampapp-92c901
10.162.3.37     peer2 tcampapp-92d901
0.0.0.0         peer1
```

2. `application`이 `0.0.0.0`으로 bind되어야함.
`127.0.0.1`로 bind되는 경우, 외부에서 접근이 안됨.
사설 IP로 bind된 경우 `localhost:8761`이 접근이 안되는 현상 발생
3. `eureka.client.service-url.defaultZone`의 `defaultZone`은 무조건 camelCase로 작성해야함(`Map<String, String>`)
4. P2P 통신을 통한 상호 업데이트의 기본 인터벌은 30초

**그 외의 설정**

https://github.com/Netflix/eureka/wiki/Overriding-Default-Configurations

# 마무리

**참고**

Spring cloud reference
- ribbon : http://cloud.spring.io/spring-cloud-static/Finchley.M7/single/spring-cloud.html#spring-cloud-ribbon
- eureka-client : http://cloud.spring.io/spring-cloud-static/Finchley.M7/single/spring-cloud.html#_service_discovery_eureka_clients
- eureka-server : http://cloud.spring.io/spring-cloud-static/Finchley.M7/single/spring-cloud.html#spring-cloud-eureka-server

서적
스프링 마이크로 서비스(라제시 RV, 2017.07.27, acorn+PACKT) : http://www.acornpub.co.kr/book/spring-microservices

예제 소스 : https://github.com/supawer0728/simple-spring-cloud/tree/eureka