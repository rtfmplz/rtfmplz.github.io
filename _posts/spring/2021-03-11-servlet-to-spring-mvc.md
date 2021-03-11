---
layout: post
title: Servlet 에서 Spring MVC 까지
tags: [web server, was, servlet, jsp, mvc, spring]
author: Jae
comments: true
---

# 이 글의 목적

- 맨날 햇갈리고 파편화 되어있던 Web Service 관련 지식들을 의식의 흐름대로 나열해본다.
- [뉴렉](https://youtube.com/user/newlec1) 강의를 정리한다.

> 저작권에 문제가 있는 경우 연락 부탁드립니다.

# 참고

- https://youtube.com/user/newlec1
- https://jongmin92.github.io/2018/03/12/Spring/spring-mvc/

# Local Program

- Console Program, GUI Program..
- Database(또는 다른 저장소)가 사용자의 Local 에 존재하고, 프로그램이 사용자의 입력에 대한 결과를 화면에 보여준다.

# Server-Client Program

- 중앙에서 네트워크를 통해서 개개인에게 데이터를 제공한다. 
- Server 프로그램이 변경되면 Client 프로그램도 대응해서 변경 되어야 한다.
- 하지만, Client 프로그램을 업데이트 또는 제거/설치 하는 과정에서 하위 호환성 문제가 발생할 소지가 있기때문에 프로그램을 업그레이드 하는것이 쉽지 않다. 
- 데이터 전송도 HTTP 프로토콜 보다는 소켓을 통해서 하는것이 일반적인데, 소켓을 이용하면 연결 상태, 데이터의 유효성 등을 일일이 체크 해줘야하는 등 부담이 있다.

# WebServer-Client Program

- Server-Client Program의 불편한 점을 해소하고자 하는 목적으로 발달했다. 
- Browser를 Client로 사용, Server가 화면까지 만들어서 Browser에게 제공한다. 
- Server 단에서 데이터와 데이터가 보여질 화면까지 만들어서 웹문서로 제공하므로 Server 프로그램이 변경 되어도 대응해서 Client인 Browser를 변경 할 필요가 없다.

# Web Server

- 정적 웹페이지를 위한 html, css, js 등 정적 파일(웹문서)을 제공한다.
- Browser는 Client로써 HTTP GET, POST 등을 이용해서 Web Server에 웹페이지를 요청하거나 정보를 저장한다. 
- 하지만, Browser로 부터 전달 받은 정보를 바탕으로 동적으로 화면을 생성해서 제공하는 것은 불가능하다.

> 대표적인 Web Server
> 
> - Apache, nginx
> - Tomcat 은 WAS지만 Web Server 기능을 포함한다.

# Web Application Server

- Web Server + Servlet Container
- 웹을 통해서 Server-Client 프로그램을 만들고자 하는 목적으로 개발 되었다. 
- Client 프로그램 없이, Browser를 Client로 사용한다. 
- Server는 미리 작성된 정적 페이지가 아닌, 사용자의 요청이 있을 때, 동적으로 웹페이지를 만들어서 제공한다. 
- 동적으로 웹페이지를 생성하기 위해서 html 페이지를 print 하는 Server Application(Servlet)을 요청이 들어오면 Servlet Container에서 실행시킨다. 

> 대표적인 WAS
> 
> - Tomcat, jetty, jboss, undertow

## Tomcat

Web Application Server (WAS)

### Web 문서를 제공하는 기능

- `${TOMCAT_HOME}/webapps/ROOT/` 하위에 있는 문서를 제공한다.
	- `localhost:8080/nana.txt`를 호출하면 `${TOMCAT_HOME}/webapps/ROOT/nana.txt` 를 제공한다.

### Servlet을 통해 동적으로 생성된 문서를 제공하는 기능

- `${TOMCAT_HOME}/webapps/ROOT/WEB-INF/classes/${pacakge}/${ClassName}.class`에 위치한 컴파일 된 Servlet 프로그램을 `${TOMCAT_HOME}/webapps/ROOT/WEB-INF/web.xml` 에 정의된 URL -> ClassName mapping 정보에 따라 제공한다.

> - servlet 3.0 이상부터 `web.xml`의 `meatdata-complete="true"`를 `false`로 변경하면 `@WebServlet("/url")`어노테이션을 이용해서 매핑정보를 제공할 수 있다.
> - `WEB-INF` 디렉토리는 서버에 의해서만 사용되는 공간으로 `localhost:8080/WEB-INF/web.xml` 등 과 같이 사용될 수 없다.

# Servlet (Server Application Let)

- 동적으로 페이지를 만들기 위한 Server Application
- Java Interface로 Implement 하고, 필요한 메서드(보통 service(), doGet, doPost())를 Override 해서 사용한다.
- Servlet을 구현한 Class와 호출될 url을 Tomcat의 web.xml에 매핑 시켜서 url이 호출되면 Servlet 클래스의 service() 메소드에 정의된 동작을 수행하게 한다.
- service() 메소드에서는 request의 내용을 확인하고, 미리 생성된 정적인 페이지를 돌려주거나, PrintWriter를 통해서 Browser에 문자열(html 코드)을 Write 할 수 있다.

```
<servlet>
    <servlet-name>hi</servlet-name>
    <servlet-class>com.playground.Hi</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>hi</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```

```
public class Hi extends HttpServlet {
    public void service(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        OutputStream os = response.getOutputStream();
        PrintStream out = new PrintStream(os, true);
        out.println("<html>");
        out.println("</html>");
    }
}
```

## Filter

- Filter Interface를 구현한다.
- Servlet의 실행 전, 후에 수행할 동작들을 정의할 수 있다.
- 필터 동작 수행 후, `chain.dofilter()`를 실행해서 Servlet으로 실행 흐름을 넘겨줄 수 있다.
- 모든 Servlet에 대해서 동작한다.

## ApplicationContext, Session, Cookie

- Servlet은 url이 호출 될 때마다 실행되고 실행 후 종료되기 때문에 stateless 한 특성을 같는다.
- Stateful한 사용자 경험을 위해서 사용자로 부터의 입력(GET이라면 query param, POST라면 body)을 통해서 받은 데이터를 Database에 저장한 후, 사용할 수도 있겠지만, Servlet은 stateful 하게 동작하기 위해서 몇가지 저장소를 사용할 수 있다.  (예를들어 사용자로 부터 입력받은 값이 새로고침 이후에도 입력창에 유지되는것 처럼 보이게 하는 것)

### ApplicationContext

- 동일한 Servlet 간 공유되는 저장 공간

### Session

- WAS에 최초 접속 시 (첫 http request 시) WAS에 Session 저장소를 생성, key로 사용할 session id를 Response header의 cookie에 담아서 돌려준다. (JSESSIONID)
- Browser(Process)가 다르면 다른 session id를 부여한다. (크롬은 창을 새로 띄워도 process가 같아서 같은 session id를 돌려준다.)
- Session id는 쿠키를 열어서 탈취가 가능하다.

### Cookies

- ApplicationContext, session과 달리 Browser의 저장소를 이용한다.
- path: cookie 마다 url을 설정해서 특정 Servlet에 종속 시킬 수 있다.
- maxAge: Browser에 저장하므로 만료시간을 길게 설정 할 수 있다.

# JSP (Java Server Page)

- Jasper를 이용한 Servlet 프로그램
- jsp 문서는 html 문서에 코드 블록을 삽입하는 형태로 구현된다.
- 사용자가 jsp 페이지를 호출하면, Jasper가 해당 jsp 파일을 빌드해서 Servlet Interface를 구현한 Java 파일과 컴파일된 Class 파일을 Tomcat Home에 생성한다.

## 코드 블럭

- jsp 파일 안에서 동적인 데이터를 생성하기 위한 Java 코드를 삽입 할 때 사용한다.
- Jasper는 jsp 파일의 내용을 _jspService()(Servlet의 service())에서 PrintStream 객체를 통해 write 하도록 Java 파일을 빌드한다. 이 때, out 객체를 통해서 write 되지 않고, Java 코드로서 사용되어야 하는 코드를 코드 블록 안에 넣는다.

### 출력 코드

- jsp 파일 내의 일반 문자열
- _jspService()에서 out.write("출력 코드")를 이용해서 출력된다.
- 일반 적인 html 코드

### 스크립트릿(Scriptlet)

- _jspService()에서 out.write("")에 의해서 출력되지 않고 Java 코드로 사용될 코드

```
<% Java Code %>
```

### 표현식(Expression)

- out.write()대신, out.print()를 이용해서 Java 변수의 값을 출력한다.

```
<%= Variable %>
```

### 선언문(Declarations)

- 코드 블럭은 _jspService() 안에 삽입 되므로, 메서드 선언을 위한 코드 블럭을 따로 사용한다. (Java는 메서드 안에 메서드를 정의할 수 없다.)

```
<%!
private int sum(int a, int b) {
	...
}
%>
```

### 지시자(Directives)

- _jspService()의 맨 앞에 오는 코드를 강제한다.

```
<%
response.setCharacterEncoding("UTF-8");
response.setContentType("text/html; charset=UTF-8");
%>
```

- 위와 같이 설정을 위한 코드는 html을 출력하는 out.write 코드보다 먼저 호출 되어야 하는데, Jasper가 이를 보장해 주지 않기 때문에, 이런 경우 코드 지시자 블럭을 이용한다.

```
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
```

## 내장 객체

- Jasper가 만들어낸 객체들
- 코드 블럭 내에서 사용할 수 있다.
- 종류
	- request
	- response
	- pageContext
	- session
	- application
	- config
	- out
	- page
	- ...


# mvc model 1 (Model, View, Controller)

- jsp 내에서 코드 블럭과 출력 코드가 섞이지 않도록 코드를 나누고, 둘 사이에 데이터 공유에 변수를 사용하는 방식
	- jsp 내에서 코드 블럭은 어느 곳에서나 사용 가능하기 때문에 코드 블록을 jsp 파일 내에서 무분별하게 사용하는 경우 읽기힘든 코드를 만들게 되며 이는 프로그램의 유지보수를 어렵게하고 버그를 발생시킬 수 있다.

```
<%--------------- Controller ---------------%>
<%
String num_ = request.getParameter("n");

if (num % 2 != 0) {
	model = "홀수";
} else {
	model = "짝수";
}
%>
<%------------------ View ------------------%>
<!DOCTYPE html>
<html>
	<head>
	</head>
	<body>
		<%=model%>
	</body>
</html>
```

- Controller: 입력, 제어
- View: HTML 코드
- Model: 출력 데이터

# mvc model 2

- 하나의 jsp 파일에서 view와 controller를 분리한 model 1 방식에서 더 나아가 물리적 파일로 분리한 방식
- Controller 역할을 하는 Servlet을 구현하고, View 역할을 하는 JSP(Servlet)을 구현해서 두 Servlet 간의 Model 역할을하는 데이터를 request의 Attribute에 담아서 `RequestDispatcher`를 통해서 포워딩한다.

> Front Controller를 Dispatcher로 사용하는 더욱 발전된 MVC Model 2 형태는 Spring MVC에서 살펴본다.

## EL(Element Language)

- View 역할을 하는 JSP에서 Java 코드 블록인 표현식을 사용하지 않도록 도와준다.
- pageContext 안에 담긴 값은 모두 EL을 사용해서 출력할 수 있다.
- 예제

```
${pageContext.request.method}
${param.n}
${header.accept}
${requestScope.result}
```

# Spring

## Spring MVC

- Servlet + JSP 를 이용한 MVC Model 2 방식에서 Front Controller로서 `DispatcherServlet`을 두어 Controller와 View를 handling 하도록 개선
- Tomcat의 web.xml의 역할 축소
	- `DispatcherServlet`이 해당 어플리케이션으로 들어오는 요청을 모두 핸들링
	- `web.xml`(Tomcat Configuration)에는 모든 url에 대해서 `DispatcherServlet`을 호출하도록 하는 설정만 남기고 `/WEB-INF/despacher-servlet.xml`(Spring Configuration)에서 url 매핑을 포함한 다른 모든 설정을 관리하므로써 Tomcat에 대한 의존성을 낮춤
- Controller를 Servlet이 아닌 POJO로 개발하여 Controller의 재사용성을 높일 수 있음

![mvc.png](:/5c8aa02d0dd64953980ab48cb0ad1460)

### 요청 처리 순서

1. DispatcherServlet가 브라우저로부터 요청을 받는다.
1. DispatcherServlet은 요청된 URL을 HandlerMapping 오브젝트에 넘겨 호출 대상이되는 컨트롤러의 오브젝트를 얻어 URL에 해당하는 메서드를 실행한다.
1. 컨트롤러는 비즈니스 로직으로 처리를 실행하고, 그 결과를 바탕으로 뷰에 전달할 데이터를 Model 오브젝트에 저장한다. 끝으로 컨트롤러 오브젝트는 처리 결과에 맞는 View 이름을 반환한다.
1. DispatcherServlet은 컨트롤러에서 반환된 View 이름을 ViewResolver에 전달해서 View 오브젝트를 얻는다.
1. DispatcherServlet은 View 오브젝트에 화면 표시를 의뢰한다.
1. View 오브젝트는 해당하는 뷰를 호출해서 화면 표시를 의뢰한다.
1. 뷰는 Model 오브젝트에서 화면 표시에 필요한 오브젝트를 가져와 화면 표시 처리한다.
1. 뷰를 브라우저로 응답한다.

> 컴포넌트 
> 
> - DispatcherServlet : 프런트 컨트롤러 담당, 모든 HTTP 요청을 받아들여 그 밖의 오브젝트 사이의 흐름을 제어, 기본적으로 스프링 MVC의 DispatcherServlet 클래스를 그대로 적용
> - HandlerMapping : 클라이언트의 요청을 바탕으로 어느 컨트롤러를 실행할지 결정
> - Model : 컨트롤러에서 뷰로 넘겨줄 오브젝트를 저장하기 위한 오브젝트, HttpServletRequest와 HttpSession처럼 String 형 키와 오브젝트를 연결해서 오브젝트를 유지
> - ViewResolver : View 이름을 바탕으로 View 오브젝트를 결정
> - View : 뷰에 화면 표시 처리를 의뢰
> - 비즈니스 로직 : 비즈니스 로직을 실행. 애플리케이션 개발자가 비즈니스 처리 사양에 맞게 작성
> - 컨트롤러(Controller) : 클라이언트 요청에 맞는 프레젠테이션 층의 애플리케이션 처리를 실행해야 함. 애플리케이션 개발자가 애플리케이션 처리 사양에 맞게 작성
> - 뷰 / JSP 등 : 클라이언트에 대해 화면 표시 처리. 자바에서는 JSP 등으로 작성하는 일이 많으며, 애플리케이션 개발자가 화면의 사양에 맞게 작성

> 용어
> 
> - POJO(Plain Old Java Object): Servlet이 아닌 일반적인 Java Object 를 나타내는 말
> - Dispatch: 보내다, 파견하다

## vs Java EE (Enterprise Edition)

- Enterprise Application을 만들 때 DI, Transaction 관리가 중요함
- Java EE 를 이용해서 DI, Transaction 관리를 처리하는 Enterprise Application 구현 가능하지만 복잡도가 높았다.
- Spring은 이것들을 library만 이용해서 간단하게 구현 가능하도록 하므로써 Java EE를 대체하게 됨

> Transaction 관리
> 
> - JDBC를 이용해서 Transacion을 관리할 수 있지만, 
> - DAO(Repository), Service layer가 있으면, Transaction이 끊어지는 함수들을 묶어서 하나의 Transaction으로 관리해야 하는 경우가 생김
> - 이런 경우 JDBC가 제공하는 connection을 공유하기가 어렵고, Transaction 관리가 불가능해짐

## DI (for MVC)

### DI(Dependency Injection)

- 의존성 주입 -> 부품 조립
- Setter Injection

```
B b = new B();
A a = new A();
a.setB(b);
```

- Construction Injection

```
B b = new B();
A a = new A(b);
```

### IoC Container (Inversion of Control)

- 부품 조립을 하는 컨테이너
- Composition HAS-A 방식에서 참조되는 객체가 생성되는 순서의 역순으로 객체가 생성하고 조립하기 때문에 "Inversion of Control" Container 라고 부름

> Composition HAS-A
> 
> ```
> class A {
> 	private B b;
> 	public A() {
> 		b = new B();
> 	}
> }
> ```

> Association HAS-A
> 
> ```
> class A {
> 	private B b;
> 	public A() {
> 	}
> 	public void setB(B b) {
>		this.b = b;
> 	}
> }
> ```

### DI in Spring

#### XML

- `ClassPathXmlApplicationContext`, `FileSystemXmlApplicationContext`, `XmlWebApplicationContext`를 통해서 설정을 `ApplicationContext`로 읽어들인다.

```
<bean id="b" class="B"/>
<bean id="a" class="A">
	// name="b"는 class A에 setB()가 존재해야 사용 가능
	<property name="b" ref="b"/>
</>
```

#### Annotation

> Annotation은 컴파일 후에도 class 파일에 남는 메타데이터로 실행환경에 의해서 읽어질 수 있다.

- `AnnotationConfigApplicationContext`를 통해서 설정을 `ApplicationContext`로 읽어들인다.
- XML 방식에서는 B2 클래스를 추가하면 XML도 수정해야하기 때문에 클래스 내부에 설정을 Annotation으로 추가해서 설정 XML을 제거할 수 있다.

```
// 설정 중, Annotation으로 된 것들이 있으니 찾아봐야 함을 지시함
<context:component-scan base-package="com.your.root.package"/> // @Configuration, @ConponentScan 으로 대체
<bean id="b" class="B"/> // @Bean으로 대체
<bean id="a" class="A"> // @Component(=Service, Controller, Repository)로 대체
	<property name="b" ref="b"/> // @Autowired로 대체
</>
```

## AOP (Aspect Oriented Programming) (for Transaction 관리)

- 관점 지향 프로그래밍
- 사용자가 관심있는 주 업무 로직(Primary/Core Concern)과 부수적인 개발자/운영자가 관심있는 부수적인 로직(Cross-cutting Concern)을 각자의 관점에 맞게 나누어서 프로그래밍 하는 개발 방법론
- Cross-cutting Concern은 로그 처리, 보안 처리, 트랜젝션 처리와 같은 작업들로, 주 업무 로직의 Before, After에서 주로 수행되어야 하는 작업들을 말한다.
- Before, After 로직들은 Proxy에 의해서 주 업무 로직 앞뒤에 배치될 수 있다.

 ### Java Proxy

- java의 기본 기능으로도 AOP를 구현할 수 있다. 

```
import java.lang.reflect.Proxy;

A a = new AA();
A proxy = Proxy.newProxyInstance(AA.class.getClassLoader(), 
new Class[] {A.class}, 
new InvocationHandler() {
    @Override
    public Object Invoke(Object proxy, Method method, Object[] args) {
        // before
        Object result = method.invoke(a, args);
        // after
        return result;
    }
}
```

> proxy: 가짜

### Spring AOP Advise

- Spring의 DI와 AOP 라이브러리를 통해서 AOP를 구현 할 수 있다.

```
<bean id="target" class="AA" />

<bean id="aroundAdvise" class="AroundAdvise" />
<bean id="beforeAdvise" class="BeforeAdvise" />
<bean id="afterAdvise" class="AfterReturningAdvise" />
<bean id="throwAdvise" class="ThrowAdvise" />

<bean id="proxy" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="target" />
    <property name="intercepterNames">
        <list>
            <value>aroundAdvise</value>
            <value>beforeAdvise</value>
            <value>afterAdvise</value>
            <value>throwAdvise</value>
        </list>
    </property>
</bean>
```

```
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

public class AroundAdvise implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        log.info("before");
        Object result = invocation.proceed();
        log.info("after");
    }
}

import java.lng.reflect.Method;
import org.springframework.aop.MethodBeforeAdvise;

public class BeforeAdvise implements MethodBeforeAdvise {
    @Override
    public Object before(Method method, Object[] args, Object target) throws Throwable {
        log.info("before");
    }
}

A proxy = context.getBean("proxy");
```

### Pointcuts, JoinPoint, Weaving

- Proxy는 모든 JoinPoint(메소드들)와 Weaving(Core Concern에 Cross Concern을 붙이는 것) 하는데, 특정 JoinPoint 만 Weaving 하도록 하는것을 PointCut 이라고한다.
- Pointcuts 은 xml 또는 Annotation을 사용해서 설정할 수 있다.

---

--- 이하 업데이트 예정 ---

## Servlet Filter (for 인증, 권한)


# Spring boot

# Client Side Rendering

