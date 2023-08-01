## Java日志体系

[toc]

### 概述

Java领域中存在多张日志框架，目前常用的框架有log4j、log4j2、commons logging、slf4j、lockback等，但是从整体上，这些日志框架可以被划分成两种类型：**记录型日志框架**和**门面型日志框架**

<table>
	<tr>
        <td></td>
		<td>日志名</td>
		<td>说明</td>
	</tr>
	<tr>
        <td rowspan="4">记录型日志</td>
		<td>JUL（Java Util Logging）</td>
        <td>JDK中的日志记录工具，自JDK1.4以来的官方日志实现</td>
	</tr>
	<tr>
		<td>Log4j</td>
        <td>Apache基金会项目中的一员，在Java程序中的应用十分广泛</td>
	</tr>
    <tr>
		<td>Log4j2</td>
        <td>Log4j的下一个版本，但是Log4j2不兼容Log4j1</td>
	</tr>
    <tr>
		<td>Logback</td>
        <td>SLF4J的实现，和SLF4J一个作者</td>
	</tr>
    <tr>
        <td rowspan="2">门面型日志</td>
        <td>JCL</td>
        <td>之前叫Jakarta Commons Logging，后更名为Commons Logging，是Apache基金会的项目，是一套Java日志接口</td>
	</tr>
	<tr>
        <td>Slf4j</td>
        <td>全称是Simple Logging Facade for Java，是一套简易的Java门面日志，本身并无日志的实现</td>
	</tr>
</table>

- 记录型日志：实际上就是实现具体的日志功能，在Java应用中打印各种类型的日志
- 门面型日志：因为有很多日志框架，不同应用中会使用不同的日志框架，非常混乱，**门面型日志的原理就是抽象一层高层API**，应用只需要依赖API，而不需要提供具体的日志实现，**具体的实现是通过SPI的方式实现的**

综上，Java日志领域被划分为两大阵营：Commons Logging阵营和Slf4j阵营





### Slf4j

因为目前基本上都使用SLF4J，所以下面我们来重点说一下SLF4J

- Slf4j的设计思想比较简洁，**使用了Facade设计模式**，Slf4j本身只提供了一个slf4j-api-version.jar包，这个jar包中主要是日志的抽象接口，jar包中本身并没有对抽象出来的接口做实现
- 对于不同的日志实现方案，比如logback、log4j等，封装出不同的桥接组件（例如logback-classic-version.jar，slf4j-log4j12-version.jar），这样使用过程中可以灵活的选取自己项目里的日志实现

<img src="../../image/Java/log1.jfif" style="zoom:50%;" />

如上图所示，**Slf4j作为门面日志提供了slf4j-api，即日志门面接口，日志门面接口本身通常并没有实际的日志输出能力，它底层还是需要去调用具体的日志框架API的，**也就是实际上它需要跟具体的日志框架结合使用，由于具体日志框架比较多，而且互相也大都不兼容，日志门面接口要想实现与任意日志框架结合可能需要对应的**桥接器**，上图红框内的组件即是对应的各种桥接器。



#### 核心类

下面列出的是slf4j-api-version.jar中的几个核心类和接口

- org.slf4j.LoggerFactory，给调用方提供的创建Logger的工厂类，在编译时绑定具体的日志实现组件

- org.slf4j.Logger，给调用方提供的日志记录抽象方法，例如info、debug、warn、error等

- org.slf4j.ILoggerFactory，获取Logger的工厂接口，具体的日志组件实现此接口

  ```java
  // 只有这一个方法
  public interface ILoggerFactory {
      Logger getLogger(String var1);
  }
  ```

- org.slf4j.helpers.NOPLogger，Slf4j的默认日志实现，是对org.slf4j.Logger接口的一个没有任何操作的实现

- **org.slf4j.impl.StaticLoggerBinder**，***与具体的日志实现组件实现的桥接类，具体的日志实现组件需要定义org.slf4j.impl包，并在org.slf4j.impl包下提供此类***，注意在slf4j-api-version.jar中不存在org.slf4j.impl.StaticLoggerBinder，在源码包slf4j-api-version-source.jar中才存在此类



#### 源码分析

##### 当没有引入任何日志实现类时的流程

如果在代码中之引入了slf4j，而没有引入其他的日志实现类，如下所示：

```xml
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>1.7.36</version>
</dependency>
```



