---
title: Java 编译器 javac 及 Lombok 实现原理解析
date: 2017-04-20 11:53:53
updated: 2022-10-31 00:55:42
categories: Java
tags: [Java, 编译器, javac, JVM, Lombok]
---

`javac` 是 Java 代码的编译器[^1][^2]，初学 Java 的时候就应该接触过。本文整理一些 `javac` 相关的高级用法。Lombok 库，大家平常一直在使用，但可能并不知道实现原理解析，其实 Lombok 实现上依赖的是 Java 编译器的注解处理 API（[JSR-296](https://www.jcp.org/en/jsr/detail?id=269)）[^3]，本文同时尝试解析 Lombok 的实现原理。

<!--more-->

先来看下 `javac` 命令行工具。`javac` 命令行工具，官方文档有完整的使用说明[^4]，当然也可以，运行 `javac -help` 或 `man javac` 查看帮助信息。下面是经典的 [hello world](https://en.wikipedia.org/wiki/%22Hello,_World!%22_program) 代码：

``` java
package com.example;
public class Greeting {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

编译与运行：

``` shell
$ tree   # 代码目录结构
.
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           └── Greeting.java
        └── resources
$ mkdir -p target/classes   # 创建 class 文件的存放目录
$ javac -d target/classes src/main/java/com/example/Greeting.java
$ java -cp target/classes com.example.Greeting
hello world
```

除了使用命令行工具编译 Java 代码，JDK 6 增加了规范“[JSR-199](https://jcp.org/en/jsr/detail?id=199): Java Compiler API”和“[JSR-296](https://www.jcp.org/en/jsr/detail?id=269): Pluggable Annotation Processing API”，开始还提供相关的 Java 编译器 API。Java 编译器的实现代码和 API 的整体结构如图所示[^2][^5]：

![Compiler Package Overview](https://static.nullwy.me/2017-04-24-1492606028088_5.png)


<span style="background-color: #cbe29a">绿色</span>标注的包是官方 API（Official API），即 JSR-199 和 JSR-296，<span style="background-color: #e4e48a;">黄色</span>标注的包为Supported API，<span style="background-color: #ccccff">紫色</span>标注的包代码全部在 `com.sun.tools.javac.*` 包下，为内部 API（Internal API）和实现类。完整的包说明如下[^2][^5][^6]：

* [javax.annotation.processing](http://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/package-summary.html) - 注解处理 ([JSR-296](https://www.jcp.org/en/jsr/detail?id=269))
* [javax.lang.model](http://docs.oracle.com/javase/8/docs/api/javax/lang/model/package-summary.html) - 注解处理和编译器 Tree API 使用的语言模型 ([JSR-296](https://www.jcp.org/en/jsr/detail?id=269))
  * [javax.lang.model.element](http://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/package-summary.html) - 语言元素。主要包含 `Element` 及其子类
  * [javax.lang.model.type](http://docs.oracle.com/javase/8/docs/api/javax/lang/model/type/package-summary.html) - 类型。主要包含 `TypeMirror` 及其子类
  * [javax.lang.model.util](http://docs.oracle.com/javase/8/docs/api/javax/lang/model/util/package-summary.html) - 语言模型工具。包含 `Elements`、`Types` 等类
* [javax.tools](http://docs.oracle.com/javase/8/docs/api/javax/tools/package-summary.html) - Java 编译器 API ([JSR-199](https://jcp.org/en/jsr/detail?id=199))
* [com.sun.source.\*](http://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/index.html) - 编译器的 Tree API，支持对抽象语法树做**只读访问**
* [com.sun.tools.javac.\*](https://www.javadoc.io/static/org.kohsuke.sorcerer/sorcerer-javac/0.11/index.html?com/sun/tools/javac/package-summary.html) - 内部 API 和实现类
  * `com.sun.tools.javac.api` - `javax.tools` 包下的 `JavaCompiler` 和其他 API 的实现
  * `com.sun.tools.javac.code` - `javax.lang.model.*` 包下的 API 的实现
  * `com.sun.tools.javac.comp` - 编译器主要处理阶段的实现
  * `com.sun.tools.javac.file` - 实现访问文件系统，包括 `javax.tools.StandardFileManager` 的实现
  * `com.sun.tools.javac.jvm` - class 文件的读写，编译器的字节码生成阶段的实现
  * `com.sun.tools.javac.main` - 代码编译的入口实现
  * `com.sun.tools.javac.model` - `javax.lang.model.*` 包的其他实现
  * `com.sun.tools.javac.parser` - 读取 Java 源代码，并生成语法树
  * `com.sun.tools.javac.processing` - 注解处理 API 的实现
  * `com.sun.tools.javac.resources` - 本地化文本和版本号的资源文件
  * `com.sun.tools.javac.tree` - 编译器语法树相关的表示类和工具类，`com.sun.source.*` 包下的 API 的实现
  * `com.sun.tools.javac.util` - 基础工具类

全部源码都位于 JDK 源码的 [langtools](https://github.com/openjdk/jdk8u/tree/master/langtools) 目录下。对外的 API，被编译到 `rt.jar`，`com.sun.source.*` 和 `com.sun.tools.javac.*` 包，被编译到 `tools.jar`，在 JDK 下的具体位置是 `$JAVA_HOME\lib\tools.jar`。值得一提的是，`langtools` 目录，除了包含 `javac` 的实现外，还实现了 `javadoc`、`javah` 等命令，编译后也是在 `tools.jar` 下。

另外，由于是内部 API 和实现类，`com.sun.tools.javac.*` 包下全部代码中都有标注警告：

> This is NOT part of any supported API. If you write code that depends on this, you do so at your own risk. This code and its internal interfaces are subject to change or deletion without notice.

# Java 编译器 API

首先，看下 [JSR-199](https://jcp.org/en/jsr/detail?id=199) 引入的 Java 编译器 API（Java Compiler API）。在没有引入 JSR-199 之前，如果要通过编程方式编译 Java 代码，只能使用 `com.sun.tools.javac.*` 包下提供内部 API。上文提到的使用命令 `javac` 编译 `Greeting.java` 的等价写法如下：

``` java
import com.sun.tools.javac.main.Main;

public class JavacMain {
    public static void main(String[] args) {
        Main compiler = new Main("javac");
        compiler.compile(new String[]{"src/main/com/example/Greeting.java", "-d", "target/classes"});
    }
}
```

事实上，`javac` 命令的底层实现就是执行 [com.sun.tools.javac.Main](https://github.com/openjdk/jdk8u/blob/master/langtools/src/share/classes/com/sun/tools/javac/Main.java) 类。执行 `javac` 命令，等价于执行 `java -cp $JAVA_HOME/lib/tools.jar com.sun.tools.javac.Main`。

``` bash
# 直接执行 com.sun.tools.javac.Main 类编译 Java 源代码
java -cp $JAVA_HOME/lib/tools.jar com.sun.tools.javac.Main -d target/classes src/main/java/com/example/Greeting.java
```

JSR-199，提供了 Java 编译器 API，对应的是 `javax.tools.*` 包。阅读包的 [javadoc](https://docs.oracle.com/javase/8/docs/api/javax/tools/package-summary.html#package.description) 容易发现，API 最核心是 [javax.tools.JavaCompiler](https://docs.oracle.com/javase/8/docs/api/javax/tools/JavaCompiler.html) 接口，该类的 javadoc 阐述了如何使用该类，可以阅读。使用 Java 编译器 API 编译 Java 源代码，示例如下：

``` java
import javax.tools.*;
import java.io.File;
import java.io.IOException;
import java.util.Arrays;

public class Jsr199Main {
    public static void main(String[] args) throws IOException {
        JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
        DiagnosticCollector<JavaFileObject> diagnostics = new DiagnosticCollector<>();

        StandardJavaFileManager fileManager = compiler.getStandardFileManager(diagnostics, null, null);

        File file = new File("src/main/java/com/example/Greeting.java");
        Iterable<? extends JavaFileObject> compilationUnits = fileManager.getJavaFileObjectsFromFiles(Arrays.asList(file));
        List<String> options = Arrays.asList("-d", "target/classes");

        compiler.getTask(null, fileManager, diagnostics, options, null, compilationUnits).call();

        fileManager.close();
    }
}
```

上述两种编程方式编译 Java 代码的方式，在 `javac` 命令的 man[^4] 文档的 “Programmatic Interface” 小节也有提及，有兴趣可以阅读。

在实际开发过程中，我们基本上都是使用 Maven 或 Gradle 编译 Java 代码。Maven 编译 Java 代码，依赖的是 Maven 的 `maven-compiler-plugin` 插件。那么 `maven-compiler-plugin` 插件底层实现是否使用了 `javax.tools.JavaCompiler` 呢？查阅官网文档后，容易发现实际情况和猜想的一样（其实也是显而易见的结论） [[doc](https://maven.apache.org/plugins/maven-compiler-plugin/)]：

> The Compiler Plugin is used to compile the sources of your project. Since 3.0, the default compiler is `javax.tools.JavaCompiler` (if you are using java 1.6) and is used to compile Java sources. If you want to force the plugin using javac, you must configure the plugin option `forceJavacCompilerUse`.

类似的，Gradle 编译 Java 代码，底层也使用了 Java 编译器 API，可以参见源码 `JdkJavaCompiler` [[github](https://github.com/gradle/gradle/blob/v7.5.0/subprojects/language-java/src/main/java/org/gradle/api/internal/tasks/compile/JdkJavaCompiler.java)]。

# javac 的编译过程
 
上文提到，[JSR-269](https://www.jcp.org/en/jsr/detail?id=269)，可插拔式注解处理 API（Pluggable Annotation Processing API）。注解处理，是编译过程中的其中一个阶段。要理解注解处理，需要先了解 Java 代码的编译过程。完整的编译过程如下图所示[^7]：

![javac-flow.png](https://static.nullwy.me/2017-04-24-javac-flow.png)

整个过程就是：

1. 源代码经过词法解析和语法解析（parse），生成抽象语法树（abstract syntax tree）。然后遍历抽象语法树，将遇到的符号填充入符号表（enter symbol table）。
2. 注解处理（annotation processing），所有注解处理器会被处理，若处理器生成新的代码或 class 文件，编译过程会重新开始，直到没有新的文件生成。每一次循环称为一个 round，也就是上图的回环过程。
3. 语义分析和字节码生成，包括标注（attribute）、数据及控制流分析（flow）、解语法糖（desugar）、字节码生成（generate）。

把上述编译过程对应到代码中，javac 编译动作的入口是 [com.sun.tools.javac.main.JavaCompiler](https://www.javadoc.io/static/org.kohsuke.sorcerer/sorcerer-javac/0.11/com/sun/tools/javac/main/JavaCompiler.html) 类，上述 3 个过程的代码逻辑集中在这个类的 compile(）和 compile2(）方法，如下图所示，整个编译过程主要的处理由图中标注的 8 个方法来完成[^8]：

<img width="700" alt="JavaCompiler compile" title="JavaCompiler compile" src="https://static.nullwy.me/JavaCompiler-compile.png">

具体来看下，词法解析和语法解析。Java 的词法和语法规则，在《Java语言规范》（[The Java Language Specification](https://docs.oracle.com/javase/specs/jls/se8/html/index.html)）中定义。从底层实现上来看，[com.sun.tools.javac.parser.Scanner](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/parser/Scanner.html) 类，按照单个字符的方式读取 Java 源文件中的关键字和标示符等内容，然后将其转换为符合 Java 语法规范（[JLS ch3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html)）的 [Token](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/parser/Tokens.Token.html) 序列。例如，针对语句 `int y = x + 1;` 的词法解析过程如下图所示[^9]：

<img width="700" alt="词法解析" title="词法解析" src="https://static.nullwy.me/javac-scanner.png">

然后，[com.sun.tools.javac.parser.JavacParser](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/parser/JavacParser.html) 类，读取 Token 序列，将 Token 序列构造为抽象语法树 [com.sun.tools.javac.tree.JCTree](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/JCTree.html)。语句 `int y = x + 1;`，生成的抽象语法树，如下图所示[^9]：

<img width="650" alt="抽象语法树" title="抽象语法树" src="https://static.nullwy.me/javac-syntax-tree.png">

该语句对应的 `JCTree.JCVariableDecl` 对象，在 IDEA 的 debug 模式下查看，如下图所示：

<img width="400" alt="IDEA debug" title="IDEA debug" src="https://static.nullwy.me/idea-debug-watch.png">

语法树中的每一个语法节点，实际上都直接或者间接地继承了 `JCTree` 类，并且都以静态内部类的形式定义在 `JCTree` 类中。Java 源文件的完整的词法解析和语法解析，由 `JavacParser` 的 `parseCompilationUnit` 方法完成。解析完成后，方法返回 `JCTree.JCCompilationUnit` 类。`JCTree.JCCompilationUnit` 类，为某个 Java 源文件解析后的整个语法树的根节点。

上文提到，`com.sun.source.*` 包下暴露的 Tree API，提供对语法树只能做只读操作。`com.sun.tools.javac.tree` 包，是 `com.sun.source.*` 包下的 API 的实现。`com.sun.source.tree.Tree` 接口对应的实现类为 `JCTree`，`Tree` 的子接口的实现类为 `JCTree` 的子类，并一一对应，比如，`com.sun.source.tree.ClassTree` 对应的实现类为 `JCTree.JCClassDecl`。`Tree` 接口及其子接口只暴露只读方法，而 `JCTree` 类及其子类，大部分的内部定义字段都是 `public`，可以直接读写。

主要的语法树节点 `JCTree` 子类，如下：

 - JCTree.[JCStatement](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/JCTree.JCStatement.html)：声明语句的语法树节点。主要的子类包括：
   - `JCTree.JCBlock`：语句块（[JLS 14.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.2)）
   - `JCTree.JCClassDecl`：类声明（[JLS 8.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.1)）
   - `JCTree.JCForLoop`：`for` 语句（[JLS 14.14.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.14.1)）
   - `JCTree.JCEnhancedForLoop`：增强for语句（[JLS 14.14.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.14.2)）
   - `JCTree.JCIf`：`if` 语句（[JLS 14.9](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.9)）
   - `JCTree.JCReturn`：`return` 语句（[JLS 14.7](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.17)）
   - `JCTree.JCVariableDecl`：变量声明，比如 `int x = 0` 语句（[JLS 14.4](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.4)）
   - 其他（不一一列举）
 - JCTree.[JCExpression](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/JCTree.JCExpression.html)：表达式的语法树节点。主要的子类包括：
   - `JCAssign`：赋值语句表达式，比如 `x = 0` 语句（[JLS 15.26](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.26)）
   - `JCIdent`：标识符表达式，比如 `x` 标识符（[JLS 3.8](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.8)）
   - `JCBinary`：二元运算符，比如 `x + 1` 语句（[JLS 15.18](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.18)）
   - `JCLiteral`：字面量运算符表达式，比如 `1` 字面量（[JLS 3.10](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10)）
   - `JCTree.JCPrimitiveTypeTree`：基础类型，比如 `int` 等类型（[JLS 4.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.2)）
 - JCTree.[JCMethodDecl](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/JCTree.JCMethodDecl.html)：方法声明（[JLS 8.4](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.4)）
 - JCTree.[JCCompilationUnit](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/JCTree.JCCompilationUnit.html)：编译单元，对应单个源文件内的全部内容（[JLS 7.3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-7.html#jls-7.3)）

全部的各个类型的树节点的类定义，可以参见 `JCTree` 和 `Tree` 类的 javadoc 或源代码。

在构造抽象语法树后，就是符号表填充阶段。在符号表填充阶段，会扫描 `JCTree` 语法树，遇到类型、变量、方法定义时，会它们的信息存储到符号表中，方便后续阶段进行快速查询。符号，对应的是 `com.sun.tools.javac.code.Symbol` 类。而 `Symbol` 类，是 `javax.lang.model` 包下 `Element` 的实现类，`Symbol` 子类是对应 `Element` 子类的实现。

`Element` 提供 `ElementKind getKind()` 方法，能获取元素类型（`ElementKind`）。全部的 `ElementKind` 共 17 种：`ANNOTATION_TYPE`（注解）、`CLASS`（类）、`CONSTRUCTOR`（构造方法）、`ENUM`（枚举）、`ENUM_CONSTANT`（枚举值）、`EXCEPTION_PARAMETER`（异常参数）、`FIELD`（字段）、`INSTANCE_INIT`（实例初始化语句块）、`INTERFACE`（接口）、`LOCAL_VARIABLE`（本地变量）、`METHOD`（方法）、`PACKAGE`（包）、`PARAMETER`（参数）、`RESOURCE_VARIABLE`（资源变量）、`STATIC_INIT`（静态初始化语句块）、`TYPE_PARAMETER`（类型参数） 以及 `OTHER`（其他）。

全部 `Element` 子类以及对应的 `Symbol` 子类，如下：

 - [PackageElement](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/PackageElement.html)：表示包 package
   - 实现类：`Symbol.PackageSymbol`
   - 元素类型 `ElementKind`：`PACKAGE`（包）
 - [TypeElement](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/TypeElement.html)：表示类 class 或接口 interface 等
   - 实现类：`Symbol.ClassSymbol`
   - 元素类型 `ElementKind`：`ANNOTATION_TYPE`（注解）、`INTERFACE`（接口）、`ENUM`（枚举）、`CLASS`（类）
 - [VariableElement](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/VariableElement.html)：表示字段、枚举值、方法参数、本地变量、资源变量、异常参数
   - 实现类：`Symbol.VarSymbol`
   - 元素类型 `ElementKind`：`EXCEPTION_PARAMETER`（异常参数）、`PARAMETER`（参数）、`ENUM_CONSTANT`（枚举值）、`RESOURCE_VARIABLE`（资源变量）、`LOCAL_VARIABLE`（本地变量）、`FIELD`（字段）
 - [ExecutableElement](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/ExecutableElement.html)：表示方法、构造方法、初始化语句块
   - 实现类：`Symbol.MethodSymbol`
   - 元素类型 `ElementKind`：`CONSTRUCTOR`（构造方法）、`STATIC_INIT`（静态初始化语句块）、`INSTANCE_INIT`（实例初始化语句块）、`METHOD`（方法）
 - [TypeParameterElement](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/element/TypeParameterElement.html)：表示参数化类型，即泛型尖括号内的类型
   - 实现类：`Symbol.TypeVariableSymbol`
   - 元素类型 `ElementKind`：`TYPE_PARAMETER`（类型参数）

在填充符号表后，就是语义分析和代码生成，包括标注（attribute）、数据及控制流分析（flow）、解语法糖（desugar）、字节码生成（generate）阶段。

在实际开发时，比如常见的“找不到符号（cannot find symbol）”编译报错，就是在标注阶段的名称消解（name resolution）时触发的。编译报错示例代码，如下：

``` java
public class CantResolve {
    int foo = bar;
}
```

编译错误的提示内容：

```
找不到符号
  符号:   变量 bar
  位置: 类 CantResolve
```

编译过程的各个阶段的更详细的阐述可以阅读书籍[^8][^10]，本文不再展开。

# 可插拔式注解处理 API

JSR-296 定义的可插拔式注解处理 API 在 `javax.annotation.processing` 包下，最核心的接口是 [javax.annotation.processing.Processor](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html)，通过实现这个接口来定义自己的注解处理器。

编译器工具与 `Processor` 实现类的交互过程是：

 - 如果存在没有被使用的 `Processor` 对象，就调用无参构造方法创建一个 `Processor` 实例。
 - 然后，编译器工具调用注解处理器的 [init](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html#init-javax.annotation.processing.ProcessingEnvironment-) 方法，初始化注解处理器，方法参数是 `ProcessingEnvironment` 对象（注解处理的执行环境，从环境中获得相关工具类，比如 `Elements`）。
 - 之后，编译器工具调用注解处理器的 [getSupportedAnnotationTypes](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html#getSupportedAnnotationTypes--)（查询该注解处理器支持的注解集合）、[getSupportedOptions](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html#getSupportedOptions--)（查询该注解处理器支持的参数选项集合）、[getSupportedSourceVersion](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html#getSupportedSourceVersion--)（查询该注解处理器支持的源代码版本）方法。
 - 最后，调用注解处理器的 [process](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html#process-java.util.Set-javax.annotation.processing.RoundEnvironment-) 方法。

注解处理会执行多轮（round），每轮都会调用 `process` 方法，调用时传入在上一轮的源代码和 class 文件中找到的该注解处理器支持的注解子集。在处理注解期间，如果任何注解处理器生成了新的源文件或 class 文件，编译器将回到解析、填充符号表、注解处理的过程，直到没有新的文件生成。

`init` 方法的参数 `ProcessingEnvironment` 对象，为注解处理的执行环境，从环境中获得相关工具类，比如，`Elements` 类，用于操作 `Element` 元素；`Filer` 类，用于生成新的文件；`Messager` 类，用于报告编译错误、告警或其他消息。另外，`ProcessingEnvironment`，也可以获得传递给注解处理器参数选项。

`AbstractProcessor` 抽象类，实现类了 `Processor` 接口，用于简化实际的注解处理器类的实现。该类通过读取 `@SupportedAnnotationTypes`、`@SupportedOptions`、`@SupportedSourceVersion` 注解值，来实现 `Processor` 接口对应的三个方法。

用命令行编译代码时，`javac` 编译器，会搜索可用的注解处理器。搜索路径可以通过参数选项 `-processorpath` 指定，如果未指定，将使用 `classpath`。注解处理器，可以通过 `-processor` 参数选项指定。若未通过 `-processor` 参数选项指定，注解处理器会使用 [SPI](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) 方式定位，在搜索路径查找 `META-INF/services/javax.annotation.processing.Processor` 文件。文件中填写的是注解处理器类名（多个的话，换行填写），编译器就会自动使用这里填写的注解处理器进行注解处理。另外，编译器 API 的 `CompilationTask` 的 `setProcessors` 方法也可以传入注解处理器。

如果注解处理器支持参数选项，编译时，参数选项可以用 `-Akey[=value]` 的方式传递[^4]。

## 扫描语法树

JDK 源码的 `langtools` 目录下，提供了示例注解处理器 [CheckNamesProcessor](https://github.com/openjdk/jdk8u/blob/master/langtools/src/share/sample/javac/processing/src/CheckNamesProcessor.java)，一个检查命名的注解处理器。`CheckNamesProcessor` 注解处理器，内部实现了 `javax.lang.model.util` 包下 [ElementScanner](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/util/ElementScanner8.html)，用来扫描 `Element` 元素符号，然后检查类命名、方法命名、字段命名、参数命名等是否符合命名规范，如果不符合命名规范，就打印编译器告警。

`javax.lang.model.util.ElementScanner8` 类用于扫描 `Element` 的核心方法：

``` java
public final R scan(Element e)
```

对语法树的扫描，`com.sun.source.util` 包下，提供了语法树扫描器 [TreeScanner](http://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/com/sun/source/util/TreeScanner.html)，用于扫描语法树上的树节点 `Tree`。类似的，`com.sun.tools.javac.tree.TreeScanner`，用于扫描语法树上的树节点 `JCTree`。

`com.sun.source.util.TreeScanner` 类用于扫描语法树的核心方法：

``` java
public R scan(Tree node, P p)
```

`com.sun.tools.javac.tree.TreeScanner` 类用于扫描语法树的核心方法：

``` java
public void scan(JCTree tree)
```

需要注意的是，注解处理器的 `process` 方法，传递过来的是 `Element` 对象，需要先获得 `Element` 对象关联的 `Tree` 或 `JCTree` 对象，才能扫描语法树。工具类 `com.sun.source.util.Trees` 提供了这样的桥接能力，该类的实现类为 `com.sun.tools.javac.api.JavacTrees`。`Trees` 的相关方法：

``` java
// 通过 ProcessingEnvironment 获得 Trees 对象
public static Trees instance(ProcessingEnvironment env)
// 通过 Element 获得 Tree
public abstract Tree getTree(Element element);
```

类似的，`JavacTrees` 的相关方法：

``` java
// 通过 ProcessingEnvironment 获得 JavacTrees 对象
public static JavacTrees instance(ProcessingEnvironment env)
// 通过 Element 获得 JCTree
public JCTree getTree(Element element)
```

使用 `ElementScanner` 或 `TreeScanner` 扫描语法树的示例注解处理器，参见 [VisitProcessor](https://github.com/yulewei/annotation-processor-demo/blob/master/processor-demo/src/main/java/com/example/visitor/VisitorProcessor.java)。

## 修改语法树

在语法解析时，`JavacParser` 类，底层实现上利用 [TreeMaker](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/TreeMaker.html) 类构造的语法树各个节点。`TreeMaker` 类，封装了创建语法树节点的方法，部分常用的方法举例：
 - TreeMaker.[Assign](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/TreeMaker.html#Assign-com.sun.tools.javac.tree.JCTree.JCExpression-com.sun.tools.javac.tree.JCTree.JCExpression-) 方法：用于生成赋值语句的语法树节点 `JCTree.JCAssign`。
 - TreeMaker.[Binary](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/TreeMaker.html#Binary-com.sun.tools.javac.tree.JCTree.Tag-com.sun.tools.javac.tree.JCTree.JCExpression-com.sun.tools.javac.tree.JCTree.JCExpression-) 方法：用于生成二元操作符的语法树节点 `JCTree.JCBinary`。
 - TreeMaker.[Block](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/TreeMaker.html#Block-long-com.sun.tools.javac.util.List-) 方法：用于生成语句块的语法树节点 `JCTree.JCBlock`。
 - TreeMaker.[VarDef](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/TreeMaker.html) 方法：用于生成变量定义的语法树节点 `JCTree.JCVariableDecl`。
 - TreeMaker.[MethodDef](https://www.javadoc.io/doc/org.kohsuke.sorcerer/sorcerer-javac/latest/com/sun/tools/javac/tree/TreeMaker.html) 方法：用于生成方法定义的语法树节点 `JCTree.JCMethodDecl`。
 - 等等

在注解处理阶段，`init` 方法传入了 `ProcessingEnvironment` 对象，通过该对象可以获得当前上下文中的 `TreeMaker` 对象，然后就可以利用 `TreeMaker` 创建新的语法树节点。

语句 `int y = x + 1;`，使用 `TreeMaker` 构造对应的 `JCTree.JCVariableDecl`，示例代码如下：

``` java
Name x = ...
Name y = names.fromString("y");
// x + 1
JCTree.JCBinary binary = maker.Binary(JCTree.Tag.PLUS, maker.Ident(x), maker.Literal(TypeTag.INT, 1));
// int y = x + 1
JCTree.JCVariableDecl decl = maker.VarDef(maker.Modifiers(0), y, maker.TypeIdent(TypeTag.INT), binary);
```

因为 `JCTree` 类及其子类的大部分的内部定义字段都是 `public`，可以直接读写，所以要想**修改语法树**，可以直接相关字段的值。比如，把 `int y = x + 1` 语句对应的 `JCTree.JCVariableDecl` 树节点改为 `int y = 42`，可以直接修改 `JCTree.JCVariableDecl` 的 `init` 字段，示例代码如下：

``` java
JCTree.JCVariableDecl decl = ...
decl.init = maker.Literal(TypeTag.INT, 42);
```

修改语法树的示例代码，参见 [PlusProcessor](https://github.com/yulewei/annotation-processor-demo/blob/master/processor-demo/src/main/java/com/example/maker/PlusProcessor.java) 注解处理器。该示例注解处理器，修改 `@PlusOne` 注解标注的方法的内部实现，改造后的方法的逻辑为，返回请求参数值加 1 后的值。比如，修改语法树前，`func` 方法实现如下：

``` java
@PlusOne
public int func(int x) {
  return x * x;
}
```

被 `PlusProcessor` 注解处理器修改语法树后，`func` 方法变成：

``` java
public int func(int x) {
  return x + 1;
}
```

修改方法内部实现的核心代码如下：

``` java
private void modifyToPlusOneMethod(JCTree.JCMethodDecl methodDecl) {
    JCTree.JCVariableDecl param = methodDecl.params.head;
    // x + 1
    JCTree.JCBinary binary = maker.Binary(JCTree.Tag.PLUS, maker.Ident(param.name), maker.Literal(TypeTag.INT, 1));
    JCTree.JCReturn ret = maker.Return(binary);
    // 修改方法内部实现
    methodDecl.body.stats = List.of(ret);
}
```

这个注解处理器仅仅用于示例，没有其他实际用途。实际开发中，Lombok 库被广泛使用，其底层实现就是利用注解处理器修改由 Lombok 注解（`@Data`、`@Getter`、`@Setter` 等）标注的代码的语法树，自动生成样板代码。针对 Lombok 库实现原理的解析，参见下文。

## 创建新文件

可插拔式注解处理 API，定义了 `javax.annotation.processing.Filer` 接口，这个接口提供了让注解处理器创建新文件的能力。`createSourceFile` 方法，用于创建新的源代码文件，`createClassFile`，用于创建新的 class 文件。

来看下示例代码，[GreetingProcessor](https://github.com/yulewei/annotation-processor-demo/blob/master/processor-demo/src/main/java/com/example/filer/GreetingProcessor.java) 注解处理器。该注解处理器功能就是基于 `Filer` 自动生成 `Greeting` 类（打印 "hello world"）。核心代码片段如下：

``` java
private boolean generateGreeting(String className) throws Exception {
    byte[] bytes = Files.readAllBytes(Paths.get(this.getClass().getResource("/Greeting.tpl").toURI()));
    String greetingTemplate = new String(bytes, StandardCharsets.UTF_8);
    String greetingSourceCode = String.format(greetingTemplate, LocalDateTime.now(), className);
    JavaFileObject fileObject = filer.createSourceFile(className);
    try (PrintWriter writer = new PrintWriter(fileObject.openWriter())) {
        writer.println(greetingSourceCode);
    }
    return true;
}
```

模板文件 `Greeting.tpl` 的内容为：

``` java
import javax.annotation.Generated;

@Generated(value = "by GreetingProcessor", date = "%s")
public class %s {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

在实际开发中，[MapStruct](https://mapstruct.org/) 是流行的用于 Bean 之间映射的工具库之一，其底层实现就是基于注解处理器 API。阅读源码，容易发现 MapStruct 库内部实现的注解处理器是 `org.mapstruct.ap.MappingProcessor`（[javadoc](https://mapstruct.org/documentation/1.5/api/org/mapstruct/ap/MappingProcessor.html)、[github](https://github.com/mapstruct/mapstruct/blob/1.5.x/processor/src/main/java/org/mapstruct/ap/MappingProcessor.java)）。`MappingProcessor` 注解处理器生成的 Mapper 实现类，底层调用的就是 `Filer` 接口的 `createSourceFile` 方法，参见源代码 [github](https://github.com/mapstruct/mapstruct/blob/1.5.x/processor/src/main/java/org/mapstruct/ap/internal/processor/MapperRenderingProcessor.java)。另外，MapStruct 库的注解处理器生成源代码文件利用了模板引擎 FreeMarker 库，可以参见 [javadoc](https://mapstruct.org/documentation/1.5/api/org/mapstruct/ap/MappingProcessor.html)、[github](https://github.com/mapstruct/mapstruct/tree/1.5.x/processor/src/main/resources/org/mapstruct/ap/internal/model)。

另外值得一提的是，除了模板引擎，生成源代码文件也可以使用 [JavaPoet](https://github.com/square/javapoet) 工具库，JavaPoet 库提供 Java API 来生成 `.java` 源文件。笔者基于 JavaPoet 库，实现了能处理类似 Lombok 的 `@Builder` 注解的 [BuilderProcessor](https://github.com/yulewei/annotation-processor-demo/blob/master/mylombok/src/main/java/com/example/filer/BuilderProcessor.java) 注解处理器，有兴趣的话可以查阅（附注：实际的 Lombok 的 `@Builder` 注解实现原理是修改语法树，并不是生成新的 `Builder` 类文件）。

# Lombok 的实现原理

依赖 JSR-269 实现的第三方工具库有很多[^11]，比如代码自动生成的 [Lombok](https://projectlombok.org/)、[MapStruct](https://mapstruct.org/) 和 Google [Auto](https://github.com/google/auto)，代码检查的 [Checker](https://checkerframework.org/) 和 Google [Error Prone](http://errorprone.info/)，编译阶段完成依赖注入的 Google [Dagger 2](https://github.com/google/dagger) 等。笔者在实际开发中就经常使用 Lombok 库和 MapStruct 库。MapStruct 库的实现原理，上文已经做了简单介绍。现在来看下 Lombok 的实现原理。

Lombok [提供](https://projectlombok.org/features/index.html) `@NonNull`、`@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@Data` 等注解，自动生成常见样板代码 boilerplate，解放开发效率。Lombok 支持 javac 和 ecj (Eclipse Compiler for Java)。对于 javac 编译器对应的注解处理器是 [LombokProcessor](https://github.com/rzwitserloot/lombok/blob/v1.16.10/src/core/lombok/javac/apt/LombokProcessor.java)，然后经过一些处理过程，每个注解都会有特定的 [handler](https://github.com/rzwitserloot/lombok/tree/v1.16.10/src/core/lombok/javac/handlers) 来处理，`@NonNull` 对应 `HandleNonNull`、`@Getter` 对应 `HandleGetter`、`@Setter` 对应 `HandleSetter`、`@ToString` 对应 `HandleToString`、`@EqualsAndHashCode` 对应 `HandleEqualsAndHashCode`、`@Data` 对应 `HandleData`。如果想要改造 Lombok 项目，让 Lombok 支持新的注解，其实就是添加新的 handler。关于 Lombok 原理以及如何为 Lombok 贡献代码，文档 “Documentation for lombok developers”[^12]，也有简单介绍，可以阅读。

阅读这些 handler 的实现，可以看到样板代码的生成依赖的就是 `com.sun.tools.javac.*` 包。最新版的 Lombok 源码太繁杂了，可以从早期版本入手，比如 [v0.8.1](https://github.com/projectlombok/lombok/tree/v0.8.1) 版本。

现在来看下如何实现 `@Getter` 注解。`@Getter` 注解的功能，就是自动生成类字段的 getter 方法，如果注解加到 class 上，就生成类的全部字段的 getter 方法。假设字段名叫 `foo`，那边生成的 getter 方法如下所示：

``` java
public int getFoo() {
  return foo;
}
```

参考 Lombok v0.8.1 和 v0.9.3 的 [HandleGetter](https://github.com/projectlombok/lombok/blob/v0.9.3/src/core/lombok/javac/handlers/HandleGetter.java) 实现源码（从 v0.9.3 版本开始，`@Getter` 注解支持加到 class 上，之前只能加到字段上），提取出其中的核心代码，实现 `@Getter` 的示例代码如下：

``` java
private void handleGetter(JCTree.JCClassDecl classDecl) {
    List<JCTree> methodDecls = List.nil();
    for (JCTree tree : classDecl.defs) {
        if (tree instanceof JCTree.JCVariableDecl) {
            // 创建 getter 方法
            JCTree.JCVariableDecl fieldDecl = (JCTree.JCVariableDecl) tree;
            String methodGetterName = Utils.toGetterName(fieldDecl);
            if (!Utils.methodExists(methodGetterName, classDecl)) {
                JCTree.JCMethodDecl methodGetter = this.createGetter(fieldDecl);
                methodDecls = methodDecls.append(methodGetter);
            }
        }
    }
    classDecl.defs = classDecl.defs.appendList(methodDecls);
}

// 生成 getter 方法
private JCTree.JCMethodDecl createGetter(JCTree.JCVariableDecl field) {
    JCTree.JCStatement returnStatement = maker.Return(maker.Ident(field));
    JCTree.JCBlock methodBody = maker.Block(0, List.of(returnStatement));
    Name methodName = names.fromString(Utils.toGetterName(field));
    JCTree.JCExpression methodType = (JCTree.JCExpression) field.getType();

    return maker.MethodDef(maker.Modifiers(Flags.PUBLIC), methodName, methodType,
            List.nil(), List.nil(), List.nil(), methodBody, null);
}
```

容易发现，实现 `@Getter` 注解依赖的 `JCTree`、`TreeMaker` 等相关类，这些类在上文都已经提及并介绍，不再复述。

为了加深对 javac 内部 API 的理解，笔者参考 Lombok 的源码，**实现了支持类似 Lombok 的 `@Data`、`@Getter`、`@Setter`、`@Slf4j` 注解的注解处理器**，[MyLombokProcessor](https://github.com/yulewei/annotation-processor-demo/blob/master/mylombok/src/main/java/com/example/processor/MyLombokProcessor.java)，代码参见 GitHub。

**附注**：本文的示例代码的完整代码，都可以在 GitHub 的 annotation-processor-demo[^13] 仓库上找到。

# 参考资料

[^1]: OpenJDK: The Java programming language Compiler Group <http://openjdk.java.net/groups/compiler/>
[^2]: The Java Programming Language Compiler, javac <https://docs.oracle.com/javase/8/docs/technotes/guides/javac/>
[^3]: 2011-05 How does lombok work? <http://stackoverflow.com/q/6107197>
[^4]: javac <https://docs.oracle.com/javase/8/docs/technotes/tools/unix/javac.html> <https://www.mankier.com/1/javac>
[^5]: OpenJDK: Compiler Package Overview <https://openjdk.org/groups/compiler/doc/package-overview/index.html>
[^6]: OpenJDK: The Hitchhiker's Guide to javac <https://openjdk.org/groups/compiler/doc/hhgtjavac/index.html>
[^7]: OpenJDK: Compilation Overview <https://openjdk.org/groups/compiler/doc/compilation-overview/index.html>
[^8]: 深入理解Java 7虚拟机，周志明 第2版2013：第10章 早期 (编译期) 优化
[^9]: 莫枢 RednaxelaFX ：JVM分享——Java程序的编译、加载与执行 <http://www.valleytalk.org/2011/07/28/java-程序的编译，加载-和-执行/>
[^10]: 深入解析Java编译器：源码剖析与实例详解，马智 2019
[^11]: Awesome Java Annotation Processing <https://github.com/gunnarmorling/awesome-annotation-processing>
[^12]: Documentation for lombok developers <https://projectlombok.org/contributing/>
[^13]: annotation-processor-demo <https://github.com/yulewei/annotation-processor-demo>






