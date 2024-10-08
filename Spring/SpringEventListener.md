# EventListener
# Overview

스프링은 이벤트를 발행하고 구독하는 기능을 제공한다. Spring Boot 1.3(Spring 4.2) 기준으로 그 사용법이 나누어지기는 하나, 기본적으로 이벤트를 발행하고 구독하는 기능은 동일하다.

이벤트를 사용하는 경우 장점은 `두 도메인 간의 의존성을 완전히 분리` 할 수 있다는 것에 있다. 결국 발행하는 A 클래스는 이를 받아 처리하는 B 클래스의 존재를 알 필요가 없으며, 이는 결국 두 클래스를 `느슨하게 결합` 하게 되고 이를 통해 재사용성이 증가하며 추후 별도의 서비스로 분리하기도 쉬워진다. 또한 모듈이 추가되거나 삭제 되어도 기존의 모듈은 영향을 받지 않게 된다.

다만 순서를 고려해야 하는 경우 유지보수가 어려워 지기도 하며, 특정 프레임워크의 API에 결합하게 된다는 단점이 존재하기도 한다.

## How To Use

### **Publish**

Spring 4.2 이전에는 반드시 이벤트 클래스가 `ApplicationEvent` 를 상속받도록 되어있다. 이벤트의 발행의 경우 `ApplicationEventPublisher` 인터페이스가 담당하는데 이를 구현하는 것은 결국 `ApplicationContext` 이며 이를 인터페이스 분리원칙(`ISP` )에 따라 이벤트 발행 처리만 분리한 것이다. 결국 해당 Interface의 `publishEvent` 메소드를 통해 사용이 가능하다.

### **Listener**

반면 이벤트 구독의 경우 발행과 마찬가지로 Spring 4.2 버전을 기준으로 이전엔 반드시 `ApplicationListener` 를 구현 했어야 했다. 그러나 4.2 버전 부터는 `@EventListener` 어노테이션을 통해 편리하게 사용할 수 있다. 실제 이벤트가 발행되면 ApplicationContext는 해당 이벤트를 구독하는 Bean들을 찾아서 `notify` 해주는데 이러한 부분은 내부적으로는 `옵저버 패턴`을 사용해 구현되어 있다.

## 동작 원리

### EventPublish 까지의 과정

위에서와 같이 `ApplicationContext`는 이벤트를 발행하는 `ApplicationEventPublisher`를 상속받아 구현하고 있다.

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
```

또한 해당 인터페이스를 `AbstractApplicationContext`가 구현하고 있으며, 실제 구현부는 아래와 같다.

```java
    protected void publishEvent(Object event, @Nullable ResolvableType typeHint) {
        Assert.notNull(event, "Event must not be null");
        ResolvableType eventType = null;
        Object applicationEvent;
        if (event instanceof ApplicationEvent applEvent) {
            applicationEvent = applEvent;
            eventType = typeHint;
        } else {
            ResolvableType payloadType = null;
            if (typeHint != null && ApplicationEvent.class.isAssignableFrom(typeHint.toClass())) {
                eventType = typeHint;
            } else {
                payloadType = typeHint;
            }

            applicationEvent = new PayloadApplicationEvent(this, event, payloadType);
        }

        if (eventType == null) {
            eventType = ResolvableType.forInstance(applicationEvent);
            if (typeHint == null) {
                typeHint = eventType;
            }
        }

        if (this.earlyApplicationEvents != null) {
            this.earlyApplicationEvents.add(applicationEvent);
        } else if (this.applicationEventMulticaster != null) {
            this.applicationEventMulticaster.multicastEvent((ApplicationEvent)applicationEvent, eventType);
        }
```

여기서 중요한 부분은 맨 마지막 부분으로 `ApplicationEventMulticaster` 에게 이벤트 처리를 위임하는 부분이다.

결국 해당 메소드를 보면 `multicastEvent` 처럼 기본적으로 스프링 이벤트 리스너는 `멀티 캐스팅 관계` 이다. 즉 `다수 수신자가 존재`할 수 있는 통신 형태로 `동일한 타입의 여러 리스너`가 등록되어 있다면 모든 리스너가 이벤트를 받게 되기에 주의해야 한다.

이때 그렇다면 기본적으로 `ApplicationContext` 가 `refresh()` 되면서 실제 `ApplicationEventMulticaster` 에 대한 구현체를 등록하게 되는데 해당 코드를 보면 아래와 같이 구성되어 있다.

```java
public void refresh() throws BeansException, IllegalStateException {
        this.startupShutdownLock.lock();

        try {
            this.startupShutdownThread = Thread.currentThread();
            StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                beanPostProcess.end();
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (Error | RuntimeException var12) {
                Throwable ex = var12;
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + ex);
                }

                this.destroyBeans();
                this.cancelRefresh(ex);
                throw ex;
            } finally {
                contextRefresh.end();
            }
        } finally {
            this.startupShutdownThread = null;
            this.startupShutdownLock.unlock();
        }
