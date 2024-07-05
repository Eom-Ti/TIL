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

이후 `finishBeanFactoryInitialization` 메소드를 통해 실행이 이어지며,
