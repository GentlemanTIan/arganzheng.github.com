---
title: BTrace实战
layout: post
tags: [java, btrace]
catalog: true
---

### 什么是BTrace

官方文档上有个非常简洁精辟的介绍。

>
BTrace is a safe, dynamic tracing tool for the Java platform. BTrace can be used to dynamically trace a running Java program (similar to DTrace for OpenSolaris applications and OS). BTrace dynamically instruments the classes of the target application to inject tracing code ("bytecode tracing"). Tracing code is expressed in Java programming language. There is also integration with DTrace for the OpenSolaris platform.

关键几点：

1. 动态增强(instrument)运行中的Java程序。普通的AOP之类的都是预先写好的脚本和配置。但是BTrace不同，他可以在运行期间植入切面代码。这也是他最大的作用。这点非常类似于JMockit。
2. 植入代码是字节码级别的跟踪(bytecode tracing，正是BTrace的名称来源)

BTrace的最大好处，是可以通过自己编写的脚本，获取应用的一切调用信息，而不需要通过修改代码（主要是打日志），不断重启应用以便修改生效。

同时，特别严格的约束，保证自己的消耗特别小，只要定义脚本时不作大死，直接在生产环境打开也没太大影响。

### 应用场景

正如其名，BytecodeTracing，可以对已经运行的java代码进行动态植入。获取运行时的一些信息。注意，只能Tracing，不能修改目标代码。只能使用`BTraceUtils`类中的方法。

下面是一些典型的应用场景：

1. 服务慢，能找出慢在哪一步，哪个函数里么？
2. 谁调用了 `System.gc()` ，调用栈如何？
3. 谁构造了一个超大的 ArrayList?
4. 什么样的入参或对象属性，导致抛出了这个异常？或进入了这个处理分支？

### 编写BTrace脚本

假设我们要拦截的方法如下：

```java
package me.arganzheng.study.btrace.target.controller;

@Controller
public class ApiDispatcher {

	public void dispatch(@PathVariable
	    	int version, @PathVariable
	    	String moduleName, @PathVariable
	    	String apiName, HttpServletRequest request, HttpServletResponse response) {
	    // ...
	    Map<String, Object> context = buildContext(request);
	    // ...
	}

	private Map<String, Object> buildContext(HttpServletRequest request) {
	        Map<String, Object> context = new HashMap<String, Object>();

        String clientIP = WebUtils.getRealIp(request);

        // 虚拟需要ClientIP做权重分配
        context.put("clientIP", clientIP);
        
        logger.info("Context.clientIP=" + clientIP);

        return context;
	}
}
```

可以编写如下脚本：

```java
package me.arganzheng.study.btrace.script;

import static com.sun.btrace.BTraceUtils.*;

import java.util.Map;

import com.sun.btrace.AnyType;
import com.sun.btrace.annotations.*;

@BTrace
public class ApiDispatcherBTrace {

    @OnMethod(clazz = "me.arganzheng.study.btrace.target.controller.ApiDispatcher", method = "buildContext", location = @Location(value = Kind.ENTRY))
    public static void onBuildContextMethodEntry(@ProbeClassName
    String className, @ProbeMethodName
    String probeMethod, AnyType[] args) {
        println(Strings.strcat("EnterMethod: ", Strings.strcat(className, Strings.strcat(".", probeMethod))));
        printArray(args);
    }

    /**
     * @param context
     */
    @OnMethod(clazz = "me.arganzheng.study.btrace.target.controller.ApiDispatcher", method = "buildContext", location = @Location(Kind.RETURN))
    public static void onBuildContextMethodReturn(@Return
    Map<String, Object> context) {
        // NOTE: Can Only user methods defined in BTraceUtils...
        // for (Entry<String, Object> entry : context.entrySet()) {
        // // ...
        // }
        String clientIP = (String) Collections.get(context, "clientIP");
        println(Strings.strcat("clientIP=", clientIP));
    }
}	
```	

**TIPS** 用 IDE 来编写 Btrace 脚本

只要把 `<btrace>/build/btrace-client.jar` 加入到 `build path` 中，就可以直接在 eclipse 中很方便的编写和编译了（只能检查 java 语法，不能检查 BTrace 脚本规范，需要使用 `<btrace>/bin/btracec` 编译 ），可以有效的减少语法错误。另外，在扔到服务器之前，先用 `<btrace>/bin/btracec` 编译一下，确保符合 BTrace 脚本规范。

