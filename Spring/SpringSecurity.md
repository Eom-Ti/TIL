# SpringSecurity
기본적으로  대부분의 시스템은 회원에 대한 관리가 필요하며, 그에 따른 `인증` 과 `인가` 에 대한 처리가 필요하다. 이를 위해 Spring은 `Spring Security` 라는 프레임워크를 통해 관련된 기능을 제공하고있다.

Spring Security는 Spring의 하위 프레임워크로 개발자가 쉽게 확장 가능할 수있으며, Spring Security의 Servlet 지원은 Servlet `Filter`를 기반으로 동작하는 방식이다.

## Filter

[Filter (Jakarta Servlet API documentation)](https://jakarta.ee/specifications/servlet/4.0/apidocs/javax/servlet/filter)

문서를 보면 필터는 리소스(서블릿 또는 정적 콘텐츠)에 대한 요청, 리소스의 응답 또는 둘 다에 대한 필터링 작업을 수행하는 객체이며. `doFilter` 메서드에서 필터링을 수행한다고 되어있다.

기본적으로 아래와 같은 메소드를 제공하고 있다.

| Modifier and Type | Method | Description |
| --- | --- | --- |
| default void | destroy() | 웹 컨테이너가 필터를 서비스에서 제거할 때 호출됩니다. |
| void | doFilter(ServletRequest request, ServletResponse response, FilterChain chain) | 클라이언트가 체인의 끝에 있는 리소스를 요청할 때마다 컨테이너가 호출합니다. |
| default void | init(FilterConfig filterConfig) | 웹 컨테이너가 필터를 서비스에 배치할 때 호출됩니다. |

위와 같은 메소드들을 통해 Filter의 수행이 가능하며, default 인 `destroy() / init()` 을 제외한 `doFilter()` 의 경우 필수 구현이 필요하다.

`doFilter` 메소드를 보면 `FilterChain` 이 파라미터로 넘어오게 되고, 이를 통해 다음 Filter로 요청과 응답을 전달 할 수 있도록 한다. 혹은 다음 체인에 응답을 전달하지 않음으로써 요청 처리를 차단할 수도 있다.
