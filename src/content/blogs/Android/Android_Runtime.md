---
title: Android Runtime
date: 2025-6-20
category: 安卓
tags: [Android, ART]
readTime: 5
excerpt: 这篇文章将与你深入探讨安卓运行时(ART)的基本结构与一些与传统JVM的异同点
reference: [
  { description: "学习整理来自小码蜂的专栏", link: "https://juejin.cn/column/7507835962337820682" },
]
---
# 一. Android Runtime（ART）架构概述

## 1.ART在Android系统中的定位
ART处于Android系统架构的核心层，向上对接应用框架层（如Activity、Service等组件），向下依赖Linux内核提供的内存管理、线程调度等基础服务。其核心职责包括：
1. **字节码执行**：解析和执行DEX（Dalvik Executable）字节码
2. **内存管理**：负责对象的分配、回收与内存优化
3. **类加载**：管理Java类的加载、验证与初始化
4. **JNI交互**：实现Java代码与Native代码的双向通信
5. **安全机制**：确保应用运行的安全性与隔离性

## 2.ART架构概述
ART的架构主要包括以下组件：
1. 类加载器系统：负责加载和验证类
2. 执行引擎：包括解释器和JIT编译器
3. 垃圾回收器：管理内存分配和回收
4. 调试和分析工具：支持应用调试和性能分析
5. 运行时服务：提供线程管理、反射等功能

## 3.ART架构设计目标
ART的设计遵循以下核心目标：
1. **高性能**：通过AOT编译、JIT即时优化提升执行效率
2. **低内存占用**：采用分代垃圾回收、指针压缩等技术优化内存
3. **兼容性**：确保与Java语言规范及Android应用的兼容性
4. **安全性**：通过沙箱机制隔离应用，防止恶意攻击
5. **可扩展性**：支持插件化、热修复等动态特性

# 二.ART初始化

## 1.Zygote进程

### Zygote进程的核心作用
Zygote进程是Android系统中的一个特殊进程，它是所有应用进程的父进程。在Android系统启动过程中，Zygote进程率先被创建并初始化，之后每当系统需要创建一个新的应用进程时，都会通过复制Zygote进程来实现，这种机制极大地提高了应用启动速度。 Zygote进程的核心作用包括：

1. 预加载和初始化Java运行时环境（包括ART虚拟机）
2. 预加载常用的Java类和资源
3. 作为应用进程的模板，通过fork()机制快速创建新的应用进程

Zygote进程的启动与ART的初始化紧密相关。在Zygote进程启动过程中，会完成ART虚拟机的初始化工作，包括内存管理系统、类加载系统、垃圾回收器等组件的初始化。ART虚拟机的高效运行是Zygote进程能够快速创建应用进程的基础。 Zygote进程在启动时会加载并初始化核心Java类库，这些类库会被映射到Zygote进程的内存空间中。当通过fork()创建新的应用进程时，这些预加载的类库和资源可以被新进程共享，无需再次加载，从而显著提高应用启动速度。

### Zygote进程的启动
Zygote进程的启动发生在Android系统启动的早期阶段。具体来说，当Linux内核启动完成后，会首先创建init进程，这是Android系统中的第一个用户空间进程。init进程会解析init.rc配置文件，并启动一系列重要的系统服务，其中就包括Zygote进程。 Zygote进程的启动是Android系统启动过程中的关键步骤，它的成功启动标志着系统Java运行时环境的准备就绪，为后续系统服务和应用的启动奠定了基础。

## 2.Linux内核启动与init进程

### Linux内核启动
Android系统的启动始于Linux内核的加载和初始化。当设备上电后，引导程序（如Bootloader）会加载Linux内核镜像到内存中，并将控制权交给内核。
Linux内核的启动流程大致如下：

1. 初始化CPU和内存子系统
2. 检测和初始化硬件设备
3. 挂载根文件系统
4. 执行第一个用户空间进程（通常是init进程）

在内核启动过程中，会执行一系列的初始化操作，包括设置中断处理、内存管理、设备驱动等。这些操作确保了系统硬件的正常工作，并为后续用户空间进程的运行提供了基础环境。

