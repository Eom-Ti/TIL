# Spring Bean Life Cycle
# Overview

`Spring Container` 는 `Bean` 의 생성과 소멸 같은 생명주기(`Life Cycel`)를 관리하며, 생성된 자바 객체들에게 추가적인 기능을 제공하는 역할을 한다.

## Bean Life Cycle

Bean Life Cycle 이란 아래와 같은 것을 의미한다.

- Spring Bean은 Java 또는 XML 정의를 기반으로 컨테이너 시작시 인스턴스화 되어야 한다.
- Bean을 사용 가능한 상태로 만들기 위해 사전, 사후 초기화 단계가 필요할 수도 있다.
- 그 후 Bena이 더 이상 필요하지 않으면 `IOC Container` 에서 제거된다.
- 다른 시스템 리소스를 해제 하기 위해 사전 및 사후 소멸 단계를 수행해야 할 수도 있다.

즉 쉽게 말해 Bean Life Cycle 이란 해당 객체가 언제, 어떻게 생성되어 소멸되기 전까지 어떤 작업을 수행하고 언제, 어떻게 소멸되는지 일련의 과정을 이르는 말이다.

Spring Container는 이런 빈 객체의 생명주기를 컨테이너의 생명주기 내에서 관리하고 객체 생성이나 소멸 시 호출될 수 있는 **콜백 메서드**를 제공하고 있다.

### Spring Container

- 초기화 : Bean을 등록, 생성, 주입
- 종료 : Bean 객체들을 소멸

### 콜백

콜백함수를 부를 때 사용되는 용어이며 콜백함수를 등록하면 특정 이벤트가 발생했을 때 해당 메소드가 호출됨 즉, 조건에 따라 실행될 수도 실행되지 않을 수도 있는 개념

이를 토대로 간단히 `Bean Life Cycle` 을 요약하면 아래와 같다.

![image](https://github.com/user-attachments/assets/10ea87aa-f166-4912-8e1f-97275471a004)

1. Spring Container 생성
2. Spring Bean 인스턴스화(빈 생성)
3. 의존성 주입
4. 초기화 Callback(`init()`) 호출
5. 사용
6. 소멸전 콜백(`destroy()`) 호출
7. 종료

## **Spring 빈 생명주기 콜백**

Spring Framework는 빈 라이프 사이클을 제어하기 위한 다음과 같은 방법을 제공한다.

1. `InitializingBean`, `DisposableBean` callback interfaces

2. 설정 정보에 초기화 메서드 init( ), 종료 메서드 destory( ) 지정하는 방법

3. `@PostConstruct`, `@PreDestroy` 애노테이션

각각의 방법에 대해 더 자세히 알아봅시다.

### InitializingBean / DisposableBean
```kotlin
@Configuration
class Test {

    @Bean
    fun beanTest(): BeanTest {
        return BeanTest()
    }
}
```

```kotlin
class BeanTest(

): InitializingBean, DisposableBean {

    override fun afterPropertiesSet() {
        println("Spring InitializingBean afterPropertiesSet")
    }

    override fun destroy() {
        println("Spring DisposableBean destroy")
    }
}
```

![image](https://github.com/user-attachments/assets/01273ca3-dfe5-4487-9f44-98bb5433389e)
- `afterPropertiesSet` 을 통해 초기화 메소드를 사용한다. 의존 관계 주입이 끝난 후 해당 메소드가 호출된다.
- `destroy` 를 통해 Spring Container가 Bean을 소멸하기 전에 수행한다.

### 특징

- 해당 인터페이스는 스프링 전용 인터페이스로 해당 코드가 인터페이스에 의존한다.
- 초기화, 소멸 메소드를 오버라이드 하기 때문에 메소드명을 변경할 수 없다
- 코드를 고칠 수 없는 외부 라이브러리에 적용 불가능하다.
### @PostConstruct, @PreDestroy 어노테이션

```kotlin
class BeanTest(

) {

    @PostConstruct
    fun init() {
        println("Spring PostConstruct init")
    }

    @PreDestroy
    fun destroy() {
        println("Spring PreDestroy destroy")
    }
}
```

![image](https://github.com/user-attachments/assets/db22566f-2c5a-4e8b-9281-5d68f3c35878)

### 특징

- *java 6 ~ 8 에선 javax.annotation* package 였으나 이후 9부터는 *javax.annotation* package 하위로 변경되었다.
- 어노테이션으로 관리가 가능하며, 메소드명 또한 별도로 지정이 간으하다.
- Spring에 종속적인 기능이 아니다.
### **설정 정보에 사용자 정의 초기화 메서드, 종료 메서드 지정**

```kotlin
@Configuration
class Test {

    @Bean(initMethod = "thisIsInit", destroyMethod = "thisIsDestroy")
    fun beanTest(): BeanTest {
        return BeanTest()
    }
}

class BeanTest(

) {

    fun thisIsInit() {
        println("Spring Bean Config init")
    }

    fun thisIsDestroy() {
        println("Spring Bean config destroy")
    }
}
```

![image](https://github.com/user-attachments/assets/a666e527-3629-4962-a28c-c9eaf5abe083)

- @Bean 어노테이션에 initMethod, destroyMethod 속성을 사용하여 초기화, 소멸 메서드를 각각 지정한다.

### 특징

- 메소드 명을 자유롭게 부여할 수있고, 스프링 코드에 의존하지 않는다.
- 설정 정보를 사용하기 때문에 코드를 고칠 수 없는 외부 라이브러리에도 초기화, 종료 메서드를 적용할 수 있다.