### 让运行中的应用加载你的BTrace脚本

通过JPS得到target JVM的pid，然后运行btrace脚本： 
	
```shell
webadmin@C2C_206_59_sles10_64:/home/arganzheng> btrace/bin/btrace 28257 /home/arganzheng/btrace_script/ApiDispatcherBTrace.java
```

当btrace脚本的条件满足时，就会执行：

```shell
EnterMethod: me.arganzheng.study.btrace.target.controller.ApiDispatcher.buildContext
[org.springframework.web.multipart.support.DefaultMultipartHttpServletRequest@26fc5b01, ]
clientIP=10.6.206.59
```

### 原理

[java.lang.instrument.Instrumentation](http://docs.oracle.com/javase/6/docs/technotes/guides/instrumentation/index.html)

### 限制

虽然这样方式很爽，不需要修改目标代码，不需要重启目标程序，完全无侵入性的AOP。但是为了保证trace语句只读, BTrace对trace脚本有一些限制(比如不能改变被trace代码中的状态) 

* BTrace class不能新建类, 新建数组, 抛异常, 捕获异常,
* 不能调用实例方法以及静态方法(com.sun.btrace.BTraceUtils除外)
* 不能将目标程序和对象赋值给BTrace的实例和静态field
* 不能定义外部, 内部, 匿名, 本地类
* 不能有同步块和方法
* 不能有循环
* 不能实现接口, 不能扩展类
* 不能使用assert语句, 不能使用class字面值

所以写起来比较蛋疼。。

当然，可以用 -u 运行在 `unsafe mode` 来规避限制，但不推荐。


### TIPS & NOTES

#### 拦截方法定位

##### 1、正则表达式定位

可以用表达式，批量定义需要监控的类与方法。正则表达式需要写在两个 "/" 中间。

下例监控 javax.swing 下的所有类的所有方法.... 可能会非常慢，建议范围还是窄些。

```java
@OnMethod(clazz="/javax\\.swing\\..*/", method="/.*/")
public static void swingMethods( @ProbeClassName String probeClass, @ProbeMethodName String probeMethod) {
   print("entered " + probeClass + "."  + probeMethod);
}
```

通过在拦截函数的定义里注入 @ProbeClassName String probeClass, @ProbeMethodName String probeMethod 参数，告诉脚本实际匹配到的类和方法名。

##### 2、按接口，父类，Annotation定位

比如我想匹配所有的 Filter 类，在接口或基类的名称前面，加个 `+` 就可以了: 

```java
@OnMethod(clazz="+life.arganzheng.study.btrace.Filter", method="doFilter")
```

具体例子可以参见: [btrace/samples/SubtypeTracer.java](https://github.com/btraceio/btrace/blob/master/samples/SubtypeTracer.java)

也可以按类或方法上的 annotaiton 匹配，前面加上 `@` 就可以了: 

```java
@OnMethod(clazz="@javax.jws.WebService", method="@javax.jws.WebMethod")
```

具体例子可以参见: [btrace/samples/WebServiceTracker.java](https://github.com/btraceio/btrace/blob/master/samples/WebServiceTracker.java)

##### 3、多个同名的函数，想区分开来，可以在拦截函数上定义不同的参数列表

如 `java.io.File` 类 的 createTempFile 方法，有两个实现：

1. public static File createTempFile(String prefix, String suffix);
2. public static File createTempFile(String prefix, String suffix, File directory);

如果要拦截第一个方法那么可以这么定义:

```java
import com.sun.btrace.AnyType;

@OnMethod(clazz = "java.io.File", method = "createTempFile", location = @Location(value = Kind.RETURN))
public static void onReturn(@Self Object self, String prefix, String suffix, @Return AnyType result)
```

如果想打印它们，首先按顺序定义:

1. 用 `@Self` 注释的 `this`
2. 完整的 参数列表 （可以按照顺序展开，也可以合并成 `AnyType[] args`）
3. 以及用 `@Return` 注释的返回值

需要使用到哪个就定义哪个，不需要的就不要定义。但定义一定要按顺序，比如参数列表不能跑到返回值的后面。

其中:

1、Self:

如果是静态函数， self为空。

前面提到，如果上述使用了非 JDK 的类，命令行里要指定 classpath。不过，如前所述，因为 BTrace 里不允许调用类的方法，所以定义具体类很多时候也没意思，所以 self 定义为 Object 就够了。

2、参数：

参数数列表要么不要定义，要定义就要定义完整，否则 BTrace 无法处理不同参数的同名函数。

如果有些参数你实在不想引入非 JDK 类，又不会造成同名函数不可区分，可以用 AnyType 来定义（不能用 Object）。

如果拦截点用正则表达式中匹配了多个函数，函数之间的参数个数不一样，你又还是想把参数打印出来时，可以用 `AnyType[] args` 来定义。

但不知道是不是当前版本的bug，AnyType[] args 不能和 location＝Kind.RETURN 同用，否则会进入一种奇怪的静默状态，只要有一个函数定义错了，整个Btrace就什么都打印不出来。

3、返回值

同理，结果也可以用 `AnyType` 来定义，特别是用正则表达式匹配多个函数的时候，连 void 都可以表示。


#### 拦截时机

可以为同一个函数的不同的Location，分别定义多个拦截函数。这点跟 Spring AOP 是完全类似的。

这个主要是通过 `@Location(Kind.XXX)`（[Kind](https://github.com/btraceio/btrace/blob/bf48290d06193e0f28c904183cb3e15906fa14f7/src/share/classes/com/sun/btrace/annotations/Kind.java)）:

* Kind.ARRAY_GET : Array element load
* Kind.ARRAY_SET : Array element store
* Kind.CALL : Method call
* Kind.CATCH : Exception catch
* Kind.CHECKCAST : Checkcast
* Kind.ENTRY : Method entry
* Kind.ERROR : "return" because of no-catch
* Kind.FIELD_GET : Getting a field value
* Kind.FIELD_SET : Setting a field value
* Kind.INSTANCEOF : instanceof check
* Kind.LINE : Source line number
* Kind.NEW : New object created
* Kind.NEWARRAY : New array created
* Kind.RETURN : Return from method
* Kind.SYNC_ENTRY : Entry into a synchronized block
* Kind.SYNC_EXIT : Exit from a synchronized block 
* Kind.THROW : Throwing an exception

虽然定义了很多，但是实际上常用的也就是下面这几个。

##### 1、`Kind.Entry` 和 `Kind.Return`

```java
@OnMethod( clazz="java.net.ServerSocket", method="bind" )
```

不写Location，默认就是刚进入函数的时候 (`Kind.ENTRY`)，表示在进入 `java.net.ServerSocket` bind 方法的时候进行拦截。

但如果你想获得函数的返回结果或执行时间，则必须把切入点定在 `Kind.RETURN` 时:

```java
OnMethod(clazz = "java.net.ServerSocket", method = "getLocalPort", location = @Location(Kind.RETURN))
public static void onGetPort(@Return int port, @Duration long duration) {
	...
}
```

**注意** : `duration` 的单位是纳秒，要除以 `1,000,000` 才是毫秒。

##### 2、`Kind.Error`、`Kind.Throw` 和 `Kind.Catch`

异常抛出(Throw)，异常被捕获(Catch)，异常没被捕获被抛出函数之外(Error)，主要用于对某些异常情况的跟踪。

在拦截函数的参数定义里注入一个Throwable的参数，代表异常。

```java
@OnMethod(clazz = "java.net.ServerSocket", method = "bind", location = @Location(Kind.ERROR))
public static void onBind(Throwable exception, @Duration long duration) {
	...
}
```

##### 3、`Kind.Call` 和 `Kind.Line`

下例定义监控 `bind()` 函数里调用的所有其他函数: 

```java
@OnMethod(clazz = "java.net.ServerSocket", method = "bind", 
	location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
public static void onBind(@Self Object self, @TargetInstance Object instance, @TargetMethodOrField String method, @Duration long duration) {
	...
}
```

所调用的类及方法名所注入到 `@TargetInstance` 与 `@TargetMethodOrField` 中。

​静态函数中，instance 的值为空。
如果想获得执行时间，必须把 Where 定义成 AFTER。

注意这里，一定不要像下面这样大范围的匹配，否则这性能是神仙也没法救了：

```java
@OnMethod(clazz = "/javax\\.swing\\..*/", method = "/.*/", 
	location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/"))
```

下例监控代码是否到达了 Socket 类的第 363 行。

```java
@OnMethod(clazz = "java.net.ServerSocket", location = @Location(value = Kind.LINE, line = 363))
public static void onBind4() {
   println("socket bind reach line:363");
}
```

line还可以为 -1，然后每行都会打印出来，加参数 int line 获得的当前行数。此时会显示函数里完整的执行路径，但肯定又非常慢。


### 一些例子

#### 1、打印慢调用

下例打印所有用时超过1毫秒的filter

```java
@OnMethod(clazz = "+life.arganzheng.study.btrace.Filter", method = "doFilter", location = @Location(Kind.RETURN))
public static void onDoFilterReturn(@ProbeClassName String pcn,  @Duration long duration) {
    if (duration > 1000000) {
        println(pcn + ",duration:" + (duration / 100000));
    }
}
```

**TIPS** 定位到某一个 Filter 慢了之后，可以直接用 `Location(Kind.CALL)` ，进一步找出它里面的哪一步慢了。

#### 2、谁调用了这个函数

比如，谁调用了 `System.gc()` ?

```java
@OnMethod(clazz = "java.lang.System", method = "gc")
public static void onSystemGC() {
    println("entered System.gc()");
    jstack();
}
```

#### 3、捕捉异常，或进入了某个特定函数/代码行时，this对象及参数的值

可以参考文章一开始的例子。


#### 4、打印函数的调用/慢调用的统计信息

BTrace 提供了如下几种机制，可以让你比较方便的统计接口的调用信息:

* [Aggregation](https://github.com/btraceio/btrace/blob/master/src/share/classes/com/sun/btrace/aggregation/Aggregation.java)
* [Sample](https://github.com/btraceio/btrace/blob/master/samples/AllMethodsSampled.java) 
* [Histogram.java](https://github.com/btraceio/btrace/blob/master/samples/Histogram.java)
* [HistoOnEvent.java](https://github.com/btraceio/btrace/blob/master/samples/HistoOnEvent.java) 
* [Profiling.java](https://github.com/btraceio/btrace/blob/master/samples/Profiling.java) 

当然你也可以用 AtomicInteger 构造计数器，然后定时 (@OnTimer)，或根据事件 (@OnEvent) 输出结果 (ctrl+c后选择发送事件)。

这里给大家介绍一下 BTrace 提供的一个专门分析性能工具: [BTraceUtils.Profiling](https://github.com/btraceio/btrace/blob/master/src/share/classes/com/sun/btrace/profiling/MethodInvocationProfiler.java)。

```java
package life.arganzheng.study;

import com.sun.btrace.BTraceUtils;
import com.sun.btrace.Profiler;
import com.sun.btrace.annotations.*;

/**
 * @author argan
 * @date 2019/03/11
 */
@BTrace
class BtraceProfiling {

    @Property
    Profiler profiler = BTraceUtils.Profiling.newProfiler();

    @OnMethod(clazz = "+life.arganzheng.study.btrace.Serializer", //
        method = "deserialize", //
        location = @Location(value = Kind.ENTRY))
    void onEntry(@ProbeMethodName(fqn = true) String probeMethod) {
        BTraceUtils.Profiling.recordEntry(profiler, probeMethod);
    }

    @OnMethod(clazz = "+life.arganzheng.study.btrace.Serializer", //
        method = "deserialize", //
        location = @Location(value = Kind.RETURN))
    void onReturn(@ProbeMethodName(fqn = true) String probeMethod, @Duration long duration) {
        BTraceUtils.Profiling.recordExit(profiler, probeMethod, duration);
    }

    @OnTimer(30000)
    void timer() {
        BTraceUtils.Profiling.printSnapshot("Serializer performance profile", profiler);
    }
}
```

**注意** class 不能是 public，否则会编译报错:

```shell
#bin/btracec script/BtraceProfiling.java
script/BtraceProfiling.java:14: error: script/BtraceProfiling.java:13:instance variables are not allowed
    Profiler profiler = BTraceUtils.Profiling.newProfiler();
             ^
script/BtraceProfiling.java:19: error: script/BtraceProfiling.java:16:instance methods are not allowed
    void onEntry(@ProbeMethodName(fqn = true) String probeMethod) {
         ^
script/BtraceProfiling.java:26: error: script/BtraceProfiling.java:23:instance methods are not allowed
    void onReturn(@ProbeMethodName(fqn = true) String probeMethod, @Duration long duration) {
         ^
script/BtraceProfiling.java:31: error: script/BtraceProfiling.java:30:instance methods are not allowed
    void timer() {
         ^
```

然后扔到服务器上执行:

```sh
[root@10.21.12.92:/usr/btrace]
#bin/btrace 30857 script/BtraceProfiling.java
Serializer performance profile
  Block                                                                                                                                                                        Invocations  SelfTime.Total  SelfTime.Avg  SelfTime.Min  SelfTime.Max  WallTime.Total  WallTime.Avg  WallTime.Min  WallTime.Max
  public volatile java.lang.Object life.arganzheng.study.btrace.serializer.impl.StringDoubleMapSerializer#deserialize(java.lang.String)                                            438466      9237018114         21066           899     647406149     22513126112         51345          3168     647413698
  public life.arganzheng.study.btrace.fealib.container.StringDoubleMap life.arganzheng.study.btrace.serializer.impl.StringDoubleMapSerializer#deserialize(java.lang.String)       438467     13276130045         30278          1610      13019459     13276130045         30278          1610      13019459
  public volatile java.lang.Object life.arganzheng.study.btrace.serializer.impl.StringSerializer#deserialize(java.lang.String)                                                    1367387     23497292267         17184           324      24884586     30843753653         22556           492      24887799
  public java.lang.String life.arganzheng.study.btrace.serializer.impl.StringSerializer#deserialize(java.lang.String)                                                             1367389      7346469993          5372           168      18949655      7346469993          5372           168      18949655
  public volatile java.lang.Object life.arganzheng.study.btrace.serializer.impl.DoubleSerializer#deserialize(java.lang.String)                                                     200279      3434033454         17146           349       1870931      4887393721         24402           643       1873112
  public java.lang.Double life.arganzheng.study.btrace.serializer.impl.DoubleSerializer#deserialize(java.lang.String)                                                              200279      1453360267          7256           294        815897      1453360267          7256           294        815897
  public volatile java.lang.Object life.arganzheng.study.btrace.serializer.impl.IntegerSerializer#deserialize(java.lang.String)                                                    197498      3476676247         17603           509      15031070      4702958072         23812           845      15050428
  public java.lang.Integer life.arganzheng.study.btrace.serializer.impl.IntegerSerializer#deserialize(java.lang.String)                                                            197498      1226281825          6209           199      15013941      1226281825          6209           199      15013941
  public volatile java.lang.Object life.arganzheng.study.btrace.serializer.impl.DoubleListSerializer#deserialize(java.lang.String)                                                 584698      9770631629         16710           467      22538002     23880497505         40842          1864      22545218
  public life.arganzheng.study.btrace.fealib.container.DoubleList life.arganzheng.study.btrace.serializer.impl.DoubleListSerializer#deserialize(java.lang.String)                 584699     14109898193         24131          1201      19619910     14109898193         24131          1201      19619910
  public volatile java.lang.Object life.arganzheng.study.btrace.serializer.impl.StringListSerializer#deserialize(java.lang.String)                                                 184585      3865020770         20938           481      12183812      5496685753         29778          1356      12268570
  public life.arganzheng.study.btrace.fealib.container.StringList life.arganzheng.study.btrace.serializer.impl.StringListSerializer#deserialize(java.lang.String)                 184585      1631664983          8839           449       3009604      1631664983          8839           449       3009604
...

^Z
[2]+  Stopped                 bin/btrace 30857 script/BtraceProfiling.java
```

所有的时间单位都是纳秒。

是不是很强大？

不过这种运行时动态增强和打印日志（这个影响更大）的方式会严重影响性能，线上需要慎用，拦截的范围要越小越好，打印的日志要越少越好。


**TIPS** 同名函数怎样精确拦截？

我们写了一个拦截：

```java
@OnMethod(clazz = "/com\\.argan\\.internet\\.ai\\.platform\\.fealib\\.feafunc\\.common..*/", //
        method = "apply", //
        location = @Location(value = Kind.ENTRY))
```
但是发现打印的 profile 日志有内部类拥有一样的函数名 apply。关键是会导致被监控到进程挂了，报如下错误：

```java
java.util.concurrent.ExecutionException: java.lang.NoClassDefFoundError: com/argan/internet/ai/platform/fealib/feafunc/common/TsToHour$Lambda$555
```

```sh
FeatureFunction performance profile
  Block                                                                                                                                  Invocations  SelfTime.Total  SelfTime.Avg  SelfTime.Min  SelfTime.Max  WallTime.Total  WallTime.Avg  WallTime.Min  WallTime.Max
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ElemSfDivRd#apply(java.lang.Object[])                              8       235321628      29415203     111947283     111947283       239585795      29948224     112008812     112008812
  public java.lang.Object com.argan.internet.ai.platform.fealib.feafunc.common.ElemSfDivRd$$Lambda$613#apply(java.lang.Object)                     48         4264167         88836          6212         22418         4264167         88836          6212         22418
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ZipWithIdx#apply(java.lang.Object[])                              12          122573         10214         13530         13530          122573         10214         13530         13530
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.Round#apply(java.lang.Object[])                                    3           61615         20538          5343         49783           61615         20538          5343         49783
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ZipSfDivRd#apply(java.lang.Object[])                               4        86264210      21566052         17405      86209646        86264210      21566052         17405      86209646
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.Sum#apply(java.lang.Object[])                                      6        23663651       3943941         23749      23527073        23913555       3985592         49094      23633671
  public java.lang.Object com.argan.internet.ai.platform.fealib.feafunc.common.Sum$$Lambda$615#apply(java.lang.Object, java.lang.Object)           36          249904          6941          3868         73407          249904          6941          3868         73407
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.GetNumBySplit#apply(java.lang.Object[])                            1        85495023      85495023      85495023      85495023        85495023      85495023      85495023      85495023
```

查看一下文档，发现 `@onMethod` 注解 有个 `type` 参数 可以指定完整的方法签名：

```
@OnMethod(clazz=<cname_spec>[, method=<mname_spec>]? [, type=<signature>]? [, location=<location_spec>]?)
```

其中 

```
signature = <return_type> ((arg_type(,arg_type)*)?
```

所以我们可以这样子拦截：

```java
package life.arganzheng.study;

import com.sun.btrace.BTraceUtils;
import com.sun.btrace.Profiler;
import com.sun.btrace.annotations.*;

/**
 * @author zhengzhibin
 * @date 2019/03/11
 */
@BTrace
class Profiling4FeatureFunction {

    @Property
    Profiler profiler = BTraceUtils.Profiling.newProfiler();

    @OnMethod(clazz = "/com\\.argan\\.internet\\.ai\\.platform\\.fealib\\.feafunc\\.common..*/", //
        method = "apply", type = "Object[] (java.lang.Object[])", //
        location = @Location(value = Kind.ENTRY))
    void onEntry(@ProbeMethodName(fqn = true) String probeMethod) {
        BTraceUtils.Profiling.recordEntry(profiler, probeMethod);
    }

    @OnMethod(clazz = "/com\\.argan\\.internet\\.ai\\.platform\\.fealib\\.feafunc\\.common..*/", //
        method = "apply", type = "Object[] (java.lang.Object[])", //
        location = @Location(value = Kind.RETURN))
    void onReturn(@ProbeMethodName(fqn = true) String probeMethod, @Duration long duration) {
        BTraceUtils.Profiling.recordExit(profiler, probeMethod, duration);
    }

    @OnTimer(60000)
    void timer() {
        BTraceUtils.Profiling.printSnapshot("FeatureFunction performance profile", profiler);
    }
}
```

果然现在不会挂了，而且只打印出精确的类:

```sh
#bin/btrace 4049 script/Profiling4FeatureFunction.java
FeatureFunction performance profile
  Block                                                                                                                   Invocations  SelfTime.Total  SelfTime.Avg  SelfTime.Min  SelfTime.Max  WallTime.Total  WallTime.Avg  WallTime.Min  WallTime.Max
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.TsToHour#apply(java.lang.Object[])             414252      2940911581          7099           518      45170177      2940911581          7099           518      45170177
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.TsToHourIdx#apply(java.lang.Object[])          414252      1015077694          2450           315      96732257      1015077694          2450           315      96732257
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.TsToWeekDay#apply(java.lang.Object[])          414252      1102223448          2660           351       9156696      1102223448          2660           351       9156696
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.IpSplit#apply(java.lang.Object[])              414252      1345673740          3248           518       3636720      1345673740          3248           518       3636720
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ZipWithIdx#apply(java.lang.Object[])         16965102     19257986784          1135            99      28691391     19257986784          1135            99      28691391
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.GetNumBySplit#apply(java.lang.Object[])       1653808      3951298130          2389           358      20588737      3951298130          2389           358      20588737
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.LogRd#apply(java.lang.Object[])               2480712      4210742105          1697           170       4443824      4210742105          1697           170       4443824
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.Split#apply(java.lang.Object[])                826905      1775562052          2147           161      21907911      1775562052          2147           161      21907911
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ParseKeyNumMap#apply(java.lang.Object[])      1240356      4819320801          3885           456      86441607      4819320801          3885           456      86441607
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ElemSfLogRd#apply(java.lang.Object[])         9802645     21945289929          2238           208     312793710     21945289929          2238           208     312793710
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.StrToNum#apply(java.lang.Object[])            1653808      2949540567          1783           163       8850330      2949540567          1783           163       8850330
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.DivideRd#apply(java.lang.Object[])             413452       537422835          1299           119        215791       537422835          1299           119        215791
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.Round#apply(java.lang.Object[])               1658184      2182575646          1316           103        269140      2182575646          1316           103        269140
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ZipSfDivRd#apply(java.lang.Object[])          4104898     10683597665          2602           171      27629351     10683597665          2602           171      27629351
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.Sum#apply(java.lang.Object[])                 4801603      5889664879          1226           110      21645135      5889664879          1226           110      21645135
  public java.lang.Object[] com.argan.internet.ai.platform.fealib.feafunc.common.ElemSfDivRd#apply(java.lang.Object[])         4801500      7326235886          1525           173       4942396      7326235886          1525           173       4942396
...
```

但是我们发现 `TsToHour` 这个算子耗时有点诡异，想看看他的调用链耗时，可以使用 `Kind.CALL` location 值:

```java
@OnMethod(clazz = "+com.argan.internet.ai.platform.fealib.feafunc.common.TsToHour", //
        method = "apply", type = "Object[] (java.lang.Object[])", //
        location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
  // @Sampled(kind = Sampled.Sampler.Adaptive)
  public static void onCall( // all calls to methods
      @Self Object self, // @Self 表示当前监控的函数所在类，如果是静态类则为空
      @TargetInstance Object instance, // @TargetInstance 表示函数中调用的方法或属性所在的类，如果是静态方法则为空
      @TargetMethodOrField(fqn = true) String method, // @TargetMethodOrField 表示调用的方法或属性
      @Duration long duration // 如果要获取执行时间，那么 where 必须设置为 Where.AFTER
  ) {
      println(strcat("self: ", str(self)));
      println(strcat("instance: ", str(instance)));
      println(strcat("method: ", str(method)));
      println(strcat("duration(ms): ", str(duration / 1000000)));

      if (duration > 1 * 100000) { // > 1ms
          println("=======invoke method:" + method + ", duration:" + (duration / 100000) + " ms");
      }
  }

  @OnError
  public static void onTraceProblem(Throwable t) {
      BTraceUtils.println("UNABLE TO TRACE: " + t);
  }
```

然而发现这个方法会卡住很久，一直没有反应。。

**TIPS**

阿里巴巴开源的Java诊断工具 [arthas](https://alibaba.github.io/arthas/) 可以替代 Btrace 的大部分功能，比如 [monitor](https://alibaba.github.io/arthas/monitor.html) 命令，跟 Btrace 的 Profiler 效果基本一样，使用起来更简单方便。不过两者都有一个问题，就是功能最强大也是最需要的 trace 功能，都有问题。btrace 是没有生效的感觉，一直卡住。arthas 是会严重影响性能，并且相当不准确。


### 参考文章

1. [BTrace](https://kenai.com/projects/btrace/pages/Home)
2. [BTrace – a Simple Way to Instrument Running Java Applications](http://piotrnowicki.com/2012/05/btrace-a-simple-way-to-instrument-running-java-applications/)
3. [Study_Java_BTrace ](http://source.hatter.me/p/hatter-source-code/wiki/Study_Java_BTrace)
4. [Btrace入门到熟练小工完全指南](http://calvin1978.blogcn.com/articles/btrace1.html)
5. [BTrace : Java 线上问题排查神器](https://cloud.tencent.com/developer/article/1015128)
6. [问题排查利器之-JVM动态追踪工具BTrace](https://coldwalker.com/2018/03//troubleshooter_btrace01/)