### init进程的创建
当Linux内核完成初始化后，会创建init进程（进程ID为1）。init进程是Android系统中所有用户空间进程的祖先，它负责启动和管理系统中的其他进程和服务。
init进程的主要作用包括：

1. 解析和执行init.rc配置文件
2. 启动系统关键服务（如Zygote、ServiceManager等）
3. 监控和管理系统服务的生命周期
4. 处理系统关机和重启操作

init进程使用的是Android特有的Init语言编写的配置文件，这些文件定义了系统服务的启动顺序、依赖关系和运行参数等信息。

## 3.Zygote进程启动流程

### 初始化与参数配置
Zygote进程的启动由app_process可执行文件触发。app_process是一个用C++编写的程序，位于/system/bin目录下。它的主要作用是初始化Java虚拟机环境，并启动Zygote进程的主类。在启动Zygote进程时，app_process会接收一系列命令行参数，这些参数定义了Zygote进程的启动模式和行为

### 启动Java虚拟机
在解析完命令行参数后，app_process会调用JNI（Java Native Interface）函数来初始化Java虚拟机（JVM）。在Android系统中，这个Java虚拟机实际上是ART（Android Runtime）。 初始化ART的关键步骤包括：

1. 创建Java虚拟机实例
2. 设置Java虚拟机参数（如堆大小、垃圾回收器类型等）
3. 注册JNI方法
4. 加载核心Java类库

### 调用ZygoteIniit.main()方法
一旦Java虚拟机初始化完成，app_process会调用Zygote的主类com.android.internal.os.ZygoteInit的main()方法。这个方法是Zygote进程的入口点，负责完成Zygote进程的剩余初始化工作。 ZygoteInit.main()方法的主要工作包括：

1. 创建并初始化Zygote服务器套接字
2. 预加载Java类和资源
3. 启动系统服务
4. 等待并处理来自系统的创建应用进程请求

## 小结
先暂停一下，我们对上述过程进行一个简单的梳理，当Android系统启动时先初始化Linux内核，接下来执行init进程，init进程中包含了Zygote进程的启动规则，然后启动Zygote进程，先配置启动参数，如Zygote进程标识，初始化完成后启动系统服务等，接下来Zygote进程去创建java虚拟机也就是ART，最后执行ZygoteInit.main()方法完成初始化，要注意其中的依赖关系，类比一下：
1. app_process 相当于 “启动器”：类似双击.exe 文件启动程序，app_process 被执行后启动 Zygote 进程。
2. Zygote 进程相当于 “容器”：容器启动后，在内部搭建 Java 虚拟机环境（如 ART），为运行 Java 代码做准备。

核心逻辑：app_process 是启动 Zygote 的工具，而 Zygote 是创建 Java 虚拟机的主体。  因此依赖关系为：

1. Zygote 进程由 app_process 程序创建：init 进程通过执行 app_process 可执行文件，创建 Zygote 进程。
2. Java 虚拟机由 Zygote 进程创建：Zygote 进程在启动时，通过代码逻辑初始化 ART/Dalvik 虚拟机

## 4.ART初始化
### Runtime初始化
在Zygote进程启动过程中，ART的初始化始于Runtime类的创建和初始化。Runtime类是ART的核心类，负责管理整个运行时环境。  
Runtime类的初始化主要包括以下步骤：

1. 解析命令行参数和环境变量
2. 初始化内存分配器
3. 设置线程管理系统
4. 初始化垃圾回收器
5. 注册JNI方法

### 类加载系统初始化
类加载系统是ART的重要组成部分，负责查找、加载和验证Java类。在Zygote进程中，类加载系统的初始化确保了核心Java类库能够被正确加载和使用。 类加载系统的初始化主要包括：

1. 创建类加载器实例
2. 初始化类加载路径
3. 注册类解析器
4. 预加载核心类

### 垃圾回收器初始化
垃圾回收器（GC）负责自动管理Java对象的内存分配和回收。在Zygote进程中，垃圾回收器的初始化设置了GC的类型、参数和工作模式。 
ART支持多种垃圾回收器，包括：
1. 标记-清除（Mark-Sweep）
2. 标记-整理（Mark-Compact）
3. G1（Garbage-First）

垃圾回收器的初始化主要包括：
1. 根据系统配置选择合适的GC类型
2. 设置GC参数（如堆大小、并发线程数等）
3. 初始化GC数据结构
4. 注册GC回调函数

