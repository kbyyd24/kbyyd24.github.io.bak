---
title: 使用 Gradle 但不使用 Java 插件构建 Java 项目
tags:
  - gradle
  - java
date: 2020-02-22 23:42:35
updated: 2020-02-22 23:42:35
---


本文目标是探索在没有使用任何额外插件的情况下，如何使用 `Gradle` 构建一个 `Java` 项目，以此对比使用 `Java` 插件时得到的好处。

# 初始化项目

使用 `Gradle Init` 插件提供的 `init` task 来创建一个 `Gradle` 项目：

```shell
gradle init --type basic --dsl groovy --project-name gradle-demo
```

运行完成后，我们将得到这些文件：

```shell
❯ tree
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── settings.gradle
```

接下来，我们将关注点放到 `build.gradle` 上面，这是接下来编写构建脚本的地方。

# Hello World

首先，我们编写一个 `Java` 的 HelloWorld，做为业务代码的代表：

```java
public class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello Wrold");
  }
}
```

然后，将这个内容保存到 `src/HelloWorld.java` 文件中，不按照 `maven` 的约定来组织项目结构。

# 编译 Java

接着，我们需要给我们的构建脚本添加任务来编译刚才写的 `Java` 文件。这里就需要使用到 `Task`。关于 `Task	`，`Gradle` 上有比较详细的文档描述如何使用它：https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task & https://docs.gradle.org/current/userguide/more_about_tasks.html。

现在，我们可以创建一个 `JavaCompile` 类型的 `Task` 对象，命名为 `compileJava` ：

```gradle
task compileJava(type: JavaCompile) {
  source fileTree("$projectDir/src")
  include "**/*.java"
  destinationDir = file("${buildDir}/classes")
  sourceCompatibility = '1.8'
  targetCompatibility = '1.8'
  classpath = files("${buildDir}/classes")
}
```

在上面的代码中，我们：

1. 通过 `source` & `include` 方法指定了要被编译的文件所在的目录和文件的扩展名
2. 通过 `destinationDir` 指定了编译后的 `class` 文件的存放目录
3. 通过 `sourceCompatibility` & `targetCompatibility` 指定了源码的 `Java` 版本和 `class` 文件的版本
4. 通过 `classpath` 指定了编译时使用的 `classpath`

那么，接下来我们就可以执行 `compileJava` 这个任务了：

```bash
❯ gradle compileJava
❯ tree build
build
├── classes
│   └── HelloWorld.class
└── tmp
    └── compileJava
❯ cd build/classes
❯ java HelloWorld
Hello World
```

我们可以看到，HelloWorld 已经编译成功，并且可以被正确执行。

# 添加第三方依赖

在实际的项目中，难免会使用到其他人开发的库。要使用别人开发的库，就需要添加依赖。在 `Gradle` 中添加依赖，需要做这样四个事情：

1. 申明 `repository`
2. 定义 `configuration`
3. 申明 `dependency`
4. 将 `dependency` 添加到 `classpath`

## 申明 `repository`

在 `Gradle` 中可以定义项目在哪些 `repository` 中寻找依赖，通过 `dependencies` 语法块申明：

```gradle
repositories {
  mavenCentral()
  maven {
    url 'https://maven.springframework.org/release'
  }
}
```

因为 `mavenCentral` 和 `jcenter` 是比较常见的两个仓库，所以 `Gradle` 提供了函数可以直接使用。而其他的仓库则需要自己指定仓库的地址。

申明了 `repository` 之后，`Gradle` 才会知道在哪里寻找申明的依赖。

## 定义 `configuration`

如果你使用过 `maven` 的话，也许 `repository` 和 `dependency` 都能理解，但对 `configuration` 却可能感到陌生。

`Configuration` 是一组为了完成一个具体目标的依赖的集合。那些需要使用依赖的地方，比如 `Task`，应该使用 `configuration`，而不是直接使用依赖。这个概念仅在依赖管理范围内适用。

