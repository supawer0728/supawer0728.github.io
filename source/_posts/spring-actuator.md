---
title: (Spring Boot)Spring Boot Actuator 소개
date: 2018-05-12 17:33:26
tags: [spring, spring-boot, spring-boot-actuator]
categories:
  - [spring, boot, actuator]
---
# 서론

웹 개발자로서 웹 애플리케이션을 만들 때 신경써야할 것은 서비스 로직 뿐만이 아니다. 웹 애플리케이션의 사용자는 누구인지(일반인? 외부시스템?), 어떤 경로로 애플리케이션에 요청을 할 지(Load Balancing, Fire Wall), 요청 수나 TPS 등 많은 것들을 고려해야한다. 이번에 소개할 `spring-boot-actuator`라는 모듈은 **애플리케이션의 상태**를 종합적으로 정리하여 우리에게 제공해준다. 본문에서는 [Spring Boot Samples](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples)에 있는 프로젝트 중에서 `spring-boot-sample-actuator-ui`를 가져와 살짝 소스를 수정하여 actuator가 무엇이고 어떤 일 해주는지 알아보려한다.

<!-- more -->

# 샘플 프로젝트로 Actuator 체험해보기

## 소스 수정

[spring-boot-sample-actuator-ui](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-actuator-ui)에서 소스를 다운받아 `pom.xml` 파일의 `parent`를 수정하여 바로 실행해볼 수 있다. 여기서는 `sprint-boot-starter-security`도 제외시키겠다.

```xml
<!-- parent를 spring-boot-starter-parent로 수정 -->
<parent>
  <!-- Your own application should inherit from spring-boot-starter-parent -->
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.0.2.RELEASE</version>
</parent>

<!-- spring-security 주석처리 -->
<!--<dependency>-->
<!--<groupId>org.springframework.boot</groupId>-->
<!--<artifactId>spring-boot-starter-security</artifactId>-->
<!--</dependency>-->
```

## applcation.properties

```properties
spring.security.user.name=user
spring.security.user.password=password
management.health.diskspace.enabled=false
management.endpoints.web.exposure.include=*
```

앞서 security 의존성을 없앴기 때문에 `spring-security`가 동작하지는 않는다.

## 실행

maven의 `spring-boot:run`이나 직접 `SampleActuatorUiApplication`의 `main()`을 호출하여 서버를 띄운 후 `GET /actuator`를 실행해보자. 아래와 같은 json 정보를 얻을 수 있다.(몇몇 부분은 생략했다) 

```json
{
   "_links":{
      "self":{"href":"http://localhost:8080/actuator"},
      "auditevents":{"href":"http://localhost:8080/actuator/auditevents"},
      "beans":{"href":"http://localhost:8080/actuator/beans"},
      "health":{"href":"http://localhost:8080/actuator/health"},
      "conditions":{"href":"http://localhost:8080/actuator/conditions"},
      "configprops":{"href":"http://localhost:8080/actuator/configprops"},
      "env":{"href":"http://localhost:8080/actuator/env"},
      "env-toMatch":{"href":"http://localhost:8080/actuator/env/{toMatch}"},
      "info":{"href":"http://localhost:8080/actuator/info"},
      "loggers":{"href":"http://localhost:8080/actuator/loggers"},
      "loggers-name":{"href":"http://localhost:8080/actuator/loggers/{name}"},
      "heapdump":{"href":"http://localhost:8080/actuator/heapdump"},
      "threaddump":{"href":"http://localhost:8080/actuator/threaddump"},
      "metrics-requiredMetricName":{"href":"http://localhost:8080/actuator/metrics/{requiredMetricName}"},
      "metrics":{"href":"http://localhost:8080/actuator/metrics"},
      "scheduledtasks":{"href":"http://localhost:8080/actuator/scheduledtasks"},
      "httptrace":{"href":"http://localhost:8080/actuator/httptrace"},
      "mappings":{"href":"http://localhost:8080/actuator/mappings"},
      "jolokia":{"href":"http://localhost:8080/actuator/jolokia"}
   }
}
```

위와 같은 내용의 애플리케이션의 상태를 확인할 수 있다. 예를 들어 `GET /actuator/health`를 실행해보자. 아래와 같이 현재 애플리케이션의 health 상태를 알 수 있다.

```json
{"status": "UP"}
```

혹은 `GET /actuator/metrics/jvm.threads.live`를 요청해보자. 현재 JVM의 활성화된 Thread의 정보를 가져올 수 있다.

```json
{  
   "name":"jvm.threads.live",
   "measurements":[  
      {  
         "statistic":"VALUE",
         "value":23
      }
   ],
   "availableTags": []
}
```

`GET /actuator/heapdump`나 `GET /actuator/threaddump`를 요청하면 dump 파일을 받을 수도 있다.

