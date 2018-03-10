---
title: (Spring Cloud) Zuul
date: 2018-03-11 02:29:18
tags: [spring-cloud, spring, netflix, zuul, api-gateway, proxy]
categories:
  - spring
  - cloud
  - netflix
---
# Zuul

Netflix에서 사용하는 JVM 기반의 라우터이자, 로드 밸런서

아래의 용도로 사용할 수 있다

- 인증과 보안 : 리소스에 대한 인증 정보를 식별하고, 인증이 되지 않는 경우 요청 거부
- 모니터링과 분석 : 서비스 상세를 파악하기 위해, 의미있는 정보와 통계를 추적
- 동적 라우팅
- 트래픽 조정
- 그 외...
<!-- more -->
![image](https://camo.githubusercontent.com/5e596c573110bffb608614a09c97611107205d0d/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d706879736963616c2d617263682e706e67)

# 기본 예제

**상황**

1. client가 zuul을 `http://zuul.com/api/members/1`을 호출하면
2. zuul은 해당 요청을 `member-api`에  `http://member-api/api/members/1`로 보낸다

> 참고로 Ribbon, Feign 등에서 대상 서버의 `spring.application.name`을 domain으로 넣으면, 로드밸런싱 후 알아서 IP로 변환된다
> `http://member-api/api/members/1` -> `http://x.x.x.x/api/members/1`

{% plantuml %}
[client] --> [zuul] : http://zuul.com/api/members/1
[zuul] --> [member-api] : http://member-api/api/members/1
{% endplantuml %}

> 예제 소스는 이전의 `Ribbon과 Eureka`에서 이어짐

## member-api

- eureka 적용
- spring-data-rest 적용(base path: /api)
- `/api/members/1` 호출 시 `{"name": "a","grade": "BRONZE","_links":...}` 응답

**application.yml**

```
spring:
  application.name: member-api
  data.rest.base-path: /api
eureka.client.service-url.defaultZone: http://localhost:8761/eureka
```

- `spring.application.name` : eureka에 등록되는 serivce id

## gateway

**의존성 추가**

```gradle
compile('org.springframework.cloud:spring-cloud-starter-zuul')
```

**자바 설정 추가**

```java
@EnableZuulProxy
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

**application.yml**

```yml
spring.application.name: api-gateway
eureka.client.service-url.defaultZone: http://localhost:8761/eureka
zuul:
  routes:
    member-api:
      path: /api/members/**
      stripPrefix: false
```

- `zuul.routes.<serviceId>.path` : 해당 path의 요청을 `<serviceId>`로 보낸다
- `zuul.routes.<serviceId>.stripPrefix`: false인 경우 uri를 모두 보내며(`/api/members/1`), true인 경우에는 matching된 값을 제외하고 보낸다(`/1`)

## 실행

1. eureka-server start
2. gateway, member-api(8083, 8084) start(순서 상관 없음)

**member-api**

`GET http://localhost:8083/api/members/1`

```
{
  "name": "a",
  "grade": "BRONZE",
  "_links":{
    "self":{
      "href": "http://localhost:8083/api/members/1"
    },
    "member":{
      "href": "http://localhost:8083/api/members/1"
    }
  }
}
```

**gateway(같은 응답)**

`GET http://localhost:8080/api/members/1`

```
{
  "name": "a",
  "grade": "BRONZE",
  "_links":{
    "self":{
      "href": "http://localhost:8083/api/members/1"
    },
    "member":{
      "href": "http://localhost:8083/api/members/1"
    }
  }
}
```

# Hystrix, Ribbon과 결합

eureka에 등록된 대상 서버의 `spring.application.name(serivceId)`를 기반으로 아래와 같이
Hystrix, Ribbon 설정을 결합할 수 있다

```
zuul:
  routes:
    member-api:
      path: /api/members/**
      stripPrefix: false

hystrix:
  command:
    member-api:
      execution:
        isolation:
          thread:
            # Ribbon의 각 timeout보다 커야 기대하는대로 동작함
            timeoutInMilliseconds: 5000 

member-api:
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    ConnectTimeout: 1000
    ReadTimeout: 3000
    MaxTotalHttpConnections: 500
    MaxConnectionsPerHost: 100
```

# Zuul Properties

`application.yml`의 `zuul`로 시작하는 property는 `ZuulProperties` 클래스에 매핑된다

**zuul properties(일부)**

- `zuul.host.maxTotalConnections` : backend에 연결할 수 있는 최대 connection 갯수, 기본 200 -> eureka를 사용하지 않을시 적용됨
- `zuul.host.maxPerRouteConnections` : `route` 별 최대 connection 갯수, 기본 20 -> eureka를 사용하지 않을 시 적용됨
- `zuul.host.socketTimeoutMillis`
- `zuul.host.connectTimeoutMillis`
- `zuul.ribbonIsolationStrategy` : Hystrix의 isolation 방식 - 기본은 `SEMAPHORE`
- `zuul.retryable` : true인 경우 Ribbon client의 설정대로, 요청이 실패시 재시도
- `zuul.addProxyHeaders` : true인 경우, `X-Forwarded-Host` 헤더가 추가됨

> eureka를 사용하면서 readTimeout, socketTimeout을 사용하려면
> `ribbon.ReadTimeout`, `ribbon.SocketTimeout`을 사용해야한다

# Zuul Http Client

zuul은 기본적으로 Apache HTTP Client를 사용

`ribbon.restclient.enabled=true` 혹은 `ribbon.okhttp.enabled=true`를 설정하여 다른 library를 사용할 수도 있다

커스터마이징한 Apache HTTP client, OK HTTP client를 쓰고 싶은 경우, `ClosableHttpClient` 혹은 `OkHttpClient`를 빈으로 등록하면 된다

# Cookie와 Sensitive Header

동일한 시스템 내에서 공유되는 요청 헤더를 외부(downstream)로 노출하지 않아야하는 경우가 있을 수 있다

`sensitiveHeaders`를 설정하여 내부에서 사용되는 헤더값이 노출되는 것을 막을 수 있다

**application.yml**

```
zuul:
  routes:
    users:
      path: /myusers/**
      sensitiveHeaders: Cookie,Set-Cookie,Authorization
      url: https://downstream
```

위 예제의 `Cookie`, `Set-Cookie`, `Authorization`은 zuul의 기본값이며, `sensitiveHeaders`를 사용하지 않으려면 명시적으로 값을 비워둬야한다

# Endpoint 관리(spring-actuator)

`@EnableZuulProxy`를 사용하면 아래의 actuator endpoint가 추가된다

- Routes
- Filters

## Routes Endpoint

### `GET /application/routes`

**eureka를 안쓰는 경우**

```
{
  /api/members/**: "http://localhost:8083"
}
```

**eureka를 쓰는 경우**

```
{
  "/api/members/**": "member-api"
}
```

### `GET /application/routes?format=details`

```
{
  "/api/members/**":{
    "id": "member-api",
    "fullPath": "/api/members/**",
    "location": "member-api",
    "path": "/api/members/**",
    "retryable": false,
    "customSensitiveHeaders": false,
    "prefixStripped": false
  }
}
```

### POST

`POST /application/routes` 요청으로 route의 속성을 변경할 수 있다

변경 못하게 하려면 `endpoints.routes.enabled=false`로 설정한다

# URL 패턴과 Local Forward

```
 zuul:
  routes:
    first:
      path: /first/**
      url: http://first.example.com
    second:
      path: /second/**
      url: forward:/second
    third:
      path: /third/**
      url: forward:/3rd
    legacy:
      path: /**
      url: http://legacy.example.com
```

- `/first/**` : 외부로 라우팅
- `/second/**` : 내부(local)에서 처리됨(spring `@RequestMapping`)
- `/third/**` : `/second/**`와 마찬가지로 내부에서 처리되나 prefix가 변경됨(`/third/foo` -> `/3rd/foo`)
- 그 외 : `legacy`로 처리

# Hystrix Fallback

`FallbackProvier` 타입의 spring bean을 등록하여, 특정 route의 fallback을 실행시킬 수 있다
route의 id를 명시해야하며, fallback에 대한 반환형으로 `ClientHttpResponse`를 써야한다.

**예제**

```java
class MyFallbackProvider implements FallbackProvider {

    // 모든 route에 대해 기본 fallback을 지정하고 싶으면, "*"이나 null을 반환
    @Override
    public String getRoute() {
        return "customers";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, final Throwable cause) {
        if (cause instanceof HystrixTimeoutException) {
            return response(HttpStatus.GATEWAY_TIMEOUT);
        } else {
            return response(HttpStatus.INTERNAL_SERVER_ERROR);
        }
    }

    private ClientHttpResponse response(final HttpStatus status) {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return status;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return status.value();
            }

            @Override
            public String getStatusText() throws IOException {
                return status.getReasonPhrase();
            }

            @Override
            public void close() {
            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("fallback".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}
```

# Zuul Filter

![image](https://camo.githubusercontent.com/4eb7754152028cdebd5c09d1c6f5acc7683f0094/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d726571756573742d6c6966656379636c652e706e67)

## Pre Filter

주로 backend에 보내줄 정보를 `RequestContext`에 담는 역할

**Payco의 AccessToken으로 email을 넘겨주는 경우**

```java
public class QueryParamPreFilter extends ZuulFilter {
  @Override
  public int filterOrder() {
    return PRE_DECORATION_FILTER_ORDER - 1; // run before PreDecoration
  }

  @Override
  public String filterType() {
    return PRE_TYPE;
  }

  @Override
  public boolean shouldFilter() {
    RequestContext context = RequestContext.getCurrentContext();
    return "member-api".equals(context.get(SERVICE_ID_KEY));
  }
    
  @Override
  public Object run() {
    RequestContext context = RequestContext.getCurrentContext();
    HttpServletRequest request = context.getRequest();
    String email = paycoTokenToEmail(request);
    context.addZuulRequestHeader("X-PAYCO-EMAIL", email);
    return null;
  }
}
```

> email은 소중한 개인 정보입니다. 다루실 때 주의하시기 바랍니다.

## Route Filter

pre filter 이후에 실행되며, 다른 서비스로 보낼 요청을 작성한다

이 필터는 주로 request, response를 client가 요구하는 모델로 변환하는 작업을 수행한다

아래의 예제는 Servlet Request를 OkHttp3 Request로 변환하고, 요청을 실행하고,

OkHttp3 Response를 Servlet Response로 변환하는 작업을 수행한다

```java
public class OkHttpRoutingFilter extends ZuulFilter {
    @Autowired
    private ProxyRequestHelper helper;

    @Override
    public String filterType() {
        return ROUTE_TYPE;
    }

    @Override
    public int filterOrder() {
        return SIMPLE_HOST_ROUTING_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().getRouteHost() != null
               && RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() {
        OkHttpClient httpClient = new OkHttpClient.Builder()
                // customize
                .build();

        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();

        String method = request.getMethod();

        String uri = this.helper.buildZuulRequestURI(request);

        Headers.Builder headers = new Headers.Builder();
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String name = headerNames.nextElement();
            Enumeration<String> values = request.getHeaders(name);

            while (values.hasMoreElements()) {
                String value = values.nextElement();
                headers.add(name, value);
            }
        }

        InputStream inputStream = request.getInputStream();

        RequestBody requestBody = null;
        if (inputStream != null && HttpMethod.permitsRequestBody(method)) {
            MediaType mediaType = null;
            if (headers.get("Content-Type") != null) {
                mediaType = MediaType.parse(headers.get("Content-Type"));
            }
            requestBody = RequestBody.create(mediaType, StreamUtils.copyToByteArray(inputStream));
        }

        Request.Builder builder = new Request.Builder()
                .headers(headers.build())
                .url(uri)
                .method(method, requestBody);

        Response response = httpClient.newCall(builder.build()).execute();

        LinkedMultiValueMap<String, String> responseHeaders = new LinkedMultiValueMap<>();

        for (Map.Entry<String, List<String>> entry : response.headers().toMultimap().entrySet()) {
            responseHeaders.put(entry.getKey(), entry.getValue());
        }

        this.helper.setResponse(response.code(), response.body().byteStream(),
                                responseHeaders);
        context.setRouteHost(null); // prevent SimpleHostRoutingFilter from running
        return null;
    }
}
```

## Post Filter

Response를 생성하는 작업을 처리한다

아래 예제는 `X-Sample` 헤더에 임의의 `UUID`를 넣는 소스이다

```java
public class AddResponseHeaderFilter extends ZuulFilter {
	@Override
	public String filterType() {
		return POST_TYPE;
	}

	@Override
	public int filterOrder() {
		return SEND_RESPONSE_FILTER_ORDER - 1;
	}

	@Override
	public boolean shouldFilter() {
		return true;
	}

	@Override
	public Object run() {
		RequestContext context = RequestContext.getCurrentContext();
    	HttpServletResponse servletResponse = context.getResponse();
		servletResponse.addHeader("X-Sample", UUID.randomUUID().toString());
		return null;
	}
}

```

# 마무리

Zuul은 내부에서 Ribbon, Hystrix 등을 사용하기 때문에, 이 자체로도 LoadBalancer로 볼 수도 있고, API Gateway라고 볼수도 있다
Netflix에서는 모든 요청에 대한 Front door라고 부른다

 주변에서 api-gateway로 각광받고 있는 Kong과 비교해달라는 요청이 있었다.
 
 사실상, Kong이 할 수 있는 일과 Zuul이 할 수 있는 일에 대해서는 거의 차이점이 없다.
 Java를 잘 다룰 줄 안다면, Zuul로 세세한 부분까지 커스터마이징해서 사용할 수 있고,
 Kong은 자체로 제공해주는 plugin들로 필요한 부분들을 쉽게 설정할 수 있다
 
`사용상의 편의`라는 점을 제외한다면, Zuul과 Kong의 비교가 아니라 Zuul과 nginx의 비교가 된다
tomcat에서 Zuul을 올려 사용한다면 thread pool 기반으로 동작할 것이고
Kong을 사용한다면 event driven 방식으로 동작할 것이다

성능상에서(`중요한 부분을 거두절미하고 말하자면 Non-Blocking은 좋은 것이다!`)
저사양 서버에서는 nginx를 사용하는 Kong이 더 좋은 성능을 보이고,
자원이 충분한 서버에서는 둘 간의 차이는 거의 없다
AWS Instance 기반의 벤치마킹 : https://engineering.opsgenie.com/comparing-api-gateway-performances-nginx-vs-zuul-vs-spring-cloud-gateway-vs-linkerd-b2cc59c65369

고무적인 것은 Zuul도 2.0 버전에서 Netty 기반의 Non-bloking을 지원할 것이라는 점이다