```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OnlySlf4j {
    final static Logger logger = LoggerFactory.getLogger(OnlySlf4j.class);

    public static void main(String[] args) {
        logger.info("hello, slf4j");
    }
}
```

程序运行之后，并没有日志打印出来，反而会出现警告信息

![](../../image/Java/log2.jfif)

根据警告信息我们也能够看出来，因为没有具体的日志实现组件，所以slf4j使用的是默认nop logger，所以不会有日志打印出来

下面我们来看一下源代码，从`LoggerFactory.getLogger(OnlySlf4j.class)`入口开始，一直能够找到`findPossibleStaticLoggerBinderPathSet`方法

<img src="../../image/Java/log3.jfif" style="zoom:150%;" />

<img src="../../image/Java/log4.jfif" style="zoom:67%;" />

`findPossibleStaticLoggerBinderPathSet`方法如下：

可以看出，**该方法的作用是查找`org/slf4j/impl`路径下的`StaticLoggerBinder`类**

```java
private static Set findPossibleStaticLoggerBinderPathSet() {
  LinkedHashSet staticLoggerBinderPathSet = new LinkedHashSet();

  try {
    ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
    Enumeration paths;
    if (loggerFactoryClassLoader == null) {
      // private static String STATIC_LOGGER_BINDER_PATH = "org/slf4j/impl/StaticLoggerBinder.class"
      paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
    } else {
      paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);
    }

    while(paths.hasMoreElements()) {
      URL path = (URL)paths.nextElement();
      staticLoggerBinderPathSet.add(path);
    }
  } catch (IOException var4) {
    Util.report("Error getting resources from path", var4);
  }

  return staticLoggerBinderPathSet;
}
```

该方法返回之后，会进入`reportMultipleBindingAmbiguity(staticLoggerBinderPathSet)`方法，这个方法很简单，其实就是看一下是否有多个具体的日志实现类，如果有多个，则返回一些警告信息

```java
private static void reportMultipleBindingAmbiguity(Set staticLoggerBinderPathSet) {
  if (isAmbiguousStaticLoggerBinderPathSet(staticLoggerBinderPathSet)) {
    Util.report("Class path contains multiple SLF4J bindings.");
    Iterator iterator = staticLoggerBinderPathSet.iterator();

    while(iterator.hasNext()) {	
      URL path = (URL)iterator.next();
      Util.report("Found binding in [" + path + "]");
    }

    Util.report("See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.");
  }
}
```

因为我们的类路径下没有日志实现类，所以`staticLoggerBinderPathSet`为空，所以`StaticLoggerBinder.getSingleton()`会返回异常

![](../../image/Java/log5.jfif)

方法返回之后，可以看到会返回一个NOPLoggerFactory对象

![](../../image/Java/log6.jfif)



##### 添加logback作为slf4j的日志实现类

 ```xml
 <dependency>
   <groupId>ch.qos.logback</groupId>
   <artifactId>logback-classic</artifactId>
   <version>1.2.3</version>
 </dependency>
 ```

![](../../image/Java/log7.jfif)

因为现在的`staticLoggerBinderPathSet`不再为空，所以`StaticLoggerBinder.getSingleton()`不会返回异常，而是会创建一个`StaticLoggerBinder`对象，随后，`bind`方法就会返回，程序进入如下位置，此时`LoggerFactory`中的`getLogger()`方法中获取到的`ILoggerFactory`实际上是`logback jar`下的`LoggerContext`，随后`LoggerFactory`调用`getLogger()`方法获取到的`Logger`实际上是`logback jar`下的`Logger`。

![](../../image/Java/log8.jfif)



##### 添加了多个日志实现类

```xml
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>1.7.36</version>
</dependency>

<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.2.3</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.7.30</version>
</dependency>
```

如果存在多个日志实现类的话，那么会出现下面的这种情况，会有警告信息，**JVM会选择pom文件中第一个日志实现类对应的StaticLoggerBinder，也就是说classpath中加载顺序靠前的框架**

![](../../image/Java/log9.jfif)

> 如果我们的应用添加了多个日志绑定包Slf4j会选择哪一个呢？**Slf4j官网给出的解答是会随机选取一个**，其实这里并不是一定随机的，作者这里的随机指的是依赖于JVM的版本，不同的版本可能绑定的日志实现是有差别的，**比如我现在用的JDK1.8版本在JVM加载类的时候如果遇见了相同包名、类名的类只会加载第一个（由上面的介绍我们知道日志绑定包的实现是由一个同包名、类名的类StaticLoggerBinder实现的），后面扫描到的类不会加载，所以在这个版本是第一个加载的类生效，**但是这个规则在后续的版本中也可能会有变化，所以在不同版本中可能是随机的。



