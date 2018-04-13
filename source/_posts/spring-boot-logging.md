---
title: (Spring Boot)Logging과 Profile 전략
date: 2018-04-07 01:45:18
tags: [spring, spring-boot, log, logging, logback, profile]
categories:
  - [spring, boot]
---

# 서론

`spring-boot`에서는 Logback을 확장해서 편리한 logging 설정을 할 수 있다. `application.yml`의 설정을 읽어 오거나 특정 profile의 동작을 수행할 수 있다. 이 글에서는 Spring Boot에서 어떻게 logging 설정을 하는지를 알아볼 것이다. 그리고 그것을 기반으로 `spring-boot`를 사용할 때 profile을 활용할 수 있는 방법을 공유하려고 한다.

> 본문에서는 logback의 기초적인 정보를 가이드하지 않는다.
> `spring-boot`는 logback 외에도 log4j 등을 지원하지만 logback만을 고려하고 작성한다.

<!-- more -->

# Logback 확장(Extentions)

`spring-boot`에서는 [Logback](https://logback.qos.ch)을 사용한다. 기존의 logback을 사용해본 사람은 알 것이다. Web Application을 가동하면 classpath 내에서 환경 설정 파일(`logback.xml(logback-text.xml)`)을 검색해서 logback을 초기화시킨다. 이때는 아직 Spring이 구동되기 전의 시점이다. `spring-boot`에서는 `logback-spring.xml`을 사용해서 Spring이 logback을 구동할 수 있도록 지원해준다. 덕분에 profile이나 `application.xml`에 설정된 properties를 읽어올 수 있다.

우선은 logger를 등록하는 방법을 알아보자.

## Logger 등록과 level 설정

기존에 logback에서 logger를 등록하던 방식을 알아보자.
우선 아래와 같이 소스 상에서 로그를 남긴다.

```java
@Slf4j
@Component
public class SomeService {
    public void log() {
        log.info("hello");
    }
}
```

`logback.xml`에서 해당 로그가 남을 수 있도록 설정을 한다.

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.parfait.study.simplelogging.service.SomeService" level="INFO"/>

    <root level="warn">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

위 설정에서는 전역으로 log level이 warn 이상인 로그만 남기고 있지만, `SomeService`의 로그는 info 이상이면 남기도록 했다.

```
02:15:04.557 [restartedMain] INFO  c.p.s.s.service.SomeService - hello
```

`spring-boot`를 사용한다면 이제 `logback.xml`은 필요 없다.

**application.yml**

```yml
logging:
  level:
    root: warn
    com.parfait.study.simplelogging.service.SomeService: info
```

보다시피 `application.yml`의 설정만으로 logger와 level을 설정할 수 있다.

## Logging 관련 Spring Boot Properties

`spring-boot`에서는 `application.yml`에 정의한 properties로도 logging이 동작하도록 지원해준다. 그중에서 활용도가 있어 보이는 몇 개를 골라보았다.

| key | default | description<td colspan=3> |
| - | - | - |
| logging.file | - <td colspan=4> 절대 경로로 표현되거나 현재 경로의 상대 경로로 로그 파일명을 지정한다. |
| logging.file.path | - <td colspan=4> `logging.file`의 값이 없을 때 동작한다. 지정된 경로에 `spring.log`로 로그를 남긴다. |
| logging.file.max-size | 10MB <td colspan=4> 로그 파일의 사이즈가 지정된 임계치를 초과하면 파일명에 index를 추가한 후 새로운 파일을 작성한다.<br>예 : spring1.log, spring2.log |
| logging.file.max-history | 0 <td colspan=4> 아래에서 따로 설명한다. |
| logging.level.* | - <td colspan=4> path 기반으로 logger의 level을 지정한다. |

**logging.file.max-history 부가 설명**

File Appender는 `RollingFileAppender`를 사용한다. 하지만 `spring-boot 1.5.x`까지만 해도 file을 rolling하는 정책이 `SizeBasedTriggeringPolicy`으로 파일 크기가 특정 값에 도달하면 새로운 log file을 남기는 기능만 지원했었다. 때문에 시간이 많이 경과한 log를 지우기 위해서는 서버에서 따로 crond를 돌리거나 별도의 `logback-spring.xml` 설정을 했어야 했다. 하지만 `spring-boot 2.0.0`부터는 rolling 정책이 `SizeAndTimeBasedRollingPolicy`로 변경되었다. 즉 기본 설정으로 로그가 일별로 남는다.(예 : spring.2018-01-01.log) 그리고 `logging.file.max-history`에 지정한 일수가 지난 로그를 자동으로 삭제해준다.

## 기본 설정 파헤치기

그렇다면 `spring-boot`는 어떻게 저런 일들을 지원하는 것일까? spring에 있는 기본 설정을 한 번 살펴보자.

**base.xml**

```xml
<included>
	<include resource="org/springframework/boot/logging/logback/defaults.xml" />
	<property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/spring.log}"/>
	<include resource="org/springframework/boot/logging/logback/console-appender.xml" />
	<include resource="org/springframework/boot/logging/logback/file-appender.xml" />
	<root level="INFO">
		<appender-ref ref="CONSOLE" />
		<appender-ref ref="FILE" />
	</root>
</included>
```

- `<include resource=".../defaults.xml">` : `defaults.xml`에서 추가로 설정을 읽어온다. Console Log Pattern과 File Log Pattern의 기본값이 정의되어 있고, 그 외의 `tomcat`이나 `hibernate` 등의 모듈의 log level 설정이 되어 있다. 궁금하면 따로 찾아보자.
- `<property name="LOG_FILE" ...>` : `${LOG_FILE}`이 없으면 `${LOG_PATH}`를 부른다. `${LOG_PATH}`가 없으면 `${LOG_TEMP}`를 부른다. 이러한 형식으로 `application.yml`의 property를 불러오고 있다는 것을 파악할 수 있다.
- `<include resource=".../console-appender.xml">` : Console Appender 설정을 읽어온다. 아래에서 살펴보자.
- `<include resource=".../file-appender.xml">` : File Appender 설정을 읽어온다. 아래에서 살펴보자.

**console-appender.xml**

```xml
<included>
	<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>${CONSOLE_LOG_PATTERN}</pattern>
		</encoder>
	</appender>
</included>
```

**file-appender.xml**

```xml
<included>
	<appender name="FILE"
		class="ch.qos.logback.core.rolling.RollingFileAppender">
		<encoder>
			<pattern>${FILE_LOG_PATTERN}</pattern>
		</encoder>
		<file>${LOG_FILE}</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
			<fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
			<maxFileSize>${LOG_FILE_MAX_SIZE:-10MB}</maxFileSize>
			<maxHistory>${LOG_FILE_MAX_HISTORY:-0}</maxHistory>
		</rollingPolicy>
	</appender>
</included>
```

- `rollingPolicy` : 앞에서도 설명했지만 `SizeAndTimeBasedRollingPolicy`로 정책이 변경되면서 날짜별로 로그를 남기고, `maxHistory`에서 지정한 일수가 지나면 자동으로 삭제한다. 만약 `2.0.x`보다 옛 버전을 사용하고 있다면 차라리 위 설정을 복사해서 쓰자.

## application.yml의 property 가져오기

애플리케이션에서 원격의 로그 서버로 로그를 보내고, 로그 서버의 주소를 `application.yml`에서 `logserver.host: http://logserver.com`으로 정의했다고 하자. 이때 아래와 같이 `application.yml`을 참조할 수 있다.

```xml
<configuration>
  
  <springProperty name="host" source="logserver.host" defaultValue="http://dev-logserver.com"/>
  
  <appender name="REMOTE_LOG_SERVER" class="xxx.yyy.RemoteLogAppender">
    <remoteHost>${host}</remoteHost>
  </appender>

  <logger name="com.parfait.study.simplelogging.service.SomeService" level="INFO"/>

  <root level="warn">
    <appender-ref ref="REMOTE_LOG_SERVER"/>
  </root>
</configuration>
```

## Profile별 환경 설정

`spring.profiles.active`에서 지정한 profile에 따라 조건들을 추가할 수 있다.

```xml
<springProfile name="dev">
  <appender name="CONSOLE" class="..."></appender>
  <root>
    <appender-ref ref="CONSOLE"/>
  </root>
</springProfile>

<springProfile name="alpha">
	<appender name="FILE" class="..."></appender>
  <root>
    <appender-ref ref="FILE"/>
  </root>
</springProfile>

<springProfile name="staging">
  <!-- staging profile인 경우 -->
</springProfile>

<springProfile name="!production">
  <!-- production profile이 아닌 경우 -->
</springProfile>
```

위 예제는 [Spring Reference에 있는 예제](https://docs.spring.io/spring-boot/docs/2.0.1.RELEASE/reference/htmlsingle/#_profile_specific_configuration)를 약간 수정한 것이다. 이 예제를 보고 아쉬움이 생긴다. `OOP`스럽지 않다는 것이다. `OOP`스러운 것은 무엇일까? 작은 문제들로 나누고 이를 조합해서 큰 문제를 해결하는 방식이다! 어떻게 profile로 그러한 사용을 할지 알아보자.

# Profile 전략

앞서 보았던 profile은 애플리케이션 배포의 주기를 그대로 따르고 있다.

- dev : 개발 단계. 여기서는 CONSOLE로 로그를 남긴다.
- alpha : 알파 테스트 단계. 여기서부터 서버에 배포하므로 FILE로 로그를 남긴다.

언뜻 보기에는 별다른 문제가 없을 것 같다. 여기에 아까 만들었던 원격 로그 서버로 로그를 내보내는 Appender를 추가해보자. beta profile에서는 FILE과 REMOTE_LOG_SERVER Appender를 사용한다.

```xml
<springProfile name="beta">
  <appender name="FILE" class="..."></appender>
  <appender name="REMOTE_LOG_SERVER" class="..."></appender>
  <root>
    <appender-ref ref="FILE"/>
    <appender-ref ref="REMOTE_LOG_SERVER"/>
  </root>
</springProfile>
```

문제가 발생했다. 중복이 생긴 것이다. alpha에서도 FILE Appender를 선언하고 beta에서도 FILE Appender를 선언했다. 어떻게 해야 할까?

```xml
<springProfile name="alpha,beta">
  <appender name="FILE" class="..."></appender>
  <root>
    <appender-ref ref="FILE"/>
  </root>
</springProfile>
<springProfile name="beta">
  <appender name="REMOTE_LOG_SERVER" class="..."></appender>
  <root>
    <appender-ref ref="REMOTE_LOG_SERVER"/>
  </root>
</springProfile>
```

위와 예제가 실행되는지는 모르겠지만, 여하튼 저렇게 해서는 안될 것 같다.

필자가 제시하고 싶은 해결책은 profile을 `조합`해서 사용하자는 것이다.

```xml
<springProfile name="console-logging">
  <appender name="CONSOLE" class="..."></appender>
</springProfile>

<springProfile name="file-logging">
	<appender name="FILE" class="..."></appender>
</springProfile>

<springProfile name="remote-logging">
  <appender name="REMOTE_LOG_SERVER" class="..."></appender>
</springProfile>

<root>
  <springProfile name="console-logging">
    <appender-ref ref="CONSOLE"/>
  </springProfile>
  
  <springProfile name="file-logging">
    <appender-ref ref="FILE"/>
  </springProfile>
  
  <springProfile name="remote-logging">
    <appender-ref ref="REMOTE_LOG_SERVER"/>
  </springProfile>
</root>
```

조합을 사용하면 훨씬 유리하게 환경을 구성할 수 있다. 배포 환경과 맞춰서 설정을 해보자.

**배포 환경에 따른 `spring.profiles.active`**

- dev : console-logging
- alpha : file-logging
- beta : file-logging,remote-logging

조합을 해서 원하는 환경을 쉽게 구성할 수 있다는 건 알겠는데 또 다른 문제가 있다. beta에서 모든 것을 다 조합해서 쓰기에는 profile을 입력하는 게 꽤 눈이 아프다.

```
$ java -Dspring.profiles.active=file-logging,remote-logging -jar simple-logging-0.0.1-SNAPSHOT.jar
```

이런 문제를 해결하기 위해 `spring.profiles.include` 설정을 걸어둘 수 있다.

**application-dev.yml**

```yml
spring.profiles.include: console-logging
```

**application-alpha.yml**

```yml
spring.profiles.include: file-logging
```

**application-beta.yml**

```yml
spring.profiles.include: file-logging,remote-logging
```

배포 단계에 맞춰 필요한 조합을 미리 구성해두고 쓰자.

이러한 조합은 비단 로깅에서만 쓸 것이 아니다. 업무를 진행하다 보면 배포 환경에 유연성이 필요함을 느낄 때가 있다. 한 마일스톤 내에 두 개의 큰 기능을 배포해야 할 경우가 있다. 예를 들어 메일링 기능 추가와 SMS 기능 추가라고 하자. QA를 진행하다가 한 쪽이 통과하지 못하면 해당 기능을 빼고 배포를 하기 위한 준비가 되어 있고, 각각 별도의 서버(alpha1, alpha2)에서 QA를 진행한다. alpha1에서는 메일링 테스트를 하는데 메일 서버는 운영 환경을 바라보도록 하고, alpha2에서 진행되는 SMS 또한 SMS 서비스만 실제 운영 환경을 바라보도록 해야 한다. 이러한 요구 사항들은 얼마든지 있을 수 있으며, 이에 대한 준비 또한 필요하다.

# 마무리

사실 Profile 전략을 공유하기 위해서 `spring-boot`에서 지원하는 로깅을 소개했다. 로깅 지원은 Spring Reference만 읽어봐도 어떤 일을 해주는지 알 수 있다. Spring Boot는 여러 자동 설정으로 우리를 간편하게 해준다. 이번에는 그 설정의 간편화로 인해 얻을 수 있는 큰 이점에 대해서 이야기해봤다. Profile을 배포 단계와 동일시하는 경향이 있다. 실제로 Spring Reference조차 profile을 그런식으로 설명하는 부분이 많으니 어쩔 수 없다. 하지만 `spring-boot`에서 `spring.profiles.include`가 생기면서 Profile을 바라보는 시각을 조금 바꿀 수 있게 된 것 같다. 우리는 이제 애플리케이션 설정을 마치 플러그인 다루듯이 할 수 있게 된 것이다.

이제 `spring-boot`를 사용할 때 자바 설정상으로는 배포 단계를 나타내지 말자. `application-{배포단계}.yml`을 작성하고 필요한 설정을 조합하여 사용하자.