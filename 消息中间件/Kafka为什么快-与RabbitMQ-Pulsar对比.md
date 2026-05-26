# Kafka 为什么快 & 与 RabbitMQ/Pulsar 对比

## Kafka 为什么快

Kafka 的高吞吐来自多个层面的设计决策协同作用。

### 1. 顺序 I/O（Sequential I/O）

Kafka 的核心存储是 append-only log。写入只追加到文件末尾，读取按偏移量顺序扫描。

**原理**：磁盘顺序读写的吞吐量可以达到 600MB/s+（SSD 更高），接近内存随机读写的速度。而随机 I/O 受限于寻道时间（HDD ~10ms/次），吞吐量骤降到几十 MB/s。Kafka 把消息存储建模为日志段文件（segment），写入永远是 append，读取通过 offset 索引定位后顺序读，完美利用了操作系统的预读（read-ahead）和写合并（write-behind）。

#### ⚠️ 顺序写的成立条件与退化

严格来说，Kafka 的"顺序写"是 **per-partition 级别的顺序**，不是磁盘级别的顺序。

当一块 HDD 上承载了 N 个活跃 Partition（每个 Partition 各自 append 到不同的 segment 文件），从磁盘视角看：

```
Partition-0 append → 文件位置 A
Partition-1 append → 文件位置 B
Partition-2 append → 文件位置 C
...交替写入...
```

磁头在 A、B、C 之间跳来跳去，本质上退化为**随机写**。Partition 数越多、流量越均匀，退化越严重。

| 条件 | 顺序写优势 |
|------|-----------|
| 少量 Partition / 单 Partition 主导流量 | 强（接近纯顺序） |
| 大量 Partition + 均匀流量 + HDD | 弱（退化为随机写） |
| 大量 Partition + 均匀流量 + SSD | 影响小（SSD 随机写性能本身就高） |
| 读取侧（消费） | 顺序读优势始终存在（单 Partition 内按 offset 顺序消费） |

#### RocketMQ 的反向设计（对比）

RocketMQ 正是看到了这个问题，选择了完全不同的存储模型：

- **CommitLog**：所有 Topic 的消息写入同一个文件，真正的单文件顺序写
- **ConsumeQueue**：每个 Topic/Queue 一个索引文件，记录消息在 CommitLog 中的偏移量

```
写入路径：所有消息 → 单一 CommitLog（磁盘级顺序写 ✓）
读取路径：ConsumeQueue 索引 → 定位 CommitLog offset → 读取消息（随机读，靠 page cache 弥补）
```

### 2. Page Cache（操作系统页缓存）

Kafka 不自己管理内存缓存，而是依赖 OS 的 page cache。

**原理**：JVM 堆内存有 GC 开销，对象头占用大。Kafka 选择把数据直接写文件，让 OS 用剩余物理内存做 page cache。好处是：

- 避免 GC 停顿
- 进程重启后缓存仍在（page cache 属于内核）
- OS 的 LRU 淘汰策略已经足够好

### 3. Zero-Copy（零拷贝）

消费者读取消息时，Kafka 使用 `sendfile()` 系统调用。

**传统路径**：磁盘 → 内核缓冲区 → 用户空间 → socket 缓冲区 → 网卡（4次拷贝，2次上下文切换）

**Zero-Copy 路径**：磁盘 → 内核缓冲区 → 网卡（2次拷贝，0次用户态切换）

省掉了数据在用户空间的中转，CPU 几乎不参与数据搬运。

### 4. 批量处理（Batching）

Producer 端攒批发送，Broker 端批量落盘，Consumer 端批量拉取。

**原理**：网络请求的开销主要在 RTT 和协议头，而非 payload。批量化把每条消息的均摊开销从"一次网络往返"降到"几乎为零"。同时批量写入让磁盘 I/O 更连续。

#### Producer 端批量控制

**Java 客户端**核心配置：

