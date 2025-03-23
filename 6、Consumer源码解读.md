# **Consumer源码解读**

**本课程的核心技术点如下：**

1、consumer初始化
2、如何选举Consumer Leader
3、Consumer Leader是如何制定分区方案

4、Consumer如何拉取数据
5、Consumer的自动偏移量提交

## Consumer初始化

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/439273e13ce94cc49f0d7e29bc356b08.png)

从KafkaConsumer的构造方法出发，我们跟踪到核心实现方法

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/609b80b7970a41be9629af99225b20d7.png)

这个方法的前面代码部分都是一些配置，我们分析源码要抓核心，我把核心代码给摘出来

### **NetworkClient**

**Consumer与Broker的核心通讯组件**

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/245206b57c7f40049332a97358f3b2f5.png)

### **ConsumerCoordinator**

**协调器，在Kafka消费中是组消费，协调器在具体进行消费之前要做很多的组织协调工作。**

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/af362192f2634f11a450acbac20d17bc.png)

### Fetcher

提取器，因为Kafka消费是拉数据的，所以这个Fetcher就是拉取数据的核心类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/98a3f9fb7bec42ed8e1fea15babb8b34.png)

而在这个核心类中，我们发现有很多很多的参数设置，这些就跟我们平时进行消费的时候配置有关系了，这里我们挑一些核心重点参数来讲一讲

#### fetch.min.bytes

每次fetch请求时，server应该返回的最小字节数。如果没有足够的数据返回，请求会等待，直到足够的数据才会返回。缺省为1个字节。多消费者下，可以设大这个值，以降低broker的工作负载。

#### fetch.max.bytes

每次fetch请求时，server应该返回的最大字节数。这个参数决定了可以成功消费到的最大数据。

比如这个参数设置的是50M，那么consumer能成功消费50M以下的数据，但是最终会卡在消费大于10M的数据上无限重试。fetch.max.bytes一定要设置到大于等于最大单条数据的大小才行。

默认是50M

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/3553f7b44045496fbd072e4e8da6a273.png)

#### fetch.wait.max.ms

如果没有足够的数据能够满足fetch.min.bytes，则此项配置是指在应答fetch请求之前，server会阻塞的最大时间。缺省为500个毫秒。和上面的fetch.min.bytes结合起来，要么满足数据的大小，要么满足时间，就看哪个条件先满足。

这里说一下参数的默认值如何去找：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/735734c77a83480a82c504fd0aaff1a5.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/fdb8f426d5c74a04a98ba8d74a32f768.png)

#### max.partition.fetch.bytes

指定了服务器从每个分区里返回给消费者的最大字节数，默认1MB。

假设一个主题有20个分区和5个消费者，那么每个消费者至少要有4MB的可用内存来接收记录，而且一旦有消费者崩溃，这个内存还需更大。注意，这个参数要比服务器的message.max.bytes更大，否则消费者可能无法读取消息。

*备注：1、Kafka入门笔记*

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/149b38d2fdb744afb7342ab18a919b5c.png)

#### max.poll.records

控制每次poll方法返回的最大记录数量。

默认是500

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/14d8ab011f9e46f7b2ff35e3d0fa0e9a.png)

## 如何选举Consumer Leader

回顾之前的内容

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/d4d5010160d64359835601bd11ac1721.png)

那么如何完成以上的逻辑的，我们跟踪代码：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/d860e57c56084f7089c68b1b96b7f086.png)

### 1、消费者协调器与组协调器的通讯

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/dd7f9ef84d0b4b40803df2666799ef1c.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/1245275c5c06426587f22ec9b5b173cc.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/9fdccd7c773a432b95b6b4330e84ef81.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/18c5f2bfe54a411b898836c3e0b939bc.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/cc02631ecf8645e1b7166b708ed61687.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/b7d3f1bebe81418aa3f25ec2fbb82919.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/f2b7b514701749bcb100917863231961.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/6454beef06114ecd80bc22fb294eff82.png)

对Broker的响应进行处理

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/399824c22ffd4e2295bd2c338be5b146.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/c48b224ebd6e44908af948c6befb67fb.png)

