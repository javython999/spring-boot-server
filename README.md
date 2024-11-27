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