### JNI环境初始化
JNI（Java Native Interface）允许Java代码与本地代码（如C/C++）进行交互。在Zygote进程中，JNI环境的初始化确保了Java代码能够正确调用本地方法。 JNI环境初始化主要包括：

1. 创建JNIEnv实例
2. 注册本地方法
3. 初始化JNI函数表
4. 设置JNI异常处理机制

## 5.创建应用进程

### 通过Zygote创建应用进程的原理
在Android系统中，应用进程是通过Zygote进程的fork()机制创建的。这种机制利用了Linux操作系统的fork()系统调用的特性，允许一个进程快速复制自身，形成一个新的子进程。当系统需要启动一个应用时，ActivityManagerService会向Zygote进程发送一个创建新进程的请求。Zygote进程接收到请求后，会调用fork()系统调用创建一个新的进程，然后在新进程中加载并启动应用的主Activity。

### 应用进程创建的详细流程
应用进程创建的详细流程如下：

1. **请求发起**：当用户点击应用图标或通过其他方式启动应用时，ActivityManagerService会向Zygote进程发送创建应用进程的请求。
2. **请求接收**：Zygote进程通过服务器套接字接收请求，并解析请求参数。
3. **进程创建**：Zygote进程调用fork()系统调用创建新的进程。由于fork()是写时复制（Copy-On-Write）的，新进程可以快速复制Zygote进程的内存空间，而不需要实际复制所有数据。
4. **环境初始化**：在新进程中，Zygote会执行一些初始化操作，如重置信号处理、设置进程名称等。
5. **应用加载**：新进程加载应用的APK文件，并初始化应用的主Activity。
6. **应用启动**：新进程调用应用的主Activity的onCreate()和onStart()方法，启动应用。

### 进程创建后的ART环境调整
虽然应用进程继承了Zygote进程的ART环境，但在创建后会进行一些特定的调整，以适应应用的需求：

1. **类加载器设置**：应用进程会创建自己的类加载器，用于加载应用的类和资源
2. **内存分配调整**：根据应用的需求，调整堆大小和其他内存分配参数
3. **垃圾回收策略调整**：根据应用的特性，调整垃圾回收器的参数和工作模式
4. **调试和分析工具配置**：如果应用处于调试模式，会配置相应的调试和分析工具

# 三. 类加载机制

## 1.dex文件

### 1.概述
Dex（Dalvik Executable）文件格式是专为Android平台设计的一种字节码格式，它起源于早期的Dalvik虚拟机时代。在Android系统发展初期，传统的Java字节码（.class文件）在移动设备上执行效率较低，因为移动设备资源有限，需要更高效的执行方式。为了解决这个问题，Google开发了Dalvik虚拟机，并设计了Dex文件格式，它将多个Java类文件整合到一个单一文件中，减少了文件I/O开销和内存占用，更适合移动设备的特性。Dex文件的主要作用是作为Android应用的可执行文件格式。当开发者编写Java或Kotlin代码并编译后，最终会生成Dex文件，这些Dex文件被打包到APK（Android Package）文件中。在应用安装时，Dex文件会被安装到设备上，并在应用运行时由Android Runtime（ART）或Dalvik虚拟机加载和执行。Dex文件中包含了应用的所有代码逻辑、资源引用等信息，是Android应用运行的核心载体 。

### 2.Dex文件与Class文件的关系
Dex文件与Java Class文件都是字节码格式，但它们在结构和设计上有很大差异。Class文件是Java虚拟机（JVM）的执行格式，每个Class文件对应一个Java类或接口，包含了该类的字节码、常量池、方法信息等。而Dex文件则是将多个Class文件整合在一起，消除了冗余信息，优化了存储结构，更适合在移动设备上使用。从转换关系来看，Java源代码首先被编译成Class文件，然后通过dx工具（Android SDK中的一个工具）将多个Class文件转换为一个或多个Dex文件。在转换过程中，dx工具会进行一系列优化，如合并常量池、优化方法调用等，使得Dex文件更加紧凑和高效。例如，多个Class文件中可能存在相同的字符串常量，在转换为Dex文件时，这些重复的常量会被合并为一个，减少了文件大小 。

