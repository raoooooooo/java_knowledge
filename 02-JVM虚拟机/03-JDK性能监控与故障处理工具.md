# JDK性能监控与故障处理工具

> 💡 **为什么重要？**
> - 这是Java程序员的"工具箱"，出问题时得知道拿什么工具干活
> - 面试高频考点："CPU飙高怎么排查？"、"OOM怎么排查？"
> - 工作必备技能：天天都要用这些工具

---

## 一、JDK工具总览

JDK的bin目录下有很多exe，这些就是我们的"武器库"：

| 工具分类 | 工具名 | 主要作用 |
|---------|--------|---------|
| **基础工具** | javac、java、jar、javap | 编译、运行、反编译 |
| **进程监控** | jps | 查看Java进程（类似Linux的ps） |
| **运行时统计** | jstat | 查看JVM运行时数据（GC、内存、类加载） |
| **配置信息** | jinfo | 查看和修改JVM参数 |
| **内存快照** | jmap | 生成堆转储快照（heap dump） |
| **堆分析** | jhat | 分析heap dump文件（已过时，用MAT代替） |
| **线程快照** | jstack | 生成线程快照（看线程状态、死锁） |
| **可视化** | jconsole、jvisualvm | 图形化监控工具 |

---

## 二、jps——查看Java进程 ⭐⭐⭐

**全称**：JVM Process Status Tool

**作用**：列出正在运行的Java进程，显示进程ID（PID）、主类名等。

> 💡 这是最常用的工具！排查问题第一步：先找到Java进程的PID！

---

### 基础用法

```bash
jps [选项]
```

### 常用参数

| 参数 | 作用 |
|------|------|
| **-l** | 显示主类的完整包名，或jar的完整路径 |
| **-v** | 显示JVM启动参数 |
| **-m** | 显示main方法的入参 |
| **-q** | 只显示PID，不显示其他信息 |

---

### 实际示例

#### 示例1：基础用法 `jps`
```bash
C:\> jps
12345 App
67890 Jps
```
- 12345：Java进程PID
- App：主类名
- Jps：jps命令本身也是个Java进程

#### 示例2：显示完整类名 `jps -l`
```bash
C:\> jps -l
12345 com.xxx.App
67890 sun.tools.jps.Jps
```
> ✅ 最常用！多个Java进程时能清楚区分是哪个应用

#### 示例3：显示JVM参数 `jps -v`
```bash
C:\> jps -v
12345 App -Xms2g -Xmx2g -XX:+UseG1GC
```
> 能看到这个JVM用了什么GC、堆大小等参数

---

### 面试/工作高频用法

```bash
# 最常用：找Java进程PID
jps -l

# 不知道PID？先jps，再用后面的工具
jps -l
jstat -gc 12345
```

---

## 三、jstat——JVM运行时统计 ⭐⭐⭐

**全称**：JVM Statistics Monitoring Tool

**作用**：实时监控JVM各种运行数据：GC情况、内存使用、类加载、编译情况等。

> 💡 最常用的功能是看GC情况！怀疑有GC问题时，第一个用的就是它！

---

### 基础用法

```bash
jstat [选项] <PID> [间隔时间 毫秒] [查询次数]
```

**说明**：
- 间隔时间：每隔多少毫秒输出一次
- 查询次数：一共输出多少次
- 不写的话只输出一次

---

### 最常用参数（GC相关）⭐⭐⭐

| 参数 | 作用 |
|------|------|
| **-gc** | 查看GC堆详细情况（最常用！） |
| **-gcutil** | GC汇总（百分比显示，更直观） |
| **-gccause** | 同gcutil，但额外显示最后一次GC的原因 |
| **-gccapacity** | 显示堆各区域的容量 |
| **-class** | 类加载统计 |
| **-compiler** | JIT编译统计 |

---

### 详细示例与输出解读

#### 示例1：查看GC汇总 `jstat -gcutil <PID>`

**这是最常用的命令！**

```bash
C:\> jstat -gcutil 12345
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  50.23  75.89  30.45  92.31  88.56   100    0.567     5    0.234    0.801
```

**逐个字段解释（重点！）**：

