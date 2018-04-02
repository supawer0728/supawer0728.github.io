---
title: Spring Framework 5.0.4.RELEASE Reference Core (Chapter2)
date: 2018-03-11 02:36:07
tags: [spring, reference]
categories:
  - spring
  - reference
---

# 글에 앞서서

- 본문은 Spring Framework Version 5의 습득을 위한 글이다.
- 이 글을 상업적 목적으로 쓰지 않았다

```
Authors
Rod Johnson , Juergen Hoeller , Keith Donald , Colin Sampaleanu , Rob Harrop , Thomas Risberg , Alef Arendsen , Darren Davison , Dmitriy Kopylenko , Mark Pollack , Thierry Templier , Erwin Vervaet , Portia Tung , Ben Hale , Adrian Colyer , John Lewis , Costin Leau , Mark Fisher , Sam Brannen , Ramnivas Laddad , Arjen Poutsma , Chris Beams , Tareq Abedrabbo , Andy Clement , Dave Syer , Oliver Gierke , Rossen Stoyanchev , Phillip Webb , Rob Winch , Brian Clozel , Stephane Nicoll , Sebastien Deleuze

Copyright © 2004-2016

Copies of this document may be made for your own use and for distribution to others, provided that you do not charge any fee for such copies and further provided that each copy contains this Copyright Notice, whether distributed in print or electronically.
```

# 2. Resources

# 2.1. Introduction

- `java.net.URL`은 low-level 자원에 대한 접근을 모두 처리해내기에는 부족하다
    - `classpath`나 `ServletContext`를 기준으로 자원에 접근하는 방법이 필요하다
<!-- more -->
# 2.2. The Resource interface

## 2.2. The Resource interface

```java
public interface Resource extends InputStreamSource {

    boolean exists();
    boolean isOpen();
    URL getURL() throws IOException;
    File getFile() throws IOException;
    Resource createRelative(String relativePath) throws IOException;
    String getFilename();
    String getDescription();
}

public interface InputStreamSource {
    InputStream getInputStream() throws IOException;
}
```

# 2.3. Built-in Resource implementations

`PropertyEditor`라는 것이 `classpath:xxx`, `https:xxx` 등의 문자열로 아래 `Resource` 구현체 중 하나로 타입을 결정한다

- `UrlResource` : `java.net.URL`의 wrapping. `http`, `ftp` 등의 자원에 접근할 때 사용
- `ClassPathResource` : classpath에 있는 자원을 가져올 때 사용, `classpath:`
- `FileSystemResource`
- `ServletContextResource` : `ServletContext(웹 환경)`의 자원에 접근할 때 사용
- `InputStreamResource`
- `ByteArrayResource`

# 2.4. The ResourceLoader

## 2.4. The ResourceLoader

모든 application context는 `ResourceLoader`를 구현하여, 문자열로 원하는 자원을 획득할 수 있다.

```java
public interface ResourceLoader {
    Resource getResource(String location);
}

public interface ResourcePatternResolver extends ResourceLoader { ... }

public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver { ... }
```

아래 소스에 대해서 어떠한 `Resource`가 반환될까?

```java
Resource template = ctx.getResource("some/resource/path/myTemplate.txt");
```

- `ClassPathXmlApplicationContext` -> `ClassPathResource`
- `FileSystemXmlApplicationContext` -> `FileSystemResource`
- `WebApplicationContext` -> `ServletContextResource`

`ApplicationContext`와 상관없이 원하는 타입의 자원을 얻기 위해서는 `classpath:`등의 prefix를 달면 된다

| Prefix | Example | Explanation |
| - | - | - |
| classpath: | classpath:com/myapp/config.xml | Loaded from the classpath |
| file: | file:///data/config.xml | Loaded as a URL, from the filesystem |
| http: | http://myserver/logo.png | Loaded as a URL |
| (none) | /data/config.xml | Depends on the underlying ApplicationContext |

# 2.7. Application contexts and Resource paths

## 2.7.1. Constructing application contexts

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```
```java
ApplicationContext ctx = new FileSystemXmlApplicationContext("conf/appContext.xml");
```
```java
ApplicationContext ctx = new FileSystemXmlApplicationContext("classpath:conf/appContext.xml");
```

## 2.7.2. Wildcards in application context constructor resource paths

### Ant-style Patterns

```java
/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml
```

### The `classpath*:` prefix

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath*:conf/appContext.xml");
```