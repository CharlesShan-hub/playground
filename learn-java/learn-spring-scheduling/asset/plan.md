# Spring Scheduling 学习与演示计划

> 定位：这个仓库不是业务系统，而是一个**可运行的「定时任务实验台」**，且只聚焦 **Spring 这一层**：
> Spring Framework 的任务抽象、`@Scheduled` 注解体系、线程模型与生命周期、动态调度，以及 Boot 自带的可观测能力。
>
> 范围边界（重要）：
> - JDK 原生调度（Timer / ScheduledExecutorService / DelayQueue）属于 **Java 并发主题**，本项目不展开，见第三部分。
> - ShedLock / Quartz / XXL-JOB 属于 **分布式调度主题**，由后续独立的 learn-xxx 项目承载，见第三部分。
>
> 目标不变：让每个知识点都有一条可以直接跑起来、看到日志、能改参数验证的命令，而不是只留一堆笔记。

---

# 第一部分 · 需要掌握的内容

> 知识地图，按依赖顺序，不要跳。这里只放 Spring 这一层的东西。

## 1.1 Spring 的任务抽象

| 概念 | 作用 | 关键类 |
| --- | --- | --- |
| `TaskExecutor` | 异步执行任务（≈ `Executor`） | `ThreadPoolTaskExecutor`、`SimpleAsyncTaskExecutor` |
| `TaskScheduler` | 按时间调度任务 | `ThreadPoolTaskScheduler`、`SimpleAsyncTaskScheduler` |
| `Trigger` | 决定下一次执行时间 | `CronTrigger`、`PeriodicTrigger`、自定义实现 |
| `ScheduledTaskRegistrar` | 注册中心，注解背后的「登记簿」 | 配合 `SchedulingConfigurer` |
| `ScheduledTaskHolder` | 运行时读取已注册任务 | 可注入，用于做任务管理接口 |

## 1.2 注解层（日常 90% 的用法）

- `@EnableScheduling`：开启调度，本质是导入 `SchedulingConfiguration`。
- `@Scheduled` 的属性逐个吃透：

| 属性 | 语义 | 易错点 |
| --- | --- | --- |
| `cron` | 6 位：秒 分 时 日 月 周 | 不是 Linux 的 5 位；日/周互斥需用 `?`；不支持「年」字段 |
| `fixedRate` | 从**上次开始**计时，固定频率 | 任务耗时 > 间隔会重叠执行 |
| `fixedDelay` | 从**上次结束**计时 | 天然不重叠，批处理首选 |
| `initialDelay` | 首次延迟 | 单独使用时（Spring 6.1+）表示「只执行一次」 |
| `zone` | cron 的时区 | 跨时区部署必须显式指定 `Asia/Shanghai` |
| `timeUnit` | 时间单位 | 与 `*String` 形式配合更好读 |
| `scheduler` | 指定用哪个 `TaskScheduler` | 多调度器隔离时使用 |
| `cron = "-"` | 关闭该任务 | 配合 `${...}` 做开关 |

- `@Scheduled` 的方法约束：`public`、无参数、返回值会被忽略（可返回 `Future`/`Publisher`）。
- 常见失效原因：类不是 Bean、不在扫描范围、方法有参数、`@EnableScheduling` 没加、被 `final`/同类内部调用绕过代理。
- 一句话理解封装边界：`fixedRate`/`fixedDelay` 的语义与 `ScheduledExecutorService` 的两个 schedule 方法一一对应——知道这个对应关系即可，JDK 层本身不在本项目展开。

## 1.3 线程模型与生命周期（区分「会写」和「写对」）

- **默认调度线程池只有 1 个线程**：一个慢任务会阻塞所有 `@Scheduled`。
- 两种扩大并发的方式：
  - `spring.task.scheduling.pool.size=N`（配置优先）
  - 实现 `SchedulingConfigurer`，注入自定义 `ThreadPoolTaskScheduler`（需要精细控制时）
