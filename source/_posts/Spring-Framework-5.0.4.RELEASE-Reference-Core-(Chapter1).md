---
title: Spring Framework 5.0.4.RELEASE Reference Core (Chapter1)
date: 2018-03-11 02:33:38
tags: [spring, reference]
categories:
  - spring
  - reference
---
# 1. The IoC container

# 1.2. Container overview

`org.springframework.context.ApplicationContext` 인터페이스는 Spring IoC container이다. 구현체들은 bean을 인스턴스화, 설정, 조합하는 역할을 책임지고 있다.

## 1.2.2. Instantiating a container

`ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");`

<!-- more -->

# 1.3. Bean overview

## 1.3.2. Instantiating beans

> `static inner class`를 bean으로 등록하기 위해서는 `FQDN$내부클래스명`으로 표기해야한다
> `com.example.Foo$Bar`

# 1.5. Bean scopes

## 1.5.1. Bean scopes

| Scope | Description |
| - | - |
| singleton | (Default) Spring IoC container 당 하나의 객체 인스턴스만 생성 |
| prototype | 다수의 객체 인스턴스를 생성`(any number)` |
| request | `HTTP Request`과 동일한 lifecycle을 가짐. web 환경의 Spring ApplicationContext에서만 동작함 |
| session | `HTTP Session`과 동일한 lifecycle을 가짐. web 환경의 Spring ApplicationContext에서만 동작함 |
| application | `ServletContext`와 동일한 lifecycle을 가짐. web 환경의 Spring ApplicationContext에서만 동작함 |
| webcoket | `WebSocket`과 동일한 lifecycle을 가짐. web 환경의 Spring ApplicationContext에서만 동작함 |

## 1.5.3. Singleton beans with prototype-bean dependencies

`singleton bean`이 `prototype bean`에 의존하고 있는 경우 어떻게 될까?
의존성은 초기화 시점(instantiation time)에 정의된다는 점을 기억해야한다.

즉, `singleton bean`이 초기화 될때 새로운 `prototype bean`이 생성되겠지만,
한 번 초기화 된 이후에는 항상 같은 `prototype bean`을 사용하게 될 것이다.

`singleton bean`이 매번 다른 인스턴스를 바라보도록 하고 싶다면 [Method Injection](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/core.html#beans-factory-method-injection)항목을 참고할 수 있다.
스포를 하자면 `IoC`를 하지 않는 것이다.

## 1.5.4. Request, session, application, and WebSocket scopes

```java
@RequestScope
@Component
public class RequestScopeBean { 
  //...
}
@SessionScope
@Component
public class RequestScopeBean { 
  //...
}
@ApplicationScope
@Component
public class RequestScopeBean { 
  //...
}
```

## Scoped beans as dependencies

`Request`, `Session` 등의 bean이 자신보다 더 긴 lifecycle을 가지고 있는 bean에 속해질 때,
Spring IoC는 `AOP Proxy`로 이를 대체하여 문제를 해결한다.
즉 실제 Instance를 주입받는 것이 아니라, Proxy 객체를 주입받게되며, 이 프록시는 실제 instance를 가리키도록 되어 있다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- an HTTP Session-scoped bean exposed as a proxy -->
    <bean id="userPreferences" class="com.foo.UserPreferences" scope="session">
        <!-- instructs the container to proxy the surrounding bean -->
        <aop:scoped-proxy/>
    </bean>

    <!-- a singleton-scoped bean injected with a proxy to the above bean -->
    <bean id="userService" class="com.foo.SimpleUserService">
        <!-- a reference to the proxied userPreferences bean -->
        <property name="userPreferences" ref="userPreferences"/>
    </bean>
</beans>
```

# 1.9. Annotation-based container configuration

## Annotation 기반 설정이 XML 기반 설정보다 더 나은가?

그런거 없음
개발자가 알아서 선택할 일

## 1.9.1 `@Required`

bean이 초기화될 때, 반드시 의존성이 설정되어야함.
속성이 없는 경우, container가 예외를 던짐.
NPE를 회피할 수 있음.

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Required
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

## 1.9.2 `@Autowired`

생성자
```java
public class MovieRecommender {
    private final CustomerPreferenceDao customerPreferenceDao;
    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }
}
```

setter
```java
public class SimpleMovieLister {
    private MovieFinder movieFinder;
    @Autowired
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```
목록(`@Order`로 순서를 정할 수 있음)
```java
public interface Adder {
}
@Component
public class AdderA implements Adder {
}
@Component
public class AdderB implements Adder {
}
public class AdderService {
  @Autowired
  Set<Adder> Adders; // Adder[] adders, List<Adder> adders
}
```

## 1.9.3. Fine-tuning annotation-based autowiring wigh `@Primary`

`Autowired`받을 수 있는 후보가 여럿 있을 때, 주로 사용할 bean을 `@Primary`로 지정 가능

```java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }

    @Bean
    public MovieCatalog secondMovieCatalog() { ... }
}
```

## 1.9.4. Fine-tuning annotation-based autowiring with qualifiers

보다 정교하게 bean selection을 제어하고 싶다면, `@Qualifier`를 사용할 수 있다

**field에 사용**

```java
public class MovieRecommender {

    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog;
}
```

**생성자에 사용**

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main")MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

## 1.9.4. Fine-tuning annotation-based autowiring with qualifiers

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/>

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/>

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

## 1.9.4. Fine-tuning annotation-based autowiring with qualifiers

`@Autowired`와 `@Qualifier`를 같이 사용하는 경우

1. `@Autowired`는 type을 기반으로 bean을 찾는다
2. `@Qualifier`는 기본적으로 bean name을 사용하여 bean을 찾는다

## 1.9.4. Fine-tuning annotation-based autowiring with qualifiers

**`@Qualifier`**를 사용한 custom annotation 작성
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {
    String value();
}
```
```java
public class MovieRecommender {

    @Autowired
    @Genre("Action")
    private MovieCatalog actionCatalog;

    private MovieCatalog comedyCatalog;

    @Autowired
    public void setComedyCatalog(@Genre("Comedy") MovieCatalog comedyCatalog) {
        this.comedyCatalog = comedyCatalog;
    }
}
```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="Genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="example.Genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