| 列名 | 含义 | 说明 |
|------|------|------|
| **S0** | Survivor 0区使用率 | 百分比，0.00表示空的 |
| **S1** | Survivor 1区使用率 | 百分比，50.23%表示用了一半 |
| **E** | Eden区使用率 | 75.89%表示Eden快满了，快Young GC了 |
| **O** | 老年代使用率 | 30.45%表示老年代还很空 |
| **M** | 元空间使用率 | 92.31%表示元空间快满了 |
| **CCS** | 压缩类空间使用率 | JDK 8+有 |
| **YGC** | Young GC次数 | 100次，系统运行以来发生了100次Young GC |
| **YGCT** | Young GC总耗时 | 0.567秒，100次GC总共花了0.567秒 |
| **FGC** | Full GC次数 | 5次，Full GC发生了5次 |
| **FGCT** | Full GC总耗时 | 0.234秒，5次Full GC花了0.234秒 |
| **GCT** | 所有GC总耗时 | 0.801秒 = YGCT + FGCT |

**怎么看有没有问题？**
- ✅ 正常情况：YGC频繁但每次快，FGC次数少
- ⚠️ 异常1：FGC次数快速增加 → 老年代满了，快OOM了
- ⚠️ 异常2：FGCT特别高 → Full GC太慢，STW时间长
- ⚠️ 异常3：O列接近100% → 老年代快满了

---

#### 示例2：连续监控GC情况

```bash
# 每1秒输出一次，连续输出10次
jstat -gcutil 12345 1000 10
```

**效果**：每秒刷新一次GC情况，能看到GC的变化趋势
- 观察E是不是一直在涨，涨到接近100%就会触发YGC
- 观察YGC后E是不是清零了，S0/S1是不是涨了
- 观察老年代O是不是一直在涨，如果是可能有内存泄漏

---

#### 示例3：查看GC详细信息 `jstat -gc <PID>`

```bash
C:\> jstat -gc 12345
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
1024.0 1024.0  0.0   512.3  8192.0   6213.4   20480.0     6236.2   4096.0 3788.5 512.0  453.4    100   0.567     5    0.234    0.801
```

**字段解释**（C=Capacity容量，U=Used已用）：

| 列名 | 含义 | 单位 |
|------|------|------|
| **S0C** | Survivor 0区容量 | KB |
| **S1C** | Survivor 1区容量 | KB |
| **S0U** | Survivor 0区已用 | KB |
| **S1U** | Survivor 1区已用 | KB |
| **EC** | Eden区容量 | KB |
| **EU** | Eden区已用 | KB |
| **OC** | 老年代容量 | KB |
| **OU** | 老年代已用 | KB |
| **MC** | 元空间容量 | KB |
| **MU** | 元空间已用 | KB |

---

#### 示例4：查看最后一次GC原因 `jstat -gccause <PID>`

```bash
C:\> jstat -gccause 12345
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC
  0.00  50.23  75.89  30.45  92.31  88.56   100    0.567     5    0.234    0.801 Allocation Failure   No GC
```

**额外字段**：
- **LGCC**（Last GC Cause）：最后一次GC原因
  - `Allocation Failure`：分配失败（最常见，就是Eden满了）
  - `Metadata GC Threshold`：元空间满了
  - `System.gc()`：有人调用了System.gc()
- **GCC**（Current GC Cause）：当前GC原因，No GC表示没在GC

---

### 工作中排查GC问题的步骤

```bash
# 第一步：先看GC汇总，有没有问题
jstat -gcutil 12345

# 第二步：如果FGC频繁，连续监控看趋势
jstat -gcutil 12345 1000  # 每秒刷一次，Ctrl+C停止

# 第三步：看每次GC的原因
jstat -gccause 12345

# 第四步：如果老年代一直在涨 → 可能内存泄漏 → 用jmap导出dump分析
```

---

## 四、jinfo——查看和修改JVM参数 ⭐⭐

**全称**：JVM Configuration Info

**作用**：查看JVM的配置参数，还可以**运行时修改**部分参数（不需要重启）。

---

### 基础用法

```bash
jinfo [选项] <PID>
```

