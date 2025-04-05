# Java基础补缺

## G1收集器和CMS收集器的区别，为什么G1被提出和取代

### G1 收集器和 CMS 收集器的区别

 **设计目标**

- **CMS（Concurrent Mark Sweep）收集器**：其主要目标是获取最短回收停顿时间，在进行垃圾回收时，尽量减少应用程序的停顿时间，以达到较好的响应性能，适合对响应时间要求较高的应用，如 Web 应用。
- **G1（Garbage - First）收集器**：面向服务端应用，目标是在满足高吞吐量的同时，尽可能地控制垃圾回收的停顿时间。它可以在大内存、多处理器的环境下表现良好，既能保证应用程序的性能，又能有效地管理内存。

**内存布局**



- **CMS 收集器**：采用传统的分代收集思想，将堆内存划分为新生代（又分为 Eden 区和两个 Survivor 区）和老年代，不同代采用不同的垃圾回收算法。
- **G1 收集器**：打破了传统的分代内存布局，将整个堆内存划分为多个大小相等的独立区域（Region），每个 Region 可以是 Eden 区、Survivor 区或者老年代。同时，还有专门的 Humongous 区域用于存储大对象（大小超过 Region 一半的对象）。

**垃圾回收算法**



- **CMS 收集器**：采用 “标记 - 清除” 算法。其过程分为初始标记、并发标记、重新标记和并发清除四个阶段。初始标记和重新标记阶段需要 STW（Stop - The - World），并发标记和并发清除阶段可以与应用程序并发执行。
- **G1 收集器**：整体上采用 “标记 - 整理” 算法，局部（Region 之间）采用 “复制” 算法。在进行垃圾回收时，G1 会优先回收垃圾最多的 Region，从而保证在有限的时间内获得最高的垃圾回收效率。



**停顿控制**

- **CMS 收集器**：虽然尽量减少停顿时间，但在并发标记阶段可能会产生浮动垃圾，需要在下次垃圾回收时处理。并且，“标记 - 清除” 算法会产生内存碎片，当内存碎片过多时，可能会导致提前进行 Full GC，从而增加停顿时间。

- **G1 收集器**：通过可预测的停顿时间模型，用户可以指定一个垃圾回收的最大停顿时间。G1 会根据这个目标来选择要回收的 Region 数量和顺序，从而更好地控制停顿时间。

  



**空间利用率**

- **CMS 收集器**：由于采用 “标记 - 清除” 算法，会产生内存碎片，随着时间的推移，可能会导致无法分配连续的大内存空间，降低了内存的利用率。
- **G1 收集器**：采用 “标记 - 整理” 和 “复制” 算法，不会产生内存碎片，能够更有效地利用内存空间。

### G1 被提出的原因



- **大内存管理难题**：随着硬件技术的发展，服务器的内存越来越大。传统的垃圾收集器在处理大内存时效率较低，例如 CMS 收集器在大内存下容易出现内存碎片问题，导致频繁的 Full GC。G1 收集器通过将堆内存划分为多个 Region，能够更好地管理大内存，提高垃圾回收的效率。
- **停顿时间控制需求**：现代应用程序对响应时间的要求越来越高，需要垃圾收集器能够在更短的时间内完成垃圾回收。G1 收集器提供了可预测的停顿时间模型，能够根据用户指定的停顿时间来调整垃圾回收的策略，满足了应用程序对低延迟的需求。

### G1 取代 CMS 的原因



- **内存碎片问题**：CMS 收集器的 “标记 - 清除” 算法会产生大量的内存碎片，而 G1 收集器的 “标记 - 整理” 和 “复制” 算法可以避免内存碎片的产生，提高了内存的利用率，减少了因内存碎片导致的 Full GC。
- **停顿时间控制**：G1 收集器能够更好地控制垃圾回收的停顿时间，特别是在大内存环境下，相比 CMS 收集器具有更稳定的性能表现。
- **适应性更强**：G1 收集器在处理不同大小的堆内存和不同类型的应用程序时，具有更好的适应性。它可以根据应用程序的运行情况动态调整垃圾回收策略，而 CMS 收集器的适应性相对较差。

