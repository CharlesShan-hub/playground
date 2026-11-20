# 学习笔记 01 · Spring 的任务抽象

> 对应 plan.md §1.1（知识点 1.1）
> 参考阅读：[spring-framework-task-execution-and-scheduling.md](spring-framework-task-execution-and-scheduling.md) —— TaskExecutor 一节、TaskScheduler / Trigger 一节、注解支持一节
> 学习方式：概念 → 最小代码跑起来 → 回答文末自测问题。这是后面所有里程碑的地基。

---

## 0. 全局心智模型

Spring 把「任务」拆成两组**正交**的抽象：

- **执行（现在就跑）**：`TaskExecutor` —— 给我一个 `Runnable`，立刻执行，不管时间。
- **调度（未来/周期跑）**：`TaskScheduler` —— 给我一个 `Runnable`，在未来的某个时间点执行一次或周期执行。

再补三个辅助角色：

- `Trigger`：调度时「**下一次什么时候跑**」的策略，从 `TaskScheduler` 里解耦出来。
- `ScheduledTaskRegistrar`：任务的「**登记簿**」，`@Scheduled` 注解背后登记任务的地方。
- `ScheduledTaskHolder`：运行时查阅登记簿的「**窗口**」，任务管理接口从这里读数据。

一句话总结：**`@Scheduled` 只是门面，背后全部落到这五个类上。**

```
@EnableScheduling
   │  导入 SchedulingConfiguration
   ▼
ScheduledAnnotationBeanPostProcessor
   │  扫描所有 Bean 的 @Scheduled 方法
   ▼
ScheduledTaskRegistrar（登记簿：cron / fixedRate / fixedDelay / Trigger 任务）
   │  容器启动时统一 schedule() 提交
   ├──► TaskScheduler.schedule(task, Trigger) ◄── 每次触发前问 Trigger.nextExecution(...)
   │
   └── 同时对外暴露 ScheduledTaskHolder（getScheduledTasks() → M3 任务中心用）
```

---

## 1. `TaskExecutor` —— 只管「现在跑」

### 是什么

接口与 `java.util.concurrent.Executor` **完全相同**，只有一个方法：

```java
public interface TaskExecutor extends Executor {
    void execute(Runnable task);
}
```

存在意义：屏蔽实现差异（是不是真的「池」、单线程还是同步、SE 还是 Jakarta EE），让 Spring 内部组件（事件广播、JMS 容器等）和你的 Bean 只依赖接口。

### 内置实现速查

| 实现 | 一句话 | 什么时候用它 |
| --- | --- | --- |
| `SyncTaskExecutor` | 调用者线程里同步跑 | 测试用例，不想要多线程 |
| `SimpleAsyncTaskExecutor` | 每次新建线程，不复用；支持并发上限；可启用虚拟线程 | 任务少、生命周期短；虚拟线程场景 |
| `ThreadPoolTaskExecutor` | 包装 `ThreadPoolExecutor`，Bean 属性可配，最常用 | **默认选择** |
| `ConcurrentTaskExecutor` | 适配任意 `Executor` | `ThreadPoolTaskExecutor` 不够灵活时 |
| `DefaultManagedTaskExecutor` | JNDI 获取容器管理的 `ManagedExecutorService` | Jakarta EE 应用服务器内 |

### 最小验证代码

```java
@Bean
ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(4);
    executor.setQueueCapacity(8);
    executor.setThreadNamePrefix("exec-");
    return executor;
}
```

```java
CommandLineRunner demo(TaskExecutor taskExecutor) {
    return args -> IntStream.range(0, 20).forEach(i ->
        taskExecutor.execute(() -> {
            log.info("task {} on {}", i, Thread.currentThread().getName());
            ThreadUtil.sleep(500); // 占住线程
        }));
}
```

观察日志，回答：线程号从 `exec-1` 到 `exec-4` 的顺序是什么？