- 线程池满 / 队列满 / 拒绝策略 / `CallerRunsPolicy`。
- 优雅停机：`setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(...)`，以及 `spring.lifecycle.timeout-per-shutdown-phase`。
- 虚拟线程：`spring.threads.virtual.enabled=true` 后，调度器会换成 `SimpleAsyncTaskScheduler`（此时 `pool.size` 等池化参数失效）。
  - **实验任务**：在本项目里分别开/关，打印 `Thread.currentThread()`，实测当前 4.1.2 的默认行为，把结论写进 README。

## 1.4 动态调度（把 `@Scheduled` 变成可管理的对象）

- `ThreadPoolTaskScheduler.schedule(Runnable, Trigger)` 返回 `ScheduledFuture`，用 `cancel()` 停止。
- 维护 `Map<taskId, ScheduledFuture>` + 任务定义，就是最小的任务中心。
- 动态 cron 的两种流派：
  - 启动时用 `SchedulingConfigurer` + 自定义 `Trigger`，每次触发前读配置；
  - 直接调用 `TaskScheduler` 动态注册/取消（更灵活，本项目采用这种）。
- 与 `@Scheduled` 的区别：注解是「启动时静态注册」，动态方案是「运行时可变」。

## 1.5 可观测性（Spring Boot 自带能力）

- Actuator + Micrometer：线程池活跃数、队列长度、任务完成数。
- 自建「任务执行历史」表：开始时间 / 结束时间 / 耗时 / 状态 / 异常摘要。
- 慢任务告警阈值。

## 1.6 边界速查：哪些不在这里学

| 主题 | 内容 | 去向 |
| --- | --- | --- |
| JDK 原生调度 | `Timer`、`ScheduledExecutorService`、`DelayQueue` | 第三部分 §3.1（独立项目） |
| 分布式锁 | ShedLock、幂等兜底 | 第三部分 §3.2 |
| 嵌入式调度框架 | Quartz、misfire、集群行锁 | 第三部分 §3.2 |
| 调度中心 | XXL-JOB、分片广播 | 第三部分 §3.2 |

---

# 第二部分 · 目前计划

## 2.1 项目现状盘点

| 项 | 现状 | 说明 |
| --- | --- | --- |
| Spring Boot | 4.1.2 | 基于 Spring Framework 7.x |
| JDK | 21 | 支持虚拟线程 |
| 现有依赖 | `spring-boot-starter`、`lombok`、`spring-boot-starter-test` | 没有 web / actuator / 数据库 |
| 配置文件 | `src/main/resources/application.yaml` | 只有 `spring.application.name` |
| 代码 | 一个空的启动类 | 从零开始 |

**结论**：现在是「最干净的起点」。前 2 个里程碑只依赖日志输出，等基础打牢再引入 Web 和内嵌数据库，避免一上来被环境和配置淹没。

## 2.2 这个仓库最终要变成什么

一句话：**一个能演示「Spring 定时任务这一层」全过程、并看清它能力边界的实验台**。

最终形态（按里程碑逐步长出来）：

1. `basic` 包：`cron` / `fixedRate` / `fixedDelay` / `initialDelay` / `zone` / `timeUnit` 的对照实验，启动就能在日志里看到差异。
2. `config` 包：自定义调度线程池、优雅停机、虚拟线程开关对比。
3. `dynamic` 包：**运行时任务中心** —— 通过 API 新增 / 修改 cron / 暂停 / 恢复 / 立即执行任务，不重启应用。
4. `persistence` 包：任务定义落库（H2），重启后自动恢复。
5. `observability` 包：Micrometer 指标 + 任务执行历史表，能在页面上看到每个任务跑了多久、成功还是失败。

> 每个包对应一个可以现场演示的结论，而不是单纯堆代码。
> 「多实例重复执行、分布式锁、Quartz/XXL-JOB 对照」已移出本项目，见第三部分。

## 2.3 里程碑路线

> 每个里程碑都以「能演示、能验收」结束，不追求一次写完。

### M1 · 注解基础实验

