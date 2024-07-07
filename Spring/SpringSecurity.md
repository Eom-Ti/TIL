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

이처럼 `Filter`는 웹 애플리케이션에서 요청이나 응답을 가로채어 무언가 작업을 수행하는 객체로 사용자 요청을 특정 서블릿이나 정적 콘텐츠로 전달하기 전에 필터가 요청을 가로채어 필요한 전처리 작업을 할 수 있다. 또는 서블릿이 응답을 클라이언트로 보내기 전에 필터가 응답을 가로채어 후처리 작업을 할 수도 있다.

`Servlet Filter` 는 `Dispatcher Servlet` 에 요청이 전달 되기 전/후에 지정한 URL 패턴에 맞는 모든 요청에 대한 부가 작업을 제공하며, `Dispatcher Servlet` 의 경우 스프링의 가장 앞단에 존재하는 프론트 컨트롤러로 결국 `Filter` 는 Spring의 범위 밖에서 처리가 된다.

이를 간단하게 도식화 한다면 아래와 같다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/e7ecc13f-ccf0-40b1-b73e-16b8fc4f4abc)

쉽게 말해 `Dispatcher Servlet`에 요청이 전달 되기 `전/후 과정`에서 **부가적인** 작업을 처리하는 객체이다.

이러한 `Filter`는 공통된 보안 및 인증/인가 관련 작업에 주로 사용되거나, 모든 요청에 대한 로깅 또는 감사등에 많이 쓰이고 있다. 즉 `스프링과 무관하게 전역적으로 처리해야 하는 작업`에 대한 처리를 담당하는 객체이다.
