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
## Servlet & Security **Architecture**

위에서 `Filter`에 대해 알아본것 처럼 Filter는 Web Context(Servlet Container) 에서 동작하며, `DispatcherServlet` 에 요청이 전달 되기 `전/후` 과정에서 동작한다.

이 말은 서블릿 컨테이너의 경우 자체 표준을 사용하여 필터 인스턴스를 등록할 수 있지만, `Spring` 에서 정의한 `Bean` 은 인식하지 못함을 의미한다.

### DelegatingFilterProxy

따라서 Spring에서 정의된 Bean을 사용하기 위해선 중간 다리가 필요하며, 이를 담당하는 것이 `DelegatingFilterProxy` 이다.

이를 통해 Spring `ApplicationContext` 에 등록된 실제 Filter Bean을 탐색하여 요청을 해당 Bean에 위임하여 처리가 가능하다. 이후 처리된 결과는 다시 `Filter Chain`을 통해 `Servlet Container`로 돌아온다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/4a20914a-3749-4bfd-bebd-20c66e510d09)

```kotlin
fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
	val delegate: Filter = getFilterBean(someBeanName) 
	delegate.doFilter(request, response) 
}
```

실제 Servlet Container의 경우 애플리케이션 시작 시 `Filter` 를 등록해야 한다. 그러나 Spring의 `ApplicationContext` 는 그보다 더 늦게 초기화가 진행된다.

위의 설명 대로라면 실제 `DelegatingFilterProxy` 에서 Spring Filter Bean에게 위임하는 것이 불가능 해보일 수 있다. 하지만, `DelegatingFilterProxy` 는 `Filter` 등록 시점을 **지연시켜 이를 해결**한다. 즉 첫 요청이 들어올 때까지 Spring Bean을 조회하지 않아도 되기 때문에 `ApplicationContext` 가 초기화 된 후에 사용이 가능하다.

DelegatigFilterProxy를 통해 Spring Filter Bean에 위임할 수 있는거 까진 확인 했다. 그럼 어떤 방식으로 `Spring Security`는 동작하는지 알아보자.

### FilterChainProxy

`DelegatingFilterProxy` 는 위의 설명처럼 Spring Filter Bean에 작업을 위임할 수 있다. 이때 `FilterChainProxy` 를 통해 위임하며, 해당 필터는 `Srping Security` 에서 제공하는 필터로 여러 보안 관련 필터 인스턴스를 `SecurityFilterChain` 을 통해 관리하고 요청이 들어오면 적절한 Filter Chain을 통해 위임한다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/bef52527-a163-4083-9316-12ec82a9faa2)

이때 `SecurityFilterChain` 은 현재 요청에 대해 호출해야 하는 Spring 보안 필터 인스턴스를 결정하기 위해 `FilterChainProxy`에서 사용된다.

### SecurityFilterChain

SecurityFilterChain의 보안 필터는 일반적으로 `Srpring Bean` 이며, `FilterChainProxy`에 등록된다.

이를 통해 `Spring Security`의 모든 서블릿 지원을 위한 시작점을 제공하며, `HttpFirewall` 적용, 또한 URL만을 기반으로 호출되던 `Servlet  Context의 Filter` 와 다르게 `HttpServletRequest` 의 모든 것을 기반으로 호출을 결정할 수 있다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/1848134e-f0c4-4842-bd3c-cb7507a1e247)

이때 위의 도식표처럼 여러개의 `SecurityFilterChain`이 등록되어 있을 때 만일 `api/messages` 라는 URL이 요청되면, `SecurityFilterChain0` 의 조건에 먼저 일치하기 때문에 `SecurityFilterChainn` 에서 일치하더라도 `0` 만 호출된다.  반대로 `/message` URL 형식이라면  `n` 외엔 일치하는 `FilterChain`이 없기 때문에 `n` 이 호출된다.

또한 각 `SecurityFilterChain` 마다 보안 필터 인스턴스를 각각 구성할 수 있으며, 만일 특정 요청을 무시하도록 하려는 경우 보안 필터 인스턴스가 `0개` 가 있을 수도 있다.

### Security Filters

