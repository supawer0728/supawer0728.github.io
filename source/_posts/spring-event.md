---
title: Spring Event + Async + AOP 적용해보기
date: 2018-03-24 00:43:45
tags: [spring, event, practice]
categories:
  - spring
  - practice
---

# 개요

원래 글을 쓰기 위해 준비하던 내용은 Event를 강조하는 것이었는데, 준비를 하다보니 Async와 AOP를 다 쓰게 되어버렸다.

이번 글에서는 하나의 transaction 안에서 많은 일을 처리하는 소스 코드를 두고, 아래와 같이 점진적으로 리팩토링해나가는 과정에 대해 이야기하려고 한다. 

1. 하나의 transaction으로 처리하기
2. transaction 내에 처리하지 않아도 될 부분은, transaction이 끝난 후에 처리하기
3. Event 사용하기
4. Event도 비동기로 처리하기
5. AOP를 사용해서 Advisor에서 Event 발급하기
6. 순서가 필요있나? Best Effort!

<!-- more -->

본문에서는 `mybatis`를 사용하며, `@Service`에서 대부분의 로직을 처리하는 방식에 대해 다루고 있다. 도메인 주도 설계(Domain-Driven Design)으로 개발을 하는 경우에는 `Domain Event` 개념을 사용할 수 있다. 기회가 된다면 `DDD Start!(최범균 저)`라는 책을 읽어볼 것을 강력히 추천한다. 필자가 알고 있는 DDD의 지식이 대부분 저 책을 기반으로 하다보니, 현재 알고 있는 내용으로 공개 블로그를 작성하기가 어렵다.
`Spring Framework 5.0.4.RELEASE`가 나온 현재까지, 간편하게 `Domain Event`를 사용할 수 있는 방법은 없어 보인다. `ThreadLocal`이나 `Configurable`의 힘을 빌려 자체적으로 구현해야한다. `DDD Start!`를 보고 Spring에서 `Domain Event`를 사용할 것이라면, 작가님이 운영하는 [블로그](http://javacan.tistory.com/entry/Handle-DomainEvent-with-Spring-ApplicationEventPublisher-EventListener-TransactionalEventListener)도 참조하자.

여기서 `Domain Event`에 대해서 다루지는 않겠지만 힌트정도는 얻어갈 수 있다고 생각한다.

# 요구 사항

- 회원이 가입하면 이메일과 SMS로 가입 축하 메시지를 보낸다.
- 이메일과 SMS는 반드시 성공하지 않아도 괜찮다.

# 구현 1. 하나의 transaction으로 처리하기

## 의존성

```gradle
dependencies {
    compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.2')
    compile('org.springframework.boot:spring-boot-starter-aop')
    compile('org.projectlombok:lombok')
    runtime('com.h2database:h2')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

## 회원 관련

### 모델

```java
@Getter
@Setter
public class Member {

    private Long id;
    private String name;
    private String email;
    private String phoneNo;

    public static Member create(@NonNull String name, @NonNull String email, @NonNull String phoneNo) {

        Member member = new Member();
        member.setName(name);
        member.setEmail(email);
        member.setPhoneNo(phoneNo);

        return member;
    }
}
```

### Service

```java
public interface MemberJoinService {
    Member join(Member member);
}
```

**설명**

`MemberJoinService`의 구현체를 여럿 생성하여, Spring의 Profile 별로 구현체를 바꿔서 사용할 것이다.

## MainClass

```java
@Slf4j
@SpringBootApplication
public class SimpleEventApplication implements CommandLineRunner {

    @Autowired
    private MemberJoinService memberJoinService;
    @Autowired
    private MemberMapper memberMapper;

    public static void main(String[] args) {
        SpringApplication.run(SimpleEventApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {

        log.info("{} is injected", memberJoinService.getClass().getCanonicalName());
        try {
            Member member = Member.create("test", "test@test.com", "012-3456-7890");
            memberJoinService.join(member);
        } catch (Exception e) {
            log.error("{} was thrown", e.getClass().getCanonicalName());
        }
        log.info("member count : {}", memberMapper.count());
    }
}
```

## 핵심 로직

```java
@Profile("simple")
@Service
@Transactional
public class SimpleMemberJoinService implements MemberJoinService {

    @Autowired
    private MemberMapper memberMapper;
    @Autowired
    private EmailService emailService;
    @Autowired
    private SmsService smsService;

    public Member join(Member member) {

        memberMapper.insert(member);
        emailService.sendEmail(member.getEmail(), EmailTemplateType.JOIN); // log만 남김
        smsService.sendSms(member.getPhoneNo(), SmsTemplateType.JOIN); // log만 남김, `fail` profile에서는 RuntimeException을 던짐

        return member;
    }
}
```

### 성공, 실패

**성공**

```
INFO  [main] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [main] c.p.s.s.service.sms.SuccessSmsService    : send JOIN sms to 012-3456-7890
INFO  [main] c.p.s.s.SimpleEventApplication           : member count : 1
```

**실패**

```
INFO  [main] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [main] c.p.s.s.service.sms.FailSmsService       : send JOIN sms to 012-3456-7890
ERROR [main] c.p.s.s.SimpleEventApplication           : java.lang.RuntimeException was thrown
INFO  [main] c.p.s.s.SimpleEventApplication           : member count : 0
```

### 설명

Application을 실행시킬 때 `spring.profiles.active=simple`로 해야 정상 동작한다.
요구 사항을 단순하게 하나의 transaction에서 구현했다. 무척이나 간단하지만 `emailService`가 실패하거나, `smsService`가 실패하면 `memberMapper.insert()`도 `rollback`된다.

# 구현 2. transaction 내에 처리하지 않아도 될 부분은, transaction이 끝난 후에 처리하기

두 가지 방법이 있을 것 같다. 하나는 `@Transactional`를 이용하는 것, 하나는 `TransactionSynchronizationAdapter`를 사용하는 것이다. 

## Propagation 사용하기

`Member`를 `insert`할 `Service`를 하나 더 두는 방식이다. 이 경우 `MemberJoinService`에서 `@Transactional` 선언을 지우거나, `MemberService.insert()`에서는 `REQUIRES_NEW`를 해서 `insert()`가 `rollback`되는 것을 막을 수 있다.

{% plantuml %}
[MemberJoinService] --> [MemberService.insert()] : 1
[MemberJoinService] --> [EmailService.send()] : 2
[MemberJoinService] --> [SmsService.send()] : 3
{% endplantuml %}

## TransactionSynchronizationAdapter 사용하기

```java
@Profile("advanced")
@Service
@Transactional
public class AdvancedMemberJoinService implements MemberJoinService {

    @Autowired
    private MemberMapper memberMapper;
    @Autowired
    private EmailService emailService;
    @Autowired
    private SmsService smsService;

    public Member join(Member member) {

        memberMapper.insert(member);
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
                emailService.sendEmail(member.getEmail(), EmailTemplateType.JOIN);
                smsService.sendSms(member.getPhoneNo(), SmsTemplateType.JOIN);
            }
        });

        return member;
    }
}
```

### 성공, 실패

**성공**

```
INFO  [main] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [main] c.p.s.s.service.sms.SuccessSmsService    : send JOIN sms to 012-3456-7890
INFO  [main] c.p.s.s.SimpleEventApplication           : member count : 1
```

**실패**

```
INFO  [main] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [main] c.p.s.s.service.sms.FailSmsService       : send JOIN sms to 012-3456-7890
ERROR [main] c.p.s.s.SimpleEventApplication           : java.lang.RuntimeException was thrown
INFO  [main] c.p.s.s.SimpleEventApplication           : member count : 1
```

`TransactionSynchronizationAdaptor`를 사용하면 위와 같이, `beforeCommit()`, `afterCommit()` 등을 구현해서 특정 로직의 실행 시점을 조절할 수 있다. `TransactionSynchronizationAdaptor`는 이름 그대로 `TransactionSynchronization`를 어댑터 패턴으로 구현해 둔 추상 클래스이다. `TransactionSynchronization`에는 각 로직이 실행될 시점이 추상화되어 있으며, 자세한 설명은 [JavaDoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html)을 확인하자.

`Spring Framework 4.2.x`부터는 더 좋은게 있으므로 길게 설명하지 않겠다.

# 구현 3. Event 사용하기

이 구현부터 본격적으로 본문의 주제에 가까워진다. 우선 소스 코드로 어떻게 Spring Event를 이용할 수 있는 지 확인해보자.

## Spring Application Event 사용하기

### Publisher

```java
@Profile("event")
@Service
@Transactional
public class EventMemberJoinService implements ApplicationEventPublisherAware, MemberJoinService {