```

여기서 `this.initApplicationEventMulticaster();` 부분을 본다면 아래와 같이  기본적으로 Spring은 별도의 설정을 하지 않는다면, `SimpleApplicationEventMulticaster.class` 를 등록하는 것을 볼 수 있다.

```java
    protected void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if (beanFactory.containsLocalBean("applicationEventMulticaster")) {
            this.applicationEventMulticaster = (ApplicationEventMulticaster)beanFactory.getBean("applicationEventMulticaster", ApplicationEventMulticaster.class);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
            }
        } else {
            this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
            beanFactory.registerSingleton("applicationEventMulticaster", this.applicationEventMulticaster);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("No 'applicationEventMulticaster' bean, using [" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
            }
        }

    }
```

if문 첫번째에서 보이듯 별도의 `Bean` 등록을 통해 Custom한 등록이 가능하기도 하다.

위까지의 내용을 도식도로 그려본다면 아래와 같이 동작한다고 이해할 수 있다.

![image](https://github.com/Eom-Ti/TIL/assets/71249347/1c29e857-a9d9-47ce-b4c8-d3f627769654)

### Event Listener 등록

그럼 `multicastEvent` 로 발행되는 Event 들은 어떻게 발행되는지 확인이 필요하다. 우선 `multicastEvent` 의 일부 코드이다.

```java
    public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = eventType != null ? eventType : ResolvableType.forInstance(event);
        Executor executor = this.getTaskExecutor();
        Iterator var5 = this.getApplicationListeners(event, type).iterator();
