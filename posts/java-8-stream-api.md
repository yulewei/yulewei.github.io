---
title: Java 8 的 Stream API 笔记
date: 2016-12-25 21:10:34 
categories: Java
tags: [Java]
---

Java 8增加了函数式编程的能力，通过流（Stream）API来支持对集合的[filter](https://en.wikipedia.org/wiki/Filter_%28higher-order_function%29)/[map](https://en.wikipedia.org/wiki/Map_%28higher-order_function%29)/[reduce](https://en.wikipedia.org/wiki/Fold_%28higher-order_function%29)操作。流是Java 8中处理集合的关键抽象概念，实现声明式的集合处理方式。

<!--more-->

# Java 8流API示例

对集合的filter、map和reduce操作，示例代码如下：

```java
// 计算偶数个数
List<Integer> list = Arrays.asList(3, 2, 12, 5, 6, 11, 13);
long count = list.stream()
        .filter(x -> x % 2 == 0).count();
System.out.println(count);

// 筛选出偶数列表
List<Integer> evenList = list.stream()
        .filter(x -> x % 2 == 0).collect(Collectors.toList());
System.out.println(evenList);

// 筛选出偶数列表, 然后全部元素加1
List<Integer> plusList = list.stream()
        .filter(x -> x % 2 == 0)
        .map(x -> x + 1).collect(Collectors.toList());
System.out.println(plusList); 

// 全部偶数求和
int sum = list.stream()
        .filter(x -> x % 2 == 0)
        .mapToInt(Integer::intValue).sum();
System.out.println(sum);

// 全部偶数求和
sum = list.stream()
        .filter(x -> x % 2 == 0)
        .reduce(0, (x, y) -> x + y);
System.out.println(sum);
```

# 流与集合的区别

流（Stream）和集合（collection）有以下区别 [[doc](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#package.description)]：
- **无存储**（no storage）。流并不是存储元素的数据结构。流，通过计算的操作流水线来传输数据源。数据源，比如数据结构、数组、生成器函数或I/O通道。
- **原生函数式**（functional in nature）。对流进行操作会产生结果，但并不会修改数据源。比如，筛选流，是生成一个不包含被筛选掉的元素新流，而不是从原集合中删除元素。
- **惰性读取**（laziness-seeking）。很多对流的操作，比如筛选（filtering）、映射（mapping）或去重（duplicate removal），能够被惰性实现，以便于性能的优化。比如，“找出第一个由三个连续单词组成的String”，并不需要检测全部的输入字符串。流操作被划分为*中间操作*（用于生成流）和*终止操作*（用于生成值或副作用）。中间操作总是惰性的。
- **可能是无界的**（possibly unbounded）。集合的大小是必须确定的，但流并不是。像limit(n) 或 findFirst()这样的逻辑短路操作，允许在有限时间内完成对无限流的计算。
- **可消耗的**（consumable）。流的元素在流的生命周期中只能被访问一次。类似于 [Iterator](https://docs.oracle.com/javase/8/docs/api/java/util/Iterator.html)，要想重新访问同一个元素，必须生成新的流。

# 流操作与流水线

当使用Stream时，会通过三个阶段来建立一个操作流水线 [[ref](https://book.douban.com/subject/26274206/)]。
1. 创建一个Stream。
2. 在一个或多个步骤中，指定将初始Stream转换为另一个Stream的`中间操作`。
3. 使用一个`终止操作`来产生一个结果。该操作会强制它之前的***延迟***操作立即执行。在这之后，该Stream就不会再被使用了。

整个流操作流水线，如图所示 [[ref](http://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/stream-api-intro/)]：

![流操作流水线](https://static.nullwy.me/2017-04-25-流水线.png)

Java 8的流创建、中间操作和终止操作的API汇总表，如下 [[ref](http://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/stream-cheat-sheet/)]

<style>
  td,th { border: 1px solid; } 
  td { vertical-align: top; }
</style>

| 创建流 | 中间操作 | 终止操作 |
| ---   | ---     | ---    |
| <p> **[Collection](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html)** <br> stream() <br> parallelStream() <p> **[Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)**, **[IntStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/IntStream.html)**, **[LongStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/LongStream.html), [DoubleStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/DoubleStream.html)** <br> static generate() *无序* <br> static of(..) <br> static empty() <br> static iterate(..) <br> static concat(..) <br> static builder() <br> <p> **[IntStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/IntStream.html)**, **[LongStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/LongStream.html)** <br> static range(..) <br> static rangeClosed(..) <p> **[Arrays](https://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html)** <br> static stream(..) <p> **[BufferedReader](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedReader.html)** <br> lines(..) <p> **[Files](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html)** <br> static list(..) <br> static walk(..) <br> static find(..) <p> **[JarFile](https://docs.oracle.com/javase/8/docs/api/java/util/jar/JarFile.html)** <br> stream() <p> **[ZipFile](https://docs.oracle.com/javase/8/docs/api/java/util/zip/ZipFile.html)** <br> stream() <p> **[Pattern](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html)** <br> splitAsStream(..) <p> **[SplittableRandom](https://docs.oracle.com/javase/8/docs/api/java/util/SplittableRandom.html)** <br> ints(..) *无序* <br> longs(..) *无序* <br> doubles(..) *无序* <p> **[Random](https://docs.oracle.com/javase/8/docs/api/java/util/Random.html)** <br> ints(..) <br> longs(..) <br> doubles(..) <p> **[ThreadLocalRandom](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadLocalRandom.html)** <br> ints() <br> longs(..) <br> doubles(..) <p> **[BitSet](https://docs.oracle.com/javase/8/docs/api/java/util/BitSet.html)** <br> stream() <p> **[CharSequence](https://docs.oracle.com/javase/8/docs/api/java/lang/CharSequence.html) (String)** <br> IntStream chars() <br> IntStream codePoints() <p> **[StreamSupport](https://docs.oracle.com/javase/8/docs/api/java/util/stream/StreamSupport.html) (low level)** <br> static doubleStream(..) <br> static intStream(..) <br> static longStream(..) <br> static stream(..) <p> | **[BaseStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/BaseStream.html)** <br> sequential() <br> parallel() <br> unordered() <br> onClose(..) <p> **[Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)** <br> filter(..) <br> map(..) <br> mapToInt(..) <br> mapToLong(..) <br> mapToDouble(..) <br> flatMap(..) <br> flatMapToInt(..) <br> flatMapToLong(..) <br> flatMapToDouble(..) <br> distinct() *有状态* <br> sorted(..) *有状态* <br> peek(..) <br> limit(..) *有状态, 逻辑短路* <br> skip(..) *有状态* <p> IntStream、LongStream和DoubleStream方法与Stream类似, 其中*额外的*方法如下：<p> **[IntStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/IntStream.html)** <br> mapToObj(..) <br> asLongStream() <br> asDoubleStream() <br> boxed() <br> <p> **[LongStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/LongStream.html)** <br> mapToObj(..) <br> asDoubleStream() <br> boxed() <p> **[DoubleStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/DoubleStream.html)** <br> mapToObj(..) <br> boxed() | <p> **[BaseStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/BaseStream.html)** <br> iterator() <br> spliterator() <p> **[Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)** <br> forEach(..) <br> forEachOrdered(..) <br> toArray(..) <br> reduce(..) <br> collect(..) <br> min(..) <br> max(..) <br> count() <br> anyMatch(..) *逻辑短路* <br> allMatch(..) *逻辑短路* <br> noneMatch(..) *逻辑短路* <br> findFirst() *逻辑短路* <br> findAny() *逻辑短路**, 不确定* <br> <p> IntStream、LongStream和DoubleStream方法与Stream类似, 其中*额外的*方法如下：<p> **[IntStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/IntStream.html), [LongStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/LongStream.html)**, **[DoubleStream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/DoubleStream.html)** <br> sum() <br> average()﻿﻿﻿﻿﻿ <br> summaryStatistics()


# 筛选与映射操作

筛选与映射操作，即上文所说的中间操作。相关的API如下：

```java
Stream<T> filter(Predicate<? super T> predicate)
<R> Stream<R> map(Function<? super T,? extends R> mapper)
<R> Stream<R> flatMap(Function<? super T,? extends Stream<? extends R>> mapper)
```

[filter(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#filter-java.util.function.Predicate-)，筛选出元素符合某个条件的新流。[map(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#map-java.util.function.Function-)，用于对元素作某种形式的转换。[flatMap(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#flatMap-java.util.function.Function-)，用于将多个子流合并为一个新流。

其他的中间操作有，提取子流，[limit(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#limit-long-)（提取前n个）和[skip(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#skip-long-)（丢弃前n个）；有状态的转换，[distinct()](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#distinct--)（过滤重复元素）和[sorted(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#sorted-java.util.Comparator-)（排序），以及主要用于日志调试的[peek(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#peek-java.util.function.Consumer-)。

```java
Stream<T> limit(long maxSize)
Stream<T> skip(long n)
Stream<T> distinct()
Stream<T> sorted(Comparator<? super T> comparator)
Stream<T> peek(Consumer<? super T> action)
```

# 归约操作与收集器

流有多种形式的通用的归约操作，[reduce(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#reduce-T-java.util.function.BinaryOperator-)和[collect(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#collect-java.util.stream.Collector-)，同时也有多种特化的归约形式，比如 [sum()](https://docs.oracle.com/javase/8/docs/api/java/util/stream/IntStream.html#sum--)、[min(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#min-java.util.Comparator-)、[max(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#max-java.util.Comparator-)或[count()](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#count--)等。[reduce(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#reduce-T-java.util.function.BinaryOperator-)和[collect(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#collect-java.util.stream.Collector-)的方法签名如下：

```java
Optional<T> reduce(BinaryOperator<T> accumulator)
T reduce(T identity, BinaryOperator<T> accumulator)
<U> U reduce(U identity, BiFunction<U,? super T,U> accumulator,
             BinaryOperator<U> combiner)
<R,A> R collect(Collector<? super T,A,R> collector)
```

[sum()](https://docs.oracle.com/javase/8/docs/api/java/util/stream/IntStream.html#sum--)、[min(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#min-java.util.Comparator-)、[max(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#max-java.util.Comparator-)或[count()](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#count--)等特化的归约操作，是[reduce(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#reduce-T-java.util.function.BinaryOperator-)归约操作的特殊化，内部实现依赖[reduce(..)](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#reduce-T-java.util.function.BinaryOperator-)，相应JDK的实现源码可以作为印证。

IntStream的sum(..)的JDK现实源码为 [[github](https://github.com/stain/jdk8u/blob/master/src/share/classes/java/util/stream/IntPipeline.java#L412)]：

```java
@Override
public final int sum() {
    return reduce(0, Integer::sum);
}
```

IntStream的min(..)的JDK现实源码为 [[github](https://github.com/stain/jdk8u/blob/master/src/share/classes/java/util/stream/IntPipeline.java#L417)]：

```java
@Override
public final OptionalInt min() {
    return reduce(Math::min);
}
```

Stream的count()的JDK现实源码为 [[github](https://github.com/stain/jdk8u/blob/master/src/share/classes/java/util/stream/ReferencePipeline.java#L524)]：

```java
@Override
public final long count() {
    return mapToLong(e -> 1L).sum();
}
```

如果不想将流归约为单个值，而只要查看集合被流操作后的结果，需要使用收集器（collector）。上文已经见到，将流收集为List：

```java
List<Integer> evenList = list.stream()
        .filter(..).collect(Collectors.toList());
```

[Collectors](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html)是收集器工具类，提供获取预定义收集器（实现接口[Collector](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html)）的静态方法。[Collectors](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html)类，除了将流收集为List，还可以是Map、Set、String，或者也能进行统计操作，如求和、计算平均值、最大值、最小值。[Collectors](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html)的全部静态方法列表如下：

- averaging[Double|Int|Long]
- collectingAndThen
- counting
- groupingBy
- groupingByConcurrent
- joining
- mapping
- maxBy
- minBy
- partitioningBy
- reducing
- summarizing[Double|Int|Long]
- summing[Double|Int|Long]
- toCollection
- toConcurrentMap
- toList
- toMap
- toSet


