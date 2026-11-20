<!-- 来源：https://docs.spring.io/spring-framework/reference/integration/scheduling.html -->
<!-- 版本：Spring Framework 7.1.0 官方文档，抓取日期 2026-11-20，仅去除页面导航；在每个英文自然段后附中文翻译（引用块），代码块保持原文 -->

> **阅读提示**：这是 Spring Framework 官方文档《Task Execution and Scheduling》章节，英文原文 + 中文对照（每段英文后的 `>` 引用块为中文翻译，代码块保持原文）。
> 与 plan.md 的对应关系：`TaskExecutor`/`TaskScheduler`/`Trigger` 抽象 → 知识点 1.1；
> `@Scheduled`/`@EnableScheduling` → 知识点 1.2；线程模型 → 1.3；`task` XML 命名空间已很少用，可略读；
> Cron Expressions → M1 直接会用到；Quartz 一节属于第三部分「后续扩展」的预习材料。

# Task Execution and Scheduling（任务执行与调度）

The Spring Framework provides abstractions for the asynchronous execution and scheduling of
tasks with the `TaskExecutor` and `TaskScheduler` interfaces, respectively. Spring also
features implementations of those interfaces that support thread pools or delegation to
CommonJ within an application server environment. Ultimately, the use of these
implementations behind the common interfaces abstracts away the differences between
Java SE and Jakarta EE environments.

> Spring Framework 分别通过 `TaskExecutor` 和 `TaskScheduler` 两个接口，为「任务的异步执行」和「按时间调度」提供了抽象。Spring 还为这些接口提供了多种实现，既支持线程池，也支持在应用服务器环境中委托给 CommonJ。由于这些实现都隐藏在统一的公共接口背后，Java SE 与 Jakarta EE 环境之间的差异就被抽象掉了。