## 统一日志方案

在实际环境中我们经常会遇到不同的组件使用的日志框架不同的情况，例如`Spring Framework`使用的是日志组件是`Commons Logging`，`XSocket`依赖的则是`Java Util Logging`。当我们在同一项目中使用不同的组件时应该如果解决不同组件依赖的日志组件不一致的情况呢？现在我们需要统一日志方案，统一使用`Slf4j`，把他们的日志输出重定向到`Slf4j`，然后`Slf4j`又会根据绑定器把日志交给具体的日志实现工具。`Slf4j`带有几个桥接模块，可以重定向`Log4j`，`JCL`和`java.util.logging`中的`Api`到`Slf4j`。

| jar包名称                    | 作用                                            |
| ---------------------------- | ----------------------------------------------- |
| log4j-over-slf4j-version.jar | 将log4j重定向到slf4j                            |
| jcl-over-slf4j-version.ja    | 将Commons Logging里的Simple Logger重定向到slf4j |
| jul-to-slf4j-version.jar     | 将Java Util Logging重定向到slf4j                |

在使用Slf4j桥接时要注意避免形成死循环，在项目依赖的jar包中不要存在以下情况

- `log4j-over-slf4j.jar`和`slf4j-log4j12.jar`同时存在，这是因为`slf4j-log4j12.jar`的存在会将所有日志调用委托给`log4j`，但由于同时有`log4j-over-slf4j.jar`的存在，会将所有对`log4j api`的调用委托给相应等值的`slf4j`，所以`log4j-over-slf4j.jar`和`slf4j-log4j12.jar`同时存在会形成死循环
- `jul-to-slf4j.jar`和`slf4j-jdk14.jar`同时存在，同理，由于`slf4j-jdk14.jar`的存在会将所有日志调用委托给`jdk`的`log`。但由于同时`jul-to-slf4j.jar`的存在，会将所有对`jul api`的调用委托给相应等值的`slf4j`，所以`jul-to-slf4j.jar`和`slf4j-jdk14.jar`同时存在会形成死循环
- `jcl-over-slf4j`和`slf4j-jcl`同时存在，同理。

<img src="../../image/Java/log10.jfif" style="zoom:67%;" />

### Case 1 新业务应用，想要使用slf4j作为日志API，使用log4j2作为日志实现

这种情况比较简单，我们只需要引入Slf4j的API包（slf4j-api-xxx.jar）、Log4j2的绑定包（log4j-slf4j-impl-xxx.jar）和Log4j2的日志实现包（log4j-core-xxx.jar）即可，比较简单就不在赘述了。



### Case 2 遗留业务应用，一个模块使用log4j作为日志框架或引用的其他第三方包中使用log4j，现在想要统一日志框架为slf4j+log4j2，如何统一输出？

![](../../image/Java/log11.jfif)

如上图所示，这种情况下需要比case 1在多引入一个Log4j桥接包（log4j-over-slf4j-xxx.jar）来把直接使用Log4j的遗留代码桥接到slf4j上，然后再引入Slf4j的API包（slf4j-api-xxx.jar）、Log4j2的绑定包（log4j-slf4j-impl-xxx.jar）和Log4j2的日志实现包（log4j-core-xxx.jar）。

注意：不同的适配器工作原理不同，被适配的日志框架并不是一定要删除的，比如上面说的这个log4j-over-slf4j这个适配器的工作原理是：内部提供了和log4j一模一样的api接口，因此在程序调用的时候，必须想办法让其走适配器的api，所以如果不删除log4j，只需要保证其在classpath中的顺序比log4j靠前就可以了。

> 为了保险起见，最好还是把原来的日志框架删除掉



### Case 3 遗留业务应用，一个模块使用log4j作为日志框架，其他第三方包中使用log4j2，现在想要统一日志框架为slf4j+log4j2，如何统一输出

其他第三方包中已经使用log4j2了，那么就不需要管了，千万不能引入log4j2的桥接包，不然会出现循环调用最终导致栈溢出。其余的就和case2一样了