    @Autowired
    private MemberMapper memberMapper;
    private ApplicationEventPublisher eventPublisher;

    public Member join(Member member) {

        memberMapper.insert(member);
        eventPublisher.publishEvent(new MemberJoinedEvent(member)); // gray zone

        return member;
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.eventPublisher = applicationEventPublisher;
    }

    public static class MemberJoinedEvent {

        @Getter
        private Member member;

        private MemberJoinedEvent(@NonNull Member member) {
            this.member = member;
        }
    }
}
```

#### 설명

`ApplicationEventPublisherAware` 인터페이스를 구현하면, Spring이 자동으로 `ApplicationEventPublisher`를 주입해준다. Event를 발급하는 방법은 너무나 간단하다. `ApplicationEventPublisher.publishEvent(Object event)`를 호출하면 끝이다.

`MemberService`는 이제 회원이 가입되었으면, 가입되었다는 Event를 발급한다.(`MemberJoinedEvent`) 어디선가 이 Event를 받아서 처리만 해주면 될 것이다. 이 Event를 어떻게 받아서 처리하는지 확인해보자.

> `// gray zone`이라고 붙인 곳은, 저 코드가 저 위치에 있는 것을 허용할 것이냐, 말 것이냐를 개발자들의 판단에 맡기고 싶어서 두었다. 본문에서는 refactoring의 대상이라 판단하고, 나중에 AOP로 제거할 것이다.