## 2.MultiDex多文件处理

### 1.概述
**主Dex文件与次级Dex文件**
在MultiDex机制中，Dex文件分为主Dex文件和次级Dex文件。主Dex文件是应用启动时首先加载的文件，它包含了应用启动所必需的核心代码，如应用入口类（继承自Application的类）、Android四大组件（Activity、Service、BroadcastReceiver、ContentProvider）的基础类等 。这些代码是应用启动和运行的基础，必须确保在应用启动时能够快速加载和执行。次级Dex文件则存储了应用的其他代码，如业务逻辑类、第三方库代码等。这些代码在应用启动时并不需要立即加载，而是在后续使用到相关功能时，由MultiDex机制动态加载。次级Dex文件的存在，使得应用能够突破单个Dex文件的方法数限制，容纳更多的代码 。

**类加载器的作用**
在MultiDex机制中，类加载器起着关键作用。Android提供了两种重要的类加载器用于处理MultiDex：PathClassLoader和DexClassLoader。PathClassLoader用于加载已经安装到设备上的应用的Dex文件，它只能加载指定目录下的Dex文件，并且这些Dex文件必须位于系统默认的应用安装路径下。在MultiDex中，PathClassLoader主要用于加载主Dex文件。DexClassLoader则更加灵活，它可以加载任意目录下的Dex文件，包括外部存储目录。在MultiDex机制中，DexClassLoader用于加载次级Dex文件。通过DexClassLoader，应用能够在运行时动态加载次级Dex文件中的类，实现代码的动态扩展 。

**MultiDex与Android Runtime的关系**
MultiDex机制与Android Runtime（ART或Dalvik）紧密协作。在应用启动时，Android Runtime首先通过PathClassLoader加载主Dex文件，并在主Dex文件中查找应用的入口类，开始执行应用的启动逻辑。当应用在运行过程中需要使用次级Dex文件中的类时，Android Runtime会借助DexClassLoader加载次级Dex文件。加载完成后，Android Runtime会将次级Dex文件中的类信息整合到运行时环境中，确保这些类能够被正确访问和调用 。同时，Android Runtime还需要处理多个Dex文件之间的类引用关系，保证代码执行的正确性和一致性 。

### 2.MultiDex的配置与构建过程
在Android项目的build.gradle文件中，需要进行相关配置以启用MultiDex功能。首先，在android闭包内的defaultConfig中添加multiDexEnabled true，这告诉Gradle在构建过程中启用MultiDex 。
```Groovy
android {    
    defaultConfig {    
        multiDexEnabled true    
    }    
}  
```
此外，还需要添加MultiDex库的依赖。在较新的Android项目中，通常使用androidx.multidex:multidex库，在dependencies闭包中添加如下依赖：
```Groovy 
dependencies {    
    implementation 'androidx.multidex:multidex:2.0.1'    
}  
```
这些配置是启用MultiDex的基础，它们指导Gradle在构建应用时进行代码拆分和Dex文件生成 。

### 3.主Dex文件的加载过程

**应用启动时的加载流程**
当用户点击应用图标启动应用时，Android系统首先会找到应用的APK文件，并解压其中的内容。系统会使用PathClassLoader来加载主Dex文件（classes.dex） PathClassLoader会从APK文件的根目录下读取classes.dex文件，并将其映射到内存中。在映射过程中，PathClassLoader会验证Dex文件的完整性，确保文件没有损坏 。如果验证通过，PathClassLoader会开始解析Dex文件的头部信息，获取文件的基本信息和各个数据区域的偏移量。接下来，PathClassLoader会根据Dex文件的结构，依次加载和解析字符串池、类型表、方法表等索引表和数据段，建立类的元数据信息 。当主Dex文件中的所有类都被加载和解析完成后，PathClassLoader会在主Dex文件中查找应用的入口类（继承自Application的类） 。

