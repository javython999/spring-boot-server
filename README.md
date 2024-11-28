# 웹 서버와 서블릿 컨테이너
## JAR, WAR
### JAR
자바는 여러 클래스와 리소스를 묶어서 `JAR`(Java Archive)라고 하는 압축 파일을 만들 수 있다.
이 파일은 JVM 위에서 직접 실행되거나 또는 다른 곳에서 사용하는 라이브러리로 제공된다.
직접 실행하는 경우 `main()` 메서드가 필요하고, `MANIFEST.MF`파일에 실행할 메인 메서드가 있는 클래스를 지정해 두어야 한다.

### WAR
WAR(Web Application Archive)라는 이름에서 알 수 있듯 WAR 파일은 웹 애플리케이션 서버(WAS)에 배포할 때 사용하는 파일이다.
JAR 파일이 JVM 위에서 실행된다면, WAR는 웹 애플리케이션 서버 위에서 실행된다.
웹 애플리케이션 서버 위에서 실행되고, HTML 같은 정적 리소스와 클래스 파일을 모두 함께 포함하기 때문에 JAR와 비교해서 구조가 더 복잡하다.

#### WAR 구조
* `WEB-INF`
  * `classes`: 실행 클래스 모음
  * `lib`: 라이브러리 모음
  * `web.xml` 웹 서버 배치 설정 파일(생략가능)
* `index.html`: 정적 리소스

`WEB-INF` 폴더 하위는 자바 클래스와 라이브러리, 그리고 설정 정보가 들어가는 곳이다.
`WEB-INF`를 제외한 나머지 영역은 HTML, CSS 같은 정적 리소스가 사용되는 영역이다.
---
## 서블릿 컨테이너 초기화
서블릿은 `ServletContainerInitializer`라는 초기화 인터페이스를 제공한다.
이름 그대로 서블릿 컨테이너를 초기화 하는 기능을 제공한다.
서블릿 컨테이너는 실행 시점에 초기화 메서드인 `onStartup()`을 호출해준다.
여기에서 애플리케이션에 필요한 기능들을 초기화 하거나 등록할 수 있다.

ServletContainerInitializer
```java
public interface ServletContainerInitializer {
    void onStartup(Set<Class<?>> var1, ServletContext var2) throws ServletException;
}
```
* `Set<Class<?>> var1`: 조금 더 유연한 초기화 기능을 제공한다. `@HandlesTypes` 애노테이션과 함께 사용한다.
* `ServletContext var2`: 서블릿 컨테이너 자체의 기능을 제공한다. 이 객체를 통해 필터나 서블릿을 등록할 수 있다.

`ServletContainerInitializer` 인터페이스를 간단히 구현해 실제 동작하는지 확인해보자.
```java
public class MyContainerInitV1 implements ServletContainerInitializer {
    
    @Override
    public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
        System.out.println("MyContainerInitV1.onStartup");
        System.out.println("set = " + set);
        System.out.println("servletContext = " + servletContext);
    }
}
```
다음 경로에 파일 생성도 필요하다.
` resources/META-INF/services/jakarta.servlet.ServletContainerInitializer`   
해당 파일 안에 `MyContainerInitV1` 클래스를 패키지 경로를 포함해서 지정해준다.
`hello.container.MyContainerInitV1`

서블릿 컨테이너 초기화를 조금 더 자세히 알아보자.
서블릿을 등록하는 방법은 2가지가 있다.
* `@WebServlet` 애노테이션
* 프로그래밍 방식

애플리케이션 초기화
서블릿 컨테이너는 조금 더 유연한 초기화 기능을 지원한다.
AppInit
```java
import jakarta.servlet.ServletContext;

public interface AppInit { 
    void onStartup(ServletContext servletContext);
}
```
* 애플리케이션 초기화를 진행하려면 먼저 인터페이스를 만들어야 한다. 내용과 형식은 상관 없고, 인터페이스는 꼭 필요하다.

