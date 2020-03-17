## 原创|面试官：线上服务器CPU占用率高如何排查定位问题？                       

​                                 

文章转载自公众号 [!Java之道](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6b3m0KWUfDb4zE5wia9fiaCUbvVRP5maNGzCKtfI3BIyGg/0)                                       

国外开发者平台 HankerRank 发布的 2018 年开发者技能调查报告中有一项关于"雇主最看重哪些核心能力"的调查，结果显示如下：



![img](https://mmbiz.qpic.cn/mmbiz_jpg/1Fk759B8md7fPl60GhTkicVV5IMdW4HwZ5TwTFPQqHHIiaMcXIVqZxyzMTJLuDNhy32jawr4CPp7QzreZJNDLxIQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

排名前几的比较受重视的能力分别为：解决问题、编程语言熟练程度、Debug、系统设计和性能优化。

解决问题的能力以超高比例排名第一，这也是为什么很多面试过程中，面试官都喜欢问如下问题：

> 1、你这个项目遇到的最大挑战是什么？如何解决的？ 
>
> 2、如果线上发生了报警你回如何排查呢？ 
>
> 3、你有解决过什么线上问题吗？ 
>
> 4、能列举几个你知道的排查Linux服务器线上问题的命令吗？

这些，都是比较常见的问题，还有一些比较具体的问题也是建议很多开发者都需要掌握的，如：

> 1、线上服务器Load飙高如何排查？ 
>
> 2、线上服务器CPU占用率高如何排查？ 
>
> 3、线上服务器频繁发生Full GC如何排查？ 
>
> 4、线上服务器发生死锁如何排查？

这些问题的回答，一方面考察了面试者是否具有很强的实战经验，另外一方面也能体现出其解决问题的能力。

毋庸置疑，作为开发人员来说，定位并解决问题的能力是至关重要的。因为一旦线上发生了问题，如CPU占用率高，如果不及时解决，很容易导致网站响应慢、服务器宕机等问题。

那么，书归正传，本文我们就来简单介绍一下，如果线上服务器发生CPU占用率过高的问题时，应该如何排查并定位问题。

***1***



**问题发现**

本文整理自一个真实的案例，是楼主负责的业务，在一次大促之前的压测时发现了这个问题。

在每次大促之前，我们的测试人员都会对网站进行压力测试，这个时候会查看服务的cpu、内存、load、rt、qps等指标。

在一次压测过程中，测试人员发现我们的某一个接口，在qps上升到500以后，CPU使用率急剧升高。

> CPU利用率，又称CPU使用率。顾名思义，CPU利用率是来描述CPU的使用情况的，表明了一段时间内CPU被占用的情况。使用率越高，说明你的机器在这个时间上运行了很多程序，反之较少。

**2**



**问题定位**

遇到这种问题，首先是登录到服务器，看一下具体情况。

**定位进程**

登录服务器，执行top命令，查看CPU占用情况：

```
$top
   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
  1893 admin     20   0 7127m 2.6g  38m S 181.7 32.6  10:20.26 java
```

> top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，类似于Windows的任务管理器。

通过以上命令，我们可以看到，进程ID为1893的Java进程的CPU占用率达到了181%，基本可以定位到是我们的Java应用导致整个服务器的CPU占用率飙升。

**定位线程**

我们知道，Java是单进程多线程的，那么，我们接下来看看PID=1893的这个Java进程中的各个线程的CPU使用情况，同样是用top命令：

```
$top -Hp 1893
   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
  4519 admin     20   0 7127m 2.6g  38m R 18.6 32.6   0:40.11 java
```

通过top -Hp 1893命令，我们可以发现，当前1893这个进程中，ID为4519的线程占用CPU最高。

**定位代码**

通过top命令，我们目前已经定位到导致CPU使用率较高的具体线程， 那么我么接下来就定位下到底是哪一行代码存在问题。

首先，我们需要把4519这个线程转成16进制：

```
$printf %x 4519
11a7
```

接下来，通过jstack命令，查看栈信息：

```
$sudo -u admin  jstack 1893 |grep -A 200 11a7
"HSFBizProcessor-DEFAULT-8-thread-5" #500 daemon prio=10 os_prio=0 tid=0x00007f632314a800 nid=0x11a2 runnable [0x000000005442a000]
   java.lang.Thread.State: RUNNABLE
  at sun.misc.URLClassPath$Loader.findResource(URLClassPath.java:684)
  at sun.misc.URLClassPath.findResource(URLClassPath.java:188)
  at java.net.URLClassLoader$2.run(URLClassLoader.java:569)
  at java.net.URLClassLoader$2.run(URLClassLoader.java:567)
  at java.security.AccessController.doPrivileged(Native Method)
  at java.net.URLClassLoader.findResource(URLClassLoader.java:566)
  at java.lang.ClassLoader.getResource(ClassLoader.java:1093)
  at java.net.URLClassLoader.getResourceAsStream(URLClassLoader.java:232)
  at org.hibernate.validator.internal.xml.ValidationXmlParser.getInputStreamForPath(ValidationXmlParser.java:248)
  at org.hibernate.validator.internal.xml.ValidationXmlParser.getValidationConfig(ValidationXmlParser.java:191)
  at org.hibernate.validator.internal.xml.ValidationXmlParser.parseValidationXml(ValidationXmlParser.java:65)
  at org.hibernate.validator.internal.engine.ConfigurationImpl.parseValidationXml(ConfigurationImpl.java:287)
  at org.hibernate.validator.internal.engine.ConfigurationImpl.buildValidatorFactory(ConfigurationImpl.java:174)
  at javax.validation.Validation.buildDefaultValidatorFactory(Validation.java:111)
  at com.test.common.util.BeanValidator.validate(BeanValidator.java:30)
```

通过以上代码，我们可以清楚的看到，BeanValidator.java的第30行是有可能存在问题的。

***3***



**问题解决**

接下来就是通过查看代码来解决问题了，我们发现，我们自定义了一个BeanValidator，封装了Hibernate的Validator，然后在validate方法中，通过Validation.buildDefaultValidatorFactory().getValidator()初始化一个Validator实例，通过分析发现这个实例化的过程比较耗时。

我们重构了一下代码，把Validator实例的初始化提到方法外，在类初始化的时候创建一次就解决了问题。

**4**



**总结**

以上，展示了一次比较完成的线上问题定位过程。主要用到的命令有:top 、printf 和 jstack

另外，线上问题排查还可以使用Alibaba开源的工具Arthas进行排查，以上问题，可以使用一下命令定位：

```
thread -n 3 //查看cpu占比前三的线程
```

以上，本文介绍了如何排查线上服务器CPU使用率过高的问题，如果大家感兴趣，后面可以再介绍一些关于LOAD飙高、频繁GC等问题的排查手段。