**Android Runtime对主Dex的处理**
Android Runtime（ART或Dalvik）在PathClassLoader加载完主Dex文件后，会接手后续处理。ART首先会对主Dex文件中的类进行验证，确保类的字节码符合语法和安全规范。验证通过后，ART会为每个类分配内存空间，并初始化类的静态字段。对于包含静态代码块的类，ART会执行静态代码块中的代码，完成类的初始化工作 。同时，ART会构建类的继承关系和方法调用关系，建立类的运行时数据结构。在完成类的初始化后，ART会调用应用入口类的onCreate()方法，开始执行应用的启动逻辑。此时，应用已经成功加载了主Dex文件中的核心代码，具备了基本的运行环境 。

**核心类的优先加载**
在主Dex文件的加载过程中，核心类会被优先加载。这些核心类包括应用入口类、Android四大组件类、与系统交互的关键类（如Context类及其子类）等 。这些类是应用启动和运行的基础，必须在应用启动时立即可用。为了确保核心类被放入主Dex文件，MultiDex.install()方法会自动分析类的依赖关系，将与核心类有直接或间接依赖关系的类也一并放入主Dex文件 。这样可以避免在应用启动时出现类找不到的问题，保证应用能够快速、稳定地启动 。同时，优先加载核心类也有助于提高应用的启动性能，减少用户等待时间

### 4.次级Dex文件的加载机制

**加载触发条件**
次级Dex文件的加载并非在应用启动时立即进行，而是在满足特定触发条件时才会发生。当应用在运行过程中需要使用次级Dex文件中的类时，就会触发次级Dex文件的加载。例如，当用户点击应用中的某个按钮，触发了一个位于次级Dex文件中的Activity时，系统会检测到该Activity类尚未加载，从而触发次级Dex文件的加载 。又或者，应用在运行过程中调用了一个位于次级Dex文件中的第三方库方法，也会触发次级Dex文件的加载 。此外，动态代码加载（如使用反射加载次级Dex文件中的类）同样会触发次级Dex文件的加载 。

**类加载的整合与管理**
当DexClassLoader成功加载次级Dex文件中的类后，需要将这些类整合到应用的运行时环境中。Android Runtime会建立新加载类与已加载类之间的引用关系，确保不同Dex文件中的类能够相互调用。为了管理多个Dex文件中的类，Android Runtime维护了一个类加载器链（双亲委派模型）。主Dex文件由PathClassLoader加载，次级Dex文件由DexClassLoader加载，DexClassLoader的父类加载器是PathClassLoader 。当应用请求加载一个类时，类加载器链会按照顺序依次尝试加载类 。首先，DexClassLoader会尝试在次级Dex文件中查找类；如果找不到，会将请求传递给父类加载器PathClassLoader，由PathClassLoader在主Dex文件中查找类 。这种类加载器链的管理方式，保证了应用能够正确加载和使用多个Dex文件中的类，避免了类冲突和找不到类的问题 。

### 5.类加载冲突与解决策略

**冲突产生的原因**
在MultiDex机制下，类加载冲突可能会因为多种原因产生。一种常见的原因是多个Dex文件中存在同名的类 。例如，应用引入了两个不同版本的第三方库，这两个库中可能包含同名但实现不同的类 。当应用在运行过程中同时使用到这两个库时，就会出现类加载冲突。另一个原因是类加载顺序问题。由于主Dex文件和次级Dex文件由不同的类加载器加载，且次级Dex文件在应用运行时才加载，如果在次级Dex文件加载之前，应用已经使用了主Dex文件中同名的类，那么当次级Dex文件加载后，可能会导致类的行为不符合预期 。此外，动态代码加载和反射的使用也可能引发类加载冲突，因为它们可能会绕过正常的类加载顺序 。

**冲突检测机制**
为了检测类加载冲突，Android Runtime在类加载过程中会进行一系列检查。当DexClassLoader加载次级Dex文件中的类时，会检查该类是否已经被其他类加载器加载过 。如果发现类已经被加载，Android Runtime会比较两个类的来源（即所属的Dex文件）。如果两个类来自不同的Dex文件，Android Runtime会判断是否存在冲突。判断的依据包括类的包名、类名以及类的字节码内容等 。如果发现冲突，Android Runtime会抛出ClassNotFoundException或NoClassDefFoundError等异常，提示开发者存在类加载冲突问题。此外，开发者也可以通过日志输出和调试工具来辅助检测类加载冲突。在关键的类加载位置添加日志输出，记录类的加载来源和加载顺序，通过分析日志可以定位到类加载冲突的具体位置 。