AppInitV1Servlet
```java
import hello.servlet.HelloServlet;
import jakarta.servlet.ServletContext;
import jakarta.servlet.ServletRegistration;
 
public class AppInitV1Servlet implements AppInit {
    @Override 
    public void onStartup(ServletContext servletContext) {
        System.out.println("AppInitV1Servlet.onStartup");
        
        //순수 서블릿 코드 등록
        ServletRegistration.Dynamic helloServlet = servletContext.addServlet("helloServlet", new HelloServlet());
        helloServlet.addMapping("/hello-servlet");
    }
 }
```
* 프로그래밍 방식으로 `HelloServlet`서블릿을 서블릿 컨테이너에 직접 등록한다.
* HTTP로 `/hello-servlet`을 호출하면 `HelloServlet`서블릿이 실행된다.

서블릿 컨테이너 초기화(`AppInit`)는 어떻게 실행되는 것일까?
MyContainerV2
```java
import jakarta.servlet.ServletContainerInitializer;
import jakarta.servlet.ServletContext;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.HandlesTypes;
import java.util.Set;

@HandlesTypes(AppInit.class)
public class MyContainerInitV2 implements ServletContainerInitializer {
    @Override 
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        System.out.println("MyContainerInitV2.onStartup");
        System.out.println("MyContainerInitV2 c = " + c);
        System.out.println("MyContainerInitV2 container = " + ctx);
        for (Class<?> appInitClass : c) {
            try {
                //new AppInitV1Servlet()과 같은 코드
                AppInit appInit = (AppInit) 
                appInitClass.getDeclaredConstructor().newInstance();
                appInit.onStartup(ctx);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
 }
```
애플리케이션 초기화 과정
1. `@HandlesTypes` 애노테이션에 애플리케이션 초기화 인터페이스를 지정한다.
   * 여기서는 앞서 만든 `AppInit.class` 인터페이스를 지정
2. 서블릿 컨테이너 초기화(`ServletContainerInitalizer`)는 파라미터로 넘어오는 `Set<Class<?>> c` 애플리케이션 초기화 인터페이스의 구현체들을 모두 찾아서 클래스 정보로 전달한다.
  * 여기서는 `@HandlesTypes(AppInit.class)`를 지정했으므로 `AppInit.class`의 구현체인 `AppInitV1Servlet.class` 정보가 전달 된다.
  * 참고로 객체 인스턴스가 아니라 클래스 정보를 전달하기 때문에 실행하려면 객체를 생성해서 사용해야 한다.
3. `appInitClass.getDeclaredConstructor().newInstance()`
  * 리플랙션을 사용해서 객체를 생성한다.
4. `appInit.onStartup(ctx)`
  * 애플리케이션 초기화 코드를 직접 실행하면서 서블릿 컨테이너 정보가 담긴 `ctx`도 함께 전달한다.

MyContainerV2 등록
`MyContainerInitV2`를 실행하려면 서블릿 컨테이너에게 알려주어야 한다.
`resources/META-INF/services/jakarta.servlet.ServletContainerInitializer` 파일에 추가한다.

초기화는 다음 순서로 진행 된다.
1. 서블릿 컨테이너 초기화 실행
   * `resources/META-INF/services/jakarta.servlet.ServletContainerInitializer`
2. 애플리케이션 초기화 실행
   * `@HandlesTypes(AppInit.class)`

#### 스프링 컨테이너 등록
AppInitV2Spring
```java
import hello.spring.HelloConfig;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;
import jakarta.servlet.ServletContext;
import jakarta.servlet.ServletRegistration;

public class AppInitV2Spring implements AppInit {
    @Override
    public void onStartup(ServletContext servletContext) {
        System.out.println("AppInitV2Spring.onStartup");
        
        //스프링 컨테이너 생성
        AnnotationConfigWebApplicationContext appContext = new AnnotationConfigWebApplicationContext();
        appContext.register(HelloConfig.class);
 
        //스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
        DispatcherServlet dispatcher = new DispatcherServlet(appContext);
 
        //디스패처 서블릿을 서블릿 컨테이너에 등록 (이름 주의! dispatcherV2)
        ServletRegistration.Dynamic servlet = servletContext.addServlet("dispatcherV2", dispatcher);
 
        // /spring/* 요청이 디스패처 서블릿을 통하도록 설정
        servlet.addMapping("/spring/*");
    }
 }
```
* `AppInitV2Spring`은 `AppInit`을 구현했다. `AppInit`을 구현하면 애플리케이션 초기화 코드가 자동으로 실행된다.

