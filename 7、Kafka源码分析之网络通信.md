# 1、生产者网络设计

## 架构设计图

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/4b6bc02f69d841c688b0385e2f0fb112.jpg)

# 2、生产者消息缓存机制

### 1、RecordAccumulator

将消息缓存到RecordAccumulator收集器中, 最后判断是否要发送。这个加入消息收集器，首先得从 Deque<RecordBatch> 里找到自己的目标分区，如果没有就新建一个批量消息 Deque 加进入

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/e82e4b9b468c48b5892ed605f82ea93b.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/dee80d3529d648079e67b86da480eb9a.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/d2c48ec28b8c490dbfe29200a7edbd49.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/81fc2531358c4a8fa58e2847711cb2be.png)

### 2、消息发送时机

如果达到发送阈值（**批次发送的条件为:缓冲区数据大小达到 batch.size 或者 linger.ms 达到上限，哪个先达到就算哪个**），唤醒Sender线程，

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/20cd7ef1988c4082b27a655d03b6a42e.png)

NetWorkClient 将 batch record 转换成 request client 的发送消息体, 并将待发送的数据按 【Broker Id <=> List】的数据进行归类

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/295fdbad2f2d49d5841478989b66fc40.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/f67309c1f8594d3c960e26eff8dda319.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/9cd086d2acec4f0f81b263430a7f9366.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/bca8ce31efa044adaf7b7c2b95463679.png)

与服务端不同的 Broker 建立网络连接，将对应 Broker 待发送的消息 List 发送出去。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/9363d1add5634550b3a106d38d00fb36.png)

9)、

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/904ea02aa0c24cb6a2f752d1b72063c4.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/735d2da9d4494f189ce55164bb985ac0.png)

经过几轮跳转

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/b989e3bbfbfe46e88c5c13f566e2e25d.png)

# 3、Kafka通讯组件解析

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1676558275075/cb489cf938f7455ba3625d186e507867.png)