**解决冲突的策略**
解决类加载冲突的策略有多种。一种策略是统一第三方库的版本，避免引入多个版本的库导致同名类冲突 。开发者可以在项目的build.gradle文件中，通过implementation或api依赖声明，指定统一的第三方库版本。另一种策略是使用ProGuard或R8等工具进行代码混淆。混淆工具可以对类名、方法名和字段名进行重命名，降低不同库中同名类冲突的概率 。在build.gradle文件中配置ProGuard或R8规则，例如：
```Groovy 
buildTypes {    
    release {    
        minifyEnabled true    
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'    
    }    
}  
```
此外，开发者还可以通过自定义类加载器来解决类加载冲突问题。自定义类加载器可以实现特定的类加载逻辑，例如优先加载某个Dex文件中的类，或者根据类的来源进行不同的处理 。通过合理运用这些策略，可以有效解决MultiDex机制下的类加载冲突问题，保证应用的稳定运行 。

## 3.ART中的双亲委派模型

### 1.类加载器的继承关系
Android的类加载器之间存在明确的继承关系 。BootClassLoader是所有类加载器的根，它没有父类加载器 。PathClassLoader和DexClassLoader都继承自BaseDexClassLoader，而BaseDexClassLoader又继承自ClassLoader类 。这种继承关系为双亲委托模型的实现提供了基础，子类加载器可以通过父类引用调用父类的加载方法，实现委托加载 。
![](/img/Android/img.png)

### 2.与Java类加载器体系的异同

Android的类加载器体系与Java标准的类加载器体系既有相似之处，也存在差异 。相似之处在于都采用了双亲委托模型，类加载器之间形成层次结构，核心类由顶层类加载器加载 。不同之处在于，Java标准体系中的顶层类加载器是Bootstrap ClassLoader（由C++实现），接下来是Extension ClassLoader和Application ClassLoader 。而Android中没有Extension ClassLoader，取而代之的是PathClassLoader和DexClassLoader，并且Android使用Dex文件格式而非Java的Class文件格式，这导致类加载的具体实现有所不同 。

### 3.三种类加载器

**1.BootClassLoader**
BootClassLoader是Android类加载器体系中的最顶层类加载器，它负责加载系统核心类库 。BootClassLoader的类加载路径通常是/system/framework目录下的核心类库文件，这些文件包含了Java和Android的基础类 。用于加载Android Framework层的class文件。

**2.PathClassLoader**
PathClassLoader是Android应用最常用的类加载器，用于加载已安装应用的Dex文件 。在实现双亲委托模型时，PathClassLoader通过其父类BaseDexClassLoader继承了ClassLoader的loadClass方法。当PathClassLoader收到类加载请求时，首先会检查该类是否已被加载，如果未加载，则将请求委托给父类加载器（通常是BootClassLoader） 。只有当父类加载器无法加载该类时，PathClassLoader才会尝试从自己的Dex文件路径中查找并加载 。

**3.DexClassLoader**
DexClassLoader与PathClassLoader类似，同样继承自BaseDexClassLoader，因此也遵循相同的双亲委托模型 。不同的是，DexClassLoader支持从任意路径加载Dex文件，适用于动态加载场景 。当DexClassLoader收到类加载请求时，同样会先委托给父类加载器，只有在父类加载器无法加载时，才会从自己指定的路径中查找类 。

## 4.小结
首先明确一个关系，DexClassLoader与PathClassLoader是平级关系**，**他们没有继承关系，而在双亲委派模型中，DexClassLoader加载类时是否会委托 PathClassLoader，取决于你是否将 PathClassLoader 设置为 parent。  
Android 的类加载器体系是基于 JVM 的双亲委派模型的改造版。双亲委派的大原则：  
classLoader.loadClass(className)    
→ **先调用 parent.loadClass(className)**  → 如果 parent 找不到才调用 findClass(className)。

DexClassLoader 会委托给它构造时传入的 **parent ClassLoader**。  这个 parent 可以是任何 ClassLoader，包括：  
PathClassLoader（这是最常见的，也是我们平时说“DexClassLoader 会先去问 PathClassLoader”的原因）  
也可以是 BootClassLoader  
甚至可以是 null（极端情况，不推荐）