```

우선 `getApplicationListeners` 를 통해 `Collection<ApplicationListener<?>>` 타입의 ApplicationListener 들을 가져온다.

```java
protected Collection<ApplicationListener<?>> getApplicationListeners(ApplicationEvent event, ResolvableType eventType) {
        Object source = event.getSource();
        Class<?> sourceType = source != null ? source.getClass() : null;
        ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
        CachedListenerRetriever newRetriever = null;
        CachedListenerRetriever existingRetriever = (CachedListenerRetriever)this.retrieverCache.get(cacheKey);
```

이때 `retrieverCache` 에서 `cacheKey` 를 통해 이미 저장된 캐시를 가져온다. 다만 없을 경우 아래와 같이  저장한 후 null을반환한다(`putIfAbsent`).  이후 `retrieveApplicationListeners` 메소드가 실행된다.

```java
        synchronized(this.defaultRetriever) {
            listeners = new LinkedHashSet(this.defaultRetriever.applicationListeners);
            listenerBeans = new LinkedHashSet(this.defaultRetriever.applicationListenerBeans);
        }

        Iterator var9 = listeners.iterator();

        while(var9.hasNext()) {
            ApplicationListener<?> listener = (ApplicationListener)var9.next();
            if (this.supportsEvent(listener, eventType, sourceType)) {
                if (retriever != null) {
                    filteredListeners.add(listener);
                }

                allListeners.add(listener);
            }
        }
```

해당 메소드를 살펴보면 `defaultRetriever` 에서 모든 `Listener` 와 `Listener Bean` 을 가져오는 것을 볼 수 있다.

실제 애플리케이션이 기동되면서 테스트 용으로 등록한 EventListner가 등록되는 것을 볼 수 있다.

![image](https://github.com/Eom-Ti/block-nonblock-test/assets/71249347/e9d37945-03cc-4e93-8e2f-ddbdc94dcbf5)

다만 아래의 `if (this.supportsEvent(listener, eventType, sourceType)) {` 부분에서 false로 나오면 실제 아래에서 처리될 `allListeners` 등에선 제외된다.

그럼 내가 등록한 `@EventListener`의 경우 어느부분에서 등록되는 것인지가 의문이다.

`AbstractApplication` 의 `refresh()` 메소드를 살펴보면`invokeBeanFactoryPostProcessors` 통해 `BeanFactoryPostProcessor`를 실행하여 빈 팩토리의 설정을 수정하는 작업을 수행한다.

즉, 빈 팩토리의 초기화 단계에서 호출되며, 빈 정의의 수정이나 추가 설정을 수행한다.

```java
    public void refresh() throws BeansException, IllegalStateException {
        this.startupShutdownLock.lock();

        try {
            this.startupShutdownThread = Thread.currentThread();
            StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
                this.invokeBeanFactoryPostProcessors(beanFactory);.
```

```java
    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
        if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null && beanFactory.containsBean("loadTimeWeaver")) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
```

실제 수행되는 작업은 `invokeBeanFactoryPostProcessors` 에서 이루어지며  이때 아래의 프로세스가 동작한다.

```java
    invokeBeanFactoryPostProcessors((Collection)registryProcessors, (ConfigurableListableBeanFactory)beanFactory);
    invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
            
    private static void invokeBeanFactoryPostProcessors(Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
        Iterator var2 = postProcessors.iterator();

        while(var2.hasNext()) {
            BeanFactoryPostProcessor postProcessor = (BeanFactoryPostProcessor)var2.next();
            StartupStep var10000 = beanFactory.getApplicationStartup().start("spring.context.bean-factory.post-process");
            Objects.requireNonNull(postProcessor);
            StartupStep postProcessBeanFactory = var10000.tag("postProcessor", postProcessor::toString);
            postProcessor.postProcessBeanFactory(beanFactory);
            postProcessBeanFactory.end();
        }
            
```

```java
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        this.beanFactory = beanFactory;
        this.originalEvaluationContext.setBeanResolver(new BeanFactoryResolver(this.beanFactory));
        Map<String, EventListenerFactory> beans = beanFactory.getBeansOfType(EventListenerFactory.class, false, false);
        List<EventListenerFactory> factories = new ArrayList(beans.values());
        AnnotationAwareOrderComparator.sort(factories);
        this.eventListenerFactories = factories;
    }
```

이때 BeanFactoryPostProcessor 구현체인 `EventListenerMethodProcessor` 까지 수행이 이어지고

`EventListenerFactory` bean type을 찾는다. 이때 기본적으로 `DefaultEventListenerFactory` 를 사용한다.

이후 `finishBeanFactoryInitialization` 메소드를 통해 실행이 이어지며, `beanFactory.preInstantiateSingletons` 메소드를 거쳐 `afterSingletonsInstantiated` 메소드를 실행한다.

```java
        while(var2.hasNext()) {
            beanName = (String)var2.next();
            Object singletonInstance = this.getSingleton(beanName);
            if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {
                StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize").tag("beanName", beanName);
                smartSingleton.afterSingletonsInstantiated();
                smartInitialize.end();
            }
        }

```

해당 메소드를 구현하고 있는 `EventListenerMethodProcessor` Class의 `processBean`를 살펴보면

```java
                while(true) {
                    while(var6.hasNext()) {
                        Method method = (Method)var6.next();
                        Iterator var8 = factories.iterator();

                        while(var8.hasNext()) {
                            EventListenerFactory factory = (EventListenerFactory)var8.next();
                            if (factory.supportsMethod(method)) {
                                Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
                                ApplicationListener<?> applicationListener = factory.createApplicationListener(beanName, targetType, methodToUse);
                                if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                                    ApplicationListenerMethodAdapter alma = (ApplicationListenerMethodAdapter)applicationListener;
                                    alma.init(context, this.evaluator);
                                }

```

`createApplicationListener` 를 통해  리스너를 생성하며 `@EventListener 타입의 Bean인 경우` 구현체인 `ApplicationListenerMethodAdapter` 를 통해  `ApplicationContext` 에 주입된다.

```java
    public ApplicationListenerMethodAdapter(String beanName, Class<?> targetClass, Method method) {
        this.beanName = beanName;
        this.method = BridgeMethodResolver.findBridgedMethod(method);
        this.targetMethod = !Proxy.isProxyClass(targetClass) ? AopUtils.getMostSpecificMethod(method, targetClass) : this.method;
        this.methodKey = new AnnotatedElementKey(this.targetMethod, targetClass);
        EventListener ann = (EventListener)AnnotatedElementUtils.findMergedAnnotation(this.targetMethod, EventListener.class);
        this.declaredEventTypes = resolveDeclaredEventTypes(method, ann);
        this.condition = ann != null ? ann.condition() : null;
        this.order = resolveOrder(this.targetMethod);
        String id = ann != null ? ann.id() : "";
        this.listenerId = !id.isEmpty() ? id : null;
```
### 실제 이벤트가 발행되었을 때의 동작원리

지금 까지 이벤트 Publish의 방법과 Listener의 Bean 등록까지를 알아봤다. 그렇다면 어떻게 많은 Event 중 자신의 타입에 맞는 Event가 발생할 때 consume이 실행되는지 알아보자.

`Publish` 때로 돌아가서 `SimpleApplicationEventMulticaster` 부터 살펴보자. 제일 먼저 볼 곳은 `multicastEvent` 이다.

```java
    public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = eventType != null ? eventType : ResolvableType.forInstance(event);
        Executor executor = this.getTaskExecutor();
        Iterator var5 = this.getApplicationListeners(event, type).iterator();

        while(true) {
            while(var5.hasNext()) {
                ApplicationListener<?> listener = (ApplicationListener)var5.next();
                if (executor != null && listener.supportsAsyncExecution()) {
                    try {
                        executor.execute(() -> {
                            this.invokeListener(listener, event);
                        });
                    } catch (RejectedExecutionException var8) {
                        this.invokeListener(listener, event);
                    }
                } else {
                    this.invokeListener(listener, event);
                }
            }

            return;
        }
    }
```

`invokeListener` 를 살펴보도록 한다.

```java
    protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
        ErrorHandler errorHandler = this.getErrorHandler();
        if (errorHandler != null) {
            try {
                this.doInvokeListener(listener, event);
            } catch (Throwable var5) {
                Throwable err = var5;
                errorHandler.handleError(err);
            }
        } else {
            this.doInvokeListener(listener, event);
        }

    }

```

에러에 대한 핸들링 부분을 볼 수 있다.  기본전략은 아무것도 없어서 커스텀을 통해 등록할 순 있다.

이후 `doInvokeListener` 메소드를 살펴보자. 이부분에서 `listener.onApplicationEvent` 를 확인할 수 있다.

```java
    private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
        try {
            listener.onApplicationEvent(event);
        } catch (ClassCastException var7) {
            ClassCastException ex = var7;
            String msg = ex.getMessage();
            if (msg != null && !this.matchesClassCastMessage(msg, event.getClass())) {
                label38: {
                    if (event instanceof PayloadApplicationEvent) {
                        PayloadApplicationEvent payloadEvent = (PayloadApplicationEvent)event;
                        if (this.matchesClassCastMessage(msg, payloadEvent.getPayload().getClass())) {
                            break label38;
                        }
```

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E event);

    default boolean supportsAsyncExecution() {
        return true;
    }

    static <T> ApplicationListener<PayloadApplicationEvent<T>> forPayload(Consumer<T> consumer) {
        return (event) -> {
            consumer.accept(event.getPayload());
        };
    }
}
```

그렇다면 아까 Listener를 Bean 등록할때 보았던 `ApplicationListenerMethodAdapter` 를 살펴보자.

```java
    public void processEvent(ApplicationEvent event) {
        Object[] args = this.resolveArguments(event);
        if (this.shouldHandle(event, args)) {
            Object result = this.doInvoke(args);
            if (result != null) {
                this.handleResult(result);
            } else {
                this.logger.trace("No result object given - no result to handle");
            }
        }

    }
```

실제 실행되는 부분이다. `resolveArguments` 를 좀더 알아보자.

```java
    protected Object[] resolveArguments(ApplicationEvent event) {
        ResolvableType declaredEventType = this.getResolvableType(event);
        if (declaredEventType == null) {
            return null;
        } else if (this.method.getParameterCount() == 0) {
            return new Object[0];
        } else {
            Class<?> declaredEventClass = declaredEventType.toClass();
            if (!ApplicationEvent.class.isAssignableFrom(declaredEventClass) && event instanceof PayloadApplicationEvent) {
                PayloadApplicationEvent<?> payloadEvent = (PayloadApplicationEvent)event;
                Object payload = payloadEvent.getPayload();
                if (declaredEventClass.isInstance(payload)) {
                    return new Object[]{payload};
                }
            }

            return new Object[]{event};
        }
    }
```

```java
    private boolean shouldHandle(ApplicationEvent event, @Nullable Object[] args) {
        if (args == null) {
            return false;
        } else {
            String condition = this.getCondition();
            if (StringUtils.hasText(condition)) {
                Assert.notNull(this.evaluator, "EventExpressionEvaluator must not be null");
                return this.evaluator.condition(condition, event, this.targetMethod, this.methodKey, args);
            } else {
                return true;
            }
        }
    }
```

우선 해당 메소드의 파라미터 정보를 불러와 0인지 아닌지를 체크한다. 이후 0이 아니라면 해당 `payload` 를 반환하고 최종적으로 Handle해야할 `Condition` 이있는지 체크한다.

이후 `processEvent` 메소드의 `this.doInvoke(args)` 를 통해 해당 이벤트를 실행한다.

```java
    @Nullable
    protected Object doInvoke(@Nullable Object... args) {
        Object bean = this.getTargetBean();
        if (bean.equals((Object)null)) {
            return null;
        } else {
            ReflectionUtils.makeAccessible(this.method);

            try {
	            return KotlinDetector.isSuspendingFunction(this.method) ? CoroutinesUtils.invokeSuspendingFunction(this.method, bean, args) : this.method.invoke(bean, args);
```

해당 메소드에선 ReflectionUtils 를 통해 실행가능하도록 허용을 한 이후 Java 라면 `this.method.invoke(bean, args)` 를 통해 메소드를 실행한다.

간단하게 위의 내용들을 정리하면 아래와 같다.

**[Publish]**

Spring 4.2 버전 기준으로 이전의 경우 `ApplicationEvent` 를 상속받아 구현해야 했지만 4.2 버전부턴 `ApplicationEventPublisher` 를 주입받아 이벤트 발행이 가능해졌다. 해당 인터페이스의 구현체는 `AbstractApplicationContext`로 publish를 진행한다.

이때 기본적으로 설정을 따로 하지 않는다면 `SimpleApplicationEventMulticaster` 를 등록하며, 이름에서 볼 수 있듯이 기본적으로 멀티케스트 방식의 이벤트 발행이 진행된다.(이는 해당 타입의 Listener들 모두가 동작할 수 있음을 말한다.)

**[Listener bean 등록]**

Listener의 경우 마찬가지로 4.2 버전을 기준으로 이전의 경우 `ApplicationListener` 를 상속받아야 했지만 4.2 버전부턴 `@EventListenr` 를 통해 bean 등록이 가능하다.

ApplicationContext refresh() 단계에서 부터 시작하며, 해당 어노테이션을 사용한 메소드를 Bean 등록하며, `ApplicationListenerMethodAdapter` 에 의해 `ApplicationContext`에 bean 등록된다.

**[EventListener 동작방식]**

다시 `AbstractApplicationContext` 로 돌아와 publish가 되면, `SimpleApplicationEventMulticaster` 를 통해 우선 캐시에서 cahceKey를 통해 리스너가 있는지 확인한다. 만일 있을 경우 Listener Collection을 받아와 iterator를 수행하며 invoke를 실행한다. 없을경우 `ApplicationListenerMethodAdapter` 를통해 `Annotaion`으로 등록된 EventBean을 찾아 해당 값을 캐시에 넣는다

이후 `ApplicationListenerMethodAdapter` 를 통해 해당 메소드의 파라미터가 0인지를 체크하고 별도의 condition을 체크해야하는지 확인한 후에, `doInvoke` 메소드를 수행한다.

이후 해당 Event를 캐시에 넣는다. 다음 호출부턴 위의 과정 없이 캐시에서 사용이 가능하다. 다만 `ApplicationListenerMethodAdapter` 을 통한 파라미터 혹은 condition 체크는 이벤트 발생시마다 확인한다.
