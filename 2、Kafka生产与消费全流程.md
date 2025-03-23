# Kafka生产与消费全流程

Kafka是一款消息中间件，消息中间件本质就是收消息与发消息，所以这节课我们会从一条消息开始生产出发，去了解生产端的运行流程，然后简单的了解一下broker的存储流程，最后这条消息是如何被消费者消费掉的。其中最核心的有以下内容。

1、Kafka客户端是如何去设计一个非常优秀的生产级的保证高吞吐的一个缓冲机制

2、消费端的原理：每个消费组的群主如何选择，消费组的群组协调器如何选择，分区分配的方法，分布式消费的实现机制，拉取消息的原理，offset提交的原理。

# Kafka一条消息发送和消费的流程(非集群)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/35d3bd71bad045dd92d5a117c71e1368.png)

## 简单入门

我们这里使用Kafka内置的客户端API开发kafka应用程序。因为我们是Java程序员，所以这里我们使用Maven，使用较新的版本

```
  <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>3.3.1</version>
  </dependency>
```

### 生产者

先创建一个主题，推荐在消息发送时创建对应的主题。当然就算没有创建主题，Kafka也能自动创建。

**auto.create.topics.enable**

是否允许自动创建主题。如果设为true，那么produce（生产者往主题写消息），consume（消费者从主题读消息）或者fetch metadata（任意客户端向主题发送元数据请求时）一个不存在的主题时，就会自动创建。缺省为true。

**num.partitions**

每个新建主题的分区个数（分区个数只能增加，不能减少 ）。这个参数默认值是1（最新版本）

#### 必选属性

创建生产者对象时有三个属性必须指定。

##### bootstrap.servers

该属性指定broker的地址清单，地址的格式为host:port。

清单里不需要包含所有的broker地址，生产者会从给定的broker里查询其他broker的信息。不过最少提供2个broker的信息(用逗号分隔，比如:127.0.0.1:9092,192.168.0.13:9092)，一旦其中一个宕机，生产者仍能连接到集群上。

##### key.serializer

生产者接口允许使用参数化类型，可以把Java对象作为键和值传broker，但是broker希望收到的消息的键和值都是字节数组，所以，必须提供将对象序列化成字节数组的序列化器。

key.serializer必须设置为实现org.apache.kafka.common.serialization.Serializer的接口类

Kafka的客户端默认提供了ByteArraySerializer,IntegerSerializer,StringSerializer，也可以实现自定义的序列化器。

##### value.serializer

同 key.serializer。

#### 三种发送方式

我们通过生成者的send方法进行发送。send方法会返回一个包含RecordMetadata的Future对象。RecordMetadata里包含了目标主题，分区信息和消息的偏移量。

##### 发送并忘记

忽略send方法的返回值，不做任何处理。大多数情况下，消息会正常到达，而且生产者会自动重试，但有时会丢失消息。

```
package com.msb.producer;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

/**
 * 类说明：kafak生产者
 */
public class HelloKafkaProducer {

    public static void main(String[] args) {
        // 设置属性
        Properties properties = new Properties();
        // 指定连接的kafka服务器的地址
        properties.put("bootstrap.servers","127.0.0.1:9092");
        // 设置String的序列化
        properties.put("key.serializer", StringSerializer.class);
        properties.put("value.serializer", StringSerializer.class);

        // 构建kafka生产者对象
        KafkaProducer<String,String> producer  = new KafkaProducer<String, String>(properties);
        try {
            ProducerRecord<String,String> record;
            try {
                // 构建消息
                record = new ProducerRecord<String,String>("msb", "teacher","lijin");
                // 发送消息
                producer.send(record);
                System.out.println("message is sent.");
            } catch (Exception e) {
                e.printStackTrace();
            }
        } finally {
            // 释放连接
            producer.close();
        }
    }


}

```

##### 同步发送

获得send方法返回的Future对象，在合适的时候调用Future的get方法。参见代码。