# 四.类加载

ART的类加载机制与标准JVM相同，也是加载，链接，初始化，链接又分为验证，准备，解析三个阶段，这里不进行赘述。

# 五.内存管理

## Alloc Space

对应JVM的堆，存储对象实例，由GC管理，ART的堆内存架构由多个核心组件协同工作：
- 内存分配器
- 垃圾回收器
- 内存映射管理器

## Image Space（映像空间）
存储预编译的类元信息、字符串等，来自 boot.art。存储 ART 启动时预加载的核心类定义和它们对应的元数据结构。是只读、共享的内存区域，多个 ART 进程可以共享这部分内存，节省系统资源。与下面的Class Table不同的是，这里只保存系统将预编译好的运行时核心类（如 java.lang.\*、android.\* 等）及其元数据，相当于只保存核心类的元数据。

## Zygote Space
存储系统初始化时预加载类与资源，用于所有 app 共享，**Zygote Space 是 ART 中属于 Heap（Java 堆）的一部分，专门用于存放在 Zygote 启动阶段分配的 Java 对象**，这些对象会被所有 app 进程通过 fork Zygote 时继承，达到共享内存、节省资源的目的。

## Native Space（JNI内存）

对应JVM的本地方法栈，由 native 层代码管理，如 C++ 对象、JNI 分配的内存

## Thread Stack

对应JVM的虚拟机栈，每个线程独立的栈，存储局部变量、方法调用信息等

## Intern Table（字符串驻留表）

类似于字符串常量池

## Class Table（类表）

对应元空间（在写的时候没搞清元空间与方法区的关系，再说一下，方法区是一个规范，规定了这个区域要存放类的元信息，而元空间是JVM的具体实现），维护当前运行时所有已加载类的元数据引用，是类的索引和快速访问结构，是类元数据存放和管理区。 再附加一条，类的静态方法和普通方法信息存储在方法区，堆中的对象实例只存储方法表的引用。

## Code Space

**再谈编译模式**
主要保存.oat文件，再详细说一下ART相关的编译，在应用安装时，不对所有的 .dex进行完全AOT，而是进行“最小 AOT + 字节码验证 + dex2oat 生成 .vdex”，AOT 编译的目标是核心类库（boot classpath 中的类）和一部分系统代码。应用自身代码 **主要还是解释执行**，等运行后再通过 JIT 编译热点方法。接下来在启动时ART会启动JIT编译器，记录热点代码，而热点代码会在运行时被编译成机器码，但是这种热点编译只放在内存中，机器码是不落盘的，但是ART会将“哪些代码是热点”的行为数据（profile）落盘，而在设备充电+空闲时，系统会调用dex2oat 工具将这些 **高频热点代码** **AOT 编译为 .odex 或 .oat 本地代码，生成可持久的机器码**，用于下次启动时直接使用。这就是 Android 的Profile-Guided Compilation（PGC，配置文件引导编译）机制，这也是为什么ATR的编译模式是JIT，AOT与配置文件混合编译

**Code Space**
他是 ART 虚拟机运行期间用于执行机器码的区域来源包括 AOT 编译的 .oat 文件 和 JIT 编译生成的 native 代码块。 应用通过fork获取自己的ART实例时使用 mmap 将 .odex/.oat 映射为“只读+可执行”的内存段（r-xp）并将其存放于ART 进程内的 Code Space。 JIT编译出的机器码都保存在Code Space中，因为Code Space是应用进程（应用ART实例）的一部分，所以应用退出之后JIT机器码也会消失。

## 小结
整体来说ART内存区域与JVM差不多，也是虚拟机栈，本地方法栈，程序计数器，方法区还有堆空间，但是ART多出了一块存放.oat文件的Code Space。在细节上Zygote Space（Zygote Heap）存放可共享的核心类的实例对象，Alloc Space存放每个应用自己的类对象(new分配出来的对象)，这两个都是堆的一部分。Image Space存放可共享的核心类的元信息，Class Table存放每个应用自己的类的元信息，这两个都是方法区的一部分。

# 六.垃圾回收（GC）

ART使用分代回收CMS，感觉流程和CMS一样，具体也不赘述了。

## GC抑制

### ART中触发GC的原因