#### 스프링 컨테이너 생성
* `AnnotationConfigWebApplicationContext`가 바로 스프링 컨테이너이다.
  * `AnnotationConfigWebApplicationContext` 부모를 따라가다 보면 `ApplicationContext` 인터페이스를 확인할 수 있다.
  * 이 구현체는 이름 그대로 애노테이션 기반 설정과 앱 기능을 지원하는 스프링 컨테이너로 이해하면 된다.
* `appContext.register(HelloConfig.class)`
  * 컨테이너에 스프링 설정을 추가 한다.

#### 스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
* `new DispatcherServlet(appContext)`
* 스프링 MVC가 제공하는 디스패처 서블릿을 생성하고, 생성자에 앞서 만든 스프링 컨테이너를 전달한다. 이렇게 하면 디스패처 서블릿과 스프링 컨테이너가 연결된다.
* 이 디스패처 서블릿에 HTTP 요청이 오면 디스패처 서블릿은 해당 스프링 컨테이너에 들어있는 컨트롤러 빈들을 호출한다.

#### 스프링 MVC 서블릿 컨테이너 초기화 지원
지금까지의 과정을 생각해보면 서블릿 컨테이너를 초기화 하기 위해 복잡한 과정을 진행했다.
* `ServletContainerInitializer` 인터페이스를 구현해서 서블릿 컨테이너 초기화 코드를 만들었다.
* 애플리케이션 초기화를 위해 `@HandlesTypes` 애노테이션을 적용
* `/META-INF/services/jakarta.servlet.ServletContainerInitalizer` 파일에 서블릿 컨테이너 초기화 클래스 경로를 등록

스프링 MVC는 이러한 서블릿 컨테이너 초기화 작업을 이미 만들어 두었다. 덕분에 서블릿 컨테이너 초기화 과정은 생략하고, 애플리케이션 초기화 코드만 작성하면된다.
스프링이 지원하는 애플리케이션 초기화를 사용하라면 다음 인터페이스를 구현하면된다.

WebApplicationInitializer
```java
package org.springframework.web;

public interface WebApplicationInitializer { 
    void onStartup(ServletContext servletContext) throws ServletException;
}
```

AppInitV3SpringMvc
```java
import hello.spring.HelloConfig;
import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;
import jakarta.servlet.ServletContext;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRegistration;

public class AppInitV3SpringMvc implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        System.out.println("AppInitV3SpringMvc.onStartup");
        
        //스프링 컨테이너 생성
        AnnotationConfigWebApplicationContext appContext = new 
        AnnotationConfigWebApplicationContext();
        appContext.register(HelloConfig.class);
        
        //스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
        DispatcherServlet dispatcher = new DispatcherServlet(appContext);
 
        //디스패처 서블릿을 서블릿 컨테이너에 등록 (이름 주의! dispatcherV3)
        ServletRegistration.Dynamic servlet = servletContext.addServlet("dispatcherV3", dispatcher);
 
        //모든 요청이 디스패처 서블릿을 통하도록 설정
        servlet.addMapping("/");
    }
 }
```
* `WebApplicationInitializer` 인터페이스를 구현한 부분을 제외하고는 이전의 `AppInitV2Spring`과 거의 같은 코드이다.
  * `WebApplicationInitializer`는 스프링이 이미 만들어둔 애플리케이션 초기화 인터페이스이다.
* `servlet.addMapping("/")` 코드를 통해 모든 요청이 해당 서블릿을 타도록 했다.