- 新增 `basic` 包，写 6 个任务类，分别覆盖 cron / fixedRate / fixedDelay / initialDelay / zone / timeUnit。
- 在 `application.yaml` 里加日志级别：`logging.level.com.charles=debug`。
- 每个任务都打印：`线程名 + 开始时间 + 结束时间 + 耗时`。
- **验收**：启动后能肉眼看出 `fixedRate` 与 `fixedDelay` 在同一段日志里的节奏差异。

### M2 · 线程模型与优雅停机

- 加一个耗时 8 秒的任务 + 一个每秒任务，先复现「被阻塞」。
- 用 `spring.task.scheduling.pool.size=4` 修复，再对比。
- 编写 `SchedulerConfig`（`SchedulingConfigurer`）演示代码式配置。
- 演示优雅停机：收到 SIGTERM 时等待任务跑完。
- 开关虚拟线程，记录两者线程名差异。
- **验收**：能口头解释「为什么默认 1 个线程会出事故」「fixedRate 为什么会重叠」。

### M3 · 动态任务中心（核心亮点）

- 引入 `spring-boot-starter-web` 与 `spring-boot-starter-actuator`。
  - 注意：Spring Boot 4 对 starter 做了模块化拆分（web starter 可能叫 `spring-boot-starter-webmvc`），先用 IDE 补全或 `mvn dependency:tree` 确认实际坐标。
- 定义 `TaskDefinition`（id、cron、描述、状态、创建时间）。
- `TaskManager`：`add / updateCron / pause / resume / runOnce / list`，内部持有 `ScheduledFuture`。
- REST API：
  - `GET /tasks` 列出所有任务（含 `@Scheduled` 注册的，通过 `ScheduledTaskHolder` 读取）
  - `POST /tasks` 新增
  - `PUT /tasks/{id}` 改 cron（先 cancel 再重新 schedule）
  - `DELETE /tasks/{id}`
  - `POST /tasks/{id}/run` 立即执行一次
- **验收**：改完 cron 不用重启就能看到新节奏；暂停后日志停止，恢复后继续。

### M4 · 持久化与重启恢复

- 引入 `spring-boot-starter-data-jpa` + H2（内存/文件模式）。
- 任务定义落库，启动时 `ApplicationRunner` 读取并重新注册。
- 增加执行历史表（M5 复用）。
- **验收**：重启后任务自动恢复；停用状态的任务不会被恢复。
- 顺带认识 Spring 层的边界：`@Scheduled` 没有任何 misfire 补偿——重启期间错过的触发就丢了，这正是扩展项目里 Quartz「misfire 策略」存在的意义。

### M5 · 可观测性收尾

- Actuator 暴露调度相关指标，`/actuator/metrics` 查看线程池数据。
- 任务执行历史落库 + 简单查询接口。
- 写一个极简 `static/index.html` 任务面板（可选，纯演示用，不引前端框架）。
- **验收**：打开页面能看到任务列表、最近执行记录、失败次数。

## 2.4 建议的目录结构

```
src/main/java/com/charles/learnspringscheduling/
├── LearnSpringSchedulingApplication.java
├── basic/              # M1 注解对照实验
├── config/             # M2 调度器 / 线程池 / 优雅停机
├── dynamic/            # M3 动态任务中心
│   ├── TaskDefinition.java
│   ├── TaskManager.java
│   └── TriggerFactory.java
├── persistence/        # M4 实体与仓储
├── observability/      # M5 指标与执行历史
└── web/                # REST 接口
```

`application.yaml` 单 `default` profile 即可；多 profile（集群节点等）由对应的扩展项目自行引入。

## 2.5 踩坑清单（Spring 层，每个都要在项目里复现一次）

1. 忘加 `@EnableScheduling`，任务静默不执行。
2. 任务类没被 Spring 管理（不是 Bean）。
3. 方法带参数 → 启动直接报错。
4. 默认单线程被慢任务阻塞，表现为「任务没跑」，实际是排队。
5. `fixedRate` 任务耗时 > 间隔 → 同机重叠执行。
6. cron 写成 5 位（Linux crontab 习惯）→ 解析失败。
7. `日` 和 `周` 同时写具体值 → 冲突报错。
8. 跨时区部署没写 `zone` → 实际执行时间偏移 8 小时。
9. 应用重启 → 错过的触发彻底丢失，无补偿。
10. 任务里抛异常被静默吞掉（没有统一异常处理 + 告警）。
11. 优雅停机没配，发布时任务执行到一半被 kill。
12. 虚拟线程开启后仍按平台线程池思路调 `pool.size`（实际被忽略）。