## redis的热key和大key概念

### 热 key 概念

**定义**

热 key 指的是在 Redis 中被频繁访问的 key。在实际应用场景中，可能会有一些热点数据，比如热门商品信息、热门新闻资讯等，这些数据对应的 Redis key 会被大量的请求频繁访问。



**产生原因**

- **业务特性**：某些业务场景下，部分数据本身就具有高热度。例如电商平台在促销活动期间，热门商品的库存信息、价格信息等会被大量用户同时访问；社交媒体平台上的热门话题相关数据也会成为热 key。

- **缓存设计**：不合理的缓存设计也可能导致热 key 的产生。比如将所有请求都集中到一个 key 上，或者没有对热点数据进行有效的拆分。

  

**影响**

- **性能瓶颈**：大量的请求集中在一个热 key 上，会导致 Redis 实例的某个节点负载过高，成为性能瓶颈。这可能会引起该节点的响应时间变长，甚至出现卡顿现象。
- **网络带宽压力**：热 key 的大量请求会占用大量的网络带宽，可能会影响其他业务的正常运行。
- **集群失衡**：在 Redis 集群环境中，热 key 可能会导致某个节点的负载远远高于其他节点，造成集群负载不均衡，降低整个集群的性能和可用性。
- 

**解决方案**

- **复制热 key**：将热 key 复制到多个 Redis 节点上，让请求分散到不同的节点，从而减轻单个节点的压力。
- **使用本地缓存**：在应用层使用本地缓存（如 Guava Cache），将热 key 的数据缓存到本地，减少对 Redis 的请求。
- **限流和熔断**：对热 key 的请求进行限流，防止过多的请求打在同一个 key 上。同时，设置熔断机制，当某个 key 的请求异常时，自动熔断对该 key 的访问。

### 大 key 概念

**定义**

大 key 通常是指存储的数据量过大的 key。在 Redis 中，不同的数据类型对于大 key 的定义有所不同：



- **字符串类型**：一般认为值的大小超过 10KB 就算是大 key。
- **哈希、列表、集合、有序集合类型**：元素数量过多（如超过 1000 个）或者整体占用内存过大也可视为大 key。

**产生原因**



- **数据未拆分**：在设计数据存储时，没有对数据进行合理的拆分，将大量的数据存储在一个 key 中。例如，将一个大的 JSON 对象或者大量的用户信息存储在一个字符串类型的 key 中。
- **批量操作**：在进行批量写入操作时，可能会不小心将大量的数据写入到一个 key 中。

**影响**



- **内存分布不均**：大 key 会占用大量的内存空间，可能导致 Redis 实例的内存分布不均衡，影响其他数据的存储。
- **删除和更新慢**：删除或更新大 key 时，会消耗大量的 CPU 时间，可能会导致 Redis 实例出现卡顿现象，影响其他操作的执行。
- **网络传输慢**：获取大 key 的数据时，会占用大量的网络带宽，导致数据传输时间变长，影响应用程序的响应速度。

**解决方案**

- **数据拆分**：将大 key 拆分成多个小 key 进行存储。例如，将一个大的哈希表拆分成多个小的哈希表，或者将一个大的列表拆分成多个小的列表。
- **定期清理**：定期清理过期或无用的大 key，释放内存空间。
- **异步处理**：对于大 key 的删除和更新操作，可以采用异步处理的方式，避免阻塞 Redis 主线程。



## 除了redis制作分布式锁还有什么方式？



### ZooKeeper

**原理**

ZooKeeper 是一个分布式协调服务，它基于树形结构的节点存储数据。实现分布式锁的基本思路是：在 ZooKeeper 中创建一个临时顺序节点，每个客户端在尝试获取锁时，会在指定的锁节点下创建一个临时顺序节点。然后，客户端获取该锁节点下的所有子节点，并判断自己创建的节点是否是序号最小的节点。如果是，则表示获取到了锁；否则，客户端需要监听比自己序号小的前一个节点的删除事件，当前一个节点被删除时，客户端再次检查自己是否是序号最小的节点，以此类推。

