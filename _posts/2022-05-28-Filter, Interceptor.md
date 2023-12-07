---
title: Filter, Interceptor
date: 2022-05-28 12:00:00 +09:00
categories: [study, cs_study]
tags: [java, spring]     
---

## 필터, 인터셉터가 필요한 이유

스프링 코드를 작성할 때 공통적으로 처리해야할 업무들이 많다. 공통업무에 관련된 코드를 모든 페이지 마다 작성해야한다면 중복된 코드가 많아지게 되면 소스관리가 힘들어진다.

즉, 공통처리를 위해 활용할 수 있는 것이 3가지가 있다.

1. Filter
2. Interceptor
3. AOP

이중에서 필터와 인터셉터에 대해서 알아보겠다.

<br>

## 필터(Filter)

필터는 J2EE 표준 스펙 기능으로 디스패처 서블릿(Dispatcher Servlet)의 앞단에서 정보를 처리한다.

디스패처 서블릿은 스프링의 가장 앞단에 존재하는 프론트 컨트롤러이므로, 필터는 스프링 범위 밖에서 처리가 되는 것이다.

스프링 컨테이너에 존재하는 빈들을 사용할 수 없어 비즈니스 로직과 연관된 작업을 수행할 수 없다.

대표적으로 인코딩 변환처리, XSS 방어, LOG, 인증등을 구현할때 사용한다.