> 与多实例 / 分布式锁相关的踩坑（原清单 9、14 条）已随主题迁出，见第三部分。

## 2.6 如何自测（每个里程碑统一用这套动作）

1. `./mvnw -q test`：单元测试通过。
2. `./mvnw spring-boot:run`：启动后观察日志，确认节奏符合预期。
3. 对照上面的「踩坑清单」，至少故意制造一次故障，再修复。
4. 把结论写进 `README.md`：一句话结论 + 关键配置 + 日志片段。

## 2.7 学习资料（只保留 Spring 相关）

### 官方文档（第一优先级，别用二手博客替代）

- Spring Framework 《Task Execution and Scheduling》
  https://docs.spring.io/spring-framework/reference/integration/scheduling.html
- `@Scheduled` Javadoc（属性语义以这里为准）
  https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/Scheduled.html
- `@EnableScheduling` Javadoc（含 `SchedulingConfigurer` 示例）
  https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html
- Spring Boot 《Task Execution and Scheduling》（`spring.task.*` 配置项）
  https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html
  - 4.x 快照中文镜像：https://www.spring-doc.cn/spring-boot/4.1.0-SNAPSHOT/reference_features_task-execution-and-scheduling.en.html
- Spring Boot 4.0 Release Notes / Spring Framework 7 What's New
  https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes
  https://github.com/spring-projects/spring-framework/wiki/What%27s-New-in-Spring-Framework-7.x

### 视频（B 站）

- 《我不允许有人还不知道单机、分布式、调度框架下的定时任务实现？》
  https://www.bilibili.com/video/BV1uqHreLEvs/
  > 适合建立「Spring 这一层的能力边界」的整体认知，看完再回来看代码。

### 文章

1. 《Spring Boot 定时任务与分布式调度全解析：从 @Scheduled 到 XXL-JOB》
   https://blog.csdn.net/weixin_31240035/article/details/170752141
   > 前半部分（@Scheduled）配合本项目阅读，后半部分算第三部分的预习。
2. 《SpringBoot 任务处理：异步任务 @Async、定时任务 @Scheduled，线程池配置》
   https://blog.csdn.net/SearchB/article/details/165229012
   > 面向 Boot 4.x，重点是虚拟线程与 `@Async` 失效场景；结论需自己实测验证。

### 工具与源码

- Cron 在线生成/校验：https://cron.qqe2.com/ 、 https://www.bejson.com/othertools/cron/
- Awaitility（测试异步任务）：https://github.com/awaitility/awaitility
- Micrometer 文档：https://docs.micrometer.io/micrometer/reference/

## 2.8 下一步

先做 **M1 + M2**：只依赖日志，不动依赖，先把「`@Scheduled` 到底怎么跑」这件事彻底搞明白。
确认 M1/M2 的节奏和写法符合预期后，再进入 M3 引入 Web 依赖。

---

# 第三部分 · 后续扩展（独立项目，不在本项目实现）

> 这些主题与「Spring 这一层」要么是不同的技术栈（JDK 并发），要么是 Spring 之上的第三方生态（分布式调度）。
> 按你「一堆 learn-xxx 项目」的规划，各自独立建仓库展开；本节只做素材归档和迁移对照，方便将来开新项目时直接搬走。

## 3.1 扩展一：JDK 原生调度（Java 并发主题）

建议项目名：`learn-jdk-scheduling`（或并入已有的并发主题项目）。

素材清单（原「第 0 层」内容原样保留）：

- `java.util.Timer` / `TimerTask`：单线程、异常会杀死整个 Timer，为什么被淘汰。
- `ScheduledExecutorService` / `ScheduledThreadPoolExecutor`：
  - `scheduleAtFixedRate` vs `scheduleWithFixedDelay` 的语义差别（正是 `@Scheduled` 两个属性对应的东西）。
  - 任务抛异常会导致后续执行被静默取消——这是最常见的线上事故。
