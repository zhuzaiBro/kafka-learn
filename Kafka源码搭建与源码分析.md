# Kafka源码分析环境搭建

使用截止目前为止Kafka的最新版本3.3.1版本的源码进行环境搭建

## Kafka源码下载

从kafka官网下载kafka-3.3.1版本的源码

[http://kafka.apache.org/downloads]()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/be2d12483ae646cea05f6dbebd36e1a2.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/b59d91d72ef641baa6c24f66570543c2.png)

解压(要放到英文目录，不然会报一些奇怪的错误)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/06fc2ab527e449d0b7835fe8bd21366a.png)

## Scala安装

因为在源码中配置的scala版本是2.13.8

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/509c3772e7174fc985ea9140581787d2.png)

以我们在win上安装Scala 2.13，上官网找到2.13.8版本对应的下载地址

https://www.scala-lang.org/download/2.13.8.html

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/2654ccbb02ce4afbb2b47c715eef6377.png)

然后就可以下载win上的安装包，scala.msi，下载好之后傻瓜式安装就可以了

在cmd中输入scala如果出现如下提示则安装成功

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/b2e12752ca90478c9b449d2d5c07fe5b.png)

## idea版本选择与Scala插件

因为最终要使用ide来导入，同时最新版本的kafka源码构建必须是gradle是高版本，所以IDE也必须是高版本才支持高版本的 gradle，所以这里推荐使用IntelliJ IDEA 2020.3.4 版本。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/ddb0e3aaf1e744a09f8d555ecb580830.png)

从这里可以看出gradle是6.7，版本已经够支持了。

安装Scala插件。

进入IntelliJ IDEA的这个界面

左侧有一个“Plugins”，搜索scala相关的插件，此时一开始是找不到的，然后点击“search in repositories”，找到一个“Scala”插件，他的类别是“Language”，在线装即可，他会下载之后安装

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/81585d23fcee48ec8cf241965fbf2755.png)

## Gradle的安装

接着需要安装Gradle，现在国外很多知名的开源项目，Kafka是用Gradle来进行项目的构建了，所以需要安装。

我的IDE的版本支持的是gradle是

Gradle来完成Kafka源码的构建，使用gradle 7.6，从官网下载，解压缩即可，然后配置GRADLE_HOME和PATH

[https://gradle.org/release]()s/

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/fc3279e965c44411b8b9550f0a70fd72.png)

配置环境变量，新建 GRADLE_HOME 环境变量指向你的 Gradle 解压路径

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/086570d27d154ca99f0ba929d29c31d9.png)

然后将 %GRADLE_HOME%\bin 添加到 Path 环境变量中，然后点击确定

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/aa5c926f7265440b9d7a8a1f42d71277.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/fc20bc65b3f5491b81a8a4486141f03b.png)

验证gradle是否安装成功，打开cmd命令行输入 gradle -v

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/790d59dd79d54f1c8ed467d7958e7375.png)

最后 验证三个基础的依赖都正确安装了

```
java -version
scala -version
gradle -version
```

## 使用Gradle来构建Kafka源码

通过win命令行进入kafka-3.3.1-src目录下，然后执行“gradle idea”为源码导入idea进行构建

这个过程会下载大量的依赖jar包，建议配置 gradle 版本库为阿里源（不然会很慢，同时还可能抛出无法下载错误），同时也要修改对应的配置文件。

编辑Kafka源码目录下的build.gradle文件

### 1、修改阿里源

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/f6be8d33c0fe48f2858da19fe0acb596.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/d7f432eae0354e38a3be36a0e0bbfd01.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/23a6cd7e7f3b4a5e9575db107dc19bfb.png)

```
maven { url 'https://maven.aliyun.com/repository/public' }
```

### 2、修改配置（防止构建报错）

```
ScalaCompileOptions.metaClass.daemonServer=true
ScalaCompileOptions.metaClass.fork=true
ScalaCompileOptions.metaClass.useAnt=false
ScalaCompileOptions.metaClass.useCompileDaemon=false
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/d82f80f1034e43af99a7ad2d2b09f679.png)

### 3、构建成功

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/1ea37d3948604b24b96be8e1ea437c93.png)

安装完了在plugins里面就可以找到scala插件了，然后点击“ok”就会提示你重启intellij idea来激活安装好的插件，然后点击里面的那个Import Project按钮即可，选择你的kafka源码所在的目录，选择你构建项目的方式是“gradle”，导入的过程也需要不少的时间，需要耐心等待，会显示的是如下的图：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/a10a7f5c9ac64526a7ac3787ebef1e22.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/0d5dce975a634dc082cfaa0ec6b50f8e.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/24c380bfeed045c68658748932ab05ef.png)

## 在IDEA中启动Kafka

我们肯定是要看到log4j输出的日志的，所以必须把config目录下的log4j.properties给放到src/main/scala目录下去，这样才能看到服务端运行起来的程序打印出来的日志![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/434c27efc35a4ca4b4d495f31b88d750.png)

另外需要修改 config目录下的server.properties

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/727ed118cb754fbe861ea4fe18e7e3d1.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/ee79978d6ef747e88f85a8bff8a8b3ad.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/c91fe2f15e3c431b9b59964aaca139d5.png)

之前IDE的Scala版本偏老，运行时会出现这个错误

那么手动更新scala的版本

[https://plugins.jetbrains.com/plugin/1347-scala/versions/stable]()

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/bd74ab533b9f4b429af605914895bd3f.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/67a995fa1ee8422e919a5e1c8db068ba.png)

缺少包slf4j-nop，导入该包。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/2354e62ef6c74c95ae6a522dd04de6de.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/3cf97cb937344fb3844a6462974b0b0f.png)

```
slf4jnop: "org.slf4j:slf4j-nop:$versions.slf4j",
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/158147ad456f4b09ba37bcac309697e8.png)

```
 compileOnly libs.slf4jnop
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/5983/1673318216074/c4344c715e514aa3a36b57c8d1824d2c.png)
