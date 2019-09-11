# 在Spring中使用事件机制

## 1. 事件机制
事件机制的底层设计模式是**观察者模式**，观察者设计模式定义了对象间的一种一对多的组合关系，以便一个对象的状态发生变化时，所有依赖于它的对象都得到通知并自动刷新，它能够将观察者和被观察者之间进行解耦。本文不具体介绍观察者模式，读者可以从其他文章进行查阅。

使用事件机制能够应对很多场景，它能够对系统业务逻辑之间的解耦。例如Swing中抽象出很多事件例如鼠标点击、键盘键入等，它通过事件派发线程不断从事件队列中获取事件并调用事件监听器的事件处理方法来处理事件。

再举一个**实际业务**中会存在的场景，我们有一个方法TradeService进行用户交易操作，此时有一个用户支付订单成功的方法paySuccess。当用户支付成功时，会有很多操作，比如向库存发送通知，如下述代码（我们对其进行简化，仅传入orderId）：

```
public class TradeServiceImpl implements TradeService {

	/**
	 * 支付成功相关操作
	 * @param orderId 订单号
	 */
	@Override
	public void paySuccess(Long orderId) {
		Preconditions.checkNotNull(orderId);
		
		// 1.修改订单状态为支付成功

		// 2.告知仓库准备发货

	}

}


```

但是此时产品经理需要我们在支付成功之后能够**通过短信告诉用户可以进行抽奖**，那我们仍需要修改该类。每次添加新功能的时候都需要修改原有的类，难以维护，这违反了设计模式的**单一职责原则**和**开闭原则**。我们可以使用事件机制，将支付操作和其他支付之后的相关操作进行分离，通过事件来解耦。下文将围绕该业务通过事件进行改造。
（注：实际上该应用场景因为通常是分布式场景所以我们需要通过消息中间件例如Kafka进行异步处理，本文由于介绍Spring内置事件机制所以利用本地事件进行简化）

## 2. Spring原生事件驱动模型
Spring提供了一些内置的事件供开发者使用：


| 事件名称              | 概述                                                         |
| --------------------- | ------------------------------------------------------------ |
| ContextRefreshedEvent | 该事件会在ApplicationContext被初始化或者更新时发布。也可以在调用ConfigurableApplicationContext 接口中的refresh()方法时被触发。 |
| ContextStartedEvent   | 当容器调用ConfigurableApplicationContext的Start()方法开始/重新开始容器时触发该事件。 |
| ContextStoppedEvent   | 当容器调用ConfigurableApplicationContext的Stop()方法停止容器时触发该事件。 |
| ContextClosedEvent    | 当ApplicationContext被关闭时触发该事件。容器被关闭时，其管理的所有单例Bean都被销毁。 |
| RequestHandledEvent   | 在Web应用中，当一个http请求（request）结束触发该事件。       |

### 2.1 同步方式
- #### 创建事件
首先我们先创建相关的事件，本场景下我们需要围绕支付成功进行相关操作，所以我们需要创建支付成功事件PaySuccessEvent，相关代码如下所示：

```
@Data
public class PaySuccessEvent extends ApplicationEvent {

	/**
	 * 订单ID
	 */
	Long orderId;

	public PaySuccessEvent(Object source, Long orderId) {
		super(source);
		this.orderId = orderId;
	}
}

```

- #### 发布事件
发布事件我们可以通过ApplicationContext或者ApplicationEventPublisher进行发布，但是如果我们只需要进行发布事件，我们只需要创建一个类实现ApplicationEventPublisherAware接口，通过回调方法，使其能够获得ApplicationEventPublish，该接口的publishEvent方法能够对事件进行发布：

```
public interface ApplicationEventPublisherAware extends Aware {

	/**
	 * Set the ApplicationEventPublisher that this object runs in.
	 * <p>Invoked after population of normal bean properties but before an init
	 * callback like InitializingBean's afterPropertiesSet or a custom init-method.
	 * Invoked before ApplicationContextAware's setApplicationContext.
	 * @param applicationEventPublisher event publisher to be used by this object
	 */
	void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);

}
```

我们常用的ApplicationContext也是继承自该接口：

```
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver
```
若需要获取ApplicationContext我们只需要实现ApplicationContextAware接口通过其回调方法即可设置上下文环境。

我们创建一个事件发布的类：

```
public class EventPublisher implements ApplicationEventPublisherAware {

	public static ApplicationEventPublisher applicationEventPublisher;

	@Override
	public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
		EventPublisher.applicationEventPublisher = applicationEventPublisher;
	}

	public static void publishEvent(ApplicationEvent applicationEvent) {
		applicationEventPublisher.publishEvent(applicationEvent);
	}
}

```
然后我们对支付成功事件进行发布，如下所示：