### 1、消费者协调器发起入组请求

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/d266f57732d94d2288f76e2d16d6cfce.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/42c86951d77d4a86a87726567983088a.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/37153d3e469a4190a7b53538fb6c6b75.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/5b3b8fa273584a81bdb66c66ba3d7e54.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/f4d91abfa5c74cf2ac696a9eb35c7552.png)

## Consumer Leader如何制定分区方案

回顾之前的内容

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/30efcaccb4314670b776e7b7545c0d8f.png)

### 消费者分区策略

消费者参数

**partition.assignment.strategy**

分区分配给消费者的策略。默认为Range。允许自定义策略。

#### **Range**

把主题的连续分区分配给消费者。（如果分区数量无法被消费者整除、第一个消费者会分到更多分区）

#### **RoundRobin**

把主题的分区循环分配给消费者。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/a5f8f124ef0e48529a46759bf1c6eb72.png)

#### StickyAssignor

初始分区和RoundRobin是一样

粘性分区：每一次分配变更相对上一次分配做最少的变动.

目标：

1、**分区的分配尽量的均衡**

2、**每一次重分配的结果尽量与上一次分配结果保持一致**

当这两个目标发生冲突时，优先保证第一个目标

比如有3个消费者（C0、C1、C2）、4个topic(T0、T1、T2、T34)，每个topic有2个分区（P1、P2）

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/0f5688b418cf4100a97af87c46c9b0c5.png)

**C0:**  **T0P0、T1P1、T3P0**

**C1: T0P1、T2P0、T3P1**

**C2: T1P0、T2P1**

如果C1下线 、如果按照RoundRobin

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/4c352e4724cf4326b51b3b4e3bfd8ec7.png)

**C0:**  **T0P0、T1P0、T2P0、T3P0**

**C2:  T0P1、T1P1、T2P1、T3P1**

对比之前

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/488fb784b67645e6b17d54e6a42aa812.png)

如果C1下线 、如果按照StickyAssignor

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/92cd0754ee2d4785aff805e0ccf2ca7e.png)

**C0:**  **T0P0、T1P1、T2P0、T3P0**

**C2:  T0P1、T1P0、T2P1、T3P1**

对比之前

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/488fb784b67645e6b17d54e6a42aa812.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/71a63c396528426f81c85c27e252066b.png)

#### 自定义策略

extends 类AbstractPartitionAssignor，然后在消费者端增加参数：

properties.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,类.class.getName());

即可。

### 消费者分区策略源码分析

接着上个章节的代码。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/2df5eb1f747e46718f854ba4a537c24b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/76f057d2b969462d8fe35120a447eeaf.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/da22757d6471436cb349cbd1f442c79d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/9c8fab4167394382acfd3096851cf13d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/88165821148b4dda8eabb85a673769d6.png)

## Consumer拉取数据

这里就是拉取数据，核心Fetch类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/d547217dfb1e47d4aed48b659eae4003.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/6741976e4860466289e0ca387cc6f283.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/74b908d7a6fd493f9651b3bb51537642.png)

## 自动提交偏移量

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/a1c6353daa0e4184bf50f402386ba38d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/68a4e535116047b8a4f75d07d8b78579.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/1345cc1f1e724f6583d8ea9a51b28fab.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/d54727b2226945cc89d27c84544a815e.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/50b6a6657e1e42b48a787c1b92526ece.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/d96d48ea405240a2be36bb6ac5f37060.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/ed503ca6d42d4b58880613d1a5893a8d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/7d12a36e5c4d4d029ce461da4212e12d.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/69b0fb87371e46c08d27e1dab133c902.png)

当然，自动提交auto.commit.interval.ms

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/38e257be210e44158cf8c340443df2b0.png)

默认5s

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1675692369064/f701faa4444a4e9ab97ba3605acae83d.png)

从源码上也可以看出

maybeAutoCommitOffsetsAsync 最后这个就是poll的时候会自动提交，而且没到auto.commit.interval.ms间隔时间也不会提交，如果没到下次自动提交的时间也不会提交。

这个autoCommitIntervalMs就是auto.commit.interval.ms设置的