### Listener

```java
@Component
public class MemberJoinedEventListener {

    @Autowired
    private SmsService smsService;
    @Autowired
    private EmailService emailService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, classes = EventMemberJoinService.MemberJoinedEvent.class)
    public void handle(EventMemberJoinService.MemberJoinedEvent event) {
        Member member = event.getMember();
        emailService.sendEmail(member.getEmail(), EmailTemplateType.JOIN);
        smsService.sendSms(member.getPhoneNo(), SmsTemplateType.JOIN);
    }
}
```

#### 설명

`@TransactionalEventListener`는 `4.2` 버전부터 사용할 수 있다. 간단하게 위에서 사용한 것과 같이, `phase`에는 실행 시점을, `classes`에는 Event의 타입을 넣어 발급된 Event를 처리할 수 있다.

**TransactionalEventListener**

| parameter | description |
| - | - |
| phase | transaction과 연관지어, 어느 시점에서 로직을 실행할 지 정할 수 있다.<br>- BEFORE_COMMIT<br>- AFTER_COMMIT(default)<br>- AFTER_ROLLBACK<br>- AFTER_COMPLETION |
| fallbackExecution | 진행 중인 transaction이 없을 때에도 실행할 지 여부. 기본값은 `false` |
| value, classes | Listener가 처리할 Event의 타입을 배열로 받는다.<br>한 종류의 타입만 지정되었을 때에는 메서드의 파라미터로 받을 수 있지만,<br>여러 타입이 선언된 경우에는 메서드의 파라미터는 비워둬야 한다. |
| condition | SpEL을 선언하여 true인 경우에 실행된다. |

#### 결과

**성공**

```
INFO  [main] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [main] c.p.s.s.service.sms.SuccessSmsService    : send JOIN sms to 012-3456-7890
INFO  [main] c.p.s.s.SimpleEventApplication           : member count : 1
```

**실패**

```
INFO  [main] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [main] c.p.s.s.service.sms.FailSmsService       : send JOIN sms to 012-3456-7890
ERROR [main] o.s.t.s.TransactionSynchronizationUtils  : TransactionSynchronization.afterCompletion threw exception
...
INFO  [main] c.p.s.s.SimpleEventApplication           : member count : 1
```

## ApplicationEvnetPublisher에 대해서

Spring에서 지원하는 `ApplicationEvnetPublisher`를 이용한 Event처리 방식은 `ApplicationContext`를 통해서 이루어진다. `ApplicationContext`가 `ApplicationEventPublisher`를 상속하고 있다. 즉 `Publisher`와 `Listener`는 Spring에 Bean으로 등록되어야 한다. Spring이 초기화될 때 `@EventListener`, `@TransactionalEventListener`를 `ApplicationContext`에 등록을 해두고 `publishEvent()`가 실행되면 등록된 Listener에 Event를 뿌려준다.

## Event에 대해서

앞서 작성했던 `EventMemberJoinService`를 다시 한 번 살펴보자.