**示例代码（使用 Apache Curator 客户端）**

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class ZooKeeperDistributedLockExample {
    private static final String ZOOKEEPER_CONNECTION_STRING = "localhost:2181";
    private static final String LOCK_PATH = "/distributed_lock";

    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient(ZOOKEEPER_CONNECTION_STRING, new ExponentialBackoffRetry(1000, 3));
        client.start();

        InterProcessMutex lock = new InterProcessMutex(client, LOCK_PATH);
        try {
            if (lock.acquire(10, java.util.concurrent.TimeUnit.SECONDS)) {
                try {
                    // 执行业务逻辑
                    System.out.println("Acquired the lock and doing business logic...");
                } finally {
                    lock.release();
                }
            }
        } finally {
            client.close();
        }
    }
}
```

**优缺点**

- **优点**：可靠性高，ZooKeeper 具有良好的容错性和一致性；支持锁的可重入性；可以实现公平锁，按照客户端请求锁的顺序依次获取锁。
- **缺点**：性能相对较低，因为涉及到网络通信和节点的创建、删除操作；部署和维护成本较高。

### 基于 `SETNX` 命令（早期实现方式）

**原理**

`SETNX`（SET if Not eXists）是 Redis 的一个原子命令，当指定的 key 不存在时，将其值设置为给定的值，并返回 1；如果 key 已经存在，则不做任何操作，并返回 0。利用这个特性，可以通过 `SETNX` 来尝试获取锁，如果返回 1 则表示获取到锁，返回 0 则表示锁已被其他客户端持有。



```java
import redis.clients.jedis.Jedis;

public class RedisLockBySetNx {
    private static final String LOCK_KEY = "distributed_lock";
    private static final String LOCK_VALUE = "locked";
    private static final int EXPIRE_TIME = 10; // 锁的过期时间，单位：秒

    public static boolean acquireLock(Jedis jedis) {
        Long result = jedis.setnx(LOCK_KEY, LOCK_VALUE);
        if (result == 1) {
            // 设置锁的过期时间，防止死锁
            jedis.expire(LOCK_KEY, EXPIRE_TIME);
            return true;
        }
        return false;
    }

    public static void releaseLock(Jedis jedis) {
        jedis.del(LOCK_KEY);
    }

    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost", 6379);
        if (acquireLock(jedis)) {
            try {
                // 执行业务逻辑
                System.out.println("Acquired the lock and doing business logic...");
            } finally {
                releaseLock(jedis);
            }
        } else {
            System.out.println("Failed to acquire the lock.");
        }
        jedis.close();
    }
}
```

**缺点**

- `SETNX` 和 `EXPIRE` 不是原子操作，如果在执行 `SETNX` 成功后，在设置过期时间之前发生异常，**可能会导致锁永远不会过期，形成死锁**。

### 基于 `SET` 命令（推荐方式）

**原理**



从 Redis 2.6.12 版本开始，`SET` 命令支持了多个可选参数，包括 `NX`（等同于 `SETNX`）和 `EX`（设置过期时间，单位为秒）或 `PX`（设置过期时间，单位为毫秒），可以将设置 key 和设置过期时间作为一个原子操作执行。

示例代码（Java 语言，使用 Jedis 客户端）

```java
import redis.clients.jedis.Jedis;

public class RedisLockBySet {
    private static final String LOCK_KEY = "distributed_lock";
    private static final String LOCK_VALUE = "locked";
    private static final int EXPIRE_TIME = 10; // 锁的过期时间，单位：秒

    public static boolean acquireLock(Jedis jedis) {
        String result = jedis.set(LOCK_KEY, LOCK_VALUE, "NX", "EX", EXPIRE_TIME);
        return "OK".equals(result);
    }