| 配置 | 默认值 | 作用 |
|------|--------|------|
| `batch.size` | 16384 (16KB) | 单个 batch 的字节上限，攒满就发 |
| `linger.ms` | 0 | 攒批等待时间，到时间就发（即使没攒满） |
| `buffer.memory` | 33554432 (32MB) | Producer 端总缓冲区大小 |
| `acks` | all | 等待多少副本确认 |

两个触发条件是 OR 关系：`batch.size` 满了，或 `linger.ms` 到了，先满足哪个就触发发送。

#### Go 客户端差异（重要）

**Sarama SyncProducer —— 真正的逐条同步，无攒批**

```go
producer, _ := sarama.NewSyncProducer(brokers, config)

// 阻塞直到这条消息被 Broker ack，不会和其他消息合并 batch
partition, offset, err := producer.SendMessage(&sarama.ProducerMessage{
    Topic: "my-topic",
    Value: sarama.StringEncoder("hello"),
})
```

Sarama 的 SyncProducer 底层虽然包装了 AsyncProducer，但它是逐条发送、逐条等待的。每调一次 `SendMessage` 就是一次完整的网络往返。这和 Java 客户端不同（Java 即使 `.get()` 阻塞，accumulator 仍然可能合并并发线程的消息）。

**Sarama AsyncProducer —— 有批量能力**

```go
config := sarama.NewConfig()
config.Producer.Flush.Frequency = 500 * time.Millisecond  // 攒批时间窗口
config.Producer.Flush.Messages = 100                       // 攒够多少条就发
config.Producer.Flush.Bytes = 16384                        // 攒够多少字节就发
config.Producer.Return.Successes = true

producer, _ := sarama.NewAsyncProducer(brokers, config)

// 非阻塞，扔进 channel
producer.Input() <- &sarama.ProducerMessage{
    Topic: "my-topic",
    Value: sarama.StringEncoder("hello"),
}

// 另一个 goroutine 收结果
select {
case msg := <-producer.Successes():
    fmt.Printf("sent to partition %d offset %d\n", msg.Partition, msg.Offset)
case err := <-producer.Errors():
    fmt.Printf("failed: %v\n", err.Err)
}
```

Flush 三个条件也是 OR 关系，**默认全是 0（不攒批，消息进来就立即发送）**。

**confluent-kafka-go（librdkafka 封装）—— 行为接近 Java**

```go
p, _ := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers": "localhost:9092",
    "batch.size":        16384,
    "linger.ms":         5,
    "acks":              "all",
})

deliveryChan := make(chan kafka.Event)

p.Produce(&kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
    Value:          []byte("hello"),
}, deliveryChan)

// 等待确认（模拟同步）
e := <-deliveryChan
msg := e.(*kafka.Message)
```

底层是 librdkafka（C 库），内部有 accumulator，永远异步。即使阻塞等 delivery channel，其他 goroutine 的消息仍然会被合并。

**Go 客户端对比总结**

| 特性 | Java Producer | Sarama SyncProducer | Sarama AsyncProducer | confluent-kafka-go |
|------|--------------|--------------------|--------------------|-------------------|
| 底层模型 | 永远异步 + accumulator | 真同步，逐条发送 | 异步 + channel | 永远异步（librdkafka） |
| 默认攒批 | 是 | 否 | 否（Flush 默认全 0） | 是 |
| 同步模式下仍有 batch | 是 | **否** | N/A | 是 |
| 性能上限 | 高 | 低（每条一次 RTT） | 高（配置 Flush 后） | 高 |

#### acks 的三档（控制可靠性，非批量行为）

```
acks=0    → fire and forget，不等任何确认（最快，可能丢消息）
acks=1    → Leader 写入本地 log 就返回（Leader 挂了可能丢）
acks=all  → 等所有 ISR 副本确认（最可靠，延迟最高）
```

一个 batch 作为整体发送，收到一个 ack 代表整个 batch 被确认。

#### Broker 端行为

Broker 端没有"攒批"逻辑 —— 它是被动接收 Producer 发来的 batch。

**Batch 原样落盘**：Broker 收到 ProduceRequest 后，不拆开 batch、不解压、不重新序列化。整个 batch 作为一个整体直接 append 到 segment 文件。Broker 的 CPU 开销极低 —— 它只是一个"搬运工"。

