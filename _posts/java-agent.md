---
title: Java Agent 学习笔记
date: 2018-10-04 23:26:20
categories: Java
tags: [Java, 字节码, javaagent, byte-buddy]
---

Java 从 1.5 开始提供了 `java.lang.instrument`（[doc](https://docs.oracle.com/javase/8/docs/technotes/guides/instrumentation/)）包，该包为检测（[instrument](https://en.wikipedia.org/wiki/Instrumentation_%28computer_programming%29)） Java 程序提供 API，比如用于监控、收集性能信息、诊断问题。通过 `java.lang.instrument` 实现工具被称为 Java Agent。Java Agent 可以修改类文件的字节码，通常是，在字节码方法插入额外的字节码来完成检测。关于如何使用 `java.lang.instrument` 包，可以参考 javadoc 的包描述（[en](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html), [zh](http://www.cjsdn.net/doc/jdk60/java/lang/instrument/package-summary.html)）。

<!--more-->

开发 Java Agent 的涉及的要点如下图所示 [ [ref](https://zeroturnaround.com/rebellabs/how-to-inspect-classes-in-your-jvm/) ]
<img width="700" alt="Java Agent" title="Java Agent" src="https://static.nullwy.me/java-agent-overview.png">

Java Agent 支持两种方式加载，启动时加载，即在 JVM 程序启动时在命令行指定一个选项来启动代理；启动后加载，这种方式使用从 JDK 1.6 开始提供的 [Attach API](https://docs.oracle.com/javase/8/docs/technotes/guides/attach/index.html) 来动态加载代理。


# 启动时加载 agent

## 最简单的例子

现在创建命名为 proj-demo 的 gradle 项目，目录布局如下：

```
$ tree proj-demo
proj-demo
├── build.gradle
└── src
    ├── main
    │   └── java
    │       └── com
    │           └── demo
    │               └── App.java
    └── test
        └── java

7 directories, 2 files
```

`com.demo.App` 类的实现：

``` java
public class App {

    public static void main(String[] args) throws InterruptedException {
        while (true) {
            System.out.println(getGreeting());
            Thread.sleep(1000L);
        }
    }

    public static String getGreeting() {
        return "hello world";
    }
}
```

运行 `com.demo.App`，每隔 1 秒输出 `hello world`：

``` sh
$ gradle build
$ java -cp "target/classes/java/main" com.demo.App
hello world
hello world
```

现在创建名称为 proj-premain 的 gradle 项目，`com.demo.MyPremain` 类实现 `premain` 方法：

``` java
package com.demo;

public class MyPremain {
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println(agentArgs);
    }
}
```

`META-INF/MANIFEST.MF` 文件指定 `Premain-Class` 属性：

``` gradle
jar {
    manifest {
        attributes 'Premain-Class': 'com.demo.MyPremain'
    }
    from {
        configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
    }
}
```

打包生成 `proj-premain.jar`，这个 jar 包就是 javaagent 代理。现在来试试运行 `com.demo.App` 时，启动这个 javaagent 代理。根据 javadoc 的描述，可以将以下选项添加到命令行来启动代理：

``` txt
    -javaagent:jarpath[=options] 
```

指定 `-javaagent:"proj-premain.jar=hello agent"`，传入的 `agentArgs` 为 `hello agent`，再次运行 `com.demo.App`：

``` sh
$ java -javaagent:"proj-premain.jar=hello agent" -cp "target/classes/java/main" com.demo.App
hello agent
hello world
hello world
```

可以看到，在运行 `main` 之前，运行了 `premain` 方法，即先输出 `hello agent`，每隔 1 秒输出 `hello world`。

## 修改字节码

在实现 `premain` 时，除了能获取 `agentArgs` 参数，还能获取 `Instrumentation` 实例。`Instrumentation` 类提供 ` addTransformer` 方法，用于注册提供的转换器 `ClassFileTransformer`：

``` java
// 注册提供的转换器
void addTransformer(ClassFileTransformer transformer)
```

`ClassFileTransformer` 是抽象接口，唯一需要实现的是 `transform` 方法。在转换器使用 `addTransformer` 注册之后，每次定义新类时（调用 `ClassLoader.defineClass`）都将调用该转换器的 `transform` 方法。该方法签名如下：

``` java
// 此方法的实现可以转换提供的类文件，并返回一个新的替换类文件
byte[] transform(ClassLoader loader,
                 String className,
                 Class<?> classBeingRedefined,
                 ProtectionDomain protectionDomain,
                 byte[] classfileBuffer)
                 throws IllegalClassFormatException
```

操作字节码可以使用 ASM、Apache BCEL、Javassist、cglib、Byte Buddy 等库。下面示例代码，使用 [BCEL](https://commons.apache.org/proper/commons-bcel/) 库实现名为 `GreetingTransformer` 转换器。该转换器实现的逻辑就是，将 `com.demo.App.getGreeting()` 方法输出的 `hello world`，替换为输出 `premain` 方法的传入的参数 `agentArgs`。

``` java
public class MyPremain {
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new GreetingTransformer(agentArgs));
    }
}
```

``` java
import org.apache.bcel.*;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

public class GreetingTransformer implements ClassFileTransformer {
    private String agentArgs;

    public GreetingTransformer(String agentArgs) {
        this.agentArgs = agentArgs;
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classfileBuffer) {
        if (!className.equals("com/demo/App")) {
            return classfileBuffer;
        }
        try {
            JavaClass clazz = Repository.lookupClass(className);
            ClassGen cg = new ClassGen(clazz);
            ConstantPoolGen cp = cg.getConstantPool();
            for (Method method : clazz.getMethods()) {
                if (method.getName().equals("getGreeting")) {
                    MethodGen mg = new MethodGen(method, cg.getClassName(), cp);
                    InstructionList il = new InstructionList();
                    il.append(new PUSH(cp, this.agentArgs));
                    il.append(InstructionFactory.createReturn(Type.STRING));
                    mg.setInstructionList(il);
                    mg.setMaxStack();
                    mg.setMaxLocals();
                    cg.replaceMethod(method, mg.getMethod());
                }
            }
            return cg.getJavaClass().getBytes();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

# 启动后加载 agent

最早 JDK 1.5发布 `java.lang.instrument` 包时，agent 是必须在 JVM 启动时，通过命令行选项附着（attach）上去。但在 JVM 正常运行时，加载 agent 没有意义，只有出现问题，需要诊断才需要附着 agent。JDK 1.6 实现了 attach-on-demand（按需附着）[ JDK-[4882798](https://bugs.openjdk.java.net/browse/JDK-4882798) ]，可以使用 Attach API 动态加载 agent [ [oracle blog](https://blogs.oracle.com/corejavatechtips/the-attach-api), [javadoc](https://docs.oracle.com/javase/8/docs/technotes/guides/attach/index.html) ]。这个 Attach API 在 `tools.jar` 中。JVM 启动时默认不加载这个 jar 包，需要在 classpath 中额外指定。使用 Attach API 动态加载 agent 的示例代码如下：

``` java
import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

public class AgentLoader {

    public static void main(String[] args) throws Exception {
        if (args.length < 2) {
            System.err.println("Usage: java -cp .:$JAVA_HOME/lib/tools.jar"
                    + " com.demo.AgentLoader <pid/name> <agent> [options]");
            System.exit(0);
        }

        String jvmPid = args[0];
        String agentJar = args[1];
        String options = args.length > 2 ? args[2] : null;
        for (VirtualMachineDescriptor jvm : VirtualMachine.list()) {
            if (jvm.displayName().contains(args[0])) {
                jvmPid = jvm.id();
                break;
            }
        }

        VirtualMachine jvm = VirtualMachine.attach(jvmPid);
        jvm.loadAgent(agentJar, options);
        jvm.detach();
    }
}
```

启动时加载 agent，`-javaagent` 传入的 jar 包需要在 `MANIFEST.MF` 中包含 `Premain-Class` 属性，此属性的值是 *代理类* 的名称，并且这个 *代理类* 要实现 `premain` 静态方法。启动后加载 agent 也是类似，通过 `Agent-Class` 属性指定 *代理类*，*代理类* 要实现 `agentemain` 静态方法。agent 被加载后，JVM 将尝试调用 `agentmain` 方法。

上文提到每次定义新类（调用 `ClassLoader.defineClass`）时，都将调用该转换器的 `transform` 方法。对于已经定义加载的类，需要使用重定义类（调用 `Instrumentation.redefineClass`）或重转换类（调用 `Instrumentation.retransformClass`）。

``` java
// 注册提供的转换器。如果 canRetransform 为 true，那么重转换类时也将调用该转换器
void addTransformer(ClassFileTransformer transformer, boolean canRetransform)
// 使用提供的类文件重定义提供的类集。新的类文件字节，通过 ClassDefinition 传入
void redefineClasses(ClassDefinition... definitions)
                     throws ClassNotFoundException, UnmodifiableClassException
// 重转换提供的类集。对于每个添加时 canRetransform 设为 true 的转换器，在这些转换器中调用 transform 方法 
void retransformClasses(Class<?>... classes)
                        throws UnmodifiableClassException
```

重定义类（redefineClass）从 JDK 1.5 开始支持，而重转换类（retransformClass）是 JDK 1.6 引入。相对来说，重转换类能力更强，当存在多个转换器时，重转换将由 transform 调用链组成，而重定义类无法组成调用链。重定义类能实现的逻辑，重转换类同样能完成，所以保留重定义类方法（`Instrumentation.redefineClass`）可能只是为了向后兼容 [ [stackoverflow](https://stackoverflow.com/q/19009583) ]。

实现 agentmain 的示例代码如下，其中 `GreetingTransformer` 转换器的类定义和上文一样。

``` java
public class MyAgentMain {

    public static void agentmain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new GreetingTransformer(agentArgs), true);
        try {
            Class clazz = Class.forName("com.demo.App");
            if (inst.isModifiableClass(clazz)) {
                inst.retransformClasses(clazz);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

`MANIFEST.MF` 文件配置：

```
jar {
    manifest {
        attributes 'Agent-Class': 'com.demo.MyAgentMain'
        attributes 'Can-Redefine-Classes' : true
        attributes 'Can-Retransform-Classes' : true
    }
    from {
        configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
    }
}
```

需要注意的是，和定义新类不同，重定义类和重转换类，可能会更改方法体、常量池和属性，但不得添加、移除、重命名字段或方法；不得更改方法签名、继承关系 [ [javadoc](http://www.cjsdn.net/doc/jdk60/java/lang/instrument/Instrumentation.html) ]。这个限制将来可能会通过 “JEP 159: Enhanced Class Redefinition” 移除 [ [ref](http://openjdk.java.net/jeps/159) ]。


# 使用 Byte Buddy

Byte Buddy（[home](http://bytebuddy.net/), [github](https://github.com/raphw/byte-buddy), [javadoc](http://bytebuddy.net/javadoc/1.8.0/)），运行时的代码生成和操作库，2015 年获得 Oracle 官方 Duke's Choice award，提供高级别的创建和修改 Java 类文件的 API，使用这个库时，不需要了解字节码。另外，对 Java Agent 的开发 Byte Buddy 也有很好的支持，可以参考 Byte Buddy 作者 Rafael Winterhalter 写的介绍文章 [ [ref1](http://www.infoq.com/cn/articles/easily-create-java-agents-with-bytebuddy), [ref2](https://www.sitepoint.com/fixing-bugs-in-running-java-code-with-dynamic-attach/) ]。

上文使用 BCEL 实现的 `GreetingTransformer`，现在改用 Byte Buddy，会变得非常简单。实现 `premain` 示例代码：

``` java
public static void premain(String agentArgs, Instrumentation inst) {
    new AgentBuilder.Default()
            .type(ElementMatchers.named("com.demo.App"))
            .transform(new AgentBuilder.Transformer() {
                @Override
                public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder,
                                                        TypeDescription typeDescription,
                                                        ClassLoader classLoader,
                                                        JavaModule module) {
                    return builder.method(ElementMatchers.named("getGreeting"))
                            .intercept(FixedValue.value(agentArgs));
                }
            }).installOn(inst);
}
```

实现 `agentmain`：

``` java
public static void agentmain(String agentArgs, Instrumentation inst) {
    new AgentBuilder.Default()
            .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
            .disableClassFormatChanges()
            .type(ElementMatchers.named("com.demo.App"))
            .transform(new AgentBuilder.Transformer() {
                @Override
                public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder,
                                                        TypeDescription typeDescription,
                                                        ClassLoader classLoader,
                                                        JavaModule module) {
                    return builder.method(ElementMatchers.named("getGreeting"))
                            .intercept(FixedValue.value(agentArgs));
                }
            }).installOn(inst);
}
```

另外，Byte Buddy 对 Attach API 作了封装，屏蔽了对 `tools.jar` 的加载，可以直接使用 `ByteBuddyAgent` 类：

``` java
ByteBuddyAgent.attach(new File(agentJar), jvmPid, options);
```

上文中的 `AgentLoader`，可以使用这个 API 简化，实现的完整示例参见 [AgentLoader2](https://github.com/yulewei/javaagent-demo/blob/master/proj-demo/src/main/java/com/demo/AgentLoader2.java)。

# 实现性能计时器

Byte Buddy 的 github 的 [README](https://github.com/raphw/byte-buddy#changing-existing-classes) 文档提供了一个性能计时拦截器的代码示例，能对某个方法的运行耗时做统计。现在我们来看下是如何实现的。假设 `com.demo.App2` 类如下：

``` java
public class App2 {

    public static void main(String[] args) {
        while (true) {
            System.out.println(getGreeting());
        }
    }

    public static String getGreeting() {
        try {
            Thread.sleep((long) (1000 * Math.random()));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello world";
    }
}
```

使用 Byte Buddy 实现计时拦截器的 agent，如下： 

``` java
public class TimerAgent {

    public static void premain(String agentArgs, Instrumentation inst) {
        new AgentBuilder.Default()
                .type(ElementMatchers.any())
                .transform((builder, type, classLoader, module) ->
                        builder.method(ElementMatchers.nameMatches(agentArgs))
                                .intercept(MethodDelegation.to(TimingInterceptor.class)))
                .installOn(inst);
    }
}
```

``` java
public class TimingInterceptor {

    @RuntimeType
    public static Object intercept(@Origin Method method, @SuperCall Callable<?> callable) throws Exception {
        long start = System.currentTimeMillis();
        try {
            return callable.call();
        } finally {
            System.out.println(method + " took " + (System.currentTimeMillis() - start) + "ms");
        }
    }
}
```

对 `getGreeting` 方法进行性能剖析，运行结果如下：

``` txt
$ java -javaagent:"proj-byte-buddy.jar=get.*" -cp "target/classes/java/main" com.demo.App2
public static java.lang.String com.demo.App2.getGreeting() took 694ms
hello world
public static java.lang.String com.demo.App2.getGreeting() took 507ms
hello world
```

示例代码中的 `premain` 参数 `agentArgs` 用于指定需要剖析性能的方法名，支持正则表达式。当实际参数传入 `get.*` 时，匹配到 `getGreeting` 方法。上面的示例，使用的是 Byte Buddy 的方法委托 Method Delegation API [ [javadoc](http://bytebuddy.net/javadoc/1.9.12/net/bytebuddy/implementation/MethodDelegation.html) ]。Delegation API 实现原理就是，将被拦截的方法委托到另一个办法上，如下左图所示（图片来自 Rafael Winterhalter 的 [slides](https://www.slideshare.net/RafaelWinterhalter/runtime-code-generation-for-the-jvm)）。这种写法会修改被代理类的类定义格式，只能用在启动时加载 agent，即 `premain` 方式代理。

若要通过 Byte Buddy 实现启动后动态加载 agent，官方提供了 Advice API [ [javadoc](http://bytebuddy.net/javadoc/1.9.12/net/bytebuddy/asm/Advice.html) ]。Advice API 实现原理上是，在被拦截方法内部的开始和结尾添加代码，如下右图所示。这样只更改了方法体，不更改方法签名，也没添加额外的方法，符合重定义类（redefineClass）和重转换类（retransformClass）的限制。

<img width="400" alt="delegation vs. advice" title="delegation vs. advice" src="https://static.nullwy.me/byte-buddy-delegation-vs-advice.png">

现在来看下使用 Advice API 实现性能定时器的代码示例：

``` java
public class TimingAdvice {

    @Advice.OnMethodEnter
    public static long enter() {
        return System.currentTimeMillis();
    }

    @Advice.OnMethodExit
    public static void exit(@Advice.Origin Method method, @Advice.Enter long start) {
        long duration = System.currentTimeMillis() - start;
        System.out.println(method + " took " + duration + "ms");
    }
}
```

``` java
public static void agentmain(String agentArgs, Instrumentation inst) {
    new AgentBuilder.Default()
            .disableClassFormatChanges()
            .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
//          .with(AgentBuilder.Listener.StreamWriting.toSystemOut())
            .type(ElementMatchers.any())
            .transform((builder, type, classLoader, module) ->
                    builder.visit(Advice.to(TimingAdvice.class)
                            .on(ElementMatchers.nameMatches(agentArgs))))
            .installOn(inst);
}
```

若对 `com.demo.App` 类，动态加载这个 Advice API 实现的 agent，`getGreeting()` 方法将会被重定义为（真正的实现可能稍有不同，但原理一致）：

``` java
public static String getGreeting() {
    long $start = System.nanoTime();
    String $result = "hello world";
    long $duration = System.nanoTime() – $start;
    System.out.println("App.getGreeting()" + " took " + $duration + "ms");
    return $result;
}
```


# 实际应用案例

Java Agent 的实际应用案例很多，举些笔者实际工作中使用到的开源软件的应用案例。

微服务是目前流行的互联网架构，实施微服务架构其中用于观察分布式服务的 APM （应用性能管理）系统是必不可缺的一环。典型的 APM 系统，如 [Pinpoint](https://naver.github.io/pinpoint/)、[SkyWalking](https://skywalking.apache.org/)，为了减少的 Java 服务应用代码的入侵，底层实现上都采用 Java Agent 技术，在 Java 服务应用启动时加载 agent，进行字节码增强技术，实现分布式追踪、服务性能监控等特性。具体可参见 Pinpoint [文档](https://naver.github.io/pinpoint/1.8.5/installation.html#5-pinpoint-agent)和 SkyWalking [文档](https://github.com/apache/skywalking/blob/master/docs/en/setup/service-agent/java-agent/README.md)。

Alibaba Java 诊断利器 [Arthas](https://github.com/alibaba/arthas)，实现上使用了动态 Attach API，相关源代码参见 [github](https://github.com/alibaba/arthas/blob/4.0.x/core/src/main/java/com/taobao/arthas/core/Arthas.java)。Arthas 4.0 开始支持 `premain` 方式启动时加载 agent，参见 issue [#550](https://github.com/alibaba/arthas/issues/550)。

---

**附注：**本文中提到的代码，可以在 github 上访问得到，[javaagent-demo](https://github.com/yulewei/javaagent-demo)。

# 参考资料

- 2016-08 Java 5 特性 Instrumentation 实践 <https://www.ibm.com/developerworks/cn/java/j-lo-instrumentation/>
- 2007-05 Java SE 6 新特性：Instrumentation 新功能 <https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html>
- 2017-08 The Attach API <https://blogs.oracle.com/corejavatechtips/the-attach-api>
- JPLIS: Java programming language agents need instrumentation support (JSR-163) <https://bugs.openjdk.java.net/browse/JDK-4882798>
- need an attach mechanism <https://bugs.openjdk.java.net/browse/JDK-6173612>
- Difference between redefine and retransform in javaagent <https://stackoverflow.com/q/19009583>
- 2015-09 JVM源码分析之javaagent原理完全解读 <http://www.infoq.com/cn/articles/javaagent-illustrated>
- 2016-02 Rafael Winterhalter：通过使用Byte Buddy，便捷地创建Java Agent <http://www.infoq.com/cn/articles/easily-create-java-agents-with-bytebuddy>
- 2017-01 Rafael Winterhalter: Fixing Bugs in Running Java Code with Dynamic Attach <https://www.sitepoint.com/fixing-bugs-in-running-java-code-with-dynamic-attach/>
- 2014-08 Making Java more dynamic: runtime code generation for the JVM <https://www.slideshare.net/RafaelWinterhalter/runtime-code-generation-for-the-jvm>
- 2016-04 Implementing a profiler agen with Byte Buddy which automatically attaches #110 <https://github.com/raphw/byte-buddy/issues/110>
- 2016-08 JavaDay Lviv 2016: Making Java more dynamic (Rafael Winterhalter) https://www.youtube.com/watch?v=jo1v8csBorw