```java
@Profile("event")
@Service
@Transactional
public class EventMemberJoinService implements ApplicationEventPublisherAware, MemberJoinService {

    // ...
    
    public Member join(Member member) {

        memberMapper.insert(member);
        eventPublisher.publishEvent(new MemberJoinedEvent(member)); // gray zone

        return member;
    }
}
```

이전까지의 소스와의 차이가 있다. 바로 `EmailService`와 `SmsService`가 없어진 것이다! 느슨한 결합으로 인한 OOP의 장점을 그대로 누릴 수 있다.

### 결합도

현재는 하나의 Application에서만 Event를 사용하고 있어서 크게 와닿지 않을 수도 있을 것 같다. 좀 더 설명하기 쉽고, 거창해 보일 수 있도록 크기를 키워보자. 만약에 MSA와 같은 구조의 시스템에서 Event를 사용한다면 어떻게 될까?

#### 시스템 간 강결합 : HTTP Rest API 호출

{% plantuml %}
[MemberServer] --> [MailServer] : 등록 `POST /type/JOIN/to/test@test.com`
[MemberServer] --> [SmsServer] : 등록 `POST /type/JOIN/to/012-3456-7890`
{% endplantuml %}

강결합 상태의 시스템의 문제를 여기서 확인할 수 있다.
`MemberServer`는 각 Server를 알고 있다. 호출해야할 URI를 스스로 가지고 있다. 만약 `MailServer`에서 메일을 보내기 위한 URI가 변경이 된다면, `MemberServer`도 수정을 해야한다. 메일 서버가 변경이 되었는 데, 회원 가입 로직이 변경되어야 한다.

#### 느슨한 결합으로 : Event 방식

{% plantuml %}
[MemberServer] --> [EventQueue] : MemberJoinedEvent
[EventQueue] --> [MailServer] : 밀어주거나
[EventQueue] <-- [MailServer] : 받아가거나
[EventQueue] --> [SmsServer] : 밀어주거나
[EventQueue] <-- [SmsServer] : 받아가거나
{% endplantuml %}

앞선 구조와 비교해보자. `MemberServer`는 회원이 가입했을 때 `MemberJoinedEvent`를 생성해서 `EventQueue`에 넣기만 하면 끝이다. 뒤에 무슨 일이 일어나든 `관심사` 밖의 일이다. `MailServer`가 `EventQueue`에서 Event를 받아가서 처리하든, `EventQueue`가 `MailServer`에 Event를 밀어주는 방식으로 처리하든 관심 없는거다.

- MailServer, SmsServer의 로직 변화가 MemberServer에 영향을 주지 않는다.
- MemberServer는 자신의 업무(도메인) 영역만 잘 처리하면 된다.

# 구현 4. Event도 비동기로 처리하기

## Spring Application 설정

```java
//..
@EnableAsync(proxyTargetClass = true)
@SpringBootApplication
public class SimpleEventApplication implements CommandLineRunner {
  // ...
}
```

## MemberJoinEvnetListener
```java
@Component
public class MemberJoinedEventListener {

    @Autowired
    private SmsService smsService;
    @Autowired
    private EmailService emailService;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, classes = EventMemberJoinService.MemberJoinedEvent.class)
    public void handle(EventMemberJoinService.MemberJoinedEvent event) {
        Member member = event.getMember();
        emailService.sendEmail(member.getEmail(), EmailTemplateType.JOIN);
        smsService.sendSms(member.getPhoneNo(), SmsTemplateType.JOIN);
    }
}
```

`@Async`를 사용하여 비동기로 구현할 수 있다.

## 결과

**성공**

```
INFO  [cTaskExecutor-1] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [cTaskExecutor-1] c.p.s.s.service.sms.SuccessSmsService    : send JOIN sms to 012-3456-7890
INFO  [           main] c.p.s.s.SimpleEventApplication           : member count : 1
```

**실패**

```
INFO  [cTaskExecutor-1] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [cTaskExecutor-1] c.p.s.s.service.sms.FailSmsService       : send JOIN sms to 012-3456-7890
INFO  [           main] c.p.s.s.SimpleEventApplication           : member count : 1
ERROR [cTaskExecutor-1] .a.i.SimpleAsyncUncaughtExceptionHandler : Unexpected error occurred invoking async method 'public void com.parfait.study.simpleevent.service.event.AsyncMemberJoinedEventListener.handle(com.parfait.study.simpleevent.service.member.AsyncEventMemberJoinService$AsyncMemberJoinedEvent)'.
```