## 1.9.7. `@Resource`

- `@Autowired`는 type으로 검색
- `@Resource`는 bean name으로 검색

## 1.9.8. @PostConstruct and @PreDestroy

```java
public class CachingMovieLister {

    @PostConstruct
    public void populateMovieCache() {
        // populates the movie cache upon initialization...
    }

    @PreDestroy
    public void clearMovieCache() {
        // clears the movie cache upon destruction...
    }
}
```

# 1.10. Classpath scanning and managed components

## 1.10.1. @Component and further stereotype annotations

- Spring은 여러 stereotype의 annotation을 제공한다 : `@Component`, `@Service`, `@Controller` 등등.
- `@Repository`, `@Service`는 `@Component`의 한정사이다
- custom annotation을 만들어 사용하거나, 기본 제공되는 stereotype annotation을 AOP를 사용하여, 초기화시 임의의 process를 수행하도록 할 수도 있다

## 1.10.2. Meta-annotations

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component // Spring will see this and treat @Service in the same way as @Component
public @interface Service {
}
```
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
	String value() default "";
}
```

## 1.10.4. Using filters to customize scanning

| Filter Type |	Example Expression |	Description |
| - | - | - |
| annotation (default) | org.example.SomeAnnotation | An annotation to be present at the type level in target components |
| assignable | org.example.SomeClass | A class (or interface) that the target components are assignable to (extend/implement) |
| aspectj | `org.example..*Service+` | An AspectJ type expression to be matched by the target components |
| regex | `org\.example\.Default.*` | A regex expression to be matched by the target components class names |
| custom | org.example.MyTypeFilter | A custom implementation of the org.springframework.core.type .TypeFilter interface |

```java
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    ...
}
```
```xml
<beans>
    <context:component-scan base-package="org.example">
        <context:include-filter type="regex"
                expression=".*Stub.*Repository"/>
        <context:exclude-filter type="annotation"
                expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
</beans>
```

# 1.12. Java-based container configuration

## 1.12.4. Using the `@Configuration` annotation

### Further information about how Java-based configuration works internally

아래 소스에서 `clientDao()`가 2번 호출됨에도 불구하고, 캐시(`CGLIB`)를 통해서 동일한 객체(`singleton`)가 주입됨

```java
@Configuration
public class AppConfig {

    @Bean
    public ClientService clientService1() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientService clientService2() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientDao clientDao() {
        return new ClientDaoImpl();
    }
}
```
## 1.12.5. Composing Java-based configurations

**`@Import`**