### 常用参数

| 参数 | 作用 |
|------|------|
| **-flags** | 查看所有JVM参数 |
| **-sysprops** | 查看系统属性（System.getProperties()） |
| **-flag <name>** | 查看某个具体参数的值 |
| **-flag [+|-]<name>** | 开启/关闭布尔类型参数 |
| **-flag <name>=<value>** | 设置参数值 |

---

### 实际示例

#### 示例1：查看所有JVM参数
```bash
jinfo -flags 12345
```
输出：
```
Non-default VM flags: -XX:InitialHeapSize=2147483648 -XX:MaxHeapSize=2147483648 -XX:+UseG1GC
```
> 能看到堆大小、用了什么GC等所有参数

#### 示例2：查看某个具体参数
```bash
# 查看最大堆
jinfo -flag MaxHeapSize 12345
-XX:MaxHeapSize=2147483648

# 查看用了什么GC
jinfo -flag UseG1GC 12345
-XX:+UseG1GC  # +号表示开启了
```

#### 示例3：运行时修改参数（不需要重启！）
```bash
# 开启GC日志
jinfo -flag +PrintGC 12345

# 关闭GC日志
jinfo -flag -PrintGC 12345

# 设置参数值
jinfo -flag HeapDumpBeforeFullGC=true 12345
```

> ⚠️ 注意：不是所有参数都能运行时修改！只有标注了 `manageable` 的参数才能改。
>
> 查看哪些参数可以改：`java -XX:+PrintFlagsFinal -version | grep manageable`

---

## 五、jmap——生成堆转储快照 ⭐⭐⭐

**全称**：Memory Map for Java

**作用**：生成堆转储文件（heap dump），还可以查看堆内存信息、对象统计。

> 💡 出OOM时、怀疑内存泄漏时，必须用jmap导出dump，然后用MAT分析！

---

### 基础用法

```bash
jmap [选项] <PID>
```

### 常用参数

| 参数 | 作用 |
|------|------|
| **-dump** | 生成堆转储快照（最常用！） |
| **-histo** | 显示堆中对象统计信息（类、实例数、占用大小） |
| **-heap** | 显示堆详细信息（用了什么GC、参数配置、分代情况） |
| **-finalizerinfo** | 显示在F-Queue中等待Finalizer的对象 |
| **-clstats** | 显示类加载器统计 |

---

### 实际示例

#### 示例1：导出堆转储文件（最常用！）⭐⭐⭐

```bash
jmap -dump:format=b,file=heap.hprof 12345
```

**参数解释**：
- `format=b`：二进制格式，固定写法
- `file=heap.hprof`：输出文件名，建议用 `.hprof` 后缀
- 12345：进程PID

**输出**：
```
Dumping heap to C:\heap.hprof ...
Heap dump file created
```

> ✅ 这个 `heap.hprof` 文件就可以用 MAT（Memory Analyzer Tool）分析了！

**常用技巧：OOM时自动导出dump**
```bash
# JVM启动时加上这个参数，OOM时自动导出dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heap.hprof
```
> 💡 生产环境一定要加上这个参数！OOM后就能拿到第一现场了。

---

#### 示例2：查看对象统计 `jmap -histo <PID>`

```bash
jmap -histo 12345 | head -20  # Windows用 | Select-Object -First 20
```

输出：
```
 num     #instances         #bytes  class name
----------------------------------------------
   1:        123456       15804544  [C          # char[]数组，字符串对象
   2:         87654        2103696  java.lang.String
   3:         45678        1096272  java.util.HashMap$Node
   4:         34567         829608  java.lang.reflect.Method
   5:         23456         562944  java.util.HashMap
```

**字段解释**：
- **#instances**：实例数量
- **#bytes**：占用字节数
- **class name**：类名，`[C`=char[]，`[B`=byte[]，`[I`=int[]

**怎么看？**
- 看有没有某个业务类的实例数量异常多
- 看String、HashMap这些是不是异常多
- 快速排查哪个对象占用内存最多

---

#### 示例3：查看堆详细信息 `jmap -heap <PID>`

```bash
jmap -heap 12345
```