Spring also features integration classes to support scheduling with the
[Quartz Scheduler](https://www.quartz-scheduler.org/).

> Spring 还提供了与 [Quartz 调度器](https://www.quartz-scheduler.org/) 集成的类，以支持基于 Quartz 的调度。

## The Spring `TaskExecutor` Abstraction（Spring 的 TaskExecutor 抽象）

Executors are the JDK name for the concept of thread pools. The “executor” naming is
due to the fact that there is no guarantee that the underlying implementation is
actually a pool. An executor may be single-threaded or even synchronous. Spring’s
abstraction hides implementation details between the Java SE and Jakarta EE environments.

> “Executor（执行器）”是 JDK 对线程池这一概念的叫法。之所以用这个名字，是因为底层实现并不保证真的是一个「池」——执行器可能是单线程的，甚至是同步执行的。Spring 的抽象屏蔽了 Java SE 与 Jakarta EE 环境之间的实现差异。

Spring’s `TaskExecutor` interface is identical to the `java.util.concurrent.Executor`
interface. In fact, originally, its primary reason for existence was to abstract away
the need for Java 5 when using thread pools. The interface has a single method
(`execute(Runnable task)`) that accepts a task for execution based on the semantics
and configuration of the thread pool.

> Spring 的 `TaskExecutor` 接口与 `java.util.concurrent.Executor` 完全相同。事实上，它最初存在的目的就是为了掩盖「使用线程池需要 Java 5」这一版本要求。该接口只有一个方法（`execute(Runnable task)`），根据线程池的语义和配置来接收一个待执行的任务。

The `TaskExecutor` was originally created to give other Spring components an abstraction
for thread pooling where needed. Components such as the `ApplicationEventMulticaster`,
JMS’s `AbstractMessageListenerContainer`, and Quartz integration all use the
`TaskExecutor` abstraction to pool threads. However, if your beans need thread pooling
behavior, you can also use this abstraction for your own needs.

> `TaskExecutor` 最初是为了给 Spring 内部其他组件提供线程池抽象而创建的。诸如 `ApplicationEventMulticaster`、JMS 的 `AbstractMessageListenerContainer` 以及 Quartz 集成等组件，都使用 `TaskExecutor` 抽象来管理线程池。不过，如果你的 Bean 也需要线程池行为，同样可以拿这个抽象来自己用。

### `TaskExecutor` Types（TaskExecutor 的内置实现）

Spring includes a number of pre-built implementations of `TaskExecutor`.
In all likelihood, you should never need to implement your own.
The variants that Spring provides are as follows:

> Spring 内置了多个 `TaskExecutor` 实现。绝大多数情况下，你不需要自己实现。Spring 提供的几种变体如下：

* `SyncTaskExecutor`:
  This implementation does not run invocations asynchronously. Instead, each
  invocation takes place in the calling thread. It is primarily used in situations
  where multi-threading is not necessary, such as in simple test cases.

  > 译：`SyncTaskExecutor`：该实现不做异步执行，每次调用都直接发生在调用者线程里。主要用于不需要多线程的场景，比如简单的测试用例。

* `SimpleAsyncTaskExecutor`:
  This implementation does not reuse any threads. Rather, it starts up a new thread
  for each invocation. However, it does support a concurrency limit that blocks
  any invocations that are over the limit until a slot has been freed up. If you
  are looking for true pooling, see `ThreadPoolTaskExecutor`, later in this list.
  This will use JDK 21’s Virtual Threads, when the "virtualThreads"
  option is enabled. This implementation also supports graceful shutdown through
  Spring’s lifecycle management.

  > 译：`SimpleAsyncTaskExecutor`：该实现不复用任何线程，而是每次调用都新起一个线程。不过它支持并发数上限（concurrency limit），超过上限的调用会被阻塞，直到有空位释放。如果你想要真正的线程池，请看本列表后面的 `ThreadPoolTaskExecutor`。启用 "virtualThreads" 选项后，它会使用 JDK 21 的虚拟线程。该实现也支持通过 Spring 生命周期管理实现优雅停机。

* `ConcurrentTaskExecutor`:
  This implementation is an adapter for a `java.util.concurrent.Executor` instance.
  There is an alternative (`ThreadPoolTaskExecutor`) that exposes the `Executor`
  configuration parameters as bean properties. There is rarely a need to use
  `ConcurrentTaskExecutor` directly. However, if the `ThreadPoolTaskExecutor` is not
  flexible enough for your needs, `ConcurrentTaskExecutor` is an alternative.

  > 译：`ConcurrentTaskExecutor`：该实现是对 `java.util.concurrent.Executor` 实例的适配器。它有一个替代品 `ThreadPoolTaskExecutor`，后者把 `Executor` 的配置参数暴露成了 Bean 属性。直接使用 `ConcurrentTaskExecutor` 的场景很少，但如果 `ThreadPoolTaskExecutor` 的灵活性不够，可以用 `ConcurrentTaskExecutor` 作为替代。

* `ThreadPoolTaskExecutor`:
  This implementation is most commonly used. It exposes bean properties for configuring
  a `java.util.concurrent.ThreadPoolExecutor` and wraps it in a `TaskExecutor`.
  If you need to adapt to a different kind of `java.util.concurrent.Executor`,
  we recommend that you use a `ConcurrentTaskExecutor` instead.
  It also provides a pause/resume capability and graceful shutdown through
  Spring’s lifecycle management.

  > 译：`ThreadPoolTaskExecutor`：这是最常用的实现。它把配置 `java.util.concurrent.ThreadPoolExecutor` 的参数暴露为 Bean 属性，并把它包装成 `TaskExecutor`。如果你需要适配其他类型的 `java.util.concurrent.Executor`，官方建议改用 `ConcurrentTaskExecutor`。它还提供暂停/恢复能力，并支持通过 Spring 生命周期管理实现优雅停机。

* `DefaultManagedTaskExecutor`:
  This implementation uses a JNDI-obtained `ManagedExecutorService` in a JSR-236
  compatible runtime environment (such as a Jakarta EE application server),
  replacing a CommonJ WorkManager for that purpose.

  > 译：`DefaultManagedTaskExecutor`：该实现使用在 JSR-236 兼容运行环境（如 Jakarta EE 应用服务器）里通过 JNDI 获取的 `ManagedExecutorService`，用来取代 CommonJ 的 WorkManager。

### Using a `TaskExecutor`（使用 TaskExecutor）

Spring’s `TaskExecutor` implementations are commonly used with dependency injection.
In the following example, we define a bean that uses the `ThreadPoolTaskExecutor`
to asynchronously print out a set of messages:

> Spring 的 `TaskExecutor` 实现通常配合依赖注入使用。下面的例子定义了一个 Bean，它使用 `ThreadPoolTaskExecutor` 异步打印一组消息：

* Java
* Kotlin

```
public class TaskExecutorExample {

	private class MessagePrinterTask implements Runnable {

		private String message;

		public MessagePrinterTask(String message) {
			this.message = message;
		}

		public void run() {
			System.out.println(message);
		}
	}

	private TaskExecutor taskExecutor;

	public TaskExecutorExample(TaskExecutor taskExecutor) {
		this.taskExecutor = taskExecutor;
	}

	public void printMessages() {
		for(int i = 0; i < 25; i++) {
			taskExecutor.execute(new MessagePrinterTask("Message" + i));
		}
	}
}
```

```
class TaskExecutorExample(private val taskExecutor: TaskExecutor) {

	private inner class MessagePrinterTask(private val message: String) : Runnable {
		override fun run() {
			println(message)
		}
	}

	fun printMessages() {
		for (i in 0..24) {
			taskExecutor.execute(
				MessagePrinterTask(
					"Message$i"
				)
			)
		}
	}
}
```

As you can see, rather than retrieving a thread from the pool and executing it yourself,
you add your `Runnable` to the queue. Then the `TaskExecutor` uses its internal rules to
decide when the task gets run.

> 可以看到，你并不需要自己从池里拿线程再执行，只要把 `Runnable` 放进队列即可。之后由 `TaskExecutor` 按它内部的规则决定任务什么时候运行。

To configure the rules that the `TaskExecutor` uses, we expose simple bean properties:

> 要配置 `TaskExecutor` 所使用的规则，可以通过暴露一些简单的 Bean 属性来完成：

* Java
* Kotlin
* Xml

```
@Bean
ThreadPoolTaskExecutor taskExecutor() {
	ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
	taskExecutor.setCorePoolSize(5);
	taskExecutor.setMaxPoolSize(10);
	taskExecutor.setQueueCapacity(25);
	return taskExecutor;
}

@Bean
TaskExecutorExample taskExecutorExample(ThreadPoolTaskExecutor taskExecutor) {
	return new TaskExecutorExample(taskExecutor);
}
```

```
@Bean
fun taskExecutor() = ThreadPoolTaskExecutor().apply {
	corePoolSize = 5
	maxPoolSize = 10
	queueCapacity = 25
}

@Bean
fun taskExecutorExample(taskExecutor: ThreadPoolTaskExecutor) = TaskExecutorExample(taskExecutor)
```

```
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
	<property name="corePoolSize" value="5"/>
	<property name="maxPoolSize" value="10"/>
	<property name="queueCapacity" value="25"/>
</bean>

<bean id="taskExecutorExample" class="TaskExecutorExample">
	<constructor-arg ref="taskExecutor"/>
</bean>
```

Most `TaskExecutor` implementations provide a way to automatically wrap tasks submitted
with a `TaskDecorator`. Decorators should delegate to the task it is wrapping, possibly
implementing custom behavior before/after the execution of the task.

> 大多数 `TaskExecutor` 实现都支持用 `TaskDecorator` 对提交的任务做自动包装。装饰器应当把调用委托给它所包装的任务，并可以在任务执行前后加入自定义行为。

Let’s consider a simple implementation that will log messages before and after the execution
or our tasks:

> 来看一个简单实现，它在任务执行前后各打印一条日志：

* Java
* Kotlin

```
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.core.task.TaskDecorator;

public class LoggingTaskDecorator implements TaskDecorator {

	private static final Log logger = LogFactory.getLog(LoggingTaskDecorator.class);

	@Override
	public Runnable decorate(Runnable runnable) {
		return () -> {
			logger.debug("Before execution of " + runnable);
			runnable.run();
			logger.debug("After execution of " + runnable);
		};
	}
}
```

```
import org.apache.commons.logging.Log
import org.apache.commons.logging.LogFactory
import org.springframework.core.task.TaskDecorator

class LoggingTaskDecorator : TaskDecorator {

	override fun decorate(runnable: Runnable): Runnable {
		return Runnable {
			logger.debug("Before execution of $runnable")
			runnable.run()
			logger.debug("After execution of $runnable")
		}
	}

	companion object {
		private val logger: Log = LogFactory.getLog(
			LoggingTaskDecorator::class.java
		)
	}
}
```

We can then configure our decorator on a `TaskExecutor` instance:

> 然后就可以在 `TaskExecutor` 实例上配置这个装饰器：

* Java
* Kotlin
* Xml

```
@Bean
ThreadPoolTaskExecutor decoratedTaskExecutor() {
	ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
	taskExecutor.setTaskDecorator(new LoggingTaskDecorator());
	return taskExecutor;
}
```

```
@Bean
fun decoratedTaskExecutor() = ThreadPoolTaskExecutor().apply {
	setTaskDecorator(LoggingTaskDecorator())
}
```

```
<bean id="decoratedTaskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
	<property name="taskDecorator" ref="loggingTaskDecorator"/>
</bean>
```

In case multiple decorators are needed, the `org.springframework.core.task.support.CompositeTaskDecorator`
can be used to execute sequentially multiple decorators.

> 如果需要叠加多个装饰器，可以使用 `org.springframework.core.task.support.CompositeTaskDecorator` 依次执行多个装饰器。

## The Spring `TaskScheduler` Abstraction（Spring 的 TaskScheduler 抽象）

In addition to the `TaskExecutor` abstraction, Spring has a `TaskScheduler` SPI with a
variety of methods for scheduling tasks to run at some point in the future. The following
listing shows the `TaskScheduler` interface definition:

> 除了 `TaskExecutor` 抽象之外，Spring 还有 `TaskScheduler` 这个 SPI，它提供了多种方法用于把任务调度到未来的某个时间点运行。下面的代码是 `TaskScheduler` 接口的定义：

```
public interface TaskScheduler {

	Clock getClock();

	ScheduledFuture schedule(Runnable task, Trigger trigger);

	ScheduledFuture schedule(Runnable task, Instant startTime);

	ScheduledFuture scheduleAtFixedRate(Runnable task, Instant startTime, Duration period);

	ScheduledFuture scheduleAtFixedRate(Runnable task, Duration period);

	ScheduledFuture scheduleWithFixedDelay(Runnable task, Instant startTime, Duration delay);

	ScheduledFuture scheduleWithFixedDelay(Runnable task, Duration delay);
}
```

The simplest method is the one named `schedule` that takes only a `Runnable` and an `Instant`.
That causes the task to run once after the specified time. All of the other methods
are capable of scheduling tasks to run repeatedly. The fixed-rate and fixed-delay
methods are for simple, periodic execution, but the method that accepts a `Trigger` is
much more flexible.

> 最简单的是那个只接收 `Runnable` 和 `Instant` 的 `schedule` 方法——它让任务在指定时间之后运行一次。其余所有方法都能把任务调度为重复执行。fixed-rate（固定频率）和 fixed-delay（固定延迟）方法用于简单的周期性执行，而接收 `Trigger` 参数的方法要灵活得多。

### `Trigger` Interface（Trigger 接口）

The `Trigger` interface is essentially inspired by JSR-236. The basic idea of the
`Trigger` is that execution times may be determined based on past execution outcomes or
even arbitrary conditions. If these determinations take into account the outcome of the
preceding execution, that information is available within a `TriggerContext`. The
`Trigger` interface itself is quite simple, as the following listing shows:

> `Trigger` 接口的设计思路基本来自 JSR-236。`Trigger` 的核心思想是：下一次执行时间可以基于过去的执行结果、甚至任意条件来决定。如果这个决策需要参考上一次执行的结果，相关信息可以从 `TriggerContext` 中拿到。`Trigger` 接口本身非常简单，如下所示：

```
public interface Trigger {

	Instant nextExecution(TriggerContext triggerContext);
}
```

The `TriggerContext` is the most important part. It encapsulates all of
the relevant data and is open for extension in the future, if necessary. The
`TriggerContext` is an interface (a `SimpleTriggerContext` implementation is used by
default). The following listing shows the available methods for `Trigger` implementations.

> `TriggerContext` 是最关键的部分。它封装了所有相关数据，并且将来如有必要还可以扩展。`TriggerContext` 是一个接口（默认实现是 `SimpleTriggerContext`）。下面的代码展示了 `Trigger` 实现可以使用的几个方法。

```
public interface TriggerContext {

	Clock getClock();

	Instant lastScheduledExecution();

	Instant lastActualExecution();

	Instant lastCompletion();
}
```

### `Trigger` Implementations（Trigger 的内置实现）

Spring provides two implementations of the `Trigger` interface. The most interesting one
is the `CronTrigger`. It enables the scheduling of tasks based on
[cron expressions](#scheduling-cron-expression).
For example, the following task is scheduled to run 15 minutes past each hour but only
during the 9-to-5 "business hours" on weekdays:

> Spring 为 `Trigger` 接口提供了两个实现。最有意思的是 `CronTrigger`，它支持基于 cron 表达式来调度任务。例如下面的任务被调度为：每个小时的第 15 分钟运行，且仅在工作日的 9 点到 17 点「上班时间」内：

```
scheduler.schedule(task, new CronTrigger("0 15 9-17 * * MON-FRI"));
```

The other implementation is a `PeriodicTrigger` that accepts a fixed
period, an optional initial delay value, and a boolean to indicate whether the period
should be interpreted as a fixed-rate or a fixed-delay. Since the `TaskScheduler`
interface already defines methods for scheduling tasks at a fixed rate or with a
fixed delay, those methods should be used directly whenever possible. The value of the
`PeriodicTrigger` implementation is that you can use it within components that rely on
the `Trigger` abstraction. For example, it may be convenient to allow periodic triggers,
cron-based triggers, and even custom trigger implementations to be used interchangeably.
Such a component could take advantage of dependency injection so that you can configure
such `Triggers` externally and, therefore, easily modify or extend them.

> 另一个实现是 `PeriodicTrigger`，它接收一个固定周期、一个可选的初始延迟，以及一个布尔值（表示周期按 fixed-rate 还是 fixed-delay 解释）。由于 `TaskScheduler` 接口本身已经定义了固定频率和固定延迟的调度方法，能直接用就尽量直接用。`PeriodicTrigger` 的价值在于：它可以用在那些依赖 `Trigger` 抽象的组件里——比如让周期触发器、cron 触发器甚至自定义触发器可以互换使用。这样的组件可以借助依赖注入，让你从外部配置这些 `Trigger`，从而轻松修改或扩展它们。

### `TaskScheduler` implementations（TaskScheduler 的内置实现）

As with Spring’s `TaskExecutor` abstraction, the primary benefit of the `TaskScheduler`
arrangement is that an application’s scheduling needs are decoupled from the deployment
environment. This abstraction level is particularly relevant when deploying to an
application server environment where threads should not be created directly by the
application itself. For such scenarios, Spring provides a `DefaultManagedTaskScheduler`
that delegates to a JSR-236 `ManagedScheduledExecutorService` in a Jakarta EE environment.

> 与 `TaskExecutor` 抽象一样，`TaskScheduler` 这套设计的主要好处是：把应用的调度需求与部署环境解耦。当部署到「不允许应用自己创建线程」的应用服务器环境时，这种抽象尤其重要。针对这种场景，Spring 提供了 `DefaultManagedTaskScheduler`，它在 Jakarta EE 环境下委托给 JSR-236 的 `ManagedScheduledExecutorService`。

Whenever external thread management is not a requirement, a simpler alternative is
a local `ScheduledExecutorService` setup within the application, which can be adapted
through Spring’s `ConcurrentTaskScheduler`. As a convenience, Spring also provides a
`ThreadPoolTaskScheduler`, which internally delegates to a `ScheduledExecutorService`
to provide common bean-style configuration along the lines of `ThreadPoolTaskExecutor`.
These variants work perfectly fine for locally embedded thread pool setups in lenient
application server environments, as well — in particular on Tomcat and Jetty.

> 如果没有「由外部管理线程」的要求，更简单的做法是在应用内部直接使用 `ScheduledExecutorService`，可以通过 Spring 的 `ConcurrentTaskScheduler` 来适配。为方便起见，Spring 还提供了 `ThreadPoolTaskScheduler`，它内部委托给 `ScheduledExecutorService`，并像 `ThreadPoolTaskExecutor` 一样提供常见的 Bean 风格配置。在环境宽松的应用服务器里，这些实现用于本地内嵌线程池也完全没问题——尤其是 Tomcat 和 Jetty。

As of 6.1, `ThreadPoolTaskScheduler` provides a pause/resume capability and graceful
shutdown through Spring’s lifecycle management. There is also a new option called
`SimpleAsyncTaskScheduler` which is aligned with JDK 21’s Virtual Threads, using a
single scheduler thread but firing up a new thread for every scheduled task execution
(except for fixed-delay tasks which all operate on a single scheduler thread, so for
this virtual-thread-aligned option, fixed rates and cron triggers are recommended).

> 从 6.1 开始，`ThreadPoolTaskScheduler` 提供了暂停/恢复能力，并支持通过 Spring 生命周期管理实现优雅停机。此外新增了一个选项 `SimpleAsyncTaskScheduler`，它与 JDK 21 的虚拟线程对齐：只用一个调度线程，但每次任务执行都会新起一个线程（fixed-delay 类任务除外，它们仍在单个调度线程上运行——所以用这个对齐虚拟线程的选项时，推荐使用 fixed-rate 和 cron 触发器）。

## Annotation Support for Scheduling and Asynchronous Execution（调度与异步执行的注解支持）

Spring provides annotation support for both task scheduling and asynchronous method
execution.

> Spring 对「任务调度」和「异步方法执行」都提供了注解支持。

### Enable Scheduling Annotations（启用调度注解）

To enable support for `@Scheduled` and `@Async` annotations, you can add `@EnableScheduling`
and `@EnableAsync` to one of your `@Configuration` classes, or `<task:annotation-driven>` element,
as the following example shows:

> 要启用 `@Scheduled` 和 `@Async` 注解的支持，可以在你的某个 `@Configuration` 类上加 `@EnableScheduling` 和 `@EnableAsync`，或者使用 `<task:annotation-driven>` 元素，如下例所示：

* Java
* Kotlin
* Xml

```
@Configuration
@EnableAsync
@EnableScheduling
public class SchedulingConfiguration {
}
```

```
@Configuration
@EnableAsync
@EnableScheduling
class SchedulingConfiguration
```

```
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:task="http://www.springframework.org/schema/task"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
	   https://www.springframework.org/schema/beans/spring-beans.xsd
	   http://www.springframework.org/schema/task
	   https://www.springframework.org/schema/task/spring-task.xsd">

	<task:annotation-driven executor="myExecutor" scheduler="myScheduler"/>
	<task:executor id="myExecutor" pool-size="5"/>
	<task:scheduler id="myScheduler" pool-size="10"/>
</beans>
```

You can pick and choose the relevant annotations for your application. For example,
if you need only support for `@Scheduled`, you can omit `@EnableAsync`. For more
fine-grained control, you can additionally implement the `SchedulingConfigurer`
interface, the `AsyncConfigurer` interface, or both. See the
[`SchedulingConfigurer`](https://docs.spring.io/spring-framework/docs/7.1.0/javadoc-api/org/springframework/scheduling/annotation/SchedulingConfigurer.html)
and [`AsyncConfigurer`](https://docs.spring.io/spring-framework/docs/7.1.0/javadoc-api/org/springframework/scheduling/annotation/AsyncConfigurer.html)
javadoc for full details.

> 你可以按需选择注解。例如只需要 `@Scheduled` 的话，可以不加 `@EnableAsync`。如果需要更细粒度的控制，可以额外实现 `SchedulingConfigurer` 接口、`AsyncConfigurer` 接口，或两者都实现。完整细节见 `SchedulingConfigurer` 和 `AsyncConfigurer` 的 Javadoc。

Note that, with the preceding XML, an executor reference is provided for handling those
tasks that correspond to methods with the `@Async` annotation, and the scheduler
reference is provided for managing those methods annotated with `@Scheduled`.

> 注意：在上面的 XML 里，executor 引用用于处理带 `@Async` 注解的方法对应的任务，scheduler 引用用于管理带 `@Scheduled` 注解的方法。

|  |  |
| --- | --- |
|  | The default advice mode for processing `@Async` annotations is `proxy` which allows for interception of calls through the proxy only. Local calls within the same class cannot get intercepted that way. For a more advanced mode of interception, consider switching to `aspectj` mode in combination with compile-time or load-time weaving. |

> 译注：处理 `@Async` 注解的默认增强（advice）模式是 `proxy`，它只能拦截通过代理发生的调用，同一类内部的本地调用无法被拦截。如果需要更高级的拦截方式，可以考虑切换到 `aspectj` 模式，并配合编译期或加载期织入（weaving）。

### The `@Scheduled` annotation（@Scheduled 注解）

You can add the `@Scheduled` annotation to a method, along with trigger metadata. For
example, the following method is invoked every five seconds (5000 milliseconds) with a
fixed delay, meaning that the period is measured from the completion time of each
preceding invocation.

> 可以在方法上添加 `@Scheduled` 注解并附上触发元数据。例如下面这个方法以固定延迟每 5 秒（5000 毫秒）调用一次——也就是说，计时起点是上一次调用完成的时间。

```
@Scheduled(fixedDelay = 5000)
public void doSomething() {
	// something that should run periodically
}
```

|  |  |
| --- | --- |
|  | By default, milliseconds will be used as the time unit for fixed delay, fixed rate, and initial delay values. If you would like to use a different time unit such as seconds or minutes, you can configure this via the `timeUnit` attribute in `@Scheduled`.  For example, the previous example can also be written as follows.  ``` @Scheduled(fixedDelay = 5, timeUnit = TimeUnit.SECONDS) public void doSomething() { 	// something that should run periodically } ``` |

> 译注：默认情况下，fixed delay、fixed rate 和 initial delay 的时间单位都是毫秒。如果想用其他时间单位（比如秒或分钟），可以通过 `@Scheduled` 的 `timeUnit` 属性配置。例如上面的例子也可以写成：`@Scheduled(fixedDelay = 5, timeUnit = TimeUnit.SECONDS)`。

If you need a fixed-rate execution, you can use the `fixedRate` attribute within the
annotation. The following method is invoked every five seconds (measured between the
successive start times of each invocation):

> 如果需要固定频率执行，可以使用注解里的 `fixedRate` 属性。下面这个方法每 5 秒调用一次（计时起点是各次调用相继的开始时间）：

```
@Scheduled(fixedRate = 5, timeUnit = TimeUnit.SECONDS)
public void doSomething() {
	// something that should run periodically
}
```

For fixed-delay and fixed-rate tasks, you can specify an initial delay by indicating
the amount of time to wait before the first execution of the method, as the following
`fixedRate` example shows:

> 对于 fixed-delay 和 fixed-rate 任务，可以通过指定首次执行前等待的时间来设置初始延迟，如下面的 `fixedRate` 例子所示：

```
@Scheduled(initialDelay = 1000, fixedRate = 5000)
public void doSomething() {
	// something that should run periodically
}
```

For one-time tasks, you can just specify an initial delay by indicating the amount
of time to wait before the intended execution of the method:

> 对于一次性任务，可以只指定一个 initial delay，表示在预期执行之前等待多长时间：

```
@Scheduled(initialDelay = 1000)
public void doSomething() {
	// something that should run only once
}
```

If simple periodic scheduling is not expressive enough, you can provide a
[cron expression](#scheduling-cron-expression).
The following example runs only on weekdays:

> 如果简单的周期调度满足不了需求，可以提供 cron 表达式。下面的例子只在工作日运行：

```
@Scheduled(cron="*/5 * * * * MON-FRI")
public void doSomething() {
	// something that should run on weekdays only
}
```

|  |  |
| --- | --- |
|  | You can also use the `zone` attribute to specify the time zone in which the cron expression is resolved. |

> 译注：还可以用 `zone` 属性指定 cron 表达式按哪个时区解析。

Notice that the methods to be scheduled must have void returns and must not accept any
arguments. If the method needs to interact with other objects from the application
context, those would typically have been provided through dependency injection.

> 注意：被调度的方法返回值必须是 void，且不能接受任何参数。如果方法需要与应用上下文中的其他对象交互，通常通过依赖注入提前注入。

`@Scheduled` can be used as a repeatable annotation. If several scheduled declarations
are found on the same method, each of them will be processed independently, with a
separate trigger firing for each of them. As a consequence, such co-located schedules
may overlap and execute multiple times in parallel or in immediate succession.
Please make sure that your specified cron expressions etc do not accidentally overlap.

> `@Scheduled` 是可重复注解（repeatable）。如果同一个方法上有多个调度声明，每个都会被独立处理，各自有单独的触发器。因此这种叠加的调度可能重叠，并行或紧跟着连续执行多次。请确保你写的 cron 表达式等不会意外重叠。

|  |  |
| --- | --- |
|  | As of Spring Framework 4.3, `@Scheduled` methods are supported on beans of any scope.  Make sure that you are not initializing multiple instances of the same `@Scheduled` annotation class at runtime, unless you do want to schedule callbacks to each such instance. Related to this, make sure that you do not use `@Configurable` on bean classes that are annotated with `@Scheduled` and registered as regular Spring beans with the container. Otherwise, you would get double initialization (once through the container and once through the `@Configurable` aspect), with the consequence of each `@Scheduled` method being invoked twice. |

> 译注：从 Spring Framework 4.3 开始，`@Scheduled` 方法支持任意 scope 的 Bean。但要确保运行时不要初始化同一个 `@Scheduled` 类的多个实例，除非你确实想给每个实例都安排调度回调。与此相关：不要在「既加了 `@Scheduled`、又作为普通 Bean 注册到容器」的类上使用 `@Configurable`，否则会双重初始化（容器一次、`@Configurable` 切面一次），导致每个 `@Scheduled` 方法被调用两次。

### The `@Scheduled` annotation on Reactive methods or Kotlin suspending functions（@Scheduled 用于响应式方法或 Kotlin 挂起函数）

As of Spring Framework 6.1, `@Scheduled` methods are also supported on several types
of reactive methods:

> 从 Spring Framework 6.1 开始，`@Scheduled` 还支持几种类型的响应式方法：

* methods with a `Publisher` return type (or any concrete implementation of `Publisher`)
  like in the following example:

  > 译：返回类型为 `Publisher`（或其任意具体实现）的方法，如下例：

```
@Scheduled(fixedDelay = 500)
public Publisher<Void> reactiveSomething() {
	// return an instance of Publisher
}
```

* methods with a return type that can be adapted to `Publisher` via the shared instance
  of the `ReactiveAdapterRegistry`, provided the type supports *deferred subscription* like
  in the following example:

  > 译：返回类型可以通过 `ReactiveAdapterRegistry` 共享实例适配成 `Publisher` 的方法，前提是该类型支持「延迟订阅」（deferred subscription），如下例：

```
@Scheduled(fixedDelay = 500)
public Single<String> rxjavaNonPublisher() {
	return Single.just("example");
}
```

|  |  |
| --- | --- |
|  | The `CompletableFuture` class is an example of a type that can typically be adapted to `Publisher` but doesn’t support deferred subscription. Its `ReactiveAdapter` in the registry denotes that by having the `getDescriptor().isDeferred()` method return `false`. |

> 译注：`CompletableFuture` 就是一个典型的「能适配成 `Publisher` 但不支持延迟订阅」的类型。注册表中通过让它的 `ReactiveAdapter` 的 `getDescriptor().isDeferred()` 返回 `false` 来标识这一点。

* Kotlin suspending functions, like in the following example:

  > 译：Kotlin 挂起函数（suspending function），如下例：

```
@Scheduled(fixedDelay = 500)
suspend fun something() {
	// do something asynchronous
}
```

* methods that return a Kotlin `Flow` or `Deferred` instance, like in the following example:

  > 译：返回 Kotlin `Flow` 或 `Deferred` 实例的方法，如下例：

```
@Scheduled(fixedDelay = 500)
fun something(): Flow<Void> {
	flow {
		// do something asynchronous
	}
}
```

All these types of methods must be declared without any arguments. In the case of Kotlin
suspending functions, the `kotlinx.coroutines.reactor` bridge must also be present to allow
the framework to invoke a suspending function as a `Publisher`.

> 这些类型的方法都必须声明为无参数。对于 Kotlin 挂起函数，还必须引入 `kotlinx.coroutines.reactor` 桥接库，框架才能把挂起函数当作 `Publisher` 来调用。

The Spring Framework will obtain a `Publisher` for the annotated method once and will
schedule a `Runnable` in which it subscribes to said `Publisher`. These inner regular
subscriptions occur according to the corresponding `cron`/`fixedDelay`/`fixedRate` configuration.

> Spring 会为注解方法获取一次 `Publisher`，然后调度一个 `Runnable`，在其中订阅这个 `Publisher`。这些内部的定期订阅会按照对应的 `cron`/`fixedDelay`/`fixedRate` 配置进行。

If the `Publisher` emits `onNext` signal(s), these are ignored and discarded (the same way
return values from synchronous `@Scheduled` methods are ignored).

> 如果 `Publisher` 发出 `onNext` 信号，这些信号会被忽略并丢弃（就像同步 `@Scheduled` 方法的返回值被忽略一样）。

In the following example, the `Flux` emits `onNext("Hello")`, `onNext("World")` every 5
seconds, but these values are unused:

> 下面的例子里，`Flux` 每 5 秒发出 `onNext("Hello")`、`onNext("World")`，但这些值并没有被使用：

```
@Scheduled(initialDelay = 5000, fixedRate = 5000)
public Flux<String> reactiveSomething() {
	return Flux.just("Hello", "World");
}
```

If the `Publisher` emits an `onError` signal, it is logged at `WARN` level and recovered.
Because of the asynchronous and lazy nature of `Publisher` instances, exceptions are
not thrown from the `Runnable` task: this means that the `ErrorHandler` contract is not
involved for reactive methods.

> 如果 `Publisher` 发出 `onError` 信号，它会以 `WARN` 级别记录日志然后恢复。由于 `Publisher` 实例的异步与惰性特性，异常不会从 `Runnable` 任务中抛出：这意味着响应式方法不走 `ErrorHandler` 那套约定。

As a result, further scheduled subscription occurs despite the error.

> 结果就是：即使出错，后续的定期订阅依然会继续。

In the following example, the `Mono` subscription fails twice in the first five seconds.
Then subscriptions start succeeding, printing a message to the standard output every five
seconds:

> 下面的例子里，`Mono` 的订阅在头 5 秒内失败了两次，之后订阅开始成功，每 5 秒往标准输出打印一条消息：

```
@Scheduled(initialDelay = 0, fixedRate = 5000)
public Mono<Void> reactiveSomething() {
	AtomicInteger countDown = new AtomicInteger(2);

	return Mono.defer(() -> {
		if (countDown.get() == 0 || countDown.decrementAndGet() == 0) {
			return Mono.fromRunnable(() -> System.out.println("Message"));
		}
		return Mono.error(new IllegalStateException("Cannot deliver message"));
	});
}
```

|  |  |
| --- | --- |
|  | When destroying the annotated bean or closing the application context, Spring Framework cancels scheduled tasks, which includes the next scheduled subscription to the `Publisher` as well as any past subscription that is still currently active (for example, for long-running publishers or even infinite publishers). |

> 译注：当注解所在的 Bean 被销毁或应用上下文关闭时，Spring 会取消已调度的任务——包括下一次对 `Publisher` 的计划订阅，以及任何仍在进行中的历史订阅（例如长时间运行甚至无限运行的 publisher）。

### The `@Async` annotation（@Async 注解）

You can provide the `@Async` annotation on a method so that invocation of that method
occurs asynchronously. In other words, the caller returns immediately upon
invocation, while the actual execution of the method occurs in a task that has been
submitted to a Spring `TaskExecutor`. In the simplest case, you can apply the annotation
to a method that returns `void`, as the following example shows:

> 在方法上标注 `@Async`，该方法的调用就会异步发生。换句话说，调用者一发起调用就立即返回，而方法的实际执行发生在提交给 Spring `TaskExecutor` 的任务里。最简单的情况是把注解加在一个返回 void 的方法上，如下例：

```
@Async
void doSomething() {
	// this will be run asynchronously
}
```

Unlike the methods annotated with the `@Scheduled` annotation, these methods can expect
arguments, because they are invoked in the “normal” way by callers at runtime rather
than from a scheduled task being managed by the container. For example, the following
code is a legitimate application of the `@Async` annotation:

> 与 `@Scheduled` 注解的方法不同，`@Async` 方法可以有参数，因为它们是调用者在运行时以「普通」方式调用的，而不是由容器管理的定时任务触发。例如下面这段代码就是 `@Async` 的合法用法：

```
@Async
void doSomething(String s) {
	// this will be run asynchronously
}
```

Even methods that return a value can be invoked asynchronously. However, such methods
are required to have a `Future`-typed return value. This still provides the benefit of
asynchronous execution so that the caller can perform other tasks prior to calling
`get()` on that `Future`. The following example shows how to use `@Async` on a method
that returns a value:

> 有返回值的方法也可以异步调用，但返回类型必须是 `Future`。这样依然保有异步执行的好处：调用者可以在对这个 `Future` 调用 `get()` 之前先去做别的事。下面的例子展示了如何在有返回值的方法上使用 `@Async`：

```
@Async
Future<String> returnSomething(int i) {
	// this will be run asynchronously
}
```

|  |  |
| --- | --- |
|  | `@Async` methods may not only declare a regular `java.util.concurrent.Future` return type but also `java.util.concurrent.CompletableFuture`, for richer interaction with the asynchronous task and for immediate composition with further processing steps. |

> 译注：`@Async` 方法不仅可以声明普通的 `java.util.concurrent.Future` 返回类型，还可以声明 `java.util.concurrent.CompletableFuture`，以便与异步任务做更丰富的交互，并能立即衔接后续处理步骤。

You can not use `@Async` in conjunction with lifecycle callbacks such as `@PostConstruct`.
To asynchronously initialize Spring beans, you currently have to use a separate
initializing Spring bean that then invokes the `@Async` annotated method on the target,
as the following example shows:

> `@Async` 不能和 `@PostConstruct` 这类生命周期回调一起使用。要异步初始化 Spring Bean，目前的做法是用一个单独的初始化 Bean 去调用目标 Bean 上的 `@Async` 方法，如下例所示：

```
public class SampleBeanImpl implements SampleBean {

	@Async
	void doSomething() {
		// ...
	}

}

public class SampleBeanInitializer {

	private final SampleBean bean;

	public SampleBeanInitializer(SampleBean bean) {
		this.bean = bean;
	}

	@PostConstruct
	public void initialize() {
		bean.doSomething();
	}

}
```

|  |  |
| --- | --- |
|  | There is no direct XML equivalent for `@Async`, since such methods should be designed for asynchronous execution in the first place, not externally re-declared to be asynchronous. However, you can manually set up Spring’s `AsyncExecutionInterceptor` with Spring AOP, in combination with a custom pointcut. |

> 译注：`@Async` 没有直接对应的 XML 配置方式，因为这类方法本来就应该被设计成异步执行，而不是从外部重新声明为异步。不过，你可以用 Spring AOP 配合自定义切点，手动装配 Spring 的 `AsyncExecutionInterceptor`。

|  |  |
| --- | --- |
|  | When `@Async` is combined with `@Retryable`, the async advice is applied outermost, so all retry attempts run on the async executor thread. See [Combining `@Retryable` with `@Async`](../core/resilience.html#resilience-annotations-retryable-combining-async) for details. |

> 译注：当 `@Async` 与 `@Retryable` 组合使用时，异步增强在最外层生效，因此所有重试尝试都跑在异步执行器线程上。详见「Combining @Retryable with @Async」相关文档。

### Executor Qualification with `@Async`（用 @Async 指定执行器）

By default, when specifying `@Async` on a method, the executor that is used is the
one [configured when enabling async support](#scheduling-enable-annotation-support),
i.e. the “annotation-driven” element if you are using XML or your `AsyncConfigurer`
implementation, if any. However, you can use the `value` attribute of the `@Async`
annotation when you need to indicate that an executor other than the default should be
used when executing a given method. The following example shows how to do so:

> 默认情况下，方法上标注 `@Async` 时使用的执行器，是启用异步支持时配置的那个——如果用 XML，就是 “annotation-driven” 元素；否则就是你的 `AsyncConfigurer` 实现（如果有的话）。不过，如果某个方法需要用默认之外的执行器来执行，可以使用 `@Async` 注解的 `value` 属性，如下例所示：

```
@Async("otherExecutor")
void doSomething(String s) {
	// this will be run asynchronously by "otherExecutor"
}
```

In this case, `"otherExecutor"` can be the name of any `Executor` bean in the Spring
container, or it may be the name of a qualifier associated with any `Executor` (for example,
as specified with the `<qualifier>` element or Spring’s `@Qualifier` annotation).

> 这里 `"otherExecutor"` 可以是 Spring 容器中任意 `Executor` Bean 的名字，也可以是与任意 `Executor` 关联的限定符（qualifier）名字（例如通过 `<qualifier>` 元素或 Spring 的 `@Qualifier` 注解指定）。

### Exception Management with `@Async`（@Async 的异常管理）

When an `@Async` method has a `Future`-typed return value, it is easy to manage
an exception that was thrown during the method execution, as this exception is
thrown when calling `get` on the `Future` result. With a `void` return type,
however, the exception is uncaught and cannot be transmitted. You can provide an
`AsyncUncaughtExceptionHandler` to handle such exceptions. The following example shows
how to do so:

> 当 `@Async` 方法有 `Future` 类型的返回值时，方法执行期间抛出的异常很容易管理——调用 `Future` 结果的 `get` 时异常会被抛出。但如果返回类型是 void，异常就没有被捕获、也无法传递出去。这时可以提供一个 `AsyncUncaughtExceptionHandler` 来处理这类异常，如下例所示：

```
public class MyAsyncUncaughtExceptionHandler implements AsyncUncaughtExceptionHandler {

	@Override
	public void handleUncaughtException(Throwable ex, Method method, Object... params) {
		// handle exception
	}
}
```

By default, the exception is merely logged. You can define a custom `AsyncUncaughtExceptionHandler`
by using `AsyncConfigurer` or the `<task:annotation-driven/>` XML element.

> 默认情况下，这类异常只会被记录日志。你可以通过 `AsyncConfigurer` 或 `<task:annotation-driven/>` XML 元素来定义自定义的 `AsyncUncaughtExceptionHandler`。

## The `task` Namespace（task XML 命名空间）

As of version 3.0, Spring includes an XML namespace for configuring `TaskExecutor` and
`TaskScheduler` instances. It also provides a convenient way to configure tasks to be
scheduled with a trigger.

> 从 3.0 版本开始，Spring 提供了一个 XML 命名空间，用于配置 `TaskExecutor` 和 `TaskScheduler` 实例；同时也提供了配置「用 Trigger 调度的任务」的便捷方式。

### The `scheduler` Element（scheduler 元素）

The following element creates a `ThreadPoolTaskScheduler` instance with the
specified thread pool size:

> 下面的元素会创建一个指定线程池大小的 `ThreadPoolTaskScheduler` 实例：

```
<task:scheduler id="scheduler" pool-size="10"/>
```

The value provided for the `id` attribute is used as the prefix for thread names
within the pool. The `scheduler` element is relatively straightforward. If you do not
provide a `pool-size` attribute, the default thread pool has only a single thread.
There are no other configuration options for the scheduler.

> `id` 属性的值会作为线程池中线程名的前缀。`scheduler` 元素比较简单：如果不提供 `pool-size` 属性，默认线程池只有 1 个线程。除此之外没有其他配置项。

### The `executor` Element（executor 元素）

The following creates a `ThreadPoolTaskExecutor` instance:

> 下面这行会创建一个 `ThreadPoolTaskExecutor` 实例：

```
<task:executor id="executor" pool-size="10"/>
```

As with the scheduler shown in the [previous section](#scheduling-task-namespace-scheduler),
the value provided for the `id` attribute is used as the prefix for thread names within
the pool. As far as the pool size is concerned, the `executor` element supports more
configuration options than the `scheduler` element. For one thing, the thread pool for
a `ThreadPoolTaskExecutor` is itself more configurable. Rather than only a single size,
an executor’s thread pool can have different values for the core and the max size.
If you provide a single value, the executor has a fixed-size thread pool (the core and
max sizes are the same). However, the `executor` element’s `pool-size` attribute also
accepts a range in the form of `min-max`. The following example sets a minimum value of
`5` and a maximum value of `25`:

> 与上一节的 scheduler 一样，`id` 属性的值会作为线程池中线程名的前缀。就线程池大小而言，`executor` 元素比 `scheduler` 元素有更多配置选项。一方面，`ThreadPoolTaskExecutor` 的线程池本身就可配置性更强：不只是单一大小，core（核心）和 max（最大）线程数可以不同。如果只给一个值，执行器就是固定大小的线程池（core 与 max 相同）。另外，`executor` 元素的 `pool-size` 属性还接受 `min-max` 形式的区间，下面的例子把最小值设为 `5`、最大值设为 `25`：

```
<task:executor
		id="executorWithPoolSizeRange"
		pool-size="5-25"
		queue-capacity="100"/>
```

In the preceding configuration, a `queue-capacity` value has also been provided.
The configuration of the thread pool should also be considered in light of the
executor’s queue capacity. For the full description of the relationship between pool
size and queue capacity, see the documentation for
[`ThreadPoolExecutor`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html).
The main idea is that, when a task is submitted, the executor first tries to use a
free thread if the number of active threads is currently less than the core size.
If the core size has been reached, the task is added to the queue, as long as its
capacity has not yet been reached. Only then, if the queue’s capacity has been
reached, does the executor create a new thread beyond the core size. If the max size
has also been reached, then the executor rejects the task.

> 上面的配置还提供了 `queue-capacity`（队列容量）。线程池的配置需要结合执行器的队列容量一起考虑。池大小与队列容量关系的完整说明见 `ThreadPoolExecutor` 的文档。核心逻辑是：任务提交时，如果当前活跃线程数小于 core 大小，执行器优先使用空闲线程；达到 core 后，只要队列还没满，任务就进入队列；只有队列也满了，执行器才会创建超过 core 的新线程；如果 max 也达到了，执行器就拒绝该任务。

By default, the queue is unbounded, but this is rarely the desired configuration,
because it can lead to `OutOfMemoryError` if enough tasks are added to that queue while
all pool threads are busy. Furthermore, if the queue is unbounded, the max size has
no effect at all. Since the executor always tries the queue before creating a new
thread beyond the core size, a queue must have a finite capacity for the thread pool to
grow beyond the core size (this is why a fixed-size pool is the only sensible case
when using an unbounded queue).

> 默认情况下队列是无界的，但这很少是想要的配置：如果所有池线程都在忙，而队列里又堆了大量任务，可能导致 `OutOfMemoryError`。而且队列无界时，max 大小完全不起作用。由于执行器总是先尝试入队、之后才创建超过 core 的新线程，队列必须有界，线程池才可能超过 core 大小增长（这就是为什么无界队列只有配合固定大小线程池才合理）。

Consider the case, as mentioned above, when a task is rejected. By default, when a
task is rejected, a thread pool executor throws a `TaskRejectedException`. However,
the rejection policy is actually configurable. The exception is thrown when using
the default rejection policy, which is the `AbortPolicy` implementation.
For applications where some tasks can be skipped under heavy load, you can instead
configure either `DiscardPolicy` or `DiscardOldestPolicy`. Another option that works
well for applications that need to throttle the submitted tasks under heavy load is
the `CallerRunsPolicy`. Instead of throwing an exception or discarding tasks,
that policy forces the thread that is calling the submit method to run the task itself.
The idea is that such a caller is busy while running that task and not able to submit
other tasks immediately. Therefore, it provides a simple way to throttle the incoming
load while maintaining the limits of the thread pool and queue. Typically, this allows
the executor to “catch up” on the tasks it is handling and thereby frees up some
capacity on the queue, in the pool, or both. You can choose any of these options from an
enumeration of values available for the `rejection-policy` attribute on the `executor`
element.

> 再看前面提到的情况：任务被拒绝时怎么办。默认情况下，任务被拒绝时线程池执行器会抛出 `TaskRejectedException`——这是默认拒绝策略 `AbortPolicy` 的行为。拒绝策略其实是可配置的。对于高负载下允许丢弃部分任务的应用，可以改用 `DiscardPolicy` 或 `DiscardOldestPolicy`。另一个适合「高负载下需要对提交任务限流」的应用的选项是 `CallerRunsPolicy`：它不抛异常也不丢弃任务，而是强制让调用 submit 方法的线程自己去执行这个任务。其思路是：调用者执行该任务期间处于忙碌状态，无法立刻提交其他任务，因此这是一种在保持线程池和队列上限的同时对进入负载做简单限流的方式。通常这让执行器能够「追上」它正在处理的任务，从而释放一些队列容量、池容量或两者兼有。这些选项都可以通过 `executor` 元素的 `rejection-policy` 属性的枚举值来选择。

The following example shows an `executor` element with a number of attributes to specify
various behaviors:

> 下面的例子展示了一个带有多个属性、用于指定各种行为的 `executor` 元素：

```
<task:executor
		id="executorWithCallerRunsPolicy"
		pool-size="5-25"
		queue-capacity="100"
		rejection-policy="CALLER_RUNS"/>
```

Finally, the `keep-alive` setting determines the time limit (in seconds) for which threads
may remain idle before being stopped. If there are more than the core number of threads
currently in the pool, after waiting this amount of time without processing a task, excess
threads get stopped. A time value of zero causes excess threads to stop
immediately after executing a task without remaining follow-up work in the task queue.
The following example sets the `keep-alive` value to two minutes:

> 最后，`keep-alive` 设置决定了线程在停止前允许空闲的时长（秒）。如果池中线程数超过 core 数量，超过部分的线程在等待这么久仍没处理到任务后就会被停止。值为 0 表示超出的线程在执行完一个任务、且队列中没有后续任务时立即停止。下面的例子把 `keep-alive` 设为两分钟：

```
<task:executor
		id="executorWithKeepAlive"
		pool-size="5-25"
		keep-alive="120"/>
```

### The `scheduled-tasks` Element（scheduled-tasks 元素）

The most powerful feature of Spring’s task namespace is the support for configuring
tasks to be scheduled within a Spring Application Context. This follows an approach
similar to other “method-invokers” in Spring, such as that provided by the JMS namespace
for configuring message-driven POJOs. Basically, a `ref` attribute can point to any
Spring-managed object, and the `method` attribute provides the name of a method to be
invoked on that object. The following listing shows a simple example:

> Spring task 命名空间最强大的功能，是支持在 Spring 应用上下文中配置要调度的任务。它的方式与 Spring 里其他「方法调用器」类似，比如 JMS 命名空间配置消息驱动 POJO 的做法。基本用法是：`ref` 属性指向任意 Spring 管理的对象，`method` 属性给出要在该对象上调用的方法名。下面是一个简单示例：

```
<task:scheduled-tasks scheduler="myScheduler">
	<task:scheduled ref="beanA" method="methodA" fixed-delay="5000"/>
</task:scheduled-tasks>

<task:scheduler id="myScheduler" pool-size="10"/>
```

The scheduler is referenced by the outer element, and each individual
task includes the configuration of its trigger metadata. In the preceding example,
that metadata defines a periodic trigger with a fixed delay indicating the number of
milliseconds to wait after each task execution has completed. Another option is
`fixed-rate`, indicating how often the method should be run regardless of how long
any previous execution takes. Additionally, for both `fixed-delay` and `fixed-rate`
tasks, you can specify an 'initial-delay' parameter, indicating the number of
milliseconds to wait before the first execution of the method. For more control,
you can instead provide a `cron` attribute to provide a
[cron expression](#scheduling-cron-expression).
The following example shows these other options:

> 外层元素引用调度器，每个单独的任务则带有自己的触发器元数据配置。上面的例子里，该元数据定义了一个固定延迟的周期触发器，表示每次任务执行完成后要等待的毫秒数。另一种选择是 `fixed-rate`，表示方法该多久运行一次，而不关心上一次执行花了多长时间。此外，对 `fixed-delay` 和 `fixed-rate` 任务，都可以指定 'initial-delay' 参数，表示方法首次执行前等待的毫秒数。若需要更多控制，可以改用 `cron` 属性提供 cron 表达式。下面的例子展示了这些其他选项：

```
<task:scheduled-tasks scheduler="myScheduler">
	<task:scheduled ref="beanA" method="methodA" fixed-delay="5000" initial-delay="1000"/>
	<task:scheduled ref="beanB" method="methodB" fixed-rate="5000"/>
	<task:scheduled ref="beanC" method="methodC" cron="*/5 * * * * MON-FRI"/>
</task:scheduled-tasks>

<task:scheduler id="myScheduler" pool-size="10"/>
```

## Cron Expressions（Cron 表达式）

All Spring cron expressions have to conform to the same format, whether you are using them in
[`@Scheduled` annotations](#scheduling-annotation-support-scheduled),
[`task:scheduled-tasks` elements](#scheduling-task-namespace-scheduled-tasks),
or someplace else. A well-formed cron expression, such as `* * * * * *`, consists of six
space-separated time and date fields, each with its own range of valid values:

> 无论是在 `@Scheduled` 注解、`task:scheduled-tasks` 元素还是其他地方使用，Spring 的 cron 表达式都必须遵循同一格式。一个合法的 cron 表达式（如 `* * * * * *`）由 6 个以空格分隔的时间日期字段组成，每个字段有自己的取值范围：

```
 ┌───────────── second (0-59)
 │ ┌───────────── minute (0 - 59)
 │ │ ┌───────────── hour (0 - 23)
 │ │ │ ┌───────────── day of the month (1 - 31)
 │ │ │ │ ┌───────────── month (1 - 12) (or JAN-DEC)
 │ │ │ │ │ ┌───────────── day of the week (0 - 7)
 │ │ │ │ │ │          (0 or 7 is Sunday, or MON-SUN)
 │ │ │ │ │ │
 * * * * * *
```

There are some rules that apply:

> 适用以下规则：

* A field may be an asterisk (`*`), which always stands for “first-last”.
  For the day-of-the-month or day-of-the-week fields, a question mark (`?`) may be used instead of an
  asterisk.

  > 译：字段可以是星号（`*`），永远表示「从头到尾」（first-last）。在「日」和「周」字段上，可以用问号（`?`）代替星号。

* Commas (`,`) are used to separate items of a list.

  > 译：逗号（`,`）用于分隔列表中的多个项。

* Two numbers separated with a hyphen (`-`) express a range of numbers.
  The specified range is inclusive.

  > 译：用连字符（`-`）分隔的两个数字表示一个数值区间，且区间是闭区间（含两端）。

* Following a range (or `*`) with `/` specifies the interval of the number’s value through the range.

  > 译：在区间（或 `*`）后面跟 `/`，指定在该范围内按什么间隔取值。

* English names can also be used for the month and day-of-week fields.
  Use the first three letters of the particular day or month (case does not matter).

  > 译：「月」和「周」字段也可以用英文缩写：取相应月份或星期几的前三个字母（大小写不限）。

* The day-of-month and day-of-week fields can contain an `L` character, which has a different meaning.

  + In the day-of-month field, `L` stands for *the last day of the month*.
    If followed by a negative offset (that is, `L-n`), it means *`n`th-to-last day of the month*.
  + In the day-of-week field, `L` stands for *the last day of the week*.
    If prefixed by a number or three-letter name (`dL` or `DDDL`), it means *the last day of week (`d`
    or `DDD`) in the month*.

  > 译：「日」和「周」字段可以包含 `L` 字符，含义各不相同。「日」字段中，`L` 表示当月最后一天；后面跟负偏移（即 `L-n`）表示当月倒数第 n 天。「周」字段中，`L` 表示一周的最后一天；如果前面加了数字或三字母缩写（`dL` 或 `DDDL`），表示当月最后一个星期 d。

* The day-of-month field can be `nW`, which stands for *the nearest weekday to day of the month `n`*.
  If `n` falls on Saturday, this yields the Friday before it.
  If `n` falls on Sunday, this yields the Monday after, which also happens if `n` is `1` and falls on
  a Saturday (that is: `1W` stands for *the first weekday of the month*).

  > 译：「日」字段可以写 `nW`，表示「离当月 n 号最近的工作日」。如果 n 是周六，则取它之前的周五；如果 n 是周日，则取它之后的周一；当 n 为 1 且恰好是周六时也是这样（即 `1W` 表示当月的第一个工作日）。

* If the day-of-month field is `LW`, it means *the last weekday of the month*.

  > 译：「日」字段写 `LW` 表示当月最后一个工作日。

* The day-of-week field can be `d#n` (or `DDD#n`), which stands for *the `n`th day of week `d`
  (or `DDD`) in the month*.

  > 译：「周」字段可以写 `d#n`（或 `DDD#n`），表示当月的第 n 个星期 d。

Here are some examples:

> 一些示例（第三列为中文含义）：

| Cron Expression | Meaning | 中文含义 |
| --- | --- | --- |
| `0 0 * * * *` | top of every hour of every day | 每天每个整点 |
| `*/10 * * * * *` | every ten seconds | 每 10 秒 |
| `0 0 8-10 * * *` | 8, 9 and 10 o’clock of every day | 每天的 8 点、9 点、10 点 |
| `0 0 6,19 * * *` | 6:00 AM and 7:00 PM every day | 每天 6:00 和 19:00 |
| `0 0/30 8-10 * * *` | 8:00, 8:30, 9:00, 9:30, 10:00 and 10:30 every day | 每天 8:00–10:30，每半小时一次 |
| `0 0 9-17 * * MON-FRI` | on the hour nine-to-five weekdays | 工作日 9 点到 17 点，每个整点 |
| `0 0 0 25 DEC ?` | every Christmas Day at midnight | 每年 12 月 25 日零点 |
| `0 0 0 L * *` | last day of the month at midnight | 每月最后一天零点 |
| `0 0 0 L-3 * *` | third-to-last day of the month at midnight | 每月倒数第三天零点 |
| `0 0 0 * * 5L` | last Friday of the month at midnight | 每月最后一个周五零点 |
| `0 0 0 * * THUL` | last Thursday of the month at midnight | 每月最后一个周四零点 |
| `0 0 0 1W * *` | first weekday of the month at midnight | 每月第一个工作日零点 |
| `0 0 0 LW * *` | last weekday of the month at midnight | 每月最后一个工作日零点 |
| `0 0 0 ? * 5#2` | the second Friday in the month at midnight | 每月第二个周五零点 |
| `0 0 0 ? * MON#1` | the first Monday in the month at midnight | 每月第一个周一零点 |

### Macros（宏）

Expressions such as `0 0 * * * *` are hard for humans to parse and are, therefore,
hard to fix in case of bugs. To improve readability, Spring supports the following
macros, which represent commonly used sequences. You can use these macros instead
of the six-digit value, thus: `@Scheduled(cron = "@hourly")`.

> 像 `0 0 * * * *` 这样的表达式对人类来说很难读懂，出了 bug 也不好修。为了提高可读性，Spring 支持以下宏，它们代表常用的固定序列。可以用宏代替 6 位表达式，例如 `@Scheduled(cron = "@hourly")`。

| Macro | Meaning | 中文含义 |
| --- | --- | --- |
| `@yearly` (or `@annually`) | once a year (`0 0 0 1 1 *`) | 每年一次 |
| `@monthly` | once a month (`0 0 0 1 * *`) | 每月一次 |
| `@weekly` | once a week (`0 0 0 * * 0`) | 每周一次 |
| `@daily` (or `@midnight`) | once a day (`0 0 0 * * *`), or | 每天一次 |
| `@hourly` | once an hour, (`0 0 * * * *`) | 每小时一次 |

## Using the Quartz Scheduler（使用 Quartz 调度器）

Quartz uses `Trigger`, `Job`, and `JobDetail` objects to realize scheduling of all
kinds of jobs. For the basic concepts behind Quartz, see the
[Quartz Web site](https://www.quartz-scheduler.org/). For convenience purposes, Spring
offers a couple of classes that simplify using Quartz within Spring-based applications.

> Quartz 通过 `Trigger`、`Job` 和 `JobDetail` 对象来实现对各种作业的调度。Quartz 的基本概念见 Quartz 官网。为了方便使用，Spring 提供了几个类，用于简化在 Spring 应用中使用 Quartz。

### Using the `JobDetailFactoryBean`（使用 JobDetailFactoryBean）

Quartz `JobDetail` objects contain all the information needed to run a job. Spring
provides a `JobDetailFactoryBean`, which provides bean-style properties for XML
configuration purposes. Consider the following example:

> Quartz 的 `JobDetail` 对象包含运行一个作业所需的全部信息。Spring 提供了 `JobDetailFactoryBean`，它以 Bean 风格的属性支持 XML 配置。看下面的例子：

```
<bean name="exampleJob" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
	<property name="jobClass" value="example.ExampleJob"/>
	<property name="jobDataAsMap">
		<map>
			<entry key="timeout" value="5"/>
		</map>
	</property>
</bean>
```

The job detail configuration has all the information it needs to run the job (`ExampleJob`).
The timeout is specified in the job data map. The job data map is available through the
`JobExecutionContext` (passed to you at execution time), but the `JobDetail` also gets
its properties from the job data mapped to properties of the job instance. So, in the
following example, the `ExampleJob` contains a bean property named `timeout`, and the
`JobDetail` has it applied automatically:

> 这个 job detail 配置已经包含了运行作业（`ExampleJob`）所需的全部信息。timeout 在 job data map 中指定。job data map 可以通过 `JobExecutionContext`（执行时传给你）访问；此外，`JobDetail` 还会把 job data 映射到作业实例的同名属性上。因此在下面的例子里，`ExampleJob` 有一个名为 `timeout` 的 Bean 属性，`JobDetail` 会自动把值赋给它：

```
package example;

public class ExampleJob extends QuartzJobBean {

	private int timeout;

	/**
	 * Setter called after the ExampleJob is instantiated
	 * with the value from the JobDetailFactoryBean.
	 */
	public void setTimeout(int timeout) {
		this.timeout = timeout;
	}

	protected void executeInternal(JobExecutionContext ctx) throws JobExecutionException {
		// do the actual work
	}
}
```

All additional properties from the job data map are available to you as well.

> job data map 中的其他附加属性也同样可用。

|  |  |
| --- | --- |
|  | By using the `name` and `group` properties, you can modify the name and the group of the job, respectively. By default, the name of the job matches the bean name of the `JobDetailFactoryBean` (`exampleJob` in the preceding example above). |

> 译注：通过 `name` 和 `group` 属性，可以分别修改作业的名称和分组。默认情况下，作业名与 `JobDetailFactoryBean` 的 Bean 名称一致（即上面例子中的 `exampleJob`）。

### Using the `MethodInvokingJobDetailFactoryBean`（使用 MethodInvokingJobDetailFactoryBean）

Often you merely need to invoke a method on a specific object. By using the
`MethodInvokingJobDetailFactoryBean`, you can do exactly this, as the following example shows:

> 很多时候你只是需要调用某个特定对象上的方法。使用 `MethodInvokingJobDetailFactoryBean` 就能直接做到，如下例所示：

```
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
	<property name="targetObject" ref="exampleBusinessObject"/>
	<property name="targetMethod" value="doIt"/>
</bean>
```

The preceding example results in the `doIt` method being called on the
`exampleBusinessObject` method, as the following example shows:

> 上面的例子会让 `exampleBusinessObject` 的 `doIt` 方法被调用，如下例所示：

```
public class ExampleBusinessObject {

	// properties and collaborators

	public void doIt() {
		// do the actual work
	}
}
```

```
<bean id="exampleBusinessObject" class="examples.ExampleBusinessObject"/>
```

By using the `MethodInvokingJobDetailFactoryBean`, you need not create one-line jobs
that merely invoke a method. You need only create the actual business object and
wire up the detail object.

> 使用 `MethodInvokingJobDetailFactoryBean`，你就不必为了「只是调一个方法」而专门创建一行代码的作业类。只需要创建真正的业务对象，再把 detail 对象装配好即可。

By default, Quartz Jobs are stateless, resulting in the possibility of jobs interfering
with each other. If you specify two triggers for the same `JobDetail`, it is possible
that the second one starts before the first job has finished. If `JobDetail` classes
implement the `Stateful` interface, this does not happen: the second job does not start
before the first one has finished.

> 默认情况下，Quartz 作业是无状态的，因此作业之间可能互相干扰。如果给同一个 `JobDetail` 指定两个触发器，第二个触发器可能在第一个作业还没跑完时就启动。如果 `JobDetail` 类实现了 `Stateful` 接口，就不会发生这种情况：第一个作业完成之前，第二个作业不会启动。

To make jobs resulting from the `MethodInvokingJobDetailFactoryBean` be non-concurrent,
set the `concurrent` flag to `false`, as the following example shows:

> 要让 `MethodInvokingJobDetailFactoryBean` 产生的作业不并发执行，把 `concurrent` 标志设为 `false` 即可，如下例所示：

```
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
	<property name="targetObject" ref="exampleBusinessObject"/>
	<property name="targetMethod" value="doIt"/>
	<property name="concurrent" value="false"/>
</bean>
```

|  |  |
| --- | --- |
|  | By default, jobs will run in a concurrent fashion. |

> 译注：默认情况下，作业是并发执行的。

### Wiring up Jobs by Using Triggers and `SchedulerFactoryBean`（用 Trigger 和 SchedulerFactoryBean 把作业装配起来）

We have created job details and jobs. We have also reviewed the convenience bean that
lets you invoke a method on a specific object. Of course, we still need to schedule the
jobs themselves. This is done by using triggers and a `SchedulerFactoryBean`. Several
triggers are available within Quartz, and Spring offers two Quartz `FactoryBean`
implementations with convenient defaults: `CronTriggerFactoryBean` and
`SimpleTriggerFactoryBean`.

> 到这里，作业详情和作业都已创建，也介绍了调用指定对象方法的便捷 Bean。当然，我们还需要调度作业本身——这要用 Trigger 和 `SchedulerFactoryBean` 来完成。Quartz 内置了多种 Trigger，Spring 提供了两个带便利默认值的 Quartz `FactoryBean` 实现：`CronTriggerFactoryBean` 和 `SimpleTriggerFactoryBean`。

Triggers need to be scheduled. Spring offers a `SchedulerFactoryBean` that exposes
triggers to be set as properties. `SchedulerFactoryBean` schedules the actual jobs with
those triggers.

> Trigger 需要被调度。Spring 提供的 `SchedulerFactoryBean` 把 trigger 暴露为可设置的属性，并用这些 trigger 对实际作业进行调度。

The following listing uses both a `SimpleTriggerFactoryBean` and a `CronTriggerFactoryBean`:

> 下面的代码同时使用了 `SimpleTriggerFactoryBean` 和 `CronTriggerFactoryBean`：

```
<bean id="simpleTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
	<!-- see the example of method invoking job above -->
	<property name="jobDetail" ref="jobDetail"/>
	<!-- 10 seconds -->
	<property name="startDelay" value="10000"/>
	<!-- repeat every 50 seconds -->
	<property name="repeatInterval" value="50000"/>
</bean>

<bean id="cronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
	<property name="jobDetail" ref="exampleJob"/>
	<!-- run every morning at 6 AM -->
	<property name="cronExpression" value="0 0 6 * * ?"/>
</bean>
```

The preceding example sets up two triggers, one running every 50 seconds with a starting
delay of 10 seconds and one running every morning at 6 AM. To finalize everything,
we need to set up the `SchedulerFactoryBean`, as the following example shows:

> 上面的例子配置了两个 trigger：一个每 50 秒运行一次、启动延迟 10 秒；另一个每天早上 6 点运行。最后还需要配置 `SchedulerFactoryBean` 把一切串起来，如下例所示：

```
<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
	<property name="triggers">
		<list>
			<ref bean="cronTrigger"/>
			<ref bean="simpleTrigger"/>
		</list>
	</property>
</bean>
```

More properties are available for the `SchedulerFactoryBean`, such as the calendars used by the
job details, properties to customize Quartz with, and a Spring-provided JDBC DataSource. See
the [`SchedulerFactoryBean`](https://docs.spring.io/spring-framework/docs/7.1.0/javadoc-api/org/springframework/scheduling/quartz/SchedulerFactoryBean.html)
javadoc for more information.

> `SchedulerFactoryBean` 还有更多可用属性，例如作业详情使用的日历（calendars）、定制 Quartz 的各种属性，以及 Spring 提供的 JDBC DataSource。更多信息见 `SchedulerFactoryBean` 的 Javadoc。

|  |  |
| --- | --- |
|  | `SchedulerFactoryBean` also recognizes a `quartz.properties` file in the classpath, based on Quartz property keys, as with regular Quartz configuration. Please note that many `SchedulerFactoryBean` settings interact with common Quartz settings in the properties file; it is therefore not recommended to specify values at both levels. For example, do not set an "org.quartz.jobStore.class" property if you mean to rely on a Spring-provided DataSource, or specify an `org.springframework.scheduling.quartz.LocalDataSourceJobStore` variant which is a full-fledged replacement for the standard `org.quartz.impl.jdbcjobstore.JobStoreTX`. |

> 译注：与常规 Quartz 配置一样，`SchedulerFactoryBean` 也能识别 classpath 下的 `quartz.properties` 文件（按 Quartz 属性键解析）。注意：`SchedulerFactoryBean` 的许多设置会与 properties 文件里的通用 Quartz 设置相互影响，因此不建议在两处都配置。例如：如果你打算依赖 Spring 提供的 DataSource，就不要再设置 "org.quartz.jobStore.class" 属性，也不要指定 `org.springframework.scheduling.quartz.LocalDataSourceJobStore` 变体——它是对标准 `org.quartz.impl.jdbcjobstore.JobStoreTX` 的完整替代。