```
package com.msb.producer;

import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;
import java.util.concurrent.Future;

/**
 * 类说明：发送消息--同步模式
 */
public class SynProducer {

    public static void main(String[] args) {
        // 设置属性
        Properties properties = new Properties();
        // 指定连接的kafka服务器的地址
        properties.put("bootstrap.servers","127.0.0.1:9092");
        // 设置String的序列化
        properties.put("key.serializer", StringSerializer.class);
        properties.put("value.serializer", StringSerializer.class);

        // 构建kafka生产者对象
        KafkaProducer<String,String> producer  = new KafkaProducer<String, String>(properties);
        try {
            ProducerRecord<String,String> record;
            try {
                // 构建消息
                record = new ProducerRecord<String,String>("msb", "teacher2333","lijin");
                // 发送消息
                Future<RecordMetadata> future =producer.send(record);
                RecordMetadata recordMetadata = future.get();
                if(null!=recordMetadata){
                    System.out.println("offset:"+recordMetadata.offset()+","
                            +"partition:"+recordMetadata.partition());
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        } finally {
            // 释放连接
            producer.close();
        }
    }




}

```

##### 异步发送

实现接口org.apache.kafka.clients.producer.Callback，然后将实现类的实例作为参数传递给send方法。

```
package com.msb.producer;

import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;
import java.util.concurrent.Future;

/**
 * 类说明：发送消息--异步模式
 */
public class AsynProducer {

    public static void main(String[] args) {
        // 设置属性
        Properties properties = new Properties();
        // 指定连接的kafka服务器的地址
        properties.put("bootstrap.servers","127.0.0.1:9092");
        // 设置String的序列化
        properties.put("key.serializer", StringSerializer.class);
        properties.put("value.serializer", StringSerializer.class);

        // 构建kafka生产者对象
        KafkaProducer<String,String> producer  = new KafkaProducer<String, String>(properties);

        try {
            ProducerRecord<String,String> record;
            try {
                // 构建消息
                record = new ProducerRecord<String,String>("msb", "teacher","lijin");
                // 发送消息
                producer.send(record, new Callback() {
                    @Override
                    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                        if (e == null){
                            // 没有异常，输出信息到控制台
                            System.out.println("offset:"+recordMetadata.offset()+"," +"partition:"+recordMetadata.partition());
                        } else {
                            // 出现异常打印
                            e.printStackTrace();
                        }
                    }
                });
            } catch (Exception e) {
                e.printStackTrace();
            }
        } finally {
            // 释放连接
            producer.close();
        }
    }




}

```

### 消费者

消费者的含义，同一般消息中间件中消费者的概念。在高并发的情况下，生产者产生消息的速度是远大于消费者消费的速度，单个消费者很可能会负担不起，此时有必要对消费者进行横向伸缩，于是我们可以使用多个消费者从同一个主题读取消息，对消息进行分流。

#### 必选属性

创建消费者对象时一般有四个属性必须指定。

bootstrap.servers、value.Deserializer key.Deserializer 含义同生产者

#### 可选属性

group.id  并非完全必需，它指定了消费者属于哪一个群组，但是创建不属于任何一个群组的消费者并没有问题。不过绝大部分情况我们都会使用群组消费。

#### 消费者群组

Kafka里消费者从属于消费者群组，一个群组里的消费者订阅的都是同一个主题，每个消费者接收主题一部分分区的消息。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/5693e8540e2b4b31b73c8b6ac6ee7038.png)

如上图，主题T有4个分区，群组中只有一个消费者，则该消费者将收到主题T1全部4个分区的消息。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/dc2400aa869a414abe8909c1e59190e5.png)

如上图，在群组中增加一个消费者2，那么每个消费者将分别从两个分区接收消息，上图中就表现为消费者1接收分区1和分区3的消息，消费者2接收分区2和分区4的消息。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image006.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/1a2816e2b9b94ffa88598d5e7e021841.png)

如上图，在群组中有4个消费者，那么每个消费者将分别从1个分区接收消息。

![](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image008.jpg)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/6992d6eacdbb4e31b1ab2facaf09dd0f.png)

但是，当我们增加更多的消费者，超过了主题的分区数量，就会有一部分的消费者被闲置，不会接收到任何消息。

往消费者群组里增加消费者是进行横向伸缩能力的主要方式。所以我们有必要为主题设定合适规模的分区，在负载均衡的时候可以加入更多的消费者。但是要记住，一个群组里消费者数量超过了主题的分区数量，多出来的消费者是没有用处的。

## 序列化

创建生产者对象必须指定序列化器，默认的序列化器并不能满足我们所有的场景。我们完全可以自定义序列化器。只要实现org.apache.kafka.common.serialization.Serializer接口即可。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/8e473e701c814e6e9b755f6fe44499b1.png)