## 설명

간단하게 spring에서 지원해주는 비동기를 사용할 수 있다. spring에서 어떻게 비동기를 지원해주는지는 본 문서에서는 설명하지 않겠다. 때문에 `TaskExecutor`도 따로 지정하지 않고 사용했다. spring에서 비동기를 사용하는 방법이 궁금하면 [레퍼런스](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/integration.html#scheduling)를 읽어보자.


# 구현 5. AOP를 사용해서 Advisor에서 Event 발급하기

앞서 Event를 발급하였을 때 `// gray zone`의 주석을 달았다. 왜 `gray zone`이라고 했을까? `MemberJoinedEvent`를 `MemberJoinedService`에서 발급하는 것이 맞을까? 개발자 취향에 따라 나뉠 수 있을 것 같다. 만약 `DDD`를 기반으로 작성한 코드였다면 100% `NO!`라고 할 수 있다. 하지만 지금의 예제는 `Service`에서 대부분의 로직을 처리한다. 때문에 누군가는 `Service`에서 Event를 발급해도 된다고할 것이고, 누군가는 Service는 핵심 로직에만 집중했으면 좋겠다라고 할 것이다.
여기서는 후자를 지지해서 AOP로 `gray zone`의 코드를 제거해보려고 한다. Event를 발급하는 행위를 하나의 공통된 방식에 따라 처리할 수 있는 관심사로 취급하겠다.

> 서비스에서 Event를 발급하겠다는 전자의 선택도 존중한다. AOP는 공통 관심사를 분리해서 더 나은 POJO를 만들 수 있게 해준다. 하지만 익숙치 않은 개발자에게는, 전체 어플리케이션 동작을 파악하는 데에 애를 먹을 수 있다. 혼자 개발하는 개발자가 아니라면, 내가 속한 팀에서 어떠한 방법의 프로그래밍이 가장 큰 효율을 낼 수 있는지 파악해봐야할 것이다.

## 의존성 설정

spring의 AOP를 사용할 것이니 관련된 의존성을 추가하자

```gradle
dependencies {
  compile('org.springframework.boot:spring-boot-starter-aop')
}
```

## Spring Application 설정

```java
//..
@EnableAspectJAutoProxy(proxyTargetClass = true)
@SpringBootApplication
public class SimpleEventApplication implements CommandLineRunner {
  // ...
}
```

## MemberJoinService

최종적으로 아래 코드가 동작하도록 만들 것이다. 기존에 `eventPublisher.publishEvent(new MemberJoinedEvent(member));`가 없어졌다. 즉 `MemberJoinService`는 더이상 `ApplicationEventPublisher`에 의존하지 않으며, 그 존재를 모르게 되었다.

```java
@Profile("aop-async-event")
@Service
@Transactional
public class AopAsyncEventMemberJoinService implements MemberJoinService {

    @Autowired
    private MemberMapper memberMapper;

    @PublishEvent(eventType = AopAsyncMemberJoinedEvent.class, params = "#{T(com.parfait.study.simpleevent.model.SendableParameter).create(email, phoneNo)}")
    public Member join(Member member) {

        memberMapper.insert(member);
        return member;
    }
}
```

AOP는 `@PublishEvent`를 대상으로 실행된다. `eventType`을 보고 어떤 타입의 Event를 발급할 것이며, `params`를 해석해서 알맞은 생성자 파라미터를 던져준다.

## 발행할 Event 정의

```java
// 해당 소스의 구현체는 목적에 따라 기본 생성자 혹은 하나의 값을 받는 생성자를 가질 것.
public interface EventHoldingValue<T> {
    T getValue();
}

public static class AopAsyncMemberJoinedEvent implements EventHoldingValue<SendableParameter> {

    @Getter
    private SendableParameter value;

    public AopAsyncMemberJoinedEvent(@NonNull SendableParameter sendableParameter) {
        this.value = sendableParameter;
    }
}

@Data
public class SendableParameter {

    private String email;
    private String phoneNo;

    public static SendableParameter create(String email, String phoneNo) {
        SendableParameter parameter = new SendableParameter();
        parameter.setEmail(email);
        parameter.setPhoneNo(phoneNo);
        return parameter;
    }
}
```

`value`를 가지는 Event를 뜻하는 `EventHoldingValue<T>`를 정의하고 이를 구현한 `AopAsyncMemberJoinedEvent`를 위와 같이 정의했다. `SendableParameter`는 앞서 봤던 `@PublishEvent(params)`에서 정의된 SpEL이 실행되어 생성된다.

## AOP관련 코드

### @PublishEvent

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface PublishEvent {

    /**
    * return이 비어있는 경우 new eventType()
    * params가 비어있는 경우 new eventType(returnValue)
    * params가 문자열인 경우 new event
    */
    Class<? extends EventHoldingValue> eventType();

    // 빈값, 문자열, SpEL('#{표현식}')을 사용할 수 있음
    String params() default "";
}
```

Pointcut을 제공할 애노테이션을 정의한다.

### Advisor

```java
@Component
@Aspect
public class PublishEventAspect implements ApplicationEventPublisherAware {

    // ...

    @Pointcut("@annotation(publishEvent)")
    public void pointcut(PublishEvent publishEvent) {
    }

    /**
    * return값 없으면 new eventType()
    * params값 없으면 new eventType(retVal)
    * params값이 문자열이면 new eventType(params)
    * params값이 SpEL이면 parse 후에 evnetType(params)
    */
    @AfterReturning(pointcut = "pointcut(publishEvent)", returning = "retVal", argNames = "publishEvent,retVal")
    public void afterReturning(PublishEvent publishEvent, Object retVal) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {

        Object event;

        if (retVal == null) {
            // method 반환 값이 void일 때에는 new eventType(this);
            event = publishEvent.eventType()
                                .getDeclaredConstructor()
                                .newInstance();

        } else if (StringUtils.isEmpty(publishEvent.params())) {
            // params가 비어 있는 경우 new eventType(retVal);
            event = publishEvent.eventType()
                                .getConstructor(retVal.getClass())
                                .newInstance(retVal);

        } else if (isSpel(publishEvent.params())) {
            // params가 spel인 경우 new eventType(parsed(publishEvent.params()))
            String spel = publishEvent.params().replaceAll(spelRegex, "$1");
            Object constructArg = expressionParser.parseExpression(spel).getValue(retVal);
            event = publishEvent.eventType()
                                .getDeclaredConstructor(constructArg.getClass())
                                .newInstance(constructArg);

        } else {
            // params가 그냥 string인 경우 new eventType(publishEvent.params());
            event = publishEvent.eventType().getConstructor(String.class).newInstance(publishEvent.params());
        }

        eventPublisher.publishEvent(event);
    }

    private boolean isSpel(String params) {
        return spelPattern.matcher(params).matches();
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.eventPublisher = applicationEventPublisher;
    }
}
```

## 실행 결과

**성공**

```
INFO  [cTaskExecutor-1] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [cTaskExecutor-1] c.p.s.s.service.sms.SuccessSmsService    : send JOIN sms to 012-3456-7890
INFO  [           main] c.p.s.s.SimpleEventApplication           : member count : 1
```

**실패**

```
INFO  [cTaskExecutor-1] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [cTaskExecutor-1] c.p.s.s.service.sms.FailSmsService       : send JOIN sms to 012-3456-7890
INFO  [           main] c.p.s.s.SimpleEventApplication           : member count : 1
ERROR [cTaskExecutor-1] .a.i.SimpleAsyncUncaughtExceptionHandler : Unexpected error occurred invoking async method 'public void com.parfait.study.simpleevent.service.event.AopAsyncMemberJoinedEventListener.handle(com.parfait.study.simpleevent.service.member.AopAsyncEventMemberJoinService$AopAsyncMemberJoinedEvent)'.
```

## 설명

Event를 발급하는 공통된 행위를 애노테이션과 AOP를 사용해서 구현해보았다. 이로써 `Service` 계층의 로직에서는 특별한 일이 아니면 Event를 발급할 소스가 등장하지 않을 것이고, `ApplicationEventPublisher`와의 의존 관계도 끊을 수 있다. 

여기서 한 번 더 리팩토링하려고 한다. 아직도 문제점이 있다. 바로 이벤트 처리에 `순서`가 존재한다는 점이다.

# 구현 6. 순서가 필요있나? Best Effort!

비동기를 처리하는 데 굳이 순서가 필요할까? 여태까지의 코드를 보면 `Email`이 성공해야 `Sms`가 성공한다. 실제로 우리의 로직은 이러한 순서를 필요로하지 않는다. `MemberJoinedEvent`를 발급했을 때, 이를 처리할 핸들러를 여럿 등록하고 각각의 thread를 격리해보자.

### 설명

`MemberJoinService`의 구현체에서 달라진 부분은 없다

## EventListeners

```java
@Service
public class EmailEventListener {

    @Autowired
    private EmailService emailService;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, classes = DistributedAopAsyncMemberJoinedEvent.class)
    public void handleMemberJoinedEvent(DistributedAopAsyncMemberJoinedEvent event) {
        SendableParameter parameter = event.getValue();
        emailService.sendEmail(parameter.getEmail(), parameter.getEmailTemplateType());
    }
}