    public static void releaseLock(Jedis jedis) {
        jedis.del(LOCK_KEY);
    }

    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost", 6379);
        if (acquireLock(jedis)) {
            try {
                // 执行业务逻辑
                System.out.println("Acquired the lock and doing business logic...");
            } finally {
                releaseLock(jedis);
            }
        } else {
            System.out.println("Failed to acquire the lock.");
        }
        jedis.close();
    }
}
```



注意事项

- 释放锁时，需要确保只有持有锁的客户端才能释放锁，避免误删其他客户端的锁。可以在设置锁的值时**，使用一个唯一的标识符**（如 UUID），在释放锁时先检查锁的值是否与自己持有的标识符一致。

## ioc容器与spring容器

IoC（Inversion of Control，控制反转）容器和 Spring 容器不是完全等同的概念，但它们之间存在紧密的联系，下面从概念、关系等方面进行详细解释：

**概念定义**

- IoC 容器
  - IoC 是一种设计原则，它将对象的创建、依赖关系的管理等控制权从代码中转移到外部容器。IoC 容器就是实现这一原则的具体工具，它负责对象的实例化、生命周期管理以及对象之间依赖关系的注入。IoC 容器是一个抽象的概念，不局限于特定的框架或技术，只要符合控制反转的思想，能完成对象管理和依赖注入功能的都可以称为 IoC 容器。
- Spring 容器
  - Spring 容器是 Spring 框架的核心组件之一，它是一个具体实现了 IoC 原则的容器。Spring 框架提供了多种类型的容器，如 `BeanFactory` 和 `ApplicationContext` ，它们能够管理 Spring 应用中的各种 Bean（组件），包括创建 Bean 实例、注入依赖、管理 Bean 的生命周期等。

**两者关系**

- Spring 容器是 IoC 容器的一种实现
  - Spring 容器是基于 IoC 思想开发的，它是众多 IoC 容器实现中的一个具体例子。Spring 容器通过 XML 配置文件、Java 注解等方式来定义和管理 Bean，实现了对象的创建和依赖注入的自动化，很好地体现了 IoC 原则。
- IoC 容器概念更宽泛
  - IoC 容器是一个通用的概念，除了 Spring 容器外，还有其他框架也实现了 IoC 容器，例如 Guice（Google 开发的轻量级 Java 依赖注入框架）、PicoContainer 等。这些框架都遵循 IoC 原则，提供了对象管理和依赖注入的功能，但它们的实现方式和特点可能与 Spring 容器有所不同。

## 如何判断自己的任务是cpu密集型还是io密集型

**定义区分**

- **CPU 密集型任务**：也称为计算密集型任务，这类任务主要特点是需要大量的 CPU 计算资源。任务执行过程中，CPU 一直处于忙碌状态，几乎没有空闲时间等待其他操作完成，例如数据加密解密、视频编码解码、科学计算等。
- **I/O 密集型任务**：这类任务主要的时间花费在 I/O 操作上，比如磁盘读写、网络请求等。在进行 I/O 操作时，CPU 处于空闲状态，等待 I/O 操作完成后再继续执行后续任务，像文件读写、数据库查询、网络爬虫等都属于 I/O 密集型任务。



## sql的in排序

数据量很多时的解决方式，批量in查询以及

## cas

## redis的solr脚本以及pilpline

## mongo和es技术选型上的区分

**数据模型与存储方式**

- MongoDB
  - **特点**：MongoDB 是面向文档的数据库，采用 BSON（二进制 JSON）格式存储数据。文档以键值对的形式组织，类似于 JSON 对象，支持嵌套结构，一个集合中的文档可以有不同的结构，具有很高的灵活性。
  - **优势**：适合存储半结构化或非结构化数据，如日志、用户信息等。开发人员可以根据业务需求动态调整文档结构，无需预先定义严格的表结构，开发效率高。
- Elasticsearch
  - **特点**：ES 是基于 Lucene 的分布式搜索引擎，数据以索引（Index）、类型（Type，在 7.x 版本后逐渐弃用）和文档（Document）的形式组织。文档是 JSON 格式，索引类似于关系数据库中的数据库，用于存储相关的文档。
  - **优势**：其倒排索引的存储结构使得全文搜索和分析非常高效，能够快速定位包含特定关键词的文档。





**查询与分析能力**

- MongoDB
  - **特点**：提供了丰富的查询操作符，支持范围查询、正则表达式查询等。可以使用聚合管道进行复杂的数据处理和分析，如分组、排序、统计等。
  - **优势**：对于需要进行复杂数据处理和聚合操作的场景，MongoDB 可以方便地实现。例如，统计某个时间段内用户的消费总额、按地区分组统计订单数量等。
- Elasticsearch
  - **特点**：强大的全文搜索功能是 ES 的核心优势，支持模糊搜索、高亮显示、自动补全、同义词搜索等。同时，ES 提供了丰富的聚合功能，如桶聚合、指标聚合等，可用于数据分析和可视化。
  - **优势**：在需要进行全文搜索和实时数据分析的场景中表现出色。例如，搜索引擎、日志分析、电商商品搜索等。



**分布式架构与扩展性**

- MongoDB
  - **特点**：支持分片集群和副本集。分片集群可以将数据分散存储在多个节点上，实现数据的水平扩展；副本集提供了数据的冗余备份和高可用性，当主节点故障时，副本节点可以自动切换为主节点。
  - **优势**：能够处理大规模数据的存储和读写请求，并且在数据量增长时可以方便地进行扩展。适用于需要处理大量数据写入和读取的场景，如电商订单系统、日志存储系统等。
- Elasticsearch
  - **特点**：天生就是分布式系统，数据会被自动分片并分布到多个节点上。ES 可以根据节点的负载情况自动进行分片的迁移和均衡，具有良好的扩展性和容错性。
  - **优势**：在处理大规模数据的搜索和分析时，能够快速响应用户的查询请求。同时，ES 支持动态扩展节点，随着数据量和查询请求的增加，可以方便地添加新的节点来提高系统的性能。



**性能特点**

- MongoDB
  - **特点**：在处理大量的写入操作时表现较好，尤其是对于批量写入和更新操作，MongoDB 可以通过批量操作减少网络开销，提高写入性能。
  - **优势**：适用于需要频繁写入数据的场景，如实时日志收集、传感器数据采集等。
- Elasticsearch
  - **特点**：在全文搜索和实时数据分析方面具有极高的性能，能够在毫秒级的时间内返回搜索结果。
  - **优势**：对于需要快速响应用户搜索请求的场景，如搜索引擎、商品搜索等，ES 是一个很好的选择。





适用场景

- MongoDB
  - 内容管理系统：存储文章、图片、视频等各种类型的内容，利用其灵活的数据模型可以方便地存储不同格式的内容信息。
  - 物联网平台：处理大量传感器产生的实时数据，MongoDB 的高写入性能和分布式架构能够满足物联网数据的存储和处理需求。
  - 移动应用后端：存储用户信息、应用日志等数据，方便开发人员根据业务需求动态调整数据结构。
- Elasticsearch
  - 搜索引擎：如电商网站的商品搜索、新闻网站的文章搜索等，利用 ES 的全文搜索功能可以快速定位用户需要的信息。
  - 日志分析系统：对系统日志、应用日志进行实时分析，通过 ES 的聚合功能可以统计错误日志的数量、分析用户行为等。
  - 商业智能分析：对业务数据进行实时分析和可视化，帮助企业快速了解业务状况，做出决策。



综上所述，MongoDB 更侧重于数据的存储和复杂数据处理，而 Elasticsearch 则专注于全文搜索和实时数据分析。在技术选型时，需要根据具体的业务需求、数据特点和性能要求来选择合适的数据库。

## AQS

## 注解的意义与区别

## spring的事务注解方式如何实现有哪几种

## JVM问题排查与调优经历&过程

**1. 确认 Java 进程内存占用情况**



- **使用 `top` 或 `htop` 命令**：可以实时查看系统中各个进程的资源占用情况，包括内存使用量。找到 Java 进程的 PID（进程 ID）。

```bash
top
# 按 'P' 键按 CPU 使用率排序，按 'M' 键按内存使用率排序
```



- **使用 `ps` 命令**：可以查看指定进程的详细信息，包括内存使用情况。

```bash
ps -p <PID> -o pid,%mem,%cpu,cmd
```

**2. 分析 Java 堆内存使用情况**

- **使用 `jstat` 命令**：可以实时监控 Java 堆内存的使用情况，包括各个区域（如 Eden、Survivor、Old 区）的使用量和垃圾回收情况。

```bash
jstat -gc <PID> <interval> <count>
# 例如，每 1 秒监控一次，共监控 10 次
jstat -gc <PID> 1000 10
```

- **使用 `jmap` 命令**：可以生成 Java 堆的转储文件（Heap Dump），用于后续的内存分析。

```bash
jmap -dump:format=b,file=heapdump.hprof <PID>
```

**3.分析堆转储文件**

- **使用 VisualVM 或 YourKit 等工具**：这些工具可以加载堆转储文件，分析内存中的对象分布、对象引用关系等，找出占用大量内存的对象和可能存在的内存泄漏问题。

**4. 检查代码中的内存泄漏问题**

- **常见的内存泄漏场景**：如未关闭的资源（如文件、数据库连接、网络连接等）、静态集合中持有对象引用、内部类持有外部类引用等。
- **使用代码分析工具**：如 SonarQube、FindBugs 等，帮助检测代码中可能存在的内存泄漏问题。

### 解决思路

1. **优化代码**

- **及时释放资源**：确保在使用完资源后及时关闭，例如使用 `try-with-resources` 语句来管理资源。

```java
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // 使用文件输入流
} catch (IOException e) {
    e.printStackTrace();
}
```



- **避免静态集合中持有大量对象引用**：如果静态集合中存储了大量对象，并且这些对象不再使用时没有及时从集合中移除，会导致内存泄漏。



```java
public class StaticCollectionExample {
    private static final List<Object> staticList = new ArrayList<>();