> **关键坑（ThreadPoolExecutor 的规矩，Spring 只是包装）**：核心线程满后**先入队**，队列（8 个）满了才创建到 `maxPoolSize`。所以日志里先看到 `exec-1/exec-2` 各扛 5 个，而不是先扩容。这解释了「为什么你的线程池参数不生效」这类线上问题。

---

## 2. `TaskScheduler` —— 管「什么时候跑」

### 是什么

一个 SPI，核心方法一览（Spring Framework 7）：

```java
public interface TaskScheduler {
    ScheduledFuture schedule(Runnable task, Trigger trigger);          // 最灵活：策略交给 Trigger
    ScheduledFuture schedule(Runnable task, Instant startTime);        // 一次性
    ScheduledFuture scheduleAtFixedRate(Runnable task, Duration period);        // ≈ fixedRate
    ScheduledFuture scheduleAtFixedRate(Runnable task, Instant start, Duration period);
    ScheduledFuture scheduleWithFixedDelay(Runnable task, Duration delay);      // ≈ fixedDelay
    ScheduledFuture scheduleWithFixedDelay(Runnable task, Instant start, Duration delay);
}
```

注意对应关系：`scheduleAtFixedRate` / `scheduleWithFixedDelay` 正是 `@Scheduled` 两个属性的底层落点。所有方法返回 `ScheduledFuture`，`cancel()` 即可停止 —— **这就是 M3 动态任务中心的根基**。

### 内置实现速查

| 实现 | 一句话 | 备注 |
| --- | --- | --- |
| `ThreadPoolTaskScheduler` | 包装 `ScheduledExecutorService`，Bean 风格配置 | **默认选择**；6.1 起支持暂停/恢复、优雅停机 |
| `SimpleAsyncTaskScheduler` | 一个调度线程 + 每次执行新起线程（对齐虚拟线程） | **fixed-delay 任务仍跑在调度线程上**，用它时推荐 fixed-rate 和 cron |
| `ConcurrentTaskScheduler` | 适配任意 `ScheduledExecutorService` | 少用 |
| `DefaultManagedTaskScheduler` | Jakarta EE 托管环境 | 少用 |

Boot 默认给一个 `pool.size=1` 的 `ThreadPoolTaskScheduler`（`spring.task.scheduling.*` 可调）—— **这就是「默认一个慢任务拖死所有 @Scheduled」的根源**，M2 重点实验。

### 最小验证代码

```java
@Bean
ThreadPoolTaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
    scheduler.setPoolSize(4);
    scheduler.setThreadNamePrefix("sched-");
    return scheduler;
}
```

四种调度方式各跑一次，看节奏差异：

```java
scheduler.schedule(task, Instant.now().plusSeconds(30));        // 30 秒后跑一次
scheduler.scheduleAtFixedRate(task, Duration.ofSeconds(5));     // 上次开始计时
scheduler.scheduleWithFixedDelay(task, Duration.ofSeconds(5));  // 上次结束计时
scheduler.schedule(task, new CronTrigger("*/5 * * * * *"));     // cron
```

---

## 3. `Trigger` —— 「下一次什么时候跑」的策略

### 是什么

```java
public interface Trigger {
    Instant nextExecution(TriggerContext triggerContext);
}
```

核心思想：**下一次执行时间可以依赖历史**。`TriggerContext` 提供三个时间戳：

| 方法 | 含义 |
| --- | --- |
| `lastScheduledExecution()` | 上次**计划**执行时间 |
| `lastActualExecution()` | 上次**实际**开始时间（可能与计划不符，比如排队等线程） |
| `lastCompletion()` | 上次**完成**时间 |

三个时间分开放，才能表达「任务实际跑到一半时，下一次该怎么算」这类问题。返回 `null` 表示不再调度。

### 内置实现

- `CronTrigger`：按 cron 表达式（6 位）计算，用于工作日时段、整点对齐等不规则节奏。
- `PeriodicTrigger`：固定周期 + 可选 initialDelay，可指定 fixed-rate / fixed-delay。`TaskScheduler` 已有对应方法时优先直接用方法；它的价值是**在依赖 Trigger 抽象的组件里互换使用**。

