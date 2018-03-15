---
title: HAProxy, Spring-boot를 이용한 WebServer-less 구조
date: 2018-03-10 03:14:40
tags: [haproxy,spring-boot,web-server,architecture]
categories:
  - architecture
---

# 개요

static resouce는 CDN에서 처리하고,
동적인 부분은 WAS(Web application server)로 처리하는 경우,
Web Server(Apache, nginx 등)이 과연 필요할까라는 의문에서,
Web Server 없이 서비스할 수 있지 않을까해서 조사해보았다

<!-- more -->

> 하나의 서버에서 Tomcat 앞에 webserver를 사용하는 것은, 대부분의 경우 기본 HTTP connector로 standalone으로 동작하는 것보다 현저히 낮은 성능을 보이며, 이는 web application의 대부분이 static file로 구성되었어도 마찬가지이다
> -Tomcat 문서 내용

**장점**
* 관리 포인트가 줄어듬
    * connection, socket timeout을 계산하기가 조금 더 수월하다
    * apache 혹은 nginx 등의 학습비용을 줄일 수 있다
    * 서버 증설시 해야할 일이 줄어든다(question)
* 성능향상
    * webserver에 사용되는 시스템리소스를 web application server에 줄 수 있다
    * single server에서 tomcat을 standalone으로 사용하는 것이, 앞단에 web server를 두는 것보다 훨씬 빠르다(고 한다)
* ajp를 사용하는 것보다 http를 사용하는 것이 개발자에게 익숙하다(apache의 경우)

# Embedded Tomcat의 성능에 대한 염려

Spring boot에서 내장하고 있는 Tomcat이 외장 Tomcat에 비해 성능이 빠를까?

## Embedded Tomcat VS Non-embedded Tomcat

- 외장 Tomcat을 사용시 .war로 압축된 소스의 내용을 loading
- 내장 Tomcat은 jar로 압축된 소스를 loading

> 성능 비교는 무의미

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

# 기존에 사용하던 Apache의 기능을 HAProxy로 Porting 할 수 있을까?

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
cl meta_inf_request path_reg (^|/)META-INF($|/)
http-request deny if meta_inf_request
= http-request allow if !meta_inf_request
```

# 기존에 사용하던 Apache의 기능을 HAProxy로 Porting 할 수 있을까?

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

# 기존에 사용하던 Apache의 기능을 HAProxy로 Porting 할 수 있을까?

## Request 재작성(mod_rewrite)

```
http-request {
  allow | tarpit | auth [realm <realm>] | redirect <rule> | deny [deny_status <status>]
  | add-header <name> <fmt> | set-header <name> <fmt> | capture <sample> [ len <length>
  | id <id> ] | del-header <name> | set-nice <nice> | set-log-level <level>
  | replace-header <name> <match-regex> <replace-fmt>
  | replace-value <name> <match-regex> <replace-fmt> | set-method <fmt> | set-path <fmt>
  | set-query <fmt> | set-uri <fmt> | set-tos <tos> | set-mark <mark>
  | add-acl(<file name>) <key fmt> | del-acl(<file name>) <key fmt>
  | del-map(<file name>) <key fmt> | set-map(<file name>) <key fmt> <value fmt>
  | set-var(<var name>) <expr> | unset-var(<var name>)
  | { track-sc0 | track-sc1 | track-sc2} <key> [table <table>]
  | sc-inc-gpc0(<sc-id>) | sc-set-gpt0(<sc-id>) <int> | silent-drop |
}
[ { if | unless } <condition> ]

www를 강제로 붙이는 예제)
http-request redirect code 301 \
location http://www.%[hdr(host)]%[capture.req.uri] \
unless { hdr_beg(host) -i www }
```

# 기존에 사용하던 Apache의 기능을 HAProxy로 Porting 할 수 있을까?

## VirtualHost

**로드밸런서에 등록된 ip 기반으로 나누는 방식(권장)**

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

# 기존에 사용하던 Apache의 기능을 HAProxy로 Porting 할 수 있을까?

## X-Forwarded-For

```
option forwardfor [ except <network> ] [ header <name> ] [ if-none ]

# Public HTTP address also used by stunnel on the same machine
frontend www
    mode http
    option forwardfor except 127.0.0.1  # stunnel already adds the header

# Those servers want the IP Address in X-Client
backend www
    mode http
    option forwardfor header X-Client
```

# 결론

static 자원을 모두 CDN에서 처리를 하며, HAProxy를 모든 서버 앞에 두게 된다면
굳이 apache, nginx와 같은 web server를 사용할 필요가 없어진다.
오히려 사용하므로써 서버 간 구조를 복잡하게 만들고,
서버 리소스를 비효율적으로 사용할 가능성을 높이며,
이는 곧 최적화, 성능 이슈를 야기하는 원인이 될 수 있다.

습관처럼 사용해오던 apache + tomcat을 왜 그렇게 사용했는지 이유를 되짚어보고,
우리의 시스템에 그 이유가 마찬가지로 적용이 되는지를 생각해본다면,
과감히 web server를 버리고 standalone tomcat을 사용하는 것이 좋다고 본다