![image](https://user-images.githubusercontent.com/70622731/156137692-b5923f77-787b-45a3-92b3-b554f581a2ba.png)

### 필터 메소드

필터를 추가하기 위해서는 javax.servlet의 Filter 인터페이스를 구현(implements)해야 하며 이는 다음의 3가지 메소드를 가지고 있다.

```java
public interface Filter {
    public default void init(FilterConfig filterConfig) throws ServletException {}
    
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException;
    
    public default void destroy() {} 
}
```

__init 메소드__

init 메소드는 필터객체를 초기화하고 서비스에 추기하기 위한 메소드이다. 웹 컨테이너가 1회 init 메소드를 호출하여 필터 객체를 초기화하면 이후의 요청들은 doFilter를 통해 처리된다.

__doFilter__

doFilter 메소드는 url-pattern에 맞는 모든 HTTP 요청이 디스패처 서블릿으로 전달되기 전에 웹 컨테이너에 의해 실행되는 메소드이다. doFilter의 파라미터로는 FilterChain이 있는데, FilterChain의 doFilter 통해 다음 대상으로 요청을 전달하게 된다. chain.doFilter() 전/후에 우리가 필요한 처리 과정을 넣어줌으로써 원하는 처리를 진행할 수 있다.

__destroy__

destroy 메소드는 필터 객체를 서비스에서 제거하고 사용하는 자원을 반환하기 위한 메소드이다. 이는 웹 컨테이너에 의해 1번 호출되며 이후에는 이제 doFilter에 의해 처리되지 않는다.

__필터 동작방식__

<img width="1022" alt="image" src="https://user-images.githubusercontent.com/70622731/156183715-8ffe93ca-a486-47cf-ba2b-906ecd75f486.png">

클라이언트에서 보낸 요청 정보는 필터에 의해 정보가 변경되며, 반대로 자원에서 보낸 정보도 필터에 의해 변경된다.

보통 클라이언트와 자원 사이에 1개가 존재하지만, 여러개의 필터가 모여서 하나의 체인을 만들 수 있다.

구성된 체인은 맨 처음 인스턴스 초기화(init())를 거친 후 각 필터에서 doFilter() 를 통해 필터링 작업을 처리하고 Destroy 된다.

이 때의 환경 설정은 주로 톰캣을 사용할 경우 web.xml 또는 Java Configuration 을 이용해서 구현하게 된다.

<br>

## 인터셉터(Interceptor)

인터셉터는 Spring이 제공하는 기술로써, 디스패처 서블릿이 컨트롤러를 호출하기 전에 요청과 응답을 참조하거나 가공할 수 있는 기능을 제공한다.

디스패처 서블릿은 핸들러 매핑을 통해 컨트롤러를 찾는데, 이때 1개 이상의 인터셉터가 등록되어 있다면 순차적으로 인터셉터들을 거쳐 컨트롤러가 실행되도록 하고, 인터셉터가 없다면 바로 컨트롤러를 실행한다.

대표적으로 인증/인가 등과 같은 공통작업, controller로 넘겨주는 정보의 가공, 로그인 처리 시 많이 사용된다.

<img width="1029" alt="image" src="https://user-images.githubusercontent.com/70622731/156138871-6ea8eacf-f74d-4765-9c21-78a9f5fda9b7.png">

### 인터셉터 메소드

인터셉터를 추가하기 위해서는 org.springframework.web.servlet의 HandlerInterceptor 인터페이스를 구현(implements)해야 하며, 이는 다음의 3가지 메소드를 가지고 있다.

```java
public interface HandlerInterceptor {
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception { return true; }
    
    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception { }
    
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception { }
}
```

__preHandle__

preHandle 메소드는 컨트롤러가 호출되기 전에 실행된다. 그렇기 때문에 컨트롤러 이전에 처리해야 하는 전처리 작업이나 요청 정보를 가공하거나 추가하는 경우에 사용할 수 있다.

__postHandle__

postHandle 메소드는 컨트롤러를 호출된 후에 실행된다. 그렇기 때문에 컨트롤러 이후에 처리해야 하는 후처리 작업이 있을 때 사용할 수 있다. 이 메소드에는 컨트롤러가 반환하는 ModelAndView 타입의 정보가 제공되는데, 최근에는 Json 형태로 데이터를 제공하는 RestAPI 기반의 컨트롤러(@RestController)를 만들면서 자주 사용되지는 않는다.

__afterCompletion__

afterCompletion 메소드는 이름 그대로 모든 뷰에서 최종 결과를 생성하는 일을 포함해 모든 작업이 완료된 후에 실행된다. 요청 처리 중에 사용한 리소스를 반환할 때 사용하기에 적합하다.

__인터셉터 동작방식__

아래 그림은 인터셉터의 정상적 흐름도이다. 컨트롤러가 호출되기 전에는 "preHandle"이 작동하고, 호출된 후에는 "postHandle"이, 마지막으로 이러한 요청들이 완료된 이후에 "afterCompletion"이 작동하는 것을 볼 수 있다

<img width="715" alt="image" src="https://user-images.githubusercontent.com/70622731/156185167-31d7ef3f-dda3-4ace-8e8e-72315bf6a226.png">

인터셉터의 예외 흐름은 preHandle이 예외를 발생시키면 뒤에 있던 postHandle이 호출되지 않는다, 이때 주의할 점이 있는데 afterCompletion은 항상 호출이 된다는 것이다 마치 자바의 try catch문에서 finally가 항상 작동하는 것을 생각하면 이해가 좀 더 쉽겠다

<img width="719" alt="image" src="https://user-images.githubusercontent.com/70622731/156185379-fde9809d-c237-4232-860b-e6381f3bbcd1.png">

이 때의 환경 설정은 주로 servletContext.xml 또는 Java Configuration 을 이용해서 구현하게 된다.

<br>

## 인터셉터와 AOP 차이

인터셉터 대신에 컨트롤러들에 적용할 부가기능을 어드바이스로 만들어 AOP(Aspect Oriented Programming, 관점 지향 프로그래밍)를 적용할 수도 있다. 하지만 다음과 같은 이유들로 컨트롤러의 호출 과정에 적용되는 부가기능들은 인터셉터를 사용하는 편이 낫다.

1. Spring의 컨트롤러는 타입과 실행 메소드가 모두 제각각이라 포인트컷(적용할 메소드 선별)의 작성이 어렵다.
2. Spring의 컨트롤러는 파라미터나 리턴 값이 일정하지 않다.

즉, 타입이 일정하지 않고 호출 패턴도 정해져 있지 않기 때문에 컨트롤러에 AOP를 적용하려면 번거로운 부가 작업들이 생기게 된다.

<br>

## 필터 인터셉터 차이

__필터와 인터셉터는 실행되는 시점이 다르다.__

필터는 Web Application에 등록을 하고, Interceptor는 Spring의 Context에 등록을 한다.

필터는 Servlet에서 처리하기 전후를 다룰 수 있다.

스프링에서는 ServletRequest/ServletResponse 객체를 조작할 수 있다는 점에서 필터를 인터셉터 보다 훨씬 강력한 기술이라고 표현하고 있다.

필터에서는 기본적으로 스프링과 무관하게 전역적으로 처리해야 하는 작업들을 처리할 수 있다.

인터셉터에서는 클라이언트의 요청과 관련되어 전역적으로 처리해야 하는 작업들을 처리할 수 있다.

<br>

---

### Reference

- https://mangkyu.tistory.com/173
- https://junghyungil.tistory.com/123
- https://maenco.tistory.com/entry/Spring-MVC-Filter-Interceptor?category=959609
- https://hidelookit.tistory.com/287