**刷盘策略**：

| 配置 | 默认值 | 含义 |
|------|--------|------|
| `log.flush.interval.messages` | Long.MAX_VALUE | 累积多少条消息后强制 fsync |
| `log.flush.interval.ms` | null | 多久强制 fsync 一次 |

默认不主动 fsync，完全依赖 OS page cache 异步刷盘。数据可靠性靠多副本（replication）保证，而不是靠单机 fsync。

#### 批量化贯穿全链路

```
Producer accumulator (攒批)
    ↓ 一个 batch = 一次网络请求
Broker (原样写入，不拆包)
    ↓ Follower 批量拉取复制
Consumer (批量 fetch，一次拉多条)
    ↓ 本地解压 + 反序列化
Application
```

batch 格式全链路一致（不需要反复序列化/反序列化）。

### 5. 分区并行（Partitioning）

一个 Topic 分成多个 Partition，每个 Partition 是独立的日志文件，可以分布在不同 Broker 上。

**原理**：吞吐量随 Partition 数线性扩展。Producer 可以并行写不同 Partition，Consumer Group 内的消费者各自负责不同 Partition，天然并行。

### 6. 压缩（Compression）

支持 gzip/snappy/lz4/zstd，在 Producer 端压缩整个 batch，Broker 原样存储，Consumer 端解压。

**原理**：同一 batch 内的消息往往结构相似，压缩比高。减少网络带宽和磁盘 I/O。

### 7. 拉模型（Pull-based Consumer）

Consumer 主动拉取，而非 Broker 推送。

**原理**：Consumer 按自己的速度消费，不会被 Broker 压垮。同时拉取时可以一次拉一大批，配合批量处理。

---

## Kafka 快的代价与缺陷

| 代价 | 根因 |
|------|------|
| 不支持灵活路由 | 只有 Topic + Partition 的简单模型，没有 exchange/binding/routing key |
| 消费模型受限 | 一个 Partition 同一时刻只能被 Consumer Group 内的一个消费者消费 |
| 消息级确认缺失 | 只有 offset 提交，不支持单条消息 ack/nack/requeue |
| 延迟不是最优 | 批量化提高吞吐但增加延迟（攒批等待时间），P99 通常 ms~十几 ms 级 |
| Topic/Partition 过多时性能退化 | 每个 Partition 是独立文件，数千个 Partition 时随机 I/O 增加 |
| 不适合任务队列场景 | 没有优先级队列、延迟队列（原生）、死信队列等传统 MQ 特性 |
| 存储与计算耦合 | Broker 既负责存储又负责服务请求，扩容需要数据迁移 |

---

## 三者对比：Kafka vs RabbitMQ vs Pulsar

### 架构模型

| 维度 | Kafka | RabbitMQ | Pulsar |
|------|-------|----------|--------|
| 核心模型 | 分布式提交日志（Distributed Log） | 消息代理（Message Broker） | 分层架构（计算存储分离） |
| 存储 | Broker 本地磁盘 segment 文件 | 内存 + 可选磁盘持久化 | BookKeeper（独立存储层） |
| 消费模型 | Pull（拉） | Push（推）为主，也支持 Pull | Pull（拉） |
| 消息路由 | Topic → Partition（简单 hash） | Exchange → Binding → Queue（灵活路由） | Topic → Subscription（多种订阅模式） |
| 协议 | 自有二进制协议 | AMQP 0.9.1（标准协议） | 自有二进制协议 |
| 多租户 | 弱（靠 ACL） | 弱 | 原生支持（namespace/tenant） |

### 性能对比

| 维度 | Kafka | RabbitMQ | Pulsar |
|------|-------|----------|--------|
| 吞吐量 | 极高（百万级 msg/s） | 中等（万~十万级 msg/s） | 高（接近 Kafka） |
| 延迟 | 低（ms 级，批量化影响） | 极低（微秒~亚毫秒，单条推送） | 低（ms 级） |
| 扩展性 | 水平扩展，但 rebalance 有代价 | 垂直为主，集群扩展较弱 | 弹性扩展（计算存储独立扩） |