`Security Filter`는 `SecurityFilterChain` 을 통해 `FilterChainProxy` 에 삽입된다. 이러한 `Security Filter` 는 `Authentication(인증)` , `Authorization(인가)` 등 다양한 용도로 사용할 수 있다.

이러한 `Security Filter` 는 특정 순서로 실행되어 적시에 호출되는 것을 보장한다. 예를 들어 인증을 수행하는 필터는 인가를 수행하는 필터보다 먼저 호출된다. 일반적으로 이 순서를 알 필요는 없으나, 필요한 경우가 있다. 이는 아래의 링크를 통해 확인이 가능하다.

https://github.com/spring-projects/spring-security/blob/6.3.1/config/src/main/java/org/springframework/security/config/annotation/web/builders/FilterOrderRegistration.java

지금 까지 기본적인 Filter 부터 Servlet Filter 그리고 이를 Spring Security Filter 와는 어떻게 활용하는지를 알아 봤다 최종적으로 도식화 한다면 아래와 같을 것이다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/a0d77228-837b-4c6a-a86c-5187fec12b29)
### Security 설정

기본적인 Spring Security 를 활용하여 JWT 토큰 인증 까지의 과정을 간단하게 프로젝트로 진행해 볼 예정이다. 다만 여기서 프로젝트는 `Kotlin` 기반으로 작성되어 있으며, `Java` 의 경우 설정 단계 까지만 작성하고 이후 구현은 따로 작성하지 않을 예정이다.

현재는 우선 Spring Security를 적용 하는 단계 까지만 알아볼 예정이다.