- `DelayQueue`、`System.nanoTime()` 与时间漂移。

形态：纯 JDK，不引 Spring，每个知识点一个 `main` 方法跑通。

## 3.2 扩展二：分布式调度（调度框架生态主题）

建议项目名：`learn-distributed-scheduling`（或拆成 `learn-quartz`、`learn-xxl-job`）。

### 方案对照（原「第 5 层」表格）

| 方案 | 一句话定位 | 解决什么 | 不解决什么 |
| --- | --- | --- | --- |
| `@Scheduled` | Spring 原生，纯内存 | 单机定时 | 多实例重复、重启丢失、无监控 |
| `@Scheduled` + ShedLock | 给注解加分布式锁 | 多实例只跑一次 | 重启补偿、可视化、分片 |
| Quartz | 老牌嵌入式调度库 | 持久化、集群、misfire 策略 | 无管理后台，表多 |
| XXL-JOB | 调度中心 + 执行器 | 可视化、动态改 cron、分片、重试、告警 | 需要单独部署 |

### 核心心法

- **幂等性才是兜底**：锁只能降低概率，唯一索引 / 状态条件更新才是最后一道防线。

### ShedLock（原 M5）

- 用两个实例同时启动，复现「同一任务跑两遍」，再引入 ShedLock（JDBC）+ `@SchedulerLock` 修掉，对比加锁前后日志。
- `lockAtMostFor`（崩溃保护）与 `lockAtLeastFor`（防抖）的区别，`usingDbTime()` 避免节点时钟不一致。

### Quartz（原 M6）

- `JobDetail` / `Trigger` / `Scheduler` 三者关系，`Job` 每次触发都会重新实例化（不能用成员变量存状态），集群靠 `QRTZ_LOCKS` 行锁。
- 运行时增删任务、持久化到数据库、错过触发的 `misfire` 策略。
- 输出一张 `@Scheduled` vs Quartz 的能力对照表，说清「什么场景该迁移」。

### XXL-JOB（原 M8）

- Docker 起 `xxl-job-admin`，项目作为执行器接入。
- `@XxlJob` 的 Bean 模式、分片参数 `getShardIndex()/getShardTotal()`，演示分片广播。

## 3.3 资源迁移对照表

| 原计划位置 | 内容 | 新去处 |
| --- | --- | --- |
| 原第 0 层 | JDK 原生调度 | §3.1 |
| 原第 5 层 + M5 | 分布式方案对照、ShedLock | §3.2 |
| 原 M6 | Quartz 对照实现 | §3.2 |
| 原 M8 | XXL-JOB 接入 | §3.2 |
| 原踩坑清单第 9、14 条 | 多实例重复执行、锁 ≠ 幂等 | §3.2 |
| 原资料：XXL-JOB 视频、ShedLock/XXL-JOB/Quartz 仓库、文章 2/3 | 分布式调度资料 | §3.4 |

## 3.4 扩展部分的学习资料（到对应项目再展开）

### 视频（B 站）

- 《XXL-JOB 任务调度实战课程（GLUE / BEAN 模式、分片）》
  https://www.bilibili.com/video/BV1VN4y1Z77n/
- 搜索关键词：`Quartz 集群 实战`、`XXL-JOB 2.4 实战`、`ShedLock 分布式定时任务`

### 文章

1. 《商城定时任务多实例重复执行：@Scheduled、ShedLock、Quartz 和 XXL-JOB 该怎么选》
   https://blog.csdn.net/kdwll/article/details/170025387
   > 讲「锁的语义边界 + 幂等兜底」。
2. 《XXL-Job、Quartz、ShedLock、@Scheduled 技术对比》
   https://www.cnblogs.com/hanease/p/19908087

### 工具与源码

- ShedLock 官方仓库：https://github.com/lukas-krecan/ShedLock
- XXL-JOB 官方仓库：https://github.com/xuxueli/xxl-job
- Quartz 文档：https://www.quartz-scheduler.org/documentation/
