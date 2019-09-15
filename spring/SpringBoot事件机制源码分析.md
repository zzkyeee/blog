#### SpringBoot自带监听器和事件发布

##### SpringApplication初始化
在SpringApplication构造函数中，会通过调用**getSpringFactoriesInstances**方法获取ApplicationContextInitializer和ApplicationListener类对应的实现类并初始化，之后会放入ApplicationContext中。主要是通过加载**META-INF/spring.factories**文件实现的，而加载的过程是由SpringFactoriesLoader加载的。从CLASSPATH下的每个Jar包中搜寻所有META-INF/spring.factories配置文件，然后将解析properties文件，找到指定名称的配置后返回。需要注意的是，其实这里不仅仅是会去ClassPath路径下查找，会扫描所有路径下的Jar包，只不过这个文件只会在Classpath下的jar包中。
```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = deduceWebApplicationType();
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

##### spring自带的spring.factories
下图是SpringBoot自带的spring.factories文件中的有关监听器，其中ApplicationListener会在实例化SpringApplication读取并放入listeners中，SpringApplicationRunListener会在run方法中获取并初始化EventPublishingRunListener，后续还会提到具体的过程。
```
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```

##### 自带监听器注册的过程
在ApplicationContext尚未完成refresh时，Spring Boot使用EventPublishingRunListener进行事件分发。在SpringApplication的run方法中，可以看到以下两行代码，主要作用是获取运行时的监听器，然后将其注册到分发器中，以待事件的触发。

```
public ConfigurableApplicationContext run(String... args) {
        ......
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		......
	}
```
**getRunListeners**进行实例化SpringApplicationRunListener接口类型的Listener，这里只有一个EventPublishingRunListener，其中会用反射初始化这个类，会把所有application.getListeners()进行初始化，其中会调用addApplicationListener，这个方法是AbstractApplicationEventMulticaster的方法，作用是Add a listener to be notified of all events.

```
public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}
```
listener.starting方法主要是用来发布**ApplicationStartingEvent**事件。发布事件是通过SimpleApplicationEventMulticaster类中的multicastEvent方法，该方法会调用父类AbstractApplicationEventMulticaster的getApplicationListeners方法，用来获取符合该Event对应的监听器。
```
@Override
public void starting() {
	this.initialMulticaster.multicastEvent(
			new ApplicationStartingEvent(this.application, this.args));
}
```
之后在run方法中的prepareEnvironment方法也会进行一次事件发布,通过listeners.environmentPrepared(environment)进行发布ApplicationEnvironmentPreparedEvent事件在prepareContext方法会调用listeners.contextLoaded(context)方法进行发布ApplicationPreparedEvent事件。过程与上述类似，故不赘述，有关SpringBoot注册监听器和发布事件的细节将在下文提到。

#### bean中的监听器注册过程
在AbstractApplicationContext的refresh方法中有一个方法叫registerListeners，它的作用是在所有注册Bean中查找Listener Bean,并注册到消息广播器中。

```
protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}
		......
	}
```
之后进行bean的初始化，加载用户的监听器，在后置处理器postProcessAfterInitialization中进行判断该bean是否是实现ApplicationListener接口，如果实现了ApplicationListener接口，最后发布ContextRefreshEvent进行唤醒对应的监听器。

#### SpringBoot注册监听器和广播事件源码
上文提到注册和发布事件主要通过SimpleApplicationEventMulticaster的addApplicationListener方法和multicastEvent方法。
##### 注册监听器

```
@Override
	public void addApplicationListener(ApplicationListener<?> listener) {
		synchronized (this.retrievalMutex) {
			// Explicitly remove target for a proxy, if registered already,
			// in order to avoid double invocations of the same listener.
			Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
			if (singletonTarget instanceof ApplicationListener) {
				this.defaultRetriever.applicationListeners.remove(singletonTarget);
			}
			this.defaultRetriever.applicationListeners.add(listener);
			this.retrieverCache.clear();
		}
	}

	@Override
	public void addApplicationListenerBean(String listenerBeanName) {
		synchronized (this.retrievalMutex) {
			this.defaultRetriever.applicationListenerBeans.add(listenerBeanName);
			this.retrieverCache.clear();
		}
	}
```

##### 广播事件
发布事件是通过SimpleApplicationEventMulticaster类中的multicastEvent方法，该方法会调用父类AbstractApplicationEventMulticaster的getApplicationListeners方法，用来获取符合该Event对应的监听器。
```
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

下述代码为AbstractApplicationEventMulticaster类的getApplicationListeners方法：
```
protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

		Object source = event.getSource();
		Class<?> sourceType = (source != null ? source.getClass() : null);
		ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

		// Quick check for existing entry on ConcurrentHashMap...
		ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
		if (retriever != null) {
			return retriever.getApplicationListeners();
		}

		if (this.beanClassLoader == null ||
				(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
						(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
			// Fully synchronized building and caching of a ListenerRetriever
			synchronized (this.retrievalMutex) {
				retriever = this.retrieverCache.get(cacheKey);
				if (retriever != null) {
					return retriever.getApplicationListeners();
				}
				retriever = new ListenerRetriever(true);
				Collection<ApplicationListener<?>> listeners =
						retrieveApplicationListeners(eventType, sourceType, retriever);
				this.retrieverCache.put(cacheKey, retriever);
				return listeners;
			}
		}
		else {
			// No ListenerRetriever caching -> no synchronization necessary
			return retrieveApplicationListeners(eventType, sourceType, null);
		}
	}
```
首先会从retrieverCache中读取，它是一个concurrentHashMap。

获取完符合该Event的Listener之后，然后进行唤醒，主要是通过doInvokeListener方法

```
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			String msg = ex.getMessage();
			if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				// -> let's suppress the exception and just log a debug message.
				Log logger = LogFactory.getLog(getClass());
				if (logger.isDebugEnabled()) {
					logger.debug("Non-matching event type for listener: " + listener, ex);
				}
			}
			else {
				throw ex;
			}
		}
	}
```

