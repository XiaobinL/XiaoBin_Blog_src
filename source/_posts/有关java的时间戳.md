---
title: 有关Java的时间戳问题
date: 2023-08-16 17:07:51
keywords: "Java,时间戳,纳秒级精度的时间戳"
tags:
   - Java
   - 时间戳
categories:
   - 开发进阶
cover: https://cdn.pixabay.com/photo/2023/06/15/17/07/sun-8066051_1280.jpg
---
# 背景
在做力扣的355题设计推特
https://leetcode.cn/problems/design-twitter/description/
时，一个主要的功能是要倒序返回一个用户以及他的所关注用户的前十条推文列表。我的做法是设计一个推文类，该类里面有一个成员变量`createdTime`，在生成一个推文类的时候记录该类生成的时间戳。然后等到返回推文列表的时候用一个`TreeSet`容器，通过自定义排序规则来实现按照生成时间倒序返回的目的。其中涉及到Java里面的时间戳的生成。我一开始使用的是`System.currentTimeMillis()`方法，这个方法返回的是一个毫秒精度的时间戳。但是这样做是有问题的，会卡在：连续调用函数生成推文类的用例那里。当我将代码搬到idea上debug的时候，发现了一个诡异的事情，直接run的时候会出bug，连续生成的两个推文类到最后返回的时候会吞掉一个；但是用debug模式的时候，有时候是正确的输出，有时候又是和run模式一样的错误的输出。一开始我以为是idea的debug模式会比run模式多执行了一些方法的问题（比如idea在debug模式下会自动调用一些集合的`toString()`方法，用于在debug的时候看到集合里面的内容，但是在run模式下不会），但是后来找很久发现并不是这里面的问题。最后无奈求助了队友才发现是上面提到的推文类里面时间戳的精度不够的问题。

# 发现问题
Java里面的`System`包下的`currentTimeMillis()`，调用之后返回的是一个以毫秒为精度的时间戳，在Java的源码中该方法的定义如下：
```Java
/**
Returns the current time in milliseconds. Note that while the unit of time of the return value is a millisecond, the granularity of the value depends on the underlying operating system and may be larger. For example, many operating systems measure time in units of tens of milliseconds.
See the description of the class Date for a discussion of slight discrepancies that may arise between "computer time" and coordinated universal time (UTC).
*/
public static native long currentTimeMillis();
```

该方法返回的是一个`long`类型的值，表示：当前时间与协调世界时1970年1月1日午夜之间的差值，以毫秒为单位。

但是：`System.currentTimeMillis()` 虽然返回值的时间单位是毫秒，但值的粒度取决于底层操作系统，在不同操作系统下的实际粒度是不同的。许多操作系统以数十毫秒为单位测量时间。比如：

1. **Windows 系统：** 在 Windows 操作系统中，默认情况下，`System.currentTimeMillis()` 的精度通常在 15 毫秒左右，即时间戳的更新间隔可能为 15 毫秒。这是因为 Windows 系统使用多媒体定时器（`Multimedia Timer`）来提供系统计时服务，而这个定时器的默认分辨率是 15 毫秒。

2. **Linux 系统：** 在 Linux 操作系统中，`System.currentTimeMillis()` 的实际精度取决于硬件、内核版本和系统配置。一般情况下，Linux 的时间戳精度可以达到几毫秒甚至更低，特别是在支持高分辨率计时器（`High-Resolution Timer`）的系统上。高分辨率计时器允许纳秒级的时间戳，但是系统配置和硬件支持可能会影响其实际精度。在一些现代的 Linux 发行版中，内核已经支持高分辨率计时器。

因此当发生连续的调用同一个方法的时候，两个方法执行时间之差小于该粒度，那么拿到的时间戳就是相同的，这就导致了后面在用`TreeSet`容器按照时间戳进行排序的时候会将其中一个去重处理，这就解释了上面的直接run会出错，但是debug模式下，在将断点打在方法内的时候，由于此时两个方法的调用时间被隔开了，因此这时拿到的时间戳就是不同的，进而没有问题的这一诡异现象。

# 解决方法
上面的问题是由于`System.currentTimeMillis()` 方法获取的时间戳粒度不够导致的，当连续两个方法调用的时间间隔小于该方法的粒度的时候，该方法拿到的是两个相同的时间戳，然后再使用`TreeSet`进行处理的时候，由于`TreeSet`容器的特性，会将时间戳相同的去重处理，因此解决方法也很简单，换一个更高精度的时间戳就可以了。一般认为达到纳秒级的时间精度就足以表示两个方法调用之间的最小时间间隔了，但是并不绝对，具体的还是要看不同的操作系统、硬件、方法调用开销以及可能的线程竞争等方面。但是在这里，我们只要将获取的时间戳的精度提高到纳秒级就可以了。

# Java的时间戳获取

一般我们获取时间戳的时候通常有以下这几种方式：

1. `System.currentTimeMillis()` 方法：返回的是一个`long`类型的值，表示：当前时间与协调世界时1970年1月1日午夜之间的差值，以毫秒为单位。

2. `new Date().getTime()`方法：该方法的底层其实还是调用的`System.currentTimeMillis()` 方法，其源码如下：

    ```Java
    public Date() {
            this(System.currentTimeMillis());
    }
    ```

3. `Calendar.getInstance().getTimeInMillis()`方法：也是返回此日历的时间值（以毫秒为单位）。