    public static void addObject(Object obj) {
        staticList.add(obj);
    }

    public static void removeObject(Object obj) {
        staticList.remove(obj);
    }
}
```



- **避免内部类持有外部类引用**：如果内部类需要长时间存活，并且持有外部类的引用，可能会导致外部类对象无法被垃圾回收。可以使用静态内部类来避免这个问题。

```java
public class OuterClass {
    private int value;

    public OuterClass(int value) {
        this.value = value;
    }

    // 静态内部类
    public static class StaticInnerClass {
        public void doSomething() {
            // 不持有外部类引用
        }
    }
}
```



2. **调整 Java 堆内存参数**

- **增加堆内存大小**：如果是因为堆内存不足导致的内存问题，可以通过调整 `-Xmx` 和 `-Xms` 参数来增加堆内存的大小。

```bash
java -Xmx2048m -Xms1024m YourMainClass
```

- **调整垃圾回收器**：不同的垃圾回收器适用于不同的场景，可以根据应用的特点选择合适的垃圾回收器。例如，对于吞吐量要求较高的应用，可以选择 `Parallel` 垃圾回收器；对于响应时间要求较高的应用，可以选择 `CMS` 或 `G1` 垃圾回收器

```bash
# 使用 G1 垃圾回收器
java -XX:+UseG1GC YourMainClass
```



3. **优化数据结构和算法**



- **选择合适的数据结构**：根据实际需求选择合适的数据结构，避免使用占用大量内存的数据结构。例如，如果只需要存储少量数据，可以使用 `ArrayList`；如果需要频繁进行插入和删除操作，可以使用 `LinkedList`。
- **优化算法复杂度**：避免使用时间复杂度和空间复杂度较高的算法，减少内存的使用。

## 数据库的一致性与cap原则

### 最终一致性



- **定义**：在分布式系统中，数据更新操作后，不同节点上的数据副本可能不会立即保持一致，但在经过一段时间的异步传播和处理后，最终会达到一致的状态。这意味着在数据更新后的短时间内，不同节点上的数据可能存在差异，但随着时间推移，系统会保证数据最终达到一致。
- **实现原理**：通常依靠各种异步复制、消息队列、分布式事务等技术来实现。例如，在分布式数据库中，当一个节点更新了数据后，会通过异步复制机制将更新操作传播到其他节点，虽然这个传播过程可能存在延迟，但最终所有节点都会应用这些更新，从而实现数据的最终一致性。

### CAP 定理

- 定义

  ：在分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition Tolerance）这三个基本需求，最多只能同时满足其中两个。

  - **一致性（Consistency）**：在分布式系统中的所有数据备份，在同一时刻是否同样的值。即更新操作成功并返回客户端完成后，所有节点在同一时间的数据完全一致。
  - **可用性（Availability）**：系统提供的服务必须一直处于可用的状态，对于用户的每一个操作请求，总是能够在有限的时间内返回结果。
  - **分区容错性（Partition Tolerance）**：分布式系统在遇到任何网络分区故障的时候，仍然能够保证对外提供满足一致性和可用性的服务，除非整个网络环境都发生了故障。

- 三者关系

  - **CA without P**：如果不考虑分区容错性，即假设网络不会出现分区故障，那么可以同时实现一致性和可用性。比如在单一的数据库系统中，没有网络分区的问题，就可以通过事务机制等保证数据的一致性，同时也能保证系统的高可用性。
  - **CP without A**：当强调一致性和分区容错性时，可能会牺牲可用性。例如，在分布式数据库中，如果发生网络分区，为了保证数据一致性，系统可能会暂停部分服务，等待数据同步完成，这期间就会影响系统的可用性。
  - **AP without C**：如果注重可用性和分区容错性，就可能无法保证强一致性，只能实现最终一致性。如一些大规模的分布式缓存系统，为了保证在网络分区等情况下的高可用性，会允许数据在一定时间内存在不一致性，通过异步的方式来逐渐实现数据的最终一致性。



在实际的数据库设计和分布式系统架构中，需要根据具体的业务需求和场景来在 CAP 的三个要素中进行权衡和选择，以达到最优的系统性能和数据可靠性。

## 面试总结

### 阿里

分布式锁实现方式，除了redis还有啥
es原理，倒排索引什么原理
redis热 key大key
G1和CMS的区别
io和cpu密集型任务的区别
jvm的元空间
一个项目里有多个spring容器
kafka的原理
hashmap数组链表，头插尾插
concurrentHashMap原理
AQS
linux查看内存进程线程,TOP命令
redis实现锁的过程
redis缓存淘汰策略lru，lfu，区别
什么是粗排精排
ioc/aop

有没有遇到过死锁，如何解决死锁

死锁在系统资源消耗上的表现为什么？在没有充足接口日志支持的情况下如何使用linux解决系统资源开销问题

观察者模式，适配器模式有了解吗

# 设计模式

## **单例模式（Singleton Pattern）**

**目的**：确保一个类只有一个实例，并提供全局访问点。

单例模式是一种创建型设计模式，**它确保一个类只有一个实例，并提供一个全局访问点来获取这个实例**。在某些场景下，如数**据库连接池、线程池、配置文件管理等，需要确保系统中某个类只有一个实例存在**，避免资源的浪费和不一致性问题。

**实现方式**·

- **饿汉式**



```java
public class EagerSingleton {
    // 在类加载时就创建实例
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    // 私有构造函数，防止外部实例化
    private EagerSingleton() {}