@Service
public class SmsEventListener {

    @Autowired
    private SmsService smsEventService;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, classes = DistributedAopAsyncMemberJoinedEvent.class)
    public void sendEmail(DistributedAopAsyncMemberJoinedEvent event) {
        SendableParameter request = event.getValue();
        smsEventService.sendSms(request.getPhoneNo(), request.getSmsTemplateType());
    }
}
```

## 결과

**성공**

```
INFO  [cTaskExecutor-2] c.p.s.s.service.sms.SuccessSmsService    : send JOIN sms to 012-3456-7890
INFO  [cTaskExecutor-1] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [           main] c.p.s.s.SimpleEventApplication           : member count : 1
```

**실패**

```
INFO  [cTaskExecutor-1] c.p.s.s.service.email.EmailService       : send JOIN email to test@test.com
INFO  [cTaskExecutor-2] c.p.s.s.service.sms.FailSmsService       : send JOIN sms to 012-3456-7890
INFO  [           main] c.p.s.s.SimpleEventApplication           : member count : 1
ERROR [cTaskExecutor-2] .a.i.SimpleAsyncUncaughtExceptionHandler : Unexpected error occurred invoking async method 'public void com.parfait.study.simpleevent.service.sms.SmsEventService.sendEmail(com.parfait.study.simpleevent.service.member.DistributedAopAsyncEventMemberJoinService$DistributedAopAsyncMemberJoinedEvent)'.