输出会显示：
- 用了什么垃圾收集器
- 堆配置参数（Xms、Xmx等）
- 新生代、老年代容量和使用情况
- 每个代的使用率

---

### 工作中使用场景

```bash
# 场景1：OOM了！赶紧导出dump分析
jmap -dump:format=b,file=oom_$(date +%Y%m%d_%H%M).hprof 12345

# 场景2：怀疑内存泄漏，先看看对象统计
jmap -histo 12345 | grep com.xxx  # 只看自己业务的类

# 场景3：看看堆内存分配情况
jmap -heap 12345
```

---

## 六、jstack——生成线程快照 ⭐⭐⭐

**全称**：Java Stack Trace

**作用**：生成JVM的线程快照（thread dump），查看每个线程的执行状态、调用栈。

> 💡 排查什么问题用？
> - CPU飙高
> - 死锁
> - 线程死循环
> - 线程卡住不响应

---

### 基础用法

```bash
jstack [选项] <PID>
```

### 常用参数

| 参数 | 作用 |
|------|------|
| **-l** | 显示锁的额外信息（死锁时必须加！） |
| **-e** | 显示更多扩展信息 |
| **-F** | 线程卡住没响应时，强制输出线程栈 |

---

### 线程状态说明

jstack输出中每个线程都有状态：

| 线程状态 | 说明 |
|---------|------|
| **RUNNABLE** | 运行中，正在跑Java代码或执行native方法 |
| **BLOCKED** | 阻塞，等待锁（synchronized没拿到锁） |
| **WAITING** | 无限等待，等待另一个线程的特定操作（Object.wait()） |
| **TIMED_WAITING** | 超时等待，指定时间后自动唤醒（Thread.sleep()） |
| **TERMINATED** | 已终止，线程执行完了 |

---

### 实际示例

#### 示例1：查看所有线程栈

```bash
jstack 12345 > thread_dump.txt  # 输出到文件，方便查看
```

输出片段：
```
"main" #1 prio=5 os_prio=0 tid=0x00000000034c4000 nid=0x3b24 runnable [0x00000000034bf000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at com.xxx.App.main(App.java:42)

   Locked ownable synchronizers:
        - None
```

**逐行解释**：
- `"main"`：线程名
- `prio=5`：线程优先级
- `tid`：JVM内的线程ID
- `nid`：操作系统原生线程ID
- `runnable`：线程状态
- 下面就是调用栈，能看到线程正在执行哪一行代码
- `Locked ownable synchronizers`：持有了哪些锁

---

#### 示例2：查看死锁（必加 -l 参数！）

```bash
jstack -l 12345
```

如果有死锁，最后会输出：
```
Found one Java-level deadlock:
=============================
"线程A":
  waiting to lock monitor 0x0000000012345678 (object 0x00000000e0000001, a java.lang.Object),
  which is held by "线程B"

"线程B":
  waiting to lock monitor 0x0000000012345679 (object 0x00000000e0000002, a java.lang.Object),
  which is held by "线程A"

Java stack information for the threads listed above:
===================================================
... 两个线程的调用栈 ...
```

> ✅ jstack能自动检测死锁！排查死锁就靠它了！

---

### 🔥 经典案例：CPU飙高排查步骤

这是面试必问题！"CPU飙高了怎么排查？"

```bash
# 第一步：找到Java进程PID
jps -l
# 假设 PID = 12345

# 第二步：看哪个线程占用CPU高（Linux用top -H -p 12345，Windows用任务管理器）
top -H -p 12345
# 假设找到 CPU 最高的线程ID是 678（十进制）

# 第三步：把线程ID转成十六进制（jstack里的nid是十六进制）
printf "%x\n" 678
# 输出 2a6

# 第四步：用jstack看这个线程在干嘛
jstack 12345 | grep -A 20 2a6

# 结果：就能看到这个高CPU线程的调用栈，知道在哪行代码死循环了！
```

> 💡 Windows没有top命令怎么办？可以用Process Explorer看线程CPU，或者用Java代码自己写个监控。

---

## 七、jhat——堆转储分析工具（了解即可）

**全称**：Java Heap Analysis Tool