`Configuration` 还可以扩展其他 `configuration`，被扩展的 `configuration` 中的依赖，都将被传递到扩展的 `configuration` 中。

我们可以来创建给 HelloWorld 程序使用的 `configuration` ：

```gradle
configurations {
  forHelloWorld
}
```

定义 `configuration` 仅仅需要定义名字，不需要进行其他配置。如果需要扩展，可以使用 `extendsFrom` 方法：

```gradle
configurations {
  testHelloWorld.extendsFrom forHelloWorld
}
```

## 申明 `dependency`

申明 `dependency` 需要使用到上一步的 `configuration`，将依赖关联到一个 `configuration` 中：

```gradle
dependencies {
  forHelloWorld 'com.google.guava:guava:28.2-jre'
}
```

通过这样的申明，在 `forHelloWorld` 这个 `configuration` 中就存在了 `guava` 这个依赖。

## 将 `dependency` 添加到 `classpath`

接下来，我们就需要将 `guava` 这个依赖添加到 `compileJava` 这个 `task` 的 `classpath` 中，这样我们在代码中使用的 `guava` 提供的代码就能在编译期被 `JVM` 识别到。

但就像在[定义 `configuration`](#定义-configuration) 中描述的那样，我们需要消费 `configuration` 以达到使用依赖的目的，而不能直接使用依赖。所以我们需要将 `compileJava.classpath` 修改成下面这样：

```gradle
classpath = files("${buildDir}/classes", configurations.forHelloWorld)
```

## 修改 HelloWorld

完成上面四步之后，我们就可以在我们的代码中使用 `guava` 的代码了：

```java
import com.google.common.collect.ImmutableMap;
public class HelloWrold {
  public static void main(String[] args) {
    ImmutableMap.of("Hello", "World")
        .forEach((key, value) -> System.out.println(key + " " + value));
  }
}
```

# 打包

前面已经了解过如何进行编译，接着我们来看看如何打包。

`Java` 打包好之后，往往有两种类型的 `Jar`：

1. 一种是普通的 `Jar`，里面不包含自己的依赖，而是在 `Jar` 文件外的一个 `metadata` 文件申明依赖，比如 `maven` 中的 `pom.xml`
2. 另一种被称作 `fatJar` (or `uberJar`) ，里面已经包含了所有的运行时需要的 `class` 文件和 `resource` 文件。

## 创建普通的 `Jar` 文件

在这个练习中，我们就只关注 `Jar` 本身，不关心 `metadata` 文件。

在这里，我们自然是要创建一个 `task`，类型就使用 `Jar` ：

```gradle
tasks.create('jar', Jar)

jar {
  archiveBaseName = 'base-name'
  archiveAppendix = 'appendix'
  archiveVersion = '0.0.1'
  from compileJava.outputs
  include "**/*.class"
  manifest {
    attributes("something": "value")
  }
  setDestinationDir file("$buildDir/lib")
}
```

在这个例子中，我们：

1. 指定了 `archiveBaseName`, `archiveAppendix`, `archiveVersion` 属性，他们和 `archiveClassfier`, `archiveExtension` 将决定最后打包好的 `jar` 文件名
2. 使用 `from` 方法，指定要从 `compileJava` 的输出中拷贝文件，这样就隐式的添加了 `jar` 对 `compileJava` 的依赖
3. 使用 `include` 要求仅复制 `class` 文件
4. 可以使用 `manifest` 给 `META-INF/MANIFEST.MF` 文件添加信息
5. `setDestinationDir` 方法已经被标记为 `deprecated` 但没有替代的方法

接着，我们就可以使用 `jar` 进行打包：

```shell
❯ gradle jar
❯ tree build
build
├── classes
│   └── HelloWorld.class
├── lib
│   └── base-name-appendix-0.0.1.jar
└── tmp
    ├── compileJava
    └── jar
        └── MANIFEST.MF
        ❯ zipinfo build/lib/base-name-appendix-0.0.1.jar
❯ zipinfo build/lib/base-name-appendix-0.0.1.jar
Archive:  build/lib/base-name-appendix-0.0.1.jar
Zip file size: 1165 bytes, number of entries: 3
drwxr-xr-x  2.0 unx        0 b- defN 20-Feb-22 23:14 META-INF/
-rw-r--r--  2.0 unx       43 b- defN 20-Feb-22 23:14 META-INF/MANIFEST.MF
-rw-r--r--  2.0 unx     1635 b- defN 20-Feb-22 23:14 HelloWorld.class
3 files, 1678 bytes uncompressed, 825 bytes compressed:  50.8%
```

## 创建 `fatJar`

接着，同样使用 `Jar` 这个类型，我们创建一个 `fatJar` 任务：

```gradle
task('fatJar', type: Jar) {
  archiveBaseName = 'base-name'
  archiveAppendix = 'appendix'
  archiveVersion = '0.0.1'
  archiveClassifier = 'boot'
  from compileJava
  from configurations.forHelloWorld.collect {
    it.isDirectory() ? it : zipTree(it)
  }
  manifest {
    attributes "Main-Class": "HelloWorld"
  }
  setDestinationDir file("$buildDir/lib")
}
```

相比于 `jar`，我们的配置变更在于：

1. 添加 `archiveClassfier` 以区别 `fatJar` 和 `jar` 产生的不同 `jar` 文件
2. 使用 `from` 将 `forHelloWorld` `configuration` 的依赖全部解压后拷贝到 `jar` 文件
3. 指定 `Main-Class` 属性，以便直接运行 `jar` 文件

然后我们再执行 `fatJar` :

```shell
❯ gradle fatJar
❯ tree build
build
├── classes
│   └── HelloWorld.class
├── lib
│   ├── base-name-appendix-0.0.1-boot.jar
│   └── base-name-appendix-0.0.1.jar
└── tmp
    ├── compileJava
    ├── fatJar
    │   └── MANIFEST.MF
    └── jar
        └── MANIFEST.MF
❯ java -jar build/lib/base-name-appendix-0.0.1-boot.jar
Hello World
```

# 总结

通过练习在不使用 `Java Plugin` 的情况下，使用 `Gradle` 来构建项目，实现了编译源码、依赖管理和打包的功能，并得到了如下完整的 `gradle.build` 文件：

```gradle
repositories {
  mavenCentral()
}

configurations {
  forHelloWorld
}

dependencies {
  forHelloWorld group: 'com.google.guava', name: 'guava', version: '28.2-jre'
}

task compileJava(type: JavaCompile) {
  source fileTree("$projectDir/src")
  include "**/*.java"
  destinationDir = file("${buildDir}/classes")
  sourceCompatibility = '1.8'
  targetCompatibility = '1.8'
  classpath = files("${buildDir}/classes", configurations.forHelloWorld)
}

compileJava.doLast {
  println 'compile success!'
}

tasks.create('jar', Jar)

jar {
  archiveBaseName = 'base-name'
  archiveAppendix = 'appendix'
  archiveVersion = '0.0.1'
  from compileJava.outputs
  include "**/*.class"
  manifest {
    attributes("something": "value")
  }
  setDestinationDir file("$buildDir/lib")
}

task('fatJar', type: Jar) {
  archiveBaseName = 'base-name'
  archiveAppendix = 'appendix'
  archiveVersion = '0.0.1'
  archiveClassifier = 'boot'
  from compileJava
  from configurations.forHelloWorld.collect {
    it.isDirectory() ? it : zipTree(it)
  }
  manifest {
    attributes "Main-Class": "HelloWorld"
  }
  setDestinationDir file("$buildDir/lib")
}
```

写了这么多构建脚本，仅仅完成了 `Java Plugin` 提供的一小点功能，伤害太明显。

完整的例子可以在这个 `repo` 的 `commit` 中找到 https://github.com/kbyyd24/gradle-practice