1. **kGcCauseForAlloc** Alloc GC, 是指线程分配对象失败触发GC, 一般是因为堆内存触顶导致的，这时候GC会直接在分配对象线程直接执行。
2. **kGcCauseBackground** Background GC是虚拟机为了保证较低的内存使用水位，分配对象时候能有足够的内存而主动触发的GC，这种类型在后台线程（HeapTaskDaemon) 执行。App 在后台运行，系统空闲时主动触发 GC。
3. **kGcCauseForNativeAlloc** 当应用求分配大量Native内存时，如果检测到内存不足可能会触发这个类型的GC，用来释放一些被Java对象引用的Native对象。
4. **kGcCauseExplicit** 代码中直接调用System.gc()显式执行的，取决于我们自己的应用代码。
5. **kGcCauseZygoteFork** Zygote 进程在 fork 应用之前清理内存。
![](/img/Android/img_1.png)
我们着重去看下**kGcCauseBackground**这种Cause怎么触发的，同时也分析一下kGcCauseForAlloc这种类型。

### 对象分配流程
  ```C++
inline mirror::Object* Heap::AllocObjectWithAllocator(Thread* self,
                                                      ObjPtr<mirror::Class> klass,
                                                      size_t byte_count,
                                                      AllocatorType allocator,
                                                      const PreFenceVisitor& pre_fence_visitor) {

      //1. 直接分配对象
      obj = TryToAllocate<kInstrumented, false>(self, allocator, byte_count, &bytes_allocated,
                                                &usable_size, &bytes_tl_bulk_allocated);
      if (UNLIKELY(obj == nullptr)) {
        //2. 分配失败，执行AllocateInternalWithGc, 里面会执行 等上次GC完成后分配、尝试几种不同程度的GC后分配、扩容分配、回收软引用并扩容分配、最终失败抛出OOM等逻辑
        obj = AllocateInternalWithGc(self,
                                     allocator,
                                     kInstrumented,
                                     byte_count,
                                     &bytes_allocated,
                                     &usable_size,
                                     &bytes_tl_bulk_allocated,
                                     &klass);

      }
      //...

          if ( IsGcConcurrent()) {
        //3. 对象分配完成之后，检查当前所使用内存是否达到并发GC出发阈值，如果达到则触发并发GC
            CheckConcurrentGC(self, new_num_bytes_allocated, &obj);
           }            


}
```
### kGcCauseForAlloc
#### Alloc GC（Allocation GC）
Alloc GC 指的是在尝试**分配内存**时，发现当前堆空间不足，于是触发的一次**紧急垃圾回收**。
**🔧 触发场景：**
应用尝试分配新的对象内存（new 操作）
发现没有足够的可用空间
ART 会先尝试执行一次 GC，以**释放堆内存**
如果 GC 后仍然没有足够空间，就会抛出 OutOfMemoryError

#### Blocking GC（阻塞式 GC）
Blocking GC 是指**需要暂停应用线程（即 Stop-The-World）**的垃圾回收操作。
**🔧 特点：**
所有的 GC（如 Alloc GC, Full GC, Zygote GC）中只要存在 "stop-the-world" 行为（暂停所有 Java 线程），就属于 **Blocking GC**。
通常用于：
内存分配失败（如上面的 Alloc GC）
内存使用率达到某个阈值
系统主动触发（例如切换 Activity 时触发）

#### 小结
对于GC抑制的目标而言，Alloc GC其实给我们两点启示：
**Alloc GC是不能进行抑制的，因为他在即将OOM时候触发，没法再抑制了**
**如果我们抑制后台GC的话，一定有个限度不能太久，在主线程触发Alloc GC反而适得其反**

### Background GC（kGcCauseBackground）
在分配完成一个对象之后会检查当前堆上分配的对象大小new_num_bytes_allocated是否超过并发GC阈值，如果超过则执行RequestConcurrentGCAndSaveObject，去提交并发GC任务。

# 七.结语
最后感觉....ART其实和JVM就差在对.dex的处理上，在一个小安卓系统上不断的提高复用率，来提升性能，在GC方面还有GC抑制的相关内容，主要是在用户做滑动等操作时，不允许GC，不然GC带来的STW会造成卡顿，给用户不好的体验。