---
title: Spring Framework 5.0.4.RELEASE Reference Core (Chapter5)
date: 2018-03-11 02:41:50
tags: [spring, reference]
categories:
  - spring
  - reference
---
# 5. Aspect Oriented Programming with Spring

# 5.1. Introduction

## 5.1.2. Spring AOP capabilities and goals

- 순수 자바로 구현하여, 특정 컴파일러에 의존하지 않음
- 메서드 실행 join point만 지원
- Spring AOP의 목적은 완벽한 AOP지원이 아닌, 엔터프라이즈 어플리케이션의 각종 이슈를 해결하기 위해 AOP와 Spring IoC를 융화시키는 것
- 여타 AOP Framework와 경쟁하지 않을 것(Spring의 Proxy 방식의 AOP와 본격적인 AspectJ 등의 프레임워크는 보완관계에 있음)
<!-- more -->
## 5.1.3. AOP Proxies

```java
public interface Adder {
  int add(int a, int b);
}

@Component
public class AdderImpl implements Adder {
  public int add(int a, int b) { return a + b;}
}

@Slf4j
@Component
public class AdderAopProxyParameterLogger implements Adder {
  @Autowired
  @Qualifier("adderImpl")
  private Adder target;

  public int add(int a, int b) {
    log.debug("a : {}, b : {}", a, b);
    return target.add(a, b);
  }
}
```

> Spring AOP에서 AOP 프록시를 사용하기 위해, 표준 JDK dynamic proxy를 사용한다

## 5.2.1. Enabling `@AspectJ` Support

- `@AspectJ` 지원은 `Java` 스타일과 `xml` 스타일로 설정할 수 있다
- `aspectjweaver.jar`가 경로내에 존재해야한다

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig { }
```

## 5.2.2. Declaring an aspect

```java
@Aspect
public class ServiceParameterLogAspect {
}
```

## 5.2.3. Declaring a pointcut

```java
@Aspect
public class ServiceParameterLogAspect {
  @Pointcut("@annotation(org.springframework.stereotype.Service)")
  public void pointcut() {
  }
}
```

pointcut 종류 : https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/core.html#aop-pointcuts-designators

## 5.2.4. Declaring a advice

```java
@Aspect
public class ServiceParameterLogAspect {
  @Pointcut("@annotation(org.springframework.stereotype.Service)")
  public void pointcut() {
  }
  
  @Before("pointcut()")
  public void before() { ... }
  
  @AfterReturning(pointcut = "pointcut()", returning = "retVal")
  public void afterReturning(Object retVal) { ... }
  
  @AfterThrowing(pointcut = "pointcut()", throwing = "ex")
  public void afterThrowing(DataAccessException ex) { ... }
  
  @Around("pointcut()")
  public Object around(ProceedingJoinPoint joinPoint) {
    return joinPoint.proceed();
  }
}
```

## 5.2.4. Declaring a advice

### Passing PArameters to advice

**Parameter**
```java
@Pointcut("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
    // ...
}
```

**Annotation**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditable {
    AuditCode value();
}

@Before("com.xyz.lib.Pointcuts.anyPublicMethod() && @annotation(auditable)")
public void audit(Auditable auditable) {
    AuditCode code = auditable.value();
    // ...
}
```

## 5.2.4. Declaring a advice

### Advice ordering

**`org.springframework.core.Ordered` 인터페이스 구현**

```java
@Aspect
public class ServiceParameterLogAspect implements Ordered {
 
  @Override
  public int getOrder() { return 0;}

  @Pointcut("@annotation(org.springframework.stereotype.Service)")
  public void pointcut() {
  }
  
  @Before("pointcut()")
  public void before() { ... }
}
```
**`@Order` 애노테이션**

```java
@Order(0)
@Aspect
public class ServiceParameterLogAspect implements Ordered {

  @Pointcut("@annotation(org.springframework.stereotype.Service)")
  public void pointcut() {
  }
  
  @Before("pointcut()")
  public void before() { ... }
}
```

## 5.6. Proxying mechanisms

Advice 대상 object가 interface를 구현하고 있다면 JDK Dynamic Proxy를, 그렇지 않다면 CGLIB를 사용한다(상속)

CGLIB로 AOP를 사용할 때에는 주의점이 몇가지 있다

- `final`메서드는 `override`가 불가하기 때문에 AOP를 사용할 수 없다
- Spring 3.2부터는 CGLIB 라이브러리가 포함되어 별도로 의존성을 추가할 필요가 없다
- Spring 4.0부터 CGLIB에 의해 대상 클래스의 생성자가 두 번 호출되는 현상이 수정되었다

강제로 CGLIB를 사용하기 위해서는 `proxyTargetClass`를 `true`로 설정한다

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```