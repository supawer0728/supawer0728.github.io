---
title: (Spring)Filter와 Interceptor의 차이
date: 2018-04-04 21:38:46
tags: [spring, filter, interceptor, spring-webmvc]
categories:
  - spring
  - mvc
---

# 서론

Spring을 익힌지 얼마되지 않았을 때, 회원 인증 로직을 구현할 일이 생겼었다. 그 인증을 구현하기 위해 Filter와 Interceptor를 조사했었다. 하지만 Filter와 Interceptor를 어떤 경우에 써야 좋은지 명확하게 알지는 못했다. 이제와서 옛 기억들이 떠올라 다시 `spring filter interceptor 차이`로 검색을 해보고 올라온 글들을 읽어봤는데, 아무래도 몇가지 아쉬운 부분들이 있었다. 이번 글에서는 Spring Web Application에서 사용하는 Filter와 Interceptor에 대해, 많은 글에서 공통적으로 이야기하는 부분과 의외로 언급하지 않는 부분 두가지를 쓰려고 한다.

<!-- more -->

# 실행 시점

| ![request-lifecycle](spring-request-lifecycle.jpg) |
| - |
| 출처 : https://justforchangesake.wordpress.com/2014/05/07/spring-mvc-request-life-cycle/ |

## 공통적으로 이야기하는 점

- Filter와 Interceptor는 실행되는 시점이 다르다.
- Filter는 Web Application에 등록을 하고, Interceptor는 Spring의 Context에 등록을 한다.

## 추가로 이야기하고 싶은 점

tomcat의 경우 `deployment descriptor(/WEB-INF/web.xml)`에 사용할 Filter를 등록한다. 때문에 애플리케이션 전역에 영향을 주는 작업은 Filter로 한다는 의견이 있다. 실제로 필자도 예전에는 그런 줄 알았다. 하지만 사실은 그렇지 않다. Filter도 Interceptor도 모두 요청에 대한 `전후처리`라고 하는 역할을 수행한다. 또한 uri기반으로 언제 실행할 것인지를 조정 가능하며, 직접 request의 내용을 파악해서 원하는 조건에 부합할 때 로직을 수행할 수 있다는 점에서 차이가 없다.

실행 시점이 다르기 때문에 가장 큰 영향을 받는 것. 내가 생각하기에는 예외 처리(Exception Handling)다.

Filter에서 예외가 발생하면 Web Application에서 처리해야 한다. tomcat을 사용한다면 `<error-page>`를 잘 선언하든가 아니면 Filter 내에서 예외를 잡아 `request.getRequestDispatcher(String)`으로 마치 핑퐁하듯이 예외 처리를 미뤄야 한다. 하지만 Interceptor에서 예외가 발생하면? Interceptor의 실행 시점을 보자, Spring의 ServletDispatcher 내에 있다. 즉 `@ControllerAdvice`에서 `@ExceptionHandler`를 사용해서 예외를 처리를 할 수 있다. Spring Web Application의 예외 처리 방법에 대해서는 서술하지 않겠다. 대신 [좋은 글](https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc)이 있으니 읽어두면 좋을 것 같다. 예외 처리가 견고한 Application은 유지보수하기 좋다. 만약 작성해야할 전후처리 로직에서 예외를 전역적으로 처리하고 싶다면 Interceptor를 사용하자.

# Interface

당연하지만 interface가 다르다. 두 interface의 실행 메서드를 비교해보자.

**Filter**

```java
public interface Filter {
  void doFilter(ServletRequest request, ServletResponse response, FilterChain chain);
}
```

**HandlerInterceptor**

```java
public interface HandlerInterceptor {
  boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler);
  void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView mav);
  void afterCompletion(HttpServletRequest request, HttpServeletResponse response, Object handler, Exception ex);
}
```

## 공통적으로 이야기하는 점

- Filter는 Servlet에서 처리하기 전후를 다룰 수 있다.
- Interceptor는 Handler를 실행하기전(preHandle), Handler를 실행한 후(postHandle), view를 렌더링한 후(afterCompletion) 등, Servlet내에서도 메서드에 따라 실행 시점을 다르게 가져간다.

## 추가로 이야기하고 싶은 점

### Interceptor에서만 할 수 있는 것

- AOP 흉내를 낼 수 있다. `@RequestMapping` 선언으로 요청에 대한 `HandlerMethod`가 정해졌다면, handler라는 이름으로 `HandlerMethod`가 들어온다. `HandlerMethod`로 메서드 시그니처 등 추가적인 정보를 파악해서 로직 실행 여부를 판단할 수 있다.
- View를 렌더링하기 전에 추가 작업을 할 수 있다. 예를 들어 웹 페이지가 권한에 따라 GNB(Global Navigation Bar)이 항목이 다르게 노출되어야 할 때 등의 처리를 하기 좋다.

### Filter에서만 할 수 있는 것

`ServletRequest` 혹은 `ServletResponse`를 교체할 수 있다. 아래와 같은 일이 가능하다.

```java
public class SomeFilter implements Filter {
  //...
  
  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    chain.doFilter(new CustomServletRequest(), new CustomResponse());
  }
}
```

설마 저런 일을 할까? 꽤 자주 있는 요구 사항이다. HttpServletRequest의 body(ServletInputStream의 내용)를 로깅하는 것을 예로 들 수 있을 것 같다. `HttpServletRequest`는 body의 내용을 한 번만 읽을 수 있다. Rest API Application을 작성할 때, 흔히 json 형식으로 요청을 받는다. `@Controller(Handler)`에 요청이 들어오면서 body를 한 번 읽게 된다. 때문에 Filter나 Interceptor에서는 body를 읽을 수 없다. `IOException`이 발생한다. body를 로깅하기 위해서는 `HttpServletRequest`를 감싸서 여러번 inputStream을 열 수 있도록 커스터마이징 된 `ServletRequest`를 쓸 수 밖에 없다.

```java
public class SomeFilter implements Filter {
  //...
  
  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    chain.doFilter(new BodyCachedServletRequestWrapper(request), response);
  }
}
```

# 마무리

Spring으로 Web Application을 작성할 때, 한 번 쯤은 Filter와 Interceptor를 선택해야할 상황에 놓일 것이다. 전후처리라는 점에서 그 역할은 비슷하나, 실제로 실행되는 시점이나 할 수 있는 것들이 다르다. 처리해야할 요구 사항에 따라서 무엇을 써도 무방할 수도 있다. 하지만 차이점을 정확히 파악하고 있다면 조금이나마 더 효과적으로 개발을 할 수 있을 것이라고 믿는다.