```kotlin
implementation("org.springframework.boot:spring-boot-starter-security")
```

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig {

    @Bean
    fun filerChain(http: HttpSecurity): SecurityFilterChain {
        http {
            csrf { disable() }
            httpBasic { disable() }
            formLogin { disable() }
            sessionManagement { sessionCreationPolicy = SessionCreationPolicy.NEVER }
            authorizeRequests {
                authorize(anyRequest, authenticated)
            }
        }
        return http.build()
    }
}
```

``` text
⚠️ IDE가 자동으로 import를 해주지 않아 `org.springframework.security.config.annotation.web.invoke` 를 별도로 import 해서 사용이 필요하다.
```

간단 하게 위에 대해 설명하자면 현재 해당 프로젝트는 `rest api` 를 이용한 서버로 Session을 사용하지 않는 stateless 한 인증 방식으로 `JWT` 를 사용할 예정이기에 별도로 작성할 필요가 없다. 마찬가지로 `httpBasic` 과 `formLogin` 또한 동일하며, `stateless` 하기에 Session을 사용하지 않는다.

## Autentication Architecutre

위에서 설명한 `SecurityFilterChain` 을 통해 우리는 `인증` 과 관련된 부분을 처리할 수 있으며, 이러한 인증과 관련된 프로세스는 아래와 같은 순서로 진행된다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/343f4509-5e48-40c4-8143-d2a7c25c2243)

구체적인 구현을 하기전 우선 각 구성요소들에 대해 간단히 알아보고 시작할 필요가 있다.

### SecurityContextHolder

Spring Security의 인증 모델의 핵심이며, 여기엔 `SecurityContext` 가 포함되어 있다.

`SecurityContextHolder` 에는 인증된 사용자에 대한 세부 정보를 저장하는 곳으로, 값이 포함되어 있으면 현재 인증된 사용자로 사용된다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/c3d720d7-01ee-4da5-97cf-cbd0615eba83)

기본적으로 `SecurityContextHolder` 는 `ThreadLocal` 을 사용하여 세부 정보를 저장하기 때문에, 명시적으로 전달되지 않더라도 **동일한 스레드**에 있는 메서드에서 항상 `SecurityContextHolder` 를 사용할 수 있음을 의미한다.

### Authentication

`Authentication Iterface`는 Spring Security 내에서 두 가지 주요 용도로 사용된다.

- 사용자가 인증을 위해 제공한 자격 증명을 제공하기 위해 `AuthenticationManager`에 입력 한다.
- 현재 인증된 사용자를 나타냅니다. `SecurityContext`에서 현재 인증을 가져올 수 있다.

또한 `Authentication Iterface` 는 아래의 3개의 구성요소를 포함하고 있다.

1. `principal` : 사용자를 식별한다.
2. `credentials` : 비밀번호인 경우가 많다.
3. `authorities` :  사용자에게 부여되는 상위 수준의 권한이다.

### **GrantedAuthority**

`ROLE_XXX` 와같은 권한에 해당된다. 이러한 권한은 UserName/Password 기반의 인증을 사용할 때 `UserDetailsService`에 의해 로드된다.

### AuthenticationManager

`AuthenticationManager`는 Spring Security의 필터가 인증을 수행하는 방법을 정의하는 API이다.

그런 다음 반환되는 인증은 `SecurityContextHolder`에 저장하여 보안 컨텍스트를 관리한다.

물론 `AuthenticationManager` 를 사용하지 않고 직접 `SecurityContextHolder`에 설정할 수 있다.

`AuthenticationManager`의 구현은 다양할 수 있지만, 가장 일반적인 구현체는 `ProviderManager` 이다.

### ProviderManager

가장 일반적으로 사용되는 `AuthenticationManager` 의 구현체로 `인증성공` `인증실패` `다음 AuthenticationProvier` 에 위임 중 하나를 수행한다.

모든 AuthenticationProvider가 인증을 할 수 없으면, ProviderManager는ProviderNotFoundException 예외를 발생시키며, 이는 제공된 인증 유형을 지원하는 AuthenticationProvider가 없음을 의미 한다.

### **AuthenticationProvider**

`ProviderManager`에 여러 인증 공급자 인스턴스를 삽입할 수 있다. 각 `AuthenticationProvider` 는 특정 유형의 인증을 수행한다. 예를 들어 `DaoAuthenticationProvider`는 사용자 이름/비밀번호 기반 인증을 지원하며, `JwtAuthenticationProvider`는 JWT 토큰 인증을 지원한다.

## 인증구현

사용자를 인증하는 가장 일반적인 방법 중 하나는 `사용자 이름`과 `비밀번호`의 유효성을 검사하는 것이다. Spring Security는 사용자 이름 및 비밀번호 인증을 위한 포괄적인 지원을 제공한다.

관련하여 Srping Security는 여러 인증 방법을 제공하고 있으며, 해당 내용은 아래와 같다.

- I want to [learn how Form Login works](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html)
- I want to [learn how HTTP Basic authentication works](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html)
- I want to [learn how `DaoAuthenticationProvider` works](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html)
- I want to [manage users in memory](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/in-memory.html)
- I want to [manage users in a database](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/jdbc.html)
- I want to [manage users in LDAP](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/ldap.html#servlet-authentication-ldap-authentication)
- I want to [publish an `AuthenticationManager` bean](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/index.html#publish-authentication-manager-bean) for custom authentication
- I want to [customize the global `AuthenticationManager`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/index.html#customize-global-authentication-manager)

이중 `Form Login` 을 사용하여 인증하는 방식을 확인해 볼 예정이며, 각 상황에 따라 여러 인증방법을 구현할 수가 있다. 꼭 위의 방법을 하지 않더라도 여러 구현 방법이 있으며 그러한 방법은 이후에 확인 예정이다

Spring Security는 기본적으로 세션 쿠키 방식의 인증이 이루어 지며, 이때 `UsernamePasswordAuthenticationFilter`  를사용하여 인증을 수행한다. 해당 Filter의 `attemptAuthentication()` 메소드를 보면 `UsernamePasswordAuthenticationToken` 클래스를 생성하는 것을 볼 수 있으며, 이때 인증되지 않은 `Authentication` 을 생성한다.

```kotlin
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        if (this.postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        } else {
            String username = this.obtainUsername(request);
            username = username != null ? username.trim() : "";
            String password = this.obtainPassword(request);
            password = password != null ? password : "";
            UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username, password);
            this.setDetails(request, authRequest);
            return this.getAuthenticationManager().authenticate(authRequest);
        }
    }
```

이후 `Spring Authentication` 프로세스대로 `AutenticationManager` 로 넘어가는 부분을 확인해보자
