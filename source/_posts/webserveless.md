---
title: HAProxy를 이용한 WebServer-less 구조
date: 2018-03-10 03:14:40
tags: [haproxy,spring-boot,web-server,architecture]
categories:
  - architecture
---

# 개요

L4의 scale-out이 어렵다는 점을 해결하기 위해, HAProxy를 사용하게 된 후, 과연 web server가 필요할까라는 의문이 들었다. `apache http + tomcat`라는 기술 스택은 마치 표준인양 Java 서버 진영에서는 많이 사용되고 있는데, HAProxy로 L4를 대체해버린 경우에도 정말 apache http가 필요한 걸까? 한 번 따져볼 일이다.

<!-- more -->

# L4 + Apache Http + Tomcat VS HAProxy + Tomcat

HAProxy로 L4/L7 switch를 대체하는 것에 대한 내용은 [Naver의 기술 블로그(L4/L7 스위치의 대안, 오픈 소스 로드 밸런서 HAProxy)](http://d2.naver.com/helloworld/284659)를 참고하자

다음은 각 구성을 다이어그램으로 표현한 것이다.

**전통적인 L4 - Server 구성**

{% plantuml %}

package "apache + tomcat 서버" {
  [apache + tomcat 1]
  [apache + tomcat 2]
}

[client] --> [L4]
[L4] --> [apache + tomcat 1]
[L4] --> [apache + tomcat 2]

{% endplantuml %}

**GSLB - HAProxy - Server 구성**

{% plantuml %}
package "apache + tomcat 서버" {
  [apache + tomcat 1]
  [apache + tomcat 2]
}
[client] --> [gslb]
[gslb] --> [haproxy1]
[gslb] --> [haproxy2]
[haproxy1] --> [apache + tomcat 1]
[haproxy1] --> [apache + tomcat 2]
[haproxy2] --> [apache + tomcat 1]
[haproxy2] --> [apache + tomcat 2]
{% endplantuml %}

## Apache Http + Tomcat 스택을 쓰는 이유

우선 왜 `apache http + tomcat`을 사용할까?
개인적으로 생각하는 `apache http + tomcat`을 쓰는 이유에 대해서 꼽아봤다.

- 정적 컨텐츠 제공
  - apache http는 빠른 성능의 정적 컨텐츠 제공이 강점이다.
  - apache http로 정적 컨텐츠를 제공하고, tomcat으로 동적 service logic을 실행한다.
- 보안
  - 여러 조건을 가지고 ACL을 걸 수 있다.
  - SSL Offloading으로 tomcat에서는 http로만 잘 동작하도록 해도 문제없다.
- Apache 모듈 지원
  - 많은 기능을 모듈화하여 필요에 따라 사용할 수 있다.
- 레퍼런스
  - `apache http + tomcat`은 사용자가 많다.
  - 많은 실패사례, 성공사례, 예제와 설명을 접할 수 있다.

적어놓고 보니 안 쓸 이유가 없어보인다.
사실 이 외에도 많은 장점이 있다.(가상호스트, 요청 편집, 프록시 등)
HAProxy를 사용한다는 가정하에 몇 가지 태클을 걸어보자.

> 만약 위의 기능이 필요하다면 `apache http + tomcat`을 쓰자

### 정적 컨텐츠 제공

과연 정적인 컨텐츠가 서버마다 있어야 할까?
`/static/a.png` 파일이 A서버에 요청할 때와, B서버에 요청할 때 다른 결과가 나와야할 컨텐츠가 있을까?
그렇지 않다면 차라리 가용성을 높게 구성한 CDN에서 관리하는게 좋다고 생각한다.

### 보안

SSL Offloading, URI, Header 등, L4/L7을 대체하는 HAProxy에서 ACL체크를 할 수 있다.

### Apache 모듈 지원

HAProxy의 주된 기능은 당연히 Proxy이다. 만약 Caching이 필요하다거나, Apache의 모듈에서만 제공하는 기능이 반드시 필요하다면 Apache를 써야한다. 하지만 인프라 구성이 HAProxy로도 충분히 해낼 수 있다면 Apache를 사용해야할 이유가 희미해진다.

### 성능

{% blockquote Tomcat Reference https://tomcat.apache.org/tomcat-8.5-doc/connectors.html Connectors How To %}
단일 서버를 사용한다면 native web server를 tomcat 앞에 두는 것은 대부분의 경우, HTTP connector로 standalone으로 동작하는 것보다 `현저하게 낮은 성능(significantly worse)`을 보인다. 이는 web application의 대부분이 정적 파일로 구성된 경우에도 마찬가지다.
{% endblockquote %}

물론 `apache http + tomcat`으로 구성할 때에는, 둘 사이의 통신을 ajp 프로토콜을 사용하여 빠른 성능을 내려고 노력하지만 **필연적으로 tomcat standalone보다는 느릴 수 밖에 없다**

### 결론

Apache http에 의존적인 서비스를 해야하는 가에 대해 확인할 필요가 있다.
꼭 Apache http를 써야할 이유가 없고, 성능이나 비용 절감의 요구가 더 크다면 과감히 WebServer-less로 전환해보는 건 어떨까?

# 기존에 사용하던 Apache의 기능을 HAProxy로 Porting 할 수 있을까?

## error 처리

**apache**

```
ErrorDocument 400 /error/invalidRequest.html 
```

**HAProxy**

```
errorfile 404 /etc/haproxy/errorfiles/404.html
```

## 특정 요청 오류 응답

**apache**

```
Redirect 404 /favicon.ico
```

**HAProxy**

```
acl favicon_request path /favicon.ico
http-request deny 404 if favicon_request
```

## 특정 경로 ACL(403)

**apache**

```
<DirectoryMatch "(^|/)META-INF($|/)">
  Order deny,allow
  deny from all
</DirectoryMatch>
```

**HAProxy**

```
acl meta_inf_request path_reg (^|/)META-INF($|/)
http-request deny if meta_inf_request # http-request allow if !meta_inf_request
```


## 응답 압축

**apache**

```
AddOutputFilterByType DEFLATE text/plain text/html text/xml
DeflateCompressionLevel 9
BrowserMatch ^Mozilla/4 gzip-only-text/html
```

**HAProxy**

```
tune.zlib.memlevel 9
compression algo gzip
comporession type text/html text/pain text/xml
```

## Request 재작성(mod_rewrite)

**www를 강제로 붙이기**

```
http-request redirect code 301 \
location http://www.%[hdr(host)]%[capture.req.uri] \
unless { hdr_beg(host) -i www }
```

## VirtualHost

**로드밸런서(GSLB)에 등록된 ip 기반으로 나누는 방식(권장)**

```
frontend a_domain
  bind 133.186.189.5:80
  bind 133.186.189.5:443 ssl crt /etc/haproxy/ssl/cert.pem
frontend b_domain
  bind 133.186.189.6:80
  bind 133.186.189.6:443 ssl crt /etc/haproxy/ssl/cert.pem
```
 
**하나의 IP로 받고 Domain name을 기준으로 front에서 나누는 방식**

```
frontend http_proxy
  bind :80
  bind :443 ssl crt /etc/haproxy/ssl/coexapi/cert.pem
 
  acl is_a hdr_beg(host) a.com
  acl is_b hdr_beg(host) b.com
  http-request deny 404 if !a and !b
  use_backend a_backend if a
  use_backend b_backend if b
```

## X-Forwarded-For

```
frontend www
    mode http
    option forwardfor except 127.0.0.1

backend www
    mode http
    option forwardfor
    # option forwardfor header X-Client : X-Client 이름의 헤더로 forwardfor 값을 넣음
```

# 마무리

관례는 좋다고 생각한다. 레퍼런스가 많고, 배우기 쉬우며, 많은 사람들이 그 기술을 사용할 때에는 다 이유가 있기 때문에 쓴다. 하지만 레퍼런스 케이스가 모든 경우에 대입할 수 있는 것은 아니다. 새로운 요구 사항들이 생기고, 새로운 기술들을 익히는 과정에서 한 번 쯤은 내가 쓰고 있던 기술들을 돌이켜 보고 더 나은 구조는 없을까 고민해보는 것도 좋은 경험인 것 같다.

한 가지 분명히 밝혀둬야할 것은 HAProxy는 Apache의 대체 기술이 아니다. Apache는 server이고, HAProxy는 proxy이다. 각자 자신만이 할 수 있는 영역이 있고, 두 기술 모두 처리할 수 있는 영역이 있다. 여기서는 교집합이 되는 영역에 대해서 이야기하고 싶었다.

본문에서는 전체 주제를 유지하기 위해 Apache의 기능에만 많은 초점을 맞추었지만, HAProxy 또한 각 node끼리 P2P로 통신하여 전체 트래픽을 제어하거나, sticky session을 구성하거나, 임계치를 걸 수 있는 등, Apache에서는 하지 못할 일들을 할 수 있다. 그러니 한 번 쯤은 HAProxy 레퍼런스를 훑어보는 것도 좋다.

HAProxy는 L4/L7에서 요청을 다룰 수 있는 기능을 제공한다. HAProxy를 사용해서 L4/L7을 대체하는 구성을 사용하게 된다면, 그동안 당연히 사용해왔던 apache http에 대해서 다시 한 번 돌아볼 필요가 있다고 생각한다.