```

## 설명

Email과 Sms를 다른 thread에서 보내고 있는 것을 확인할 수 있다.
말만 어렵게 했지 내용은 간단하다. Event에 대한 `Listener(subscriber, 혹은 handler)`는 필요하면 언제든지 추가하면 된다. 또한 `@Async` 지원도 가능하다.

# 마무리

Spring에서 지원하는 Event 처리 방법과 실제로 어떻게 사용할 수 있을지 살펴보았다. 가장 중요한 것은 Event를 사용하는 이유를 깨닫는 것이다. 가장 큰 이유는 Event 방식은 `시스템 간의 결합도를 낮춰주는 것`이다. 결합도를 낮춰서 서비스 로직에 집중하고, 장애 전파에 강한 어플리케이션을 작성할 수 있다. 비동기로 실행하는 것도 Spring의 지원을 받아 간단하게 처리할 수 있다. 

물론 이러한 Event 방식도 문제는 있다. Global Transaction을 어떻게 쥐고 갈 것인지 하는 것이다. 본문에서는 하나의 Application 내에서 Event를 발생시키고 처리했는데, Event-Driven Architecture의 MSA 환경을 구성하는 경우, 어떻게 Transaction과 Latency 사이에서 타협점을 찾을 것인가는, 현재 이 업계의 큰 이슈 중 하나가 아닐까? 아직까지는 Event를 사용하면서 완벽한 Transaction을 지원하는 것은 어려워 보인다. 서비스 간의 결합도를 떨어뜨리기 위해, Event를 위한 별도의 모듈(RabbitMQ, Kafka)을 사용하는 중에 데이터 손실 등 넘어야할 난관도 많이 있다. 하지만 Event를 사용하는 구조 자체는 매우 매력적이며, 개발자에게 더 많은 선택지를 가져다 준다.

Spring은 개발자로 하여금 POJO를 작성하여 OOP를 할 수 있도록 유도한다. Spring Event 또한 개발자가 OOP를 더 잘할 수 있도록 도와주는 장치다. 더 유연한 구조로 나아가는 방법 중의 하나로 충분히 써먹을 수 있을 것이라 믿어 의심치 않는다.

소스 코드 : https://github.com/supawer0728/simple-event