```
public class TradeServiceImpl implements TradeService {

	/**
	 * 支付成功相关操作
	 * @param orderId 订单号
	 */
	@Override
	public void paySuccess(Long orderId) {
		Preconditions.checkNotNull(orderId);
		EventPublisher.publishEvent(new PaySuccessEvent(this, orderId));
	}

}
```

- #### 事件监听
我们可以使用EventListener注解或者实现ApplicationListener接口来对事件进行监听。
- #### 1. 使用EventListener注解

```
@Component
public class PaySuccessEventListener {

	/**
	 * 修改订单状态
	 * @param paySuccessEvent 支付成功事件
	 */
	@EventListener
	public void modifyOrderStatus(PaySuccessEvent paySuccessEvent)
	{
		System.out.println("修改订单状态成功！orderId:" + paySuccessEvent.getOrderId());
	}

	/**
	 * 发送短信告知用户进行抽奖
	 * @param paySuccessEvent 支付成功事件
	 */
	@EventListener
	public void sendSMS(PaySuccessEvent paySuccessEvent)
	{
		System.out.println("发送短信成功！orderId:" + paySuccessEvent.getOrderId());
	}


}

```

- #### 2. 实现ApplicationListener接口

```
public class PaySuccessListener implements ApplicationListener<PaySuccessEvent> {

	@Override
	public void onApplicationEvent(PaySuccessEvent event) {
		System.out.println("告知仓库准备发货成功！" + event.getOrderId());
	}

}
```

### 2.2 异步方式
Spring通过ApplicationEventMulticaster提供异步侦听事件的方式，但是注册 ApplicationEventMulticaster Bean 后所有的事件侦听处理都会变成的异步形式，如果需要针对特定的事件侦听采用异步方式的话：可以使用@EnableAsync和@Async组合来实现。
我们可以通过@EnableAsync注解开启异步方式，之后我们在需要异步监听的方法上加上@Async注解，这样能够使得发布事件时，发布方能异步调用监听器，主方法不会阻塞。以下是EnableAsync的部分注释：

```
 * <p>By default, Spring will be searching for an associated thread pool definition:
 * either a unique {@link org.springframework.core.task.TaskExecutor} bean in the context,
 * or an {@link java.util.concurrent.Executor} bean named "taskExecutor" otherwise. If
 * neither of the two is resolvable, a {@link org.springframework.core.task.SimpleAsyncTaskExecutor}
 * will be used to process async method invocations. Besides, annotated methods having a
 * {@code void} return type cannot transmit any exception back to the caller. By default,
 * such uncaught exceptions are only logged.
 
  * <p>To customize all this, implement {@link AsyncConfigurer} and provide:
 * <ul>
 * <li>your own {@link java.util.concurrent.Executor Executor} through the
 * {@link AsyncConfigurer#getAsyncExecutor getAsyncExecutor()} method, and</li>
 * <li>your own {@link org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler
 * AsyncUncaughtExceptionHandler} through the {@link AsyncConfigurer#getAsyncUncaughtExceptionHandler
 * getAsyncUncaughtExceptionHandler()}
 * method.</li>
 * </ul>
```
简而言之就是Spring默认情况下会先搜索TaskExecutor或者名称为taskExecutor的Executor类型的bean,若都不存在那么Spring会用SimpleAsyncTaskExecutor去执行异步方法。此外我们还可以通过实现AsyncConfigurer接口去自定义异步配置。

我们新建一个**异步配置类AsyncConfig**，使其具备异步能力：

```
@Configuration
@EnableAsync
@Slf4j
public class AsyncConfig implements AsyncConfigurer {

	@Override
	@Bean(name = "taskExecutor")
	public Executor getAsyncExecutor() {
		ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
		taskExecutor.setCorePoolSize(5);
		taskExecutor.setMaxPoolSize(10);
		taskExecutor.setQueueCapacity(25);
		taskExecutor.initialize();
		return taskExecutor;
	}

	@Override
	public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
		return new MyAsyncExceptionHandler();
	}

	/**
	 * 自定义异常处理类
	 */
	class MyAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

		//手动处理捕获的异常
		@Override
		public void handleUncaughtException(Throwable throwable, Method method, Object... obj) {
			System.out.println("------catched exception!------");
			log.info("Exception message - " + throwable.getMessage());
			log.info("Method name - " + method.getName());
			for (Object param : obj) {
				log.info("Parameter value - " + param);
			}
		}
	}
}
```
在开启异步化之后，我们只需要在监听方法上添加@Async注解就可以实现事件的异步调用。

## 3. 使用Guava中的EventBus