**作用**：分析jmap生成的heap dump文件，自带一个Web服务器，可以在浏览器里看分析结果。

### 用法

```bash
jhat heap.hprof
```

然后访问 http://localhost:7000 就能看分析结果。

> ⚠️ **注意**：现在已经没人用jhat了！
> - 功能太弱，不好用
> - 分析大堆时自己会OOM
> - **推荐用 MAT（Memory Analyzer Tool）** 或 JVisualVM
> - IDEA 也可以直接打开 .hprof 文件分析

---

## 八、jconsole——可视化监控工具 ⭐⭐

**作用**：JDK自带的图形化JVM监控工具，能看：
- 内存使用情况（实时曲线图）
- 线程情况
- 类加载情况
- GC情况
- MBean管理

### 启动

直接在命令行输入：
```bash
jconsole
```

然后选择要监控的Java进程，图形化界面就出来了！

**界面说明**：
- **概览**：内存、线程、类、CPU使用率四个曲线图
- **内存**：堆、老年代、新生代、元空间的使用情况
- **线程**：所有线程列表，能看每个线程的栈，能检测死锁
- **类**：已加载类数量
- **VM摘要**：JVM信息、参数
- **MBean**：管理JMX Bean

> 💡 适合开发环境用，看实时情况很方便。生产环境一般不用，因为要开JMX端口，有安全风险。

---

## 九、jvisualvm——全能可视化工具 ⭐⭐⭐

**全称**：Java VisualVM

**作用**：JDK自带的最强大的可视化工具！集成了上面所有命令行工具的功能：
- 监控CPU、内存、线程
- 生成和分析heap dump
- 生成和分析thread dump
- 还有很多插件可以扩展功能

### 启动

```bash
jvisualvm
```

**主要功能**：
1. **监视**：CPU、堆、元空间、线程数的实时曲线图
2. **线程**：查看所有线程，能检测死锁
3. **堆Dump**：点一下按钮就生成，还能直接分析
4. **抽样器**：CPU抽样（看哪个方法占用CPU）、内存抽样（看哪个类对象最多）
5. **Profiler**：性能分析，更详细的CPU和内存分析

> 💡 **开发环境强烈推荐！** 比jconsole好用10倍，功能强大还免费！
>
> ⚠️ JDK 9之后jvisualvm不再默认自带，需要单独下载。

---

## 十、JMC——Java Mission Control ⭐⭐

**定位**：Oracle出品的更高级的性能分析工具，以前是商业软件，现在开源了。

**核心功能**：
1. **JFR（Java Flight Recorder）**：Java飞行记录器
   - 超低开销（<1%），生产环境可以一直开着
   - 记录JVM的所有运行数据
   - 出问题后回放分析，像"黑匣子"一样
2. **JMC客户端**：分析JFR记录的文件

**JFR开启方式**：
```bash
# 开启JFR，启动JVM时加参数
-XX:+UnlockCommercialFeatures -XX:+FlightRecorder

# 运行时开启记录
jcmd <PID> JFR.start duration=60s filename=recording.jfr
```

> 💡 JFR是现在最牛逼的Java性能分析工具！生产环境也能开，几乎不影响性能，出问题后能回溯分析。JDK 11之后开源免费了。

---

## 十一、jcmd——多功能命令（JDK 7+）

**作用**：JDK 7之后新增的"万金油"命令，可以代替上面的大部分命令。

### 常用功能

```bash
# 查看所有Java进程（代替jps）
jcmd

# 导出heap dump（代替jmap）
jcmd <PID> GC.heap_dump heap.hprof

# 查看GC情况（代替jstat）
jcmd <PID> GC.class_histogram

# 导出线程栈（代替jstack）
jcmd <PID> Thread.print

# 执行GC
jcmd <PID> GC.run

# 查看JVM参数（代替jinfo）
jcmd <PID> VM.flags
```

> 💡 趋势：jcmd是官方推荐的，慢慢会代替jps、jmap、jstack这些工具。

---

## 十二、生产环境故障排查流程总结 ⭐⭐⭐

