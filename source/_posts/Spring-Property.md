---
title: Spring Property 관리
date: 2018-03-11 16:51:00
tags: [spring, spring-boot, reference]
categories:
  - spring
  - reference
---

# Spring boot enviroment abstraction

```java
@Component
public class MyBean {

    @Value("${name}")
    private String name;
}
```

classpath 내의 application.properties(yml)을 사용하여 어플리케이션에서 사용할 프로퍼티를 정의할 수 있다.
<!-- more -->

## application.properties
Srping boot enviroment abstraction은 다음 순서로 application.properties를 검색한다

1. 현재 위치에서 /config 디렉토리
2. 현재 위치
3. classpath 내의 /config 패키지
4. classpath 내의 루트

> properties 파일을 대체하여 YAML file(.yml)을 사용할 수 있다.

application.properties라는 이름 외의 다른 이름을 사용하고자 한다면 JVM 환경을 설정하여 변경할 수 있다.

```shell
$ java -jar myproject.jar --spring.config.name=myproject

$ java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```

## Profile-specific properties

`application-{profile}.properties(.yml)`의 네이밍 컨벤션을 가진 properties 파일로 profile별로 사용할 수 있다.

복수의 profile이 적용이 된다면 last wins 전략이 사용된다.

`spring.config.location`이 사용된 경우 `profile-specific properties` 전략은 무시된다.

## Placeholder in properties

application.properties 내의 값은 이전의 정의된 값을 참조할 수 있다.

```properties
app.name=MyApp
app.description=${app.name} is a Spring Boot application
```

# .properties

```
# spring
spring.server.port=8080
spring.server.domain=www.example.com
 
# jdbc
jdbc.url=jdbc:mysql://0.0.0.1:3306/example?a=b&c=d
jdbc.username=root
jdbc.password=root
 
# list
list[0]=a
list[1]=b
list[2]=c
```

## 장점

- escape문자열 필요 없다.
- 선언부가 필요 없다.
- 단순하다.

## 단점
- 가독성이 떨어질 수 있다.
- 키-값 1:1 매칭

# .xml

```xml
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
  <!-- srping -->
  <entry key="spring.server.port">8080</entry>
  <entry key="spring.server.domain">www.example.com</entry>
  
  <!-- jdbc -->
  <entry key="jdbc.url">jdbc:mysql://0.0.0.1:3306/example?a=b&amp;c=d</entry>
  <entry key="jdbc.username">root</entry>
  <entry key="jdbc.password">root</entry>
 
  <!-- list -->
  <entry key="color">
    <color key="r">0</color>
    <color key="g">100</color>
    <color key="b">0</color>
  </entry>
</properties>
```

## 장점

- 1:N 형식 매핑이 가능하다.

## 단점

- 선언부(`<!DOCTYPE...>`)가 존재한다.
- 쳐야하는 문자 수가 많다(혹은 반복되는 ctrl+c, ctrl+v, 심지어 주석 처리에도...)
- 가독성(escape문자)이 떨어진다.

# .yml

```yml
# spring
spring:
  server:
    port: 8080
    domain: www.example.com
 
# jdbc
jdbc:
  url: jdbc:mysql://0.0.0.1:3306/example?a=b&c=d
  username: root
  password: root
 
# list
list:
  - apple
  - banana
  - orange
 
# profiles
---
spring:
    profiles: develop
jdbc:
    username: dev1
---
spring:
    profiles: release
jdbc:
    username: release1
```

## 장점

- escape 문자열 필요 없다
- 1:N 형식 매핑이 가능하다
- 선언부가 없다.
- profile과 관련해서 spring의 지원을 받을 수 있다.

## 단점

- 생소함?(학습비용?)

## 참조

https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-yaml

# Spring Profile

**jvm 옵션, spring boot 실행 명령에 추가**

```
spring.profiles.active=dev(local,dev,test 등)
```

**xml bean 설정**

```xml
<beans profile="local">
  <bean id="aesCrypto">
    <property name="privateKeyPath">/a/a.cert</property>
    <property name="publicKeyPath">/a/b.cert</property>
  </bean>
</beans>
<beans profile="dev">
  <bean id="aesCrypto">
    <property name="privateKeyPath">/c/c.cert</property>
    <property name="publicKeyPath">/c/d.cert</property>
  </bean>
</beasn>
```