### 自定义序列化

代码见：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/8eb6dfa47624416fb16ed4b02739ee5d.png)

代码中使用到了自定义序列化。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/0a38704c657a48d38c2f4e55102547d6.png)

id的长度4个字节，字符串的长度描述4个字节， 字符串本身的长度nameSize个字节

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/a6d7fd2e37924e61bb63ccad4dd0c657.png)

自定义序列化容易导致程序的脆弱性。举例，在我们上面的实现里，我们有多种类型的消费者，每个消费者对实体字段都有各自的需求，比如，有的将字段变更为long型，有的会增加字段，这样会出现新旧消息的兼容性问题。特别是在系统升级的时候，经常会出现一部分系统升级，其余系统被迫跟着升级的情况。

解决这个问题，可以考虑使用自带格式描述以及语言无关的序列化框架。比如Protobuf，Kafka官方推荐的Apache Avro

## 分区

因为在Kafka中一个topic可以有多个partition，所以当一个生产发送消息，这条消息应该发送到哪个partition，这个过程就叫做分区。

当然，我们在新建消息的时候，我们可以指定partition，只要指定partition，那么分区器的策略则失效。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/65ed2b6059b746e2b43532ca18c0369a.png)

### 系统分区器

在我们的代码中可以看到，生产者参数中是可以选择分区器的。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/c088dd4584bb4285b282a306abef438b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/af5e0e57c3ca412aad7e668cf2142f8e.png)

#### DefaultPartitioner 默认分区策略

全路径类名：org.apache.kafka.clients.producer.internals.DefaultPartitioner

* 如果消息中指定了分区，则使用它
* 如果未指定分区但存在key，则根据序列化key使用murmur2哈希算法对分区数取模。
* 如果不存在分区或key，则会使用**粘性分区策略**

采用默认分区的方式，键的主要用途有两个：

一，用来决定消息被写往主题的哪个分区，拥有相同键的消息将被写往同一个分区。

二，还可以作为消息的附加消息。

#### RoundRobinPartitioner 分区策略

全路径类名：org.apache.kafka.clients.producer.internals.RoundRobinPartitioner

* 如果消息中指定了分区，则使用它
* 将消息平均的分配到每个分区中。

即key为null，那么这个时候一般也会采用RoundRobinPartitioner

#### UniformStickyPartitioner 纯粹的粘性分区策略

全路径类名：org.apache.kafka.clients.producer.internals.UniformStickyPartitioner

他跟**DefaultPartitioner** 分区策略的唯一区别就是。

**DefaultPartitionerd 如果有key的话,那么它是按照key来决定分区的,这个时候并不会使用粘性分区**
**UniformStickyPartitioner 是不管你有没有key, 统一都用粘性分区来分配**

#### 另外关于粘性分区策略

从客户端最新的版本上来看（3.3.1），有两个序列化器已经进入 弃用阶段。

这个客户端在3.1.0都还不是这样。关于粘性分区策略

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/c9e0c8d6613b4b73b87e320823f76391.png)

如果感兴趣可以看下这篇文章

[https://bbs.huaweicloud.com/blogs/348729?utm_source=oschina&amp;utm_medium=bbs-ex&amp;utm_campaign=other&amp;utm_content=content]()

### 自定义分区器

我们完全可以去实现Partitioner接口，去实现有一个自定义的分区器

```
package com.msb.selfpartition;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.utils.Utils;

import java.util.List;
import java.util.Map;

/**
 * 类说明：自定义分区器，以value值进行分区
 */
public class SelfPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitionInfos = cluster.partitionsForTopic(topic);
        int num = partitionInfos.size();
        int parId = Utils.toPositive(Utils.murmur2(valueBytes)) % num;//来自DefaultPartitioner的处理
        return parId;
    }

    public void close() {
        //do nothing
    }

    public void configure(Map<String, ?> configs) {
        //do nothing
    }

}

```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/f07c49100ca44560975e7f94f00713c4.png)

## 生产缓冲机制

客户端发送消息给kafka服务器的时候、消息会先写入一个内存缓冲中，然后直到多条消息组成了一个Batch，才会一次网络通信把Batch发送过去。主要有以下参数：

**buffer.memory**

