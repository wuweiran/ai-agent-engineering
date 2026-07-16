---
layout: default
title: Java 技术栈常见面试题
parent: 后端面试题
grand_parent: 面试题库
nav_order: 1
permalink: /docs/interview/backend/java-jvm/
---

# Java 技术栈常见面试题

Java 后端面试通常从语言语义、集合与并发开始，再进入 Spring、JVM、垃圾回收和生产排障。回答实现细节时要说明适用的 Java、JDK 或 Spring 版本，避免把某个版本的内部结构说成永远不变的规范。

## Java 核心机制

## Java 代码从编译到执行经历什么？
{: #java-compilation-execution }

Java 编译器先把源码编译成平台无关的字节码，类加载器再把需要的类加载到 JVM。执行时，解释器可以立即运行字节码；热点代码会被 JIT 编译成本地机器码，以提高后续性能。

因此 Java 通常被称为“编译与解释并存”。AOT 可以提前生成本地代码，减少启动和预热时间，但会牺牲一部分运行时画像驱动的优化，并增加构建与平台适配成本。

## 对象相等与哈希集合怎样共同工作？
{: #java-object-equality-hashing }

- **`==`**：比较两个引用是否指向同一对象；
- **`equals`**：表达业务上的逻辑相等；
- **`hashCode` 约定**：`equals` 相等的对象必须具有相同哈希值。

HashMap 和 HashSet 先用 `hashCode` 定位桶，再用 `equals` 确认对象。只重写 `equals` 不重写 `hashCode`，会破坏查找和去重；作为 Key 的对象也不应修改参与判等和哈希计算的字段。

## String 的不可变性解决了什么问题？
{: #java-string-immutability }

String 创建后内容不能改变，因此能够安全共享、缓存哈希值并进入字符串常量池。类名、文件路径和网络地址等值也不会在被其他代码持有后悄悄变化。

少量拼接或编译期常量可以直接使用 String；循环或批量拼接通常使用 StringBuilder。StringBuffer 提供同步方法，但更常见的做法是避免在线程间共享同一个字符串构建器。

## 基本类型、包装类型和自动装箱有哪些关键差异？
{: #java-primitive-wrapper-boxing }

基本类型直接表示数值或布尔值，不能为 null；包装类型是对象，可用于泛型和集合，也会增加对象、拆装箱和空值处理成本。

自动装箱由编译器转换成包装创建或 `valueOf` 调用，拆箱则读取其中的基本值。对 null 包装类型拆箱会抛出 NullPointerException，频繁拆装箱还可能制造额外分配。部分包装值存在缓存，因此不能用 `==` 判断包装对象的数值相等。

## 为什么金额计算通常使用 BigDecimal？
{: #java-bigdecimal-money }

`float` 和 `double` 使用二进制浮点表示，许多十进制小数无法被精确保存，连续运算会出现舍入误差。

金额通常使用 BigDecimal 或最小货币单位的整数表示。创建 BigDecimal 时应优先使用字符串或 `valueOf`，并明确精度与舍入规则；比较数值大小通常使用 `compareTo`，因为 `equals` 还会比较小数位数。

## Java 泛型擦除带来哪些限制？
{: #java-type-erasure }

Java 泛型主要在编译期提供类型检查。编译后，大部分类型参数会被擦除为其上界，必要位置由编译器插入类型转换和桥接方法。

因此运行时通常不能直接区分 `List<String>` 与 `List<Integer>`，也不能直接创建类型参数实例或泛型数组。框架需要保留泛型信息时，会显式传入 Class、Type 或类型令牌。

## 反射适合什么场景？有什么代价？
{: #java-reflection }

反射让程序在运行时检查类、构造器、方法、字段和注解，并进行动态调用。依赖注入、序列化、测试和插件框架经常使用它。

代价包括更弱的编译期检查、较难重构、额外调用开销和更复杂的模块访问控制。反射适合框架边界，不应替代本来可以直接调用的清晰业务接口。

## checked exception 和 unchecked exception 怎样划分？
{: #java-exception-design }

- **checked exception**：必须捕获或声明，适合调用方能够合理恢复的外部失败；
- **unchecked exception**：继承 RuntimeException，编译器不强制处理，常用于参数错误、非法状态和程序缺陷。

划分重点不是语法，而是**调用方能否采取明确行动**。权限不足、远程超时和结果未知要保留不同语义，不能全部包装成同一个 RuntimeException；文件和连接使用 try-with-resources 关闭。

## Java 17/21 中哪些语言与运行时特性值得关注？
{: #java-modern-features }

常见特性包括 record、sealed class、模式匹配、文本块，以及 Java 21 的虚拟线程。record 适合表达以数据为主的浅不可变载体；sealed class 限制允许的继承层次；模式匹配减少类型判断样板代码。

虚拟线程降低大量阻塞任务的线程成本，但不会提高数据库、下游服务和 CPU 的真实容量，也不应与平台线程池的调参方法直接混为一谈。

## Java 集合

## List、Set、Queue 和 Map 的选择依据是什么？
{: #java-collection-selection }

List 保留元素顺序并允许重复；Set 表达唯一元素集合；Queue 和 Deque 表达排队或双端操作；Map 保存 Key 到 Value 的映射。

选择时先看业务语义，再看访问模式、顺序、并发和内存。集合接口表达需要什么能力，具体实现决定性能与线程安全。

## ArrayList 和 LinkedList 怎样选择？
{: #java-list-choice }

ArrayList 基于连续数组，随机访问快、内存局部性好，尾部追加通常高效；扩容和中间插入需要复制或移动元素。LinkedList 基于双向链表，按索引访问需要遍历，每个节点还有额外引用开销。

即使经常在中间插入，LinkedList 也不一定更快，因为找到位置仍要遍历。大多数列表场景优先使用 ArrayList，再根据真实访问模式和性能数据判断。

## HashMap 的查询、冲突和扩容机制是什么？
{: #java-hashmap-mechanics }

HashMap 使用数组保存桶，根据 Key 的哈希值定位桶，再通过哈希值和 `equals` 确认 Key。不同 Key 落入同一桶时形成冲突，现代 JDK 在极端冲突下可能把链式结构转换为红黑树。

元素超过负载条件后，HashMap 会创建更大的桶数组并重新分配节点。合理初始容量可以减少扩容，但树化阈值、扩容细节等属于 JDK 实现，不是 Map 接口保证。

## HashMap 与 ConcurrentHashMap 的并发边界有什么不同？
{: #java-hashmap-concurrenthashmap }

- **HashMap**：不提供线程安全保证，并发修改可能造成覆盖、丢失更新或不稳定读取；
- **ConcurrentHashMap**：通过 CAS、桶级同步和可见性控制协调更新，读取通常不锁整张表。

但**单个方法线程安全不等于组合操作原子**。“没有则创建”应使用 `computeIfAbsent` 或 `putIfAbsent`，不能先 `get` 再 `put`。

## 迭代集合时修改数据会发生什么？
{: #java-collection-iteration }

普通集合的迭代器通常采用 fail-fast 检查，检测到迭代期间出现非预期结构修改时，尽快抛出 ConcurrentModificationException。它用于暴露错误，不是并发安全保证。

需要删除当前元素时使用迭代器提供的操作；并发读写时选择合适的并发集合或快照策略。不同并发集合可能提供弱一致性迭代，不保证观察到同一时刻的完整快照。

## BlockingQueue 在生产者消费者模型中解决什么问题？
{: #java-blocking-queue }

BlockingQueue 在队列为空时阻塞消费者，在容量受限且队列满时阻塞或拒绝生产者，减少手写等待通知逻辑。

有界队列还能把下游容量反馈给生产者；无界队列则可能把过载隐藏成持续积压。ArrayBlockingQueue 使用固定数组，LinkedBlockingQueue 使用链式节点并可设置容量，选择时要考虑内存、吞吐和背压需求。

## Java 并发

## 线程和进程有什么区别？
{: #java-process-thread }

进程拥有独立的地址空间和系统资源，是操作系统隔离程序的主要边界。线程属于进程，共享堆和打开的资源，但各自拥有程序计数器、栈和执行状态。

线程间通信成本较低，同时也更容易发生数据竞争；进程隔离更强，通信和切换通常更重。

## 为什么业务代码通常使用线程池，而不是不断创建线程？
{: #java-thread-pool }

线程池复用线程，控制并发数量，并通过队列和拒绝策略处理暂时无法执行的任务。不断创建线程会增加启动、栈内存和上下文切换成本，还可能耗尽操作系统资源。

线程池不是越大越好。它必须与 CPU、下游连接池和任务期限共同设计，否则只是把压力更快送到瓶颈。

## `synchronized` 和 `Lock` 有什么区别？
{: #java-synchronized-lock }

`synchronized` 是语言内置的互斥与可见性机制，进入和退出代码块时自动获取、释放监视器。Lock 是显式接口，可以支持可中断等待、超时获取、多个条件队列和公平策略。

Lock 必须在 `finally` 中可靠释放。需求简单时优先使用 `synchronized`；确实需要高级等待控制时再使用 Lock。

## `volatile` 保证了什么？不能保证什么？
{: #java-volatile }

`volatile` 保证一个线程写入后，其他线程能够看到最新值，并限制相关内存操作的重排序。它适合状态标记和单次读写的共享变量。

它不能让“读取、计算、写回”这样的复合操作自动变成原子操作。`count++` 仍可能在并发下丢失更新。

## Java 内存模型与 happens-before 解决什么问题？
{: #java-memory-model-happens-before }

Java 内存模型规定线程之间何时必须看到彼此的写入，以及编译器和处理器可以怎样重排序。happens-before 描述一组可见性和顺序关系。

锁释放先于后来对同一锁的获取，volatile 写先于后来对同一变量的读，线程启动和结束也有相应规则。没有这些关系时，代码不能依赖某次写入一定被其他线程及时看到。

## CAS 是什么？ABA 问题是什么？
{: #java-cas-aba }

CAS 会比较某个位置是否仍等于预期值，相等时才原子地写入新值。它是许多无锁算法和原子类的基础，冲突时通常通过重试推进。

ABA 指值从 A 变成 B 又回到 A，CAS 只看当前值时会误以为没有变化。可以加入版本号或标记解决，但高冲突下持续自旋仍会消耗 CPU。

## ThreadLocal 适合什么场景？有什么风险？
{: #java-threadlocal }

ThreadLocal 为每个线程保存独立变量，适合在线程内传递请求上下文或保存不应共享的临时对象。

线程池会长期复用线程。如果任务结束后没有调用 `remove`，旧数据可能泄漏内存，也可能被同一线程执行的下一个请求读到。请求上下文还可能在异步切换线程后丢失，需要显式传播。

## CompletableFuture 适合什么场景？
{: #java-completable-future }

CompletableFuture 适合组合多个异步结果，例如并行查询订单、库存和物流，再汇总输出。它提供转换、组合、异常处理和超时控制。

默认线程池不一定适合阻塞 I/O 或关键业务。生产代码应明确执行器、异常传播、取消和整体截止时间，避免异步任务静默失败或占满公共线程池。

## 线程池的核心参数怎样共同工作？
{: #java-thread-pool-parameters }

核心线程数决定常驻执行能力，最大线程数限制扩展并发，工作队列保存等待任务，存活时间控制非核心线程回收，线程工厂负责命名和异常处理，拒绝策略处理无法接收的新任务。

不同队列会改变最大线程数何时生效。使用无界队列时，任务通常持续排队，线程池可能永远不会扩展到最大线程数。

## 线程池大小怎样选择？
{: #java-thread-pool-sizing }

CPU 密集任务的并发通常接近可用核心数，避免过多上下文切换。I/O 密集任务可以有更多并发，因为线程会等待外部资源，但上限仍受数据库连接、下游配额、内存和任务期限约束。

公式只能作为起点。最终要通过压测观察吞吐、队列、尾部延迟、资源等待和拒绝情况。

相关内容：[容量、排队与背压]({{ site.baseurl }}/docs/backend/capacity-backpressure/)。

## 死锁怎样产生、发现和避免？
{: #java-deadlock }

多个线程分别持有对方需要的资源，并形成循环等待时会发生死锁。典型现场是线程长期处于 BLOCKED，线程 Dump 能显示锁拥有者和等待关系。

常见预防方式包括统一加锁顺序、缩短锁范围、避免持锁等待远程调用，以及使用可超时或可中断的锁。检测到死锁后通常要终止或重启受影响执行，并修复锁顺序。

## Spring 与 Spring Boot

## IoC、DI 和 Spring Bean 是什么关系？
{: #spring-ioc-di }

IoC 把对象的创建、装配和生命周期交给 Spring 容器；DI 是容器把依赖交给对象的主要方式；被容器管理的对象就是 Spring Bean。

构造器注入通常最清楚：必需依赖在创建时就完整存在，字段可以保持 `final`，测试也能直接构造对象。Spring 默认 singleton 只表示共享一个实例，不保证 Bean 内部线程安全。

## Spring AOP 与动态代理是什么关系？
{: #spring-aop-proxy }

Spring AOP 通常通过代理拦截 Bean 方法，在调用前后加入事务、权限、日志和指标等横切逻辑。JDK 动态代理基于接口，CGLIB 通过生成目标类子类实现代理。

两种方式的共同边界是调用必须经过代理。`final`、`private` 等无法被对应代理方式拦截的方法，以及对象内部的 `this.method()` 自调用，都可能绕过 Advice。

## `@Transactional` 的原理和常见失效场景是什么？
{: #spring-transactional }

**原理：**`@Transactional` 提供事务元数据，Spring 代理在方法调用前创建或加入事务，正常结束提交，异常符合规则时回滚。

**常见失效场景：**

- 对象不是 Spring Bean；
- 同类 `this.method()` 自调用绕过代理；
- 方法无法被代理；
- 异常被内部吞掉，或异常类型不满足回滚规则；
- 异步线程没有继承当前事务。

默认 RuntimeException 和 Error 回滚，普通 checked exception 默认不回滚。

## `REQUIRED` 和 `REQUIRES_NEW` 有什么区别？
{: #spring-transaction-propagation }

`REQUIRED` 是默认传播行为：已有事务就加入，没有则新建。内部参与者把事务标记为 rollback-only 后，外层提交可能收到 UnexpectedRollbackException。

`REQUIRES_NEW` 会挂起外层事务并创建独立物理事务，能够独立提交或回滚，但会额外占用数据库连接。它不能让两个事务自动形成可靠的跨步骤业务结果。

## Spring 循环依赖应该怎样理解？
{: #spring-circular-dependency }

循环依赖表示 Bean 对象图中存在相互引用。构造器循环依赖无法在任何一方创建前取得完整对象；部分 singleton 字段或 Setter 循环在特定条件下可以通过提前引用处理。

Spring Boot 近年默认不鼓励循环依赖。比“三级缓存怎样解决”更重要的是判断职责是否耦合，应优先拆分服务、提取第三个协作者或改为事件交互。

## Spring Boot 自动配置怎样工作？
{: #spring-boot-auto-configuration }

Spring Boot 根据类路径依赖、配置属性和容器中已有 Bean 判断哪些自动配置满足条件。用户显式提供相关 Bean 后，默认自动配置通常会退让。

`@SpringBootApplication` 组合了配置类、组件扫描和自动配置入口。排查时可以查看 Conditions Evaluation Report；不需要的自动配置可以显式排除。

## JVM 内存与类加载

## JVM 运行时内存区域有哪些？
{: #jvm-runtime-memory }

堆主要保存对象实例；虚拟机栈保存每个线程的方法栈帧、局部变量和调用状态；程序计数器记录当前执行位置；本地方法栈服务本地方法；方法区保存类元数据等信息，HotSpot 通常使用 Metaspace 实现。

不同区域的生命周期和异常不同。排查内存问题时，先确认增长的是 Java 堆、Metaspace、线程栈还是直接内存。

## 堆和虚拟机栈有什么区别？
{: #jvm-heap-stack }

堆由线程共享，主要保存对象并由垃圾回收器管理。每个线程拥有独立虚拟机栈，方法调用创建栈帧，方法结束后栈帧退出。

堆不足常见为 OOM，递归过深或栈空间不足可能导致 StackOverflowError。对象引用可以位于栈帧中，实际对象通常位于堆中。

## 对象一定分配在堆上吗？什么是逃逸分析？
{: #jvm-escape-analysis }

从 Java 语言语义看，对象不承诺具体物理分配位置。JIT 可以通过逃逸分析判断对象是否会被方法或线程外部使用，并进行标量替换、锁消除等优化。

因此不能仅凭源码中的 `new` 判断一定发生了可观察的堆分配。具体优化取决于 JVM、代码形态和运行时编译结果。

## 类加载过程包括哪些阶段？
{: #jvm-class-loading }

类从字节码进入可用状态，通常经历加载、验证、准备、解析和初始化。加载取得类的二进制表示；验证检查字节码安全；准备分配类变量并设置初始值；解析把符号引用转换为直接引用；初始化执行类初始化逻辑。

加载、连接和初始化的具体时机可能是惰性的，不代表应用启动时会立即初始化所有类。

## 双亲委派模型解决什么问题？
{: #jvm-parent-delegation }

类加载器收到请求时，通常先委派父加载器尝试加载，父加载器无法完成后才由自己处理。这样可以避免核心类被应用中的同名类随意替换，并减少重复加载。

插件系统、模块化容器和线程上下文类加载器可能采用不同委派策略，以支持隔离或加载应用提供的实现。

## `ClassNotFoundException` 和 `NoClassDefFoundError` 有什么区别？
{: #jvm-class-not-found }

ClassNotFoundException 通常发生在代码主动按名称加载类却没有找到时，是可以捕获的异常。NoClassDefFoundError 表示 JVM 在运行时需要某个类，但该类无法被定义或初始化，是错误类型。

后者不一定只是缺少 JAR；类初始化曾经失败，后续再次使用时也可能出现 NoClassDefFoundError。

## OOM 和 StackOverflowError 有什么区别？
{: #jvm-oom-stack-overflow }

OOM 表示 JVM 无法为某类内存请求提供空间，可能发生在堆、Metaspace、直接内存或线程创建等位置。StackOverflowError 通常表示单个线程的调用栈耗尽，常见原因是无限递归或调用层次过深。

排查时不能只看异常名称，要结合错误消息、堆 Dump、线程数量和 JVM 参数确认耗尽的区域。

## 垃圾回收

## JVM 怎样判断对象可以回收？
{: #jvm-reachability-analysis }

主流 JVM 使用可达性分析：从 GC Roots 出发遍历引用关系，不可到达的对象才可能被回收。

引用计数实现简单，却难以处理循环引用。可达性分析能够识别相互引用但整体已经不可达的对象。

## 常见 GC Roots 包括哪些对象？
{: #jvm-gc-roots }

常见 GC Roots 包括线程栈中的局部引用、已加载类的静态引用、JNI 引用以及 JVM 内部持有的对象。

GC Roots 数量和引用链会影响对象是否存活。排查泄漏时，堆分析工具通常沿保留路径找到谁仍在持有目标对象。

## 标记清除、复制和标记整理有什么区别？
{: #jvm-gc-algorithms }

标记清除回收不可达对象，但容易产生内存碎片。复制算法把存活对象移动到另一块区域，分配简单，但需要额外空间。标记整理把存活对象向一端移动，减少碎片，但移动成本较高。

现代收集器会根据分区和对象年龄组合这些方法，而不是整堆只使用一种算法。

## Minor GC、Major GC 和 Full GC 有什么区别？
{: #jvm-gc-types }

Minor GC 通常指年轻代回收；Major GC 常被用来指老年代回收；Full GC 通常处理整个堆，并可能同时处理类元数据等区域。

这些名称在不同收集器和日志中并不完全统一。生产排查应以具体 JVM 的 GC 日志事件、停顿范围和触发原因判断。

## Stop-The-World 是什么？
{: #jvm-stop-the-world }

Stop-The-World 表示 JVM 暂停应用线程，让垃圾回收或其他运行时操作在稳定状态下工作。不同收集器会把更多阶段并发执行，以缩短单次停顿。

低停顿不等于没有停顿。选择收集器时要在吞吐量、延迟、内存开销和 CPU 消耗之间取舍。

## G1、ZGC 和 Shenandoah 的目标有什么不同？
{: #jvm-gc-collectors }

G1 把堆划分为 Region，在可预测停顿目标下分阶段回收，适合较大堆和通用服务。ZGC 与 Shenandoah 把更多标记、转移工作并发化，重点降低大堆下的停顿时间，但会使用更多 CPU、内存和读屏障能力。

选择应依据 JDK 版本、堆大小、延迟目标和真实压测，不应只按“更新的收集器一定更好”判断。

## 内存泄漏和内存溢出有什么区别？
{: #jvm-memory-leak-oom }

内存泄漏指不再有业务价值的对象仍被引用，导致内存无法回收。内存溢出是内存分配最终失败的结果，可能由泄漏引起，也可能只是容量不足或瞬时流量过大。

泄漏通常表现为多次 GC 后存活集合持续增长，需要通过堆 Dump 和引用链找到异常持有者。

## 生产排障

## Java 进程 CPU 100% 怎样排查？
{: #java-high-cpu-troubleshooting }

先确认进程和高 CPU 线程，再把操作系统线程 ID 对应到线程 Dump 中的 Java 线程，检查其调用栈。常见原因包括死循环、频繁自旋、正则回溯、序列化热点和频繁 GC。

一次线程 Dump 可能只是瞬时现场，应连续采样并结合火焰图、请求 Trace 和 GC 指标确认持续热点。

## 频繁 Full GC 怎样排查？
{: #java-full-gc-troubleshooting }

先查看 GC 日志中的触发原因、停顿时间、回收前后占用和对象晋升情况，再判断是堆过小、对象分配过快、长期存活对象增长、Metaspace 压力还是显式调用 GC。

只增加堆可能延迟故障并放大停顿。怀疑泄漏时，应比较多个堆 Dump 的存活对象和引用链。

## 堆内存持续增长怎样判断是否泄漏？
{: #java-memory-leak-troubleshooting }

观察完整 GC 后的基线是否持续上升，而不是只看进程使用量。保留堆 Dump 后，按对象数量、占用和支配关系找到异常增长类型，再沿 GC Root 引用路径定位持有者。

缓存增长可能是正常容量变化，也可能缺少边界。还要结合业务负载、缓存策略和对象生命周期判断。

## 接口卡住但 CPU 很低，怎样检查？
{: #java-low-cpu-stuck-request }

CPU 低通常说明线程正在等待，而不是没有问题。线程 Dump 可以检查 BLOCKED、WAITING 和 TIMED_WAITING 状态，Trace 则能定位连接池、锁、远程调用或队列等待。

还要观察线程池队列、数据库连接池和下游并发。盲目增加线程可能把等待放大到更多请求。

相关内容：[超时、重试与故障隔离]({{ site.baseurl }}/docs/backend/service-timeouts/)、[容量、排队与背压]({{ site.baseurl }}/docs/backend/capacity-backpressure/)。

## 服务发生 OOM 后应该保留哪些现场？
{: #java-oom-evidence }

至少保留完整错误信息、JVM 参数、GC 日志、堆 Dump、线程 Dump、进程和容器内存指标，以及事故前后的流量与版本变化。

需要提前配置安全的 Dump 路径和磁盘容量。重启可以恢复服务，但没有现场就难以区分 Java 堆泄漏、直接内存、线程过多或容器限制。