    // 提供公共的静态方法获取实例
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```



- **懒汉式（线程不安全）**



```java
public class LazySingleton {
    private static LazySingleton instance;

    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();
        }
        return instance;
    }
}
```



- **懒汉式（线程安全，使用 synchronized）**



```java
public class ThreadSafeLazySingleton {
    private static ThreadSafeLazySingleton instance;

    private ThreadSafeLazySingleton() {}

    public static synchronized ThreadSafeLazySingleton getInstance() {
        if (instance == null) {
            instance = new ThreadSafeLazySingleton();
        }
        return instance;
    }
}
```



- **双重检查锁定（DCL）**

```java
public class DoubleCheckedLockingSingleton {
    private static volatile DoubleCheckedLockingSingleton instance;

    private DoubleCheckedLockingSingleton() {}

    public static DoubleCheckedLockingSingleton getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckedLockingSingleton.class) {
                if (instance == null) {
                    instance = new DoubleCheckedLockingSingleton();
                }
            }
        }
        return instance;
    }
}
```



- **静态内部类**

```java
public class StaticInnerClassSingleton {
    private StaticInnerClassSingleton() {}

    private static class SingletonHolder {
        private static final StaticInnerClassSingleton INSTANCE = new StaticInnerClassSingleton();
    }

    public static StaticInnerClassSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

**应用场景**：

- 数据库连接池。
- 配置文件管理器。

## **工厂模式（Factory Pattern）**

**目的**：定义一个创建对象的接口，但由子类决定实例化哪个类。

工厂模式是一种创建型设计模式，它**提供了一种创建对象的方式，将对象的创建和使用分离**。通过使用工厂模式，可以将对象的创建逻辑封装在一个工厂类中，使得代码更具可维护性和可扩展性。

实现方式

- **简单工厂模式**

```java
// 产品接口
interface Shape {
    void draw();
}

// 具体产品类
class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a circle");
    }
}

class Rectangle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a rectangle");
    }
}

// 简单工厂类
class ShapeFactory {
    public static Shape getShape(String shapeType) {
        if ("circle".equalsIgnoreCase(shapeType)) {
            return new Circle();
        } else if ("rectangle".equalsIgnoreCase(shapeType)) {
            return new Rectangle();
        }
        return null;
    }
}
```



- **工厂方法模式**

```java
// 产品接口
interface Product {
    void use();
}

// 具体产品类
class ConcreteProductA implements Product {
    @Override
    public void use() {
        System.out.println("Using product A");
    }
}

class ConcreteProductB implements Product {
    @Override
    public void use() {
        System.out.println("Using product B");
    }
}

// 工厂接口
interface Factory {
    Product createProduct();
}

// 具体工厂类
class ConcreteFactoryA implements Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductA();
    }
}

class ConcreteFactoryB implements Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductB();
    }
}
```



**应用场景**：

- 日志记录器。
- 数据库驱动加载。

## **抽象工厂模式（Abstract Factory Pattern）**

**目的**：提供一个创建一系列相关或相互依赖对象的接口，而无需指定具体类。

**经典案例**：

java

复制

```
interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

class WinFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WinButton();
    }
    @Override
    public Checkbox createCheckbox() {
        return new WinCheckbox();
    }
}

class MacFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
}
```

**应用场景**：

- 跨平台 UI 组件库。
- 数据库连接工厂。

## **适配器模式（Adapter Pattern）**

**定义**：适配器模式是一种结构型设计模式，它允许将一个类的接口转换成客户希望的另一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

**实现方式**  ·



- **类适配器模式**

```java
// 目标接口
interface Target {
    void request();
}

// 适配者类
class Adaptee {
    public void specificRequest() {
        System.out.println("Specific request");
    }
}

// 适配器类
class ClassAdapter extends Adaptee implements Target {
    @Override
    public void request() {
        specificRequest();
    }
}
```



- **对象适配器模式**

```java
// 目标接口
interface Target {
    void request();
}

// 适配者类
class Adaptee {
    public void specificRequest() {
        System.out.println("Specific request");
    }
}

// 适配器类
class ObjectAdapter implements Target {
    private Adaptee adaptee;

    public ObjectAdapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }

    @Override
    public void request() {
        adaptee.specificRequest();
    }
}
```

**应用场景**：

- 集成第三方库。
- 兼容旧系统。

## 观察者模式

**定义**：观察者模式是一种行为型设计模式，它定义了一种一对多的依赖关系，让多个观察者对象同时监听一个主题对象。这个主题对象在状态发生变化时，会通知所有观察者对象，使它们能够自动更新自己的状态。

**实现方式**

```java
import java.util.ArrayList;
import java.util.List;

// 主题接口
interface Subject {
    void registerObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers();
}

// 具体主题类
class ConcreteSubject implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private int state;

    public int getState() {
        return state;
    }

    public void setState(int state) {
        this.state = state;
        notifyObservers();
    }

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(state);
        }
    }
}

// 观察者接口
interface Observer {
    void update(int state);
}

// 具体观察者类
class ConcreteObserver implements Observer {
    private int observerState;

    @Override
    public void update(int state) {
        observerState = state;
        System.out.println("Observer state updated to: " + observerState);
    }
}
```