```java
@Configuration
public class ConfigA {
    @Bean
    public A a() {
        return new A();
    }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {
    @Bean
    public B b() {
        return new B();
    }
}
```

# 1.13. Environment abstraction

`Environment`는 `profiles`와 `properties`를 추상화하였다

## 1.13.1. Bean definition profiles

### `@Profile`

```java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}

@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}

// meta-annotation으로 사용
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}

@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Development
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Production
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

### Activating a profile

```
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```
```
-Dspring.profiles.active="profile1,profile2"
```

# 1.15. Additional capabilities of the ApplicationContext

## 1.15.1. Internationalization using MessageSource

**`MessageSource` 인터페이스**

- `String getMessage(String code, Object[] args, String default, Locale loc)`
- `String getMessage(String code, Object[] args, Locale loc)`
- `String getMessage(MessageSourceResolvable resolvable, Locale locale)` - `BindingResult`의 `FieldError`가 `MessageSourceResolvable`을 구현하고 있음

```
<beans>
    <bean id="messageSource"
            class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>
                <value>exceptions</value>
                <value>windows</value>
            </list>
        </property>
    </bean>
</beans>
```

## 1.15.2. Standard and custom events

**Built-in Events**

| Event | Explanation |
| - | - |
| ContextRefreshedEvent | `ApplicationContext`가 초기화되거나, 갱신(refresh)될 때 발행됨.<br>`ConfigurableApplicationContext` 인터페이스의 `refresh()` 메서드가 호출되는 상황을 예로 들 수 있음 |
| ContextStartedEvent | `ApplicationContext`가 시작될 때 발행됨<br>`ConfigurableApplicationContext` 인터페이스의 `start()`메서드가 호출되는 상황을 예로 들 수 있음 |
| ContextStoppedEvent | `ApplicationContext`가 `ConfigurableApplicationContext` 인터페이스의 `stop()`이 호출되어 정지되는 경우 발행됨 |
| ContextClosedEvent | `ApplicationContext`가 `ConfigurableApplicationContext`의 `close()`가 호출되어 `close`되는 경우에 발행됨 |
| RequestHandledEvent | 웹 환경에서 `HTTP request`를 처리했을 때 발행됨<br>요청 완료 후 발행<br>`DispatcherServlet`을 사용하는 웹 어플리케이션만 적용 |

**custom event**

```java

// Custom Event
public class BlackListEvent extends ApplicationEvent {

    private final String address;
    private final String test;

    public BlackListEvent(Object source, String address, String test) {
        super(source);
        this.address = address;
        this.test = test;
    }
}

// Publish Event
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blackList;
    private ApplicationEventPublisher publisher;

    public void setBlackList(List<String> blackList) {
        this.blackList = blackList;
    }

    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String text) {
        if (blackList.contains(address)) {
            BlackListEvent event = new BlackListEvent(this, address, text);
            publisher.publishEvent(event);
            return;
        }
        // send email...
    }
}

// Handle Event
public class BlackListNotifier implements ApplicationListener<BlackListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlackListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```
## 1.15.2. Standard and custom events

### Annotation-based event listeners

```java
public class BlackListNotifier {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    @EventListener
    public void processBlackListEvent(BlackListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}

// 여러 타입의 이벤트 처리
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
    ...
}

// SpEL 사용
@EventListener(condition = "#blEvent.test == 'foo'")
public void processBlackListEvent(BlackListEvent blEvent) {
    // notify appropriate parties via notificationAddress...
}
```

### Asynchronous Listeners

```java
@EventListener
@Async
public void processBlackListEvent(BlackListEvent event) {
    // BlackListEvent is processed in a separate thread
}
```
> 비동기 이벤트 리스너에서 예외가 발생시, caller로 전파되지 않는다. `AsyncUncaughtExceptionHandler` 참고할 것
> 처리 결과를 받기 위해서는 다시 비동기로 이벤트를 발생시켜야 한다

### Ordering listeners

```java
@EventListener
@Order(42)
public void processBlackListEvent(BlackListEvent event) {
    // notify appropriate parties via notificationAddress...
}
```

### Generic events

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    ...
}

public class EntityCreatedEvent<T>
        extends ApplicationEvent implements ResolvableTypeProvider {

    public EntityCreatedEvent(T entity) {
        super(entity);
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(),
                ResolvableType.forInstance(getSource()));
    }
}
```