上面的三个返回的都是以毫秒值为单位的时间戳，但是他们在效率上还是有差异的：
- `System.currentTimeMillis()` 方法是调用本地方法，且是静态方法，不需要创建对象，因此是最快的；
- 通过`new Date().getTime()`方法获取时间戳，它的底层还是调用的`System.currentTimeMillis()` 方法，但是要创建Date对象，因此效率比不上`System.currentTimeMillis()` ；
- `Calendar.getInstance().getTimeInMillis()`方法由于考虑的因素比较多（比如考虑了时区的影响）因此在效率上表现最差。

# 精度更高的时间戳的获取

上面的三种获取时间戳的方法都是只能获取到毫秒精度的时间戳，这里的毫秒级精度并不是说两个值之间最小相差1毫秒，事实上有些系统的粒度甚至是几十毫秒。那我们想要获取到精度更高的时间戳（比如纳秒级的精度的时间戳）怎么办呢？

下面介绍两个我目前知道的方法。

## 通过System.nanoTime()方法

在System类的`currentTimeMillis()`这个方法的下面紧接着就定义着一个`nanoTime()`方法，该方法返回的是当前时间与某个特定起点时间之间的纳秒数差。它通常用于计算不同时间片段之间的时间差，而不是作为全局唯一的时间戳。在源码中是这样定义的：

```Java
/**
Returns the current value of the running Java Virtual Machine's high-resolution time source, in nanoseconds.

This method can only be used to measure elapsed time and is not related to any other notion of system or wall-clock time. The value returned represents nanoseconds since some fixed but arbitrary origin time (perhaps in the future, so values may be negative). The same origin is used by all invocations of this method in an instance of a Java virtual machine; other virtual machine instances are likely to use a different origin.

This method provides nanosecond precision, but not necessarily nanosecond resolution (that is, how frequently the value changes) - no guarantees are made except that the resolution is at least as good as that of currentTimeMillis().

Differences in successive calls that span greater than approximately 292 years (263 nanoseconds) will not correctly compute elapsed time due to numerical overflow.

The values returned by this method become meaningful only when the difference between two such values, obtained within the same instance of a Java virtual machine, is computed.

For example, to measure how long some code takes to execute:
  long startTime = System.nanoTime();   // ... the code being measured ...   
  long estimatedTime = System.nanoTime() - startTime;
  
To compare two nanoTime values
  long t0 = System.nanoTime();   ...   
  long t1 = System.nanoTime();
one should use t1 - t0 < 0, not t1 < t0, because of the possibility of numerical overflow.
*/
public static native long nanoTime();
```

根据上面的定义的注释，在使用这个方法之前要注意以下的几个方面：

1. 关于返回值：返回正在运行的Java虚拟机的高分辨率时间源的当前值，单位为纳秒。但是和`currentTimeMillis()`一样，虽然是纳秒级的精度，但并不意味着最小两个值之间的差是1纳秒。除了保证能提供比`currentTimeMillis()`方法更高的精度之外，并不保证返回的两个值之间的任何关系。
2. 这种方法只能用于测量经过的时间，与系统或挂钟时间的任何其他概念无关。返回的值表示自某个固定但任意的起始时间（时间原点）以来的纳秒（可能在将来，因此值可能为负数）。在Java虚拟机的实例中，此方法的所有调用都使用相同的原点；其他虚拟机实例可能使用不同的来源。
3. 由于数值溢出的问题，跨度超过约292年，或者小于263纳秒的连续调用之间的差异将无法正确计算所用时间。

##  java.time 包下的 `Instant` 类

Java 8 引入的 `java.time` 包提供了更强大且易于使用的工具。`Instant` 类是其中的一部分，用于表示时间轴上的一个特定瞬时点，而且提供了以毫秒和纳秒为单位的时间戳。下面我将详细介绍 `Instant` 类的创建、使用以及与其他方法的对比。

### **创建 Instant 实例：**

 创建 `Instant` 实例非常简单，可以使用静态方法 `Instant.now()` 获取当前的时间点。也可以通过提供秒数和纳秒数来创建一个特定的瞬时点。

```Java
// 获取当前时间的 Instant
Instant now = Instant.now();

// 创建指定时间点的 Instant
Instant specificInstant = Instant.ofEpochSecond(seconds, nanos);
```

### **获取时间戳：** 

`Instant` 实例提供了两种方式获取时间戳，分别是毫秒值和纳秒值。

```Java
long timestampMillis = instant.toEpochMilli(); // 获取毫秒级别的时间戳

long timestampNanos = instant.getNano(); // 获取纳秒级别的时间戳
```

### **与其他方法的对比：** 

`Instant` 类在 `java.time` 包中是一个非常有用的工具，用于表示高精度的时间戳和瞬时点。它在许多方面比传统的时间处理方法更优越，尤其是在精度、易用性和线程安全性方面。在与其他时间处理方法对比时，`Instant` 类在精度和易用性方面都有优势。

- 与 `System.currentTimeMillis()` 对比：`Instant` 类提供了更高精度的时间戳，可以获取纳秒级别的时间戳，而不仅仅是毫秒。同时，`Instant` 是不依赖于操作系统精度的。
- 与 `System.nanoTime()` 对比：`Instant` 类的 `getNano()` 方法可以用来获取当前纳秒值，类似于 `System.nanoTime()`，但需要注意 `System.nanoTime()` 更适合用于测量时间间隔，而不是绝对时间点。
- 与 `Date` 和 `Calendar` 对比：`Instant` 是 `java.time` 包的一部分，与传统的 `Date` 和 `Calendar` 相比，它更加现代且线程安全，同时提供了更好的易用性和精度。