### 场景1：CPU飙高
```bash
1. jps -l                    # 找到Java进程PID
2. top -H -p <PID>           # 找到CPU最高的线程ID
3. printf "%x\n" <线程ID>    # 转十六进制
4. jstack <PID> | grep -A 20 <十六进制ID>  # 看这个线程在干嘛
```

### 场景2：OOM / 内存泄漏
```bash
1. jmap -dump:format=b,file=oom.hprof <PID>  # 导出dump
2. 用 MAT / JVisualVM / IDEA 打开dump分析
3. 看哪个对象最多，谁引用了它，找到泄漏点
```

### 场景3：死锁
```bash
1. jstack -l <PID>  # 直接输出，最后会自动检测死锁
```

### 场景4：系统响应慢
```bash
1. jstat -gcutil <PID> 1000  # 看是不是GC太频繁
2. jstack <PID>              # 看线程是不是都卡住了
3. jmap -histo <PID>         # 看是不是对象太多了
```

---

## 十三、常见面试题

### Q1：CPU飙高了怎么排查？（必问！）

**参考答案**：
```
1. 先用jps找到Java进程PID
2. 用top -H -p <PID>找到占用CPU最高的线程ID
3. 把线程ID转成十六进制（jstack里的nid是十六进制）
4. 用jstack <PID> | grep -A 20 <十六进制ID> 查看线程栈
5. 根据调用栈找到是哪行代码在死循环或大量计算
```

### Q2：OOM了怎么排查？

**参考答案**：
```
1. 首先JVM启动参数要加 -XX:+HeapDumpOnOutOfMemoryError，OOM时自动导出dump
2. 如果没加，就用jmap -dump:format=b,file=heap.hprof <PID>手动导出
3. 用MAT（Memory Analyzer Tool）或IDEA打开dump文件
4. 分析：
   - 看Histogram：哪个类的实例最多、占内存最大
   - 看Dominator Tree：哪些大对象被谁引用着
   - 看Leak Suspects Report：MAT自动检测的内存泄漏嫌疑点
5. 找到是哪个对象没有被释放，顺藤摸瓜找到代码问题
```

### Q3：怎么检测死锁？

**参考答案**：
```
用 jstack -l <PID> 命令，jstack会自动检测死锁并在最后输出死锁报告，
显示哪个线程持有了什么锁，又在等什么锁，还会输出两个线程的调用栈。
```

### Q4：jmap和jhat有什么区别？现在还常用吗？

**参考答案**：
```
- jmap：用来导出堆转储文件（heap dump）
- jhat：用来分析heap dump文件，自带Web服务器

现在jhat已经很少用了，功能太弱，分析大堆时自己还会OOM。
推荐用 MAT（Memory Analyzer Tool）、JVisualVM 或 IDEA 来分析dump文件。
```

### Q5：JFR是什么？有什么优势？

**参考答案**：
```
JFR（Java Flight Recorder）是JVM内置的"黑匣子"：
- 性能开销极低（<1%），生产环境可以一直开着
- 记录JVM的所有运行数据：GC、线程、锁、内存、方法执行等
- 出问题后可以回溯分析，还原事故现场
- JDK 11及之后开源免费，是目前最强大的Java性能分析工具
```

---

## 十四、工具选择指南

| 场景 | 首选工具 | 次选工具 |
|------|---------|---------|
| 找Java进程 | jps / jcmd | 任务管理器 |
| 看GC情况 | jstat -gcutil | jvisualvm |
| 看JVM参数 | jinfo -flags | jcmd |
| 导出heap dump | jmap / jcmd | jvisualvm |
| 分析heap dump | MAT / IDEA | jvisualvm |
| 导出thread dump | jstack / jcmd | jvisualvm |
| 死锁检测 | jstack -l | jvisualvm / jconsole |
| 实时监控 | jvisualvm / JMC | jconsole |
| 生产性能分析 | JFR | 无替代 |
| CPU飙高排查 | top + jstack | jvisualvm抽样器 |

---

> 💡 **最后建议**：这些工具不用死记硬背，知道有什么工具、能解决什么问题就行，用的时候查文档。但是**CPU飙高排查步骤、OOM排查步骤**这两个必须背下来，面试必问！
