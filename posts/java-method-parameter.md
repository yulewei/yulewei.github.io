---
title: Java 运行时获取方法参数名 
date: 2017-05-02 15:09:51
categories: Java
tags: [Java, 字节码, JVM]
---

本文整理 Java 运行时获取方法参数名的两种方法，Java 8 的最新的方法和 Java 8 之前的方法。

# Java 8 的新特性

翻阅 Java 8 的[新特性](http://openjdk.java.net/projects/jdk8/features)，可以看到有这么一条“[JEP 118](http://openjdk.java.net/jeps/118): Access to Parameter Names at Runtime”。这个特性就是为了能运行时获取参数名新加的。这个 [JEP](https://en.wikipedia.org/wiki/JDK_Enhancement_Proposal) 只是功能增强的提案，并没有最终实现的 JDK 相关的 API 的介绍。查看“[Enhancements to the Reflection API](http://docs.oracle.com/javase/8/docs/technotes/guides/reflection/enhancements.html)” 会看到如下介绍：

<!--more-->

> **Enhancements in Java SE 8**
> Method Parameter Reflection: You can obtain the names of the formal parameters of any method or constructor with the method [java.lang.reflect.Executable.getParameters](http://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Executable.html#getParameters--). However, `.class` files do not store formal parameter names by default. To store formal parameter names in a particular `.class` file, and thus enable the Reflection API to retrieve formal parameter names, compile the source file with the `-parameters` option of the `javac` compiler.

<style>
blockquote {
    font-size: 12px
    font-family: Monaco,Menlo,Consolas,monospace;
}
</style>

`javac` 文档中关于 `-parameters` 的介绍如下 [ [doc](http://docs.oracle.com/javase/8/docs/technotes/tools/unix/javadoc.html) [man](https://www.mankier.com/1/javac) ]：

> **-parameters**
>    Stores formal parameter names of constructors and methods in the generated class file so that the method `java.lang.reflect.Executable.getParameters` from the Reflection API can retrieve them.

现在试验下这个特性。有如下两个文件：

``` java
package com.test;

public class TestClass {
    public int sum(int num1, int num2) {
        return num1 + num2;
    }
}
```

``` java
package com.test;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

public class Java8Main {
    public static void main(String[] args) throws NoSuchMethodException {
        Method method = TestClass.class.getDeclaredMethod("sum", int.class, int.class);
        Parameter[] parameters = method.getParameters();
        for (Parameter parameter : parameters) {
            System.out.println(parameter.getType().getName() + " " + parameter.getName());
        }
    }
}
```

先试试 `javac` 不加 `-parameters` 编译，结果如下：

``` shell
$ javac -d "target/classes" src/main/java/com/test/*.java
$ java -cp "target/classes" com.test.Java8Main
int arg0
int arg1
```

加上 `-parameters` 后，运行结果如下：

``` shell
$ javac -d "target/classes" -parameters src/main/java/com/test/*.java
$ java -cp "target/classes" com.test.Java8Main
int num1
int num2
```

可以看到，加上 `-parameters` 后，正确获得了参数名。实际开发中，很少直接用命令行编译 Java 代码，项目一般都会用 maven 管理。在 maven 下，只需修改 pom 文件的 `maven-compiler-plugin` 插件配置即可，就是加上了 `compilerArgs` 节点 [ [doc](https://maven.apache.org/plugins/maven-compiler-plugin/examples/pass-compiler-arguments.html) ]，如下：

``` xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <compilerArgs>
            <arg>-parameters</arg>
        </compilerArgs>
    </configuration>
</plugin>
```

## 实现原理

“Enhancements in Java SE 8”提到，参数名信息回存储在 class 文件中。现在试试用 `javap`（[ doc ](http://docs.oracle.com/javase/8/docs/technotes/tools/unix/javap.html) [man](https://www.mankier.com/1/javap)）命令反编译生成的 class 文件。反编译 class 文件：

``` shell
$ javap -v -cp "target/classes" com.test.TestClass
Classfile /Users/yulewei/IdeaProjects/hellojava/target/classes/com/test/TestClass.class
  Last modified 2017-5-2; size 305 bytes
  MD5 checksum 24b99fec7f3062f5de1c3ca4270a1d36
  Compiled from "TestClass.java"
public class com.test.TestClass
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#15         // java/lang/Object."<init>":()V
   #2 = Class              #16            // com/test/TestClass
   #3 = Class              #17            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               sum
   #9 = Utf8               (II)I
  #10 = Utf8               MethodParameters
  #11 = Utf8               num1
  #12 = Utf8               num2
  #13 = Utf8               SourceFile
  #14 = Utf8               TestClass.java
  #15 = NameAndType        #4:#5          // "<init>":()V
  #16 = Utf8               com/test/TestClass
  #17 = Utf8               java/lang/Object
{
  public com.test.TestClass();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public int sum(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn
      LineNumberTable:
        line 6: 0
    MethodParameters:
      Name                           Flags
      num1
      num2
}
SourceFile: "TestClass.java"
```

在结尾的 `MethodParameters` 属性就是，实现运行时获取方法参数的核心。这个属性是 Java 8 的 class 文件新加的，具体介绍可以参考官方“Java 虚拟机官方”文档的介绍，“4.7.24. The MethodParameters Attribute”，[doc](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.24)。


# class 文件中的调试信息

上文介绍了 Java 8 通过新增的反射 API 运行时获取方法参数名。那么在 Java 8 之前，有没有办法呢？或者在编译时没有开启 `-parameters` 参数，又如何动态获取方法参数名呢？其实 class 文件中保存的调试信息就可以包含方法参数名。

`javac` 的 `-g` 选项可以在 class 文件中生成调试信息，官方文档介绍如下 [ [doc](http://docs.oracle.com/javase/8/docs/technotes/tools/unix/javac.html#BHCDIFEE) [man](https://www.mankier.com/1/javac#-g) ]：

> **-g**
> Generates all debugging information, including local variables. By default, only line number and source file information is generated.
> **-g:none**
> Does not generate any debugging information.
> **-g:[keyword list]**
> Generates only some kinds of debugging information, specified by a comma separated list of keywords. Valid keywords are:
> &nbsp;&nbsp; **source**
> &nbsp;&nbsp;&nbsp;&nbsp; Source file debugging information.
> &nbsp;&nbsp; **lines**
> &nbsp;&nbsp;&nbsp;&nbsp; Line number debugging information.
> &nbsp;&nbsp; **vars**
> &nbsp;&nbsp;&nbsp;&nbsp; Local variable debugging information.

可以看到默认是包含源代码信息和行号信息的。现在试验下不生成调试信息的情况：

``` shell
$ javac -d "target/classes" src/main/java/com/test/*.java -g:none
$ javap -v -cp "target/classes" com.test.TestClass
Classfile /Users/yulewei/IdeaProjects/hellojava/target/classes/com/test/TestClass.class
  Last modified 2017-5-2; size 177 bytes
  MD5 checksum 559f5448154e4d7dd089f8155d8d0f55
public class com.test.TestClass
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#9          // java/lang/Object."<init>":()V
   #2 = Class              #10            // com/test/TestClass
   #3 = Class              #11            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               sum
   #8 = Utf8               (II)I
   #9 = NameAndType        #4:#5          // "<init>":()V
  #10 = Utf8               com/test/TestClass
  #11 = Utf8               java/lang/Object
{
  public com.test.TestClass();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return

  public int sum(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn
}
```

对比上文的反编译结果，可以看到，输出结果中的 `Compiled from "TestClass.java"` 没了，`Constant pool` 中也不再有 `LineNumberTable` 和 `SourceFile`，`code` 属性里的 `LocalVariableTable` 属性也没了（当然，因为编译时没加 `-parameters` 参数，`MethodParameters` 属性自然也没了）。若选择不生成这两个属性，对程序运行产生的最主要的影响就是，当抛出异常时，堆栈中将不会显示出错代码所属的文件名和出错的行号，并且在调试程序的时候，也无法按照源码行来设置断点。

``` shell
$ javac -d "target/classes" src/main/java/com/test/*.java -g:vars
$ javap -v -cp "target/classes" com.test.TestClass
Classfile /Users/yulewei/IdeaProjects/hellojava/target/classes/com/test/TestClass.class
  Last modified 2017-5-2; size 302 bytes
  MD5 checksum d430f817e0e2cfafc9095279c67aaa72
public class com.test.TestClass
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#15         // java/lang/Object."<init>":()V
   #2 = Class              #16            // com/test/TestClass
   #3 = Class              #17            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LocalVariableTable
   #8 = Utf8               this
   #9 = Utf8               Lcom/test/TestClass;
  #10 = Utf8               sum
  #11 = Utf8               (II)I
  #12 = Utf8               num1
  #13 = Utf8               I
  #14 = Utf8               num2
  #15 = NameAndType        #4:#5          // "<init>":()V
  #16 = Utf8               com/test/TestClass
  #17 = Utf8               java/lang/Object
{
  public com.test.TestClass();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/test/TestClass;

  public int sum(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  this   Lcom/test/TestClass;
            0       4     1  num1   I
            0       4     2  num2   I
}
```

可以看到，`code` 属性里的出现了 `LocalVariableTable` 属性，这个属性保存的就是方法参数和方法内的本地变量。在演示代码的 `sum` 方法中没有定义本地变量，若存在的话，也将会保存在 `LocalVariableTable` 中。

`javap` 的 `-v` 选项会输出全部反编译信息，若只想看行号和本地变量信息，改用 `-l` 即可。输出结果如下：

``` shell
$ javap -l -cp "target/classes" com.test.TestClass
public class com.test.TestClass {
  public com.test.TestClass();
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       5     0  this   Lcom/test/TestClass;

  public int sum(int, int);
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       4     0  this   Lcom/test/TestClass;
          0       4     1  num1   I
          0       4     2  num2   I
}
```

若要全部生成全部提示信息，编译参数需要改为 `-g:source,lines,vars`。一般在 IDE 下调试代码都需要调试信息，所以这三个参数默认都会开启。IDEA 下的 javac 默认参数设置，如图：

![IDEA 默认的 javac 设置](https://static.nullwy.me/2017-05-06-Jietu20170506-174601.png)

若使用 maven，maven 的默认的编译插件 `maven-compiler-plugin` 也会**默认开启**这三个参数 [[doc](https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#debug)]，经实际验证也包括了`LocalVariableTable`。

同样的，gradle **默认也会包含**调试信息 [ [doc](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.compile.CompileOptions.html#org.gradle.api.tasks.compile.CompileOptions:debug) ]。

# 代码如何实现

上文中讲了 class 文件中的调试信息中 `LocalVariableTable` 属性里就包含方法名参数，这就是运行时获取方法参数名的方法。读取这个属性，JDK 并没有提供 API，只能借助第三方库解析 class 文件实现。

要解析 class 文件典型的工具库有 ObjectWeb 的 ASM（[wiki](https://en.wikipedia.org/wiki/ObjectWeb_ASM)，[home](http://asm.ow2.org/)，[mvn](http://mvnrepository.com/artifact/org.ow2.asm/asm)，[javadoc](https://javadoc.io/doc/org.ow2.asm/asm)）、Apache 的 Commons BCEL（[wiki](https://en.wikipedia.org/wiki/Byte_Code_Engineering_Library)，[home](http://commons.apache.org/proper/commons-bcel/)，[mvn](http://mvnrepository.com/artifact/org.apache.bcel/bcel)，[javadoc](http://commons.apache.org/proper/commons-bcel/apidocs/)）、 日本教授开发的 Javassist（[wiki](https://en.wikipedia.org/wiki/Javassist)，[github](https://github.com/jboss-javassist/javassist)，[mvn](http://mvnrepository.com/artifact/org.javassist/javassist)，[javadoc](https://javadoc.io/doc/org.javassist/javassist)）等。其中 ASM 使用最广，使用 ASM 的知名开源项目有，AspectJ, CGLIB, Clojure, Groovy, JRuby, Jython, TopLink等等 [ [ref](http://asm.ow2.org/users.html) ]。当然使用 BCEL 的项目也很多 [ [ref](http://commons.apache.org/proper/commons-bcel/projects.html) ]。ASM 相对其他库的 jar 更小，运行速度更快 [ [javadoc](https://static.javadoc.io/org.ow2.asm/asm/5.2/org/objectweb/asm/package-summary.html#package.description) ]。目前 asm-5.0.1.jar 文件大小 53 KB，BCEL 5.2 版本文件大小 520 KB，javassist-3.20.0-GA.jar 文件大小 751 KB。jar 包文件小，自然意味着代码量更少，提供的功能自然也少了。

## BCEL

先来看看用 BCEL 获取方法参数名的写法，代码如下：

``` java
package com.test;
import org.apache.bcel.Repository;
import org.apache.bcel.classfile.JavaClass;
import org.apache.bcel.classfile.LocalVariable;
import org.apache.bcel.classfile.LocalVariableTable;
import org.apache.bcel.classfile.Method;
import org.apache.bcel.generic.Type;

public class BcelMain {

    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException {
        java.lang.reflect.Method m = TestClass.class.getDeclaredMethod("sum", int.class, int.class);
        JavaClass clazz = Repository.lookupClass("com.test.TestClass");
        Method bcelMethod = clazz.getMethod(m);
        LocalVariableTable lvt = bcelMethod.getLocalVariableTable();
        for (LocalVariable lv : lvt.getLocalVariableTable()) {
            System.out.println(lv.getName() + "  " + lv.getSignature() + "  " + Type.getReturnType(lv.getSignature()));
        }
    }
}
```

输出结果：

``` txt
this  Lcom/test/TestClass;  com.test.TestClass
num1  I  int
num2  I  int
```

## ASM

ASM 的写法如下：

``` java
package com.test;
import org.objectweb.asm.*;

public class AsmMain {

    public static void main(String[] args) throws Exception {
        ClassReader classReader = new ClassReader("com.test.TestClass");
        classReader.accept(new ParameterNameDiscoveringVisitor("sum", "(II)I"), 0);
    }

    private static class ParameterNameDiscoveringVisitor extends ClassVisitor {
        private final String methodName;
        private final String methodDesc;

        public ParameterNameDiscoveringVisitor(String name, String desc) {
            super(Opcodes.ASM5);
            this.methodName = name;
            this.methodDesc = desc;
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
            if (name.equals(this.methodName) && desc.equals(methodDesc))
                return new LocalVariableTableVisitor();
            return null;
        }
    }

    private static class LocalVariableTableVisitor extends MethodVisitor {

        public LocalVariableTableVisitor() {
            super(Opcodes.ASM5);
        }

        @Override
        public void visitLocalVariable(String name, String description, String signature, Label start, Label end, int index) {
            System.out.println(name + "  " + description);
        }
    }
}
```

## Spring 框架

若使用 Spring 框架，对于运行时获取参数名，Spring 提供了内建支持，对应的实现类为 `DefaultParameterNameDiscoverer` （[javadoc](http://docs.spring.io/spring/docs/4.3.x/javadoc-api/org/springframework/core/DefaultParameterNameDiscoverer.html)）。该类先尝试用 Java 8 新的反射 API 获取方法参数名，若无法获取，则使用 ASM 库读取 class 文件的 `LocalVariableTable`，对应的代码分别为 [StandardReflectionParameterNameDiscoverer](https://github.com/spring-projects/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/StandardReflectionParameterNameDiscoverer.java) 和 [LocalVariableTableParameterNameDiscoverer](https://github.com/spring-projects/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/LocalVariableTableParameterNameDiscoverer.java)。


# 参考资料

* 2014-10 Java 8 Named Method Parameters <https://www.beyondjava.net/blog/reading-java-8-method-parameter-named-reflection/>
* JEP 118: Access to Parameter Names at Runtime <http://openjdk.java.net/jeps/118>
* Enhancements to the Reflection API <http://docs.oracle.com/javase/8/docs/technotes/guides/reflection/enhancements.html>