设置生产者内存缓冲区的大小，生产者用它缓冲要发送到服务器的消息。如果数据产生速度大于向broker发送的速度，导致生产者空间不足，producer会阻塞或者抛出异常。缺省33554432 (32M)

buffer.memory: 所有缓存消息的总体大小超过这个数值后，就会触发把消息发往服务器。此时会忽略batch.size和linger.ms的限制。
buffer.memory的默认数值是32 MB，对于单个 Producer 来说，可以保证足够的性能。 需要注意的是，如果您在同一个JVM中启动多个 Producer，那么每个 Producer 都有可能占用 32 MB缓存空间，此时便有可能触发 OOM。

**batch.size**

当多个消息被发送同一个分区时，生产者会把它们放在同一个批次里。该参数指定了一个批次可以使用的内存大小，按照字节数计算。当批次内存被填满后，批次里的所有消息会被发送出去。但是生产者不一定都会等到批次被填满才发送，半满甚至只包含一个消息的批次也有可能被发送（linger.ms控制）。缺省16384(16k) ，如果一条消息超过了批次的大小，会写不进去。

**linger.ms**

指定了生产者在发送批次前等待更多消息加入批次的时间。它和batch.size以先到者为先。也就是说，一旦我们获得消息的数量够batch.size的数量了，他将会立即发送而不顾这项设置，然而如果我们获得消息字节数比batch.size设置要小的多，我们需要“linger”特定的时间以获取更多的消息。这个设置默认为0，即没有延迟。设定linger.ms=5，例如，将会减少请求数目，但是同时会增加5ms的延迟，但也会提升消息的吞吐量。

### 为何要设计缓冲机制

1、减少IO的开销（单个 ->批次）但是这种情况基本上也只是linger.ms配置>0的情况下才会有，因为默认inger.ms=0的，所以基本上有消息进来了就发送了，跟单条发送是差不多！！

2、减少Kafka中Java客户端的GC。

比如缓冲池大小是32MB。然后把32MB划分为N多个内存块，比如说一个内存块是16KB（batch.size），这样的话这个缓冲池里就会有很多的内存块。

你需要创建一个新的Batch，就从缓冲池里取一个16KB的内存块就可以了，然后这个Batch就不断的写入消息

下次别人再要构建一个Batch的时候，再次使用缓冲池里的内存块就好了。这样就可以利用有限的内存，对他不停的反复重复的利用。因为如果你的Batch使用完了以后是把内存块还回到缓冲池中去，那么就不涉及到垃圾回收了。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1670654953054/5638282571c44238b3b362e3985468d5.png)

## 消费者偏移量提交

一般情况下，我们调用poll方法的时候，broker返回的是生产者写入Kafka同时kafka的消费者提交偏移量，这样可以确保消费者消息消费不丢失也不重复，所以一般情况下Kafka提供的原生的消费者是安全的，但是事情会这么完美吗？

### 自动提交

最简单的提交方式是让消费者自动提交偏移量。 如果enable.auto.commit被设为 true，消费者会自动把从poll()方法接收到的**最大**偏移量提交上去。提交时间间隔由auto.commit.interval.ms控制，默认值是5s。

自动提交是在轮询里进行的，消费者每次在进行轮询时会检査是否该提交偏移量了，如果是，那么就会提交从上一次轮询返回的偏移量。

不过,在使用这种简便的方式之前,需要知道它将会带来怎样的结果。

假设我们仍然使用默认的5s提交时间间隔, 在最近一次提交之后的3s发生了再均衡，再均衡之后,消费者从最后一次提交的偏移量位置开始读取消息。这个时候偏移量已经落后了3s，所以在这3s内到达的消息会被重复处理。可以通过修改提交时间间隔来更频繁地提交偏移量, 减小可能出现重复消息的时间窗, 不过这种情况是无法完全避免的。

在使用自动提交时,每次调用轮询方法都会把上一次调用返回的最大偏移量提交上去,它并不知道具体哪些消息已经被处理了,所以在再次调用之前最好确保所有当前调用返回的消息都已经处理完毕(enable.auto.comnit被设为 true时，在调用 close()方法之前也会进行自动提交)。一般情况下不会有什么问题,不过在处理异常或提前退出轮询时要格外小心。

#### 消费者的配置参数

**auto.offset.reset**

earliest
当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
latest
当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据

只要group.Id不变，不管auto.offset.reset 设置成什么值，都从上一次的消费结束的地方开始消费。