### 功能对比

| 功能 | Kafka | RabbitMQ | Pulsar |
|------|-------|----------|--------|
| 消息确认 | Offset 提交（批量） | 单条 ack/nack/requeue | 单条 ack + 累积 ack |
| 延迟消息 | 不原生支持 | 插件支持（delayed exchange） | 原生支持 |
| 死信队列 | 不原生（需自建） | 原生支持 | 原生支持 |
| 优先级队列 | 不支持 | 原生支持 | 不支持 |
| 消息回溯 | 天然支持（offset 回退） | 不支持（消费即删） | 天然支持 |
| Schema 管理 | Schema Registry（独立组件） | 无 | 内置 Schema Registry |
| 流处理 | Kafka Streams / ksqlDB | 无 | Pulsar Functions |
| 事务 | 支持（Exactly-Once） | 支持（AMQP 事务） | 支持 |
| 地理复制 | MirrorMaker（异步） | Shovel/Federation | 原生内置（同步/异步） |

---

## 为什么会产生这些优劣？原理分析

### Kafka 吞吐高但功能简单 —— 日志模型的本质

Kafka 把一切建模为**不可变的有序日志**。这个抽象极其简单，所以能做到极致优化（顺序 I/O、zero-copy、批量化）。但简单也意味着：

- 没有"队列"语义（不能单条 ack 后删除）
- 没有路由逻辑（日志就是日志，没有 exchange 概念）
- 消费者必须自己管理 offset

### RabbitMQ 功能丰富但吞吐受限 —— Broker 中心化的代价

RabbitMQ 的 AMQP 模型是：Producer → Exchange → Binding Rule → Queue → Consumer。

**为什么功能丰富**：Exchange 有 direct/topic/fanout/headers 四种类型，binding 支持通配符路由，Queue 支持优先级、TTL、死信。这些都是 Broker 在内存中维护的状态机。

**为什么吞吐受限**：

1. 每条消息都要经过路由决策（CPU 开销）
2. 消息存在 Queue 中，消费后要标记删除（随机 I/O）
3. 推模型下 Broker 要维护每个 Consumer 的状态
4. Erlang VM 的轻量进程模型适合高并发连接，但单进程吞吐不如 JVM
5. 持久化消息需要 fsync，不像 Kafka 可以依赖 page cache 异步刷盘

**适用场景**：需要复杂路由、任务分发、RPC 模式、消息级可靠性的业务系统。

### Pulsar 试图兼得 —— 计算存储分离的设计哲学

Pulsar 的核心创新是**分层架构**：

- **Broker 层**：无状态，只负责协议处理和消息路由
- **BookKeeper 层**：负责持久化存储（分布式日志存储）

**为什么能接近 Kafka 的吞吐**：BookKeeper 也是 append-only 写入（Journal + Ledger），支持批量化和分区并行。

**为什么比 Kafka 更灵活**：

- Broker 无状态 → 扩缩容不需要数据迁移
- 支持单条 ack（通过 cursor 追踪）
- 原生多租户、地理复制
- Segment 级别的存储管理，不受 Partition 数限制

**Pulsar 的代价**：

- 架构复杂度高（Broker + BookKeeper + ZooKeeper）
- 多一跳网络延迟（Broker → BookKeeper）
- 运维成本高，组件多
- 社区和生态不如 Kafka 成熟
- 极端吞吐场景下，多一层网络跳转的开销使其略逊于 Kafka

---

## 选型总结

| 场景 | 选择 | 原因 |
|------|------|------|
| 大数据流、日志采集、事件流 | Kafka | 吞吐为王，生态成熟（Flink/Spark 集成） |
| 业务系统、任务队列、RPC | RabbitMQ | 灵活路由、消息级可靠、协议标准 |
| 云原生、多租户、弹性伸缩 | Pulsar | 计算存储分离、原生跨地域、功能全面 |