이러한 애플리케이션의 상태 정보를 아무에게나 노출하면 안될 것 같다. 확인된 사용자에게만 actuator 정보를 노출하고자 할 때에는 간단하게 두 가지 방법을 고려할 수 있다.

1. `management.server.port`, `management.server.address` 값을 수정해서 해당 ip, address에 한해 ACL를 걺
2. `spring-security`를 이용하여 `management.endpoints.web.base-path(기본값 /actuator)`에 대해 권한을 확인

물론 두 방법을 동시에 사용할 수도 있다.

# 그래서 Spring Boot Actuator란

간단히 말하자면 Spring Boot Application의 상태를 관리해준다.

- Spring Boot Application의 상태 정보(health, properties, beans, 구동된 AutoConfiguration 목록 등)를 다룰 수 있도록 자동 설정.
- 각종 추상화 클래스(HealthIndicator 등)을 제공하여, 상태 정보를 변경할 수 있도록 Service를 제공.

# 써먹을 만한 설정과 사용자화 가능한 것들

## 노출할 항목 설정

앞서 본 샘플에서는 많은 정보들이 노출되고 있었다. `auditevents`, `beans`, `health`, `env` 등등, 필요해 보이는 것부터 T.M.I.까지 있다. `application.properties`에서 `management.endpoints.web.exposure.include`에 필요한 endpoint의 id를 설정할 수 있다. endpoint의 목록은 [레퍼런스 문서](https://docs.spring.io/spring-boot/docs/2.0.2.RELEASE/reference/htmlsingle/#production-ready-endpoints)를 참고하자.

**예제: health와 metrics 정보만 노출**

```
management.endpoints.web.exposure.include=health,metrics
```

## Endpoint 경로 설정

앞에서 소개한 것과 같이 `management.endpoint.web.base-path(기본값 /actuator)`를 수정하여 base-path를 수정할 수 있다. 그 외에도 `management.endpoints.web.path-mapping.<id>`값을 수정하여, 특정 id의 endpoint의 경로를 수정할 수 있다.

**예제: health의 경로를 /monitor/healthcheck로 변경**

```
management.endpoints.web.base-path=/monitor
management.endpoints.web.path-mapping.health=healthcheck
```

## CORS

`spring-boot-actuator`는 기본적으로 **클라우드 환경에서 관리자가 각종 애플리케이션의 상태를 파악하기 쉽도록 설계되어 있다.** 때문에 필요한 경우 외부 UI를 구성한 다른 도메인명을 가진 Web Application에서 각각의 서비스 애플리케이션의 상태를 파악하기 위해서 actuator 정보를 요청할 수도 있다. 이럴때 CORS를 설정하여 사용할 수 있다.

**예제: http://other-domain.com 의 GET, POST요청을 허용**

```
management.endpoints.web.cors.allowed-origins=http://other-domain.com
management.endpoints.web.cors.allowed-methods=GET,POST
```

## Health Endpoint

### Details

Health Endpoint는 위 샘플에서 소개된 것 외에도 사실상 많은 정보들을 내보일 수 있다. 이전의 샘플 소스로 돌아가서 `pom.xml`과 `application.properties`를 아래와 같이 수정해보자.

**pom.xml: jpa와 h2 database 의존성 추가**

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
</dependency>
```

**application.properties: 상세 정보를 노출하도록 변경**

```
management.endpoint.health.show-details=always
```

다시 애플리케이션을 실행시킨 후 `GET /actuator/health` 요청을 보내보면 아래와 같이 응답이 온다.

```
{  
   "status":"UP",
   "details":{  
      "db":{  
         "status":"UP",
         "details":{  
            "database":"H2",
            "hello":1
         }
      }
   }
}
```

애플리케이션에서 의존하고 있는 시스템의 Health check도 함께 파악할 수 있다. `DataSource` 외에도 `Disk`, `Cassandra`, `ElasticSearch`, `Redis` 등의 health check를 확인할 수 있다.

### 사용자 정의 Health Indicator

Health Endpoint의 내용은 `HealthIndicator`를 구현하여 정의할 수 있다.

**HealthIndicator**

```java
@FunctionalInterface
public interface HealthIndicator {
    Health health();
}
```

아래는 간단하게 health indicator를 구현한 예제이다.

- `PUT /actuator/health/up` : UP 상태로 변경
- `PUT /actuator/health/down` : DOWN 상태로 변경
- `PUT /actuator/health/maintenance` : 점검 상태로 변경

```java
@RestController
@RequestMapping("${management.endpoints.web.base-path:/actuator}")
public class CustomHealthIndicator implements HealthIndicator {

    private final AtomicReference<Health> health = new AtomicReference<>(Health.up().build());

    @Override
    public Health health() {
        return health.get();
    }

    @PutMapping("${management.endpoints.web.path-mapping.health:health}/up")
    public Health up() {
        Health up = Health.up().build();
        this.health.set(up);
        return up;
    }

    @PutMapping("${management.endpoints.web.path-mapping.health:health}/down")
    public Health down() {
        Health down = Health.down().build();
        this.health.set(down);
        return down;
    }

    @PutMapping("${management.endpoints.web.path-mapping.health:health}/maintenance")
    public Health maintenance() {
        Health maintenance = Health.status(new Status("MAINTENANCE", "점검중")).build();
        this.health.set(maintenance);
        return maintenance;
    }
}
```

각 URL에 맞춰서 `health`의 값을 변경한다. 기본적으로 `up`, `down`, `out-of-service`, `unkwon`의 상태값이 존재하지만 `maintenance`의 상태값은 없다. 사실 의미상 `out-of-service`와 동일하지만, 사용자화를 할 수 있다는 걸 보이기 위해서 새로 만들었다. 새로운 상태값을 만들어 actuator에서 인식하도록 만들려면 `application.properties`를 수정해야한다.

```
# 심각도에 따른 순서
management.health.status.order=DOWN, MAINTENANCE, UNKOWN, UP

# 각 상태별 Http 응답 코드
management.health.status.http-mapping.DOWN=503
management.health.status.http-mapping.MAINTENANCE=503
management.health.status.http-mapping.UNKNOWN=200
management.health.status.http-mapping.UP=200
```

이제 애플리케이션을 실행하여 `PUT /actuator/health/maintenance`를 요청한 후 `GET /actuator/health`로 확인해보자 아래와 같이 나온다.

![점검중](maintenance.png)

#### HealthIndicator를 구현하며, `management.endpoint.health.show-details=always`가 설정된 경우

사용자화한 항목을 하나의 상세로 표시해준다.

```json
{  
   "status":"UP",
   "details":{  
      "custom":{  
         "status":"UP"
      }
   }
}
```

## Metrics

간단히 말하자면 시계열 지표로 활용할 수 있는 정보를 관리한다. 아래는 그 일부이다.

- JVM 정보
  - thread 수
  - GC 정보
  - heap 정보
- DBCP 정보
- PROCESS 관련 정보
- CPU
  - USAGE
  - LOAD
- FILE
  - 최대 사용가능한 File Descriptor 수
  - 현재 사용중인 File Descriptor 수

### 사용자 정의 Metircs

Metrics의 정보도 사용자 정의할 수 있다. 예를 들어 현재 처리중인 동시 요청 수나 분당 요청 처리량 등을 표시할 수 있다. 방법은 간단하다. `MeterRegistry`를 주입받아 사용하면 된다.  아래는 처리중인 동시 요청 수에 대한 예제이다.

```
public class ConcurrentTransactionCountInterceptor extends HandlerInterceptorAdapter {

    private final Counter counter;

    public ConcurrentTransactionCountInterceptor(MeterRegistry meterRegistry) {
        this.counter = meterRegistry.counter("transaction.current.count");
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        counter.increment();
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        counter.increment(-1d);
    }
}
```

`GET /actuator/metrics/transaction.current.count`를 호출하면 아래와 같이 나온다.

```json
{
  "name": "transaction.current.count",
  "measurements":[
    {
      "statistic": "COUNT",
      "value": 0
    }
  ]
}
```

# 마무리

Spring Boot Actuator에 대해서 간략하게 살펴보았다. actuator에서는 이외에도 많은 기능을 제공해주지만, 자주 사용할 것 같은 기능 위주로 살펴봤다.

actuator는 기본적으로 애플리케이션의 상태를 조회하고, 변경할 수 있도록 기능을 추상화하여 정의하고 있다. 추상화된 기능들에 대한 간단한 구현체들을 제공해주며, 개발자로 하여금 사용자 정의를 통해 더욱 상세한 설정을 할 수 있도록 가능성을 열어두었다. actuator로 인해 프로젝트 내에서 애플리케이션 관리를 통합해서 사용할 수 있으며, 이는 클라우드 환경에서 애플리케이션을 관리하는 데 매우 유용하게 사용할 수 있다. Load Balancing은 물론이거니와 필요시 Instance를 자동으로 배포, 제거하는 데에 유용한 정보들을 제공해줄 수 있다.

애플리케이션 개발팀과 인프라스트럭처 관리팀이 나누어진 경우, 인프라스트럭처 관리팀에서 정해둔 정책이 있다면 `spring-boot-starter`를 사용자 정의하여 `actuator`를 회사, 혹은 팀 단위에 맞춰서 미리 정의해서 사용해보자. 그렇게 되면 개발팀은 서비스 로직에만 신경쓸 수 있게 될 것이다.