### 自定义 Trigger（M3 动态任务中心的原型）

```java
// 每次触发前重读配置里的 cron —— 改配置不用重启
Trigger dynamicCron = context -> {
    String cron = taskConfigService.currentCron(taskId);
    return new CronTrigger(cron).nextExecution(context);
};
scheduler.schedule(task, dynamicCron);
```

同一个 `schedule(task, Trigger)` 入口，能塞 `CronTrigger`、`PeriodicTrigger`、自定义实现 —— 依赖注入让它们可以互换，这就是 Trigger 抽象的意义。

---

## 4. `ScheduledTaskRegistrar` —— 注解背后的「登记簿」

### 是什么

负责**收集**任务定义（cron / fixedRate / fixedDelay / Trigger 任务），在容器启动时**统一提交**给 `TaskScheduler`。有两个使用者：

1. **`ScheduledAnnotationBeanPostProcessor`**：扫描每个 Bean 的 `@Scheduled` 方法，逐条登记进 registrar（注解路线）。
2. **`SchedulingConfigurer.configureTasks(registrar)`**：给你一把钩子，手工登记任务（编程路线）。

```java
@Configuration
@EnableScheduling
class SchedulerConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.addTriggerTask(task(), trigger());
        // 还有 addCronTask / addFixedRateTask / addFixedDelayTask
    }
}
```

> 注意：`configureTasks` 在启动时调用**一次**，之后 registrar 在初始化完成时才真正把任务交给调度器。想运行时增删任务（M3），要走 `TaskScheduler.schedule()` + 自己管理 `ScheduledFuture`，而不是回头改 registrar。

---

## 5. `ScheduledTaskHolder` —— 运行时的任务清单窗口

### 是什么

6.0 引入的只读接口，`ScheduledTaskRegistrar` 和 `ScheduledAnnotationBeanPostProcessor` 都实现了它：

```java
public interface ScheduledTaskHolder {
    Set<ScheduledTask> getScheduledTasks();
}
```

直接注入即可看到**所有**已注册任务（包括 `@Scheduled` 注册的）：

```java
@Component
class TaskInspector {

    TaskInspector(ScheduledTaskHolder holder) {
        holder.getScheduledTasks().forEach(t -> log.info("registered: {}", t));
    }
}
```

M3 的 `GET /tasks` 之所以能列出 `@Scheduled` 静态任务 + 动态注册任务，靠的就是这个接口（注解任务从 `ScheduledAnnotationBeanPostProcessor` 读，动态任务从自己的 `Map<String, ScheduledFuture>` 读）。

---

## 6. 自测问题（答不上来就回去重读）

1. `TaskExecutor` 和 `TaskScheduler` 的分界线是什么？`@Async` 用哪个？`@Scheduled` 用哪个？
2. `TriggerContext` 为什么要把「计划时间」「实际开始时间」「完成时间」分开存？
3. 为什么默认情况下一个耗时 8 秒的 `@Scheduled` 任务会阻塞其他任务？（提示：Boot 默认的调度线程池大小）
4. `SimpleAsyncTaskScheduler` 上为什么 fixed-delay 任务不会受益于「每次新起线程」？
5. `SchedulingConfigurer.configureTasks()` 拿到的 registrar 是启动期一次性的，那 M3 的运行时改 cron 应该怎么做？
6. 注入 `ScheduledTaskHolder` 后，能拿到哪些任务？能拿到 `ScheduledFuture` 吗？（拿不到 —— 它只给任务定义，句柄要自己存）

## 7. 与里程碑的对应

| 本笔记内容 | 服务于 |
| --- | --- |
| TaskExecutor 线程池规矩（队列满才扩容） | M2 线程模型 |
| TaskScheduler 四种 schedule + `ScheduledFuture.cancel()` | M3 动态任务中心 |
| 自定义 Trigger 每次触发前读配置 | M3 动态 cron |
| SchedulingConfigurer + 自定义调度器 Bean | M2 代码式配置 |
| ScheduledTaskHolder | M3 `GET /tasks` |
