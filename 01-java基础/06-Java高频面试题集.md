# Java 高频面试题集

> 📌 **一句话理解**：本文是 Java 后端面试**速查题集**，聚焦 Java 基础 / 并发 / JVM 三大核心，每题给「要点答案 + 易错点/勘误」。与 `01-java基础`、`02-JVM虚拟机` 的深度知识点详解互补--那里讲原理，这里查题目。
>
> 整理依据：两份参考资料（java面试宝典、咕泡大厂高频面试题）。已按官方文档与业界共识**逐题纠错**（铁律3），资料中的错误就地修正并在该题 `⚠️` 行说明，文末「资料勘误与重点提醒」集中汇总。

---

## 第一章 Java 基础

### Q：JDK、JRE、JVM 的关系是什么？Java 如何实现跨平台？
- JVM（Java 虚拟机）是运行字节码的抽象计算机，不同平台有各自的 JVM 实现
- JRE = JVM + 核心类库（如 java.lang），是运行 Java 程序的最小环境
- JDK = JRE + 开发工具（javac、jar 等），是开发所需的完整环境
- 跨平台原理：源码经 javac 编译成平台无关的字节码（.class），再由各平台 JVM 解释/JIT 执行。"一次编译到处运行"靠 JVM 屏蔽 OS 差异
> ⚠️ 易错点：跨平台靠的是 JVM 而非 Java 语言本身；字节码只面向 JVM，不面向任何特定处理器

### Q：什么是字节码？采用字节码的好处？
- 字节码是源码编译后生成的 `.class` 中的虚拟指令，不面向特定处理器，只面向 JVM
- 好处：兼顾执行效率与可移植性。JVM 解释执行 + JIT 编译热点为本地码，比纯解释型快；又不绑定具体机器，可跨平台
- 执行链路：源码 -> 编译器 -> 字节码 -> JVM（解释器 + JIT）-> 机器码 -> 运行

### Q：Java 有哪些基本数据类型？switch 支持哪些类型？
- 8 大基本类型：byte(1)、short(2)、int(4)、long(8)、float(4)、double(8)、char(2)、boolean(JVM 无明确规定)
- 引用类型：类、接口、数组、枚举（Java 5+）
- switch 支持：byte、short、char、int 及其包装类、枚举（Java 5+）、String（Java 7+）
- switch **不支持** long、float、double、boolean
> ⚠️ 易错点：String 不是基本数据类型；switch 不支持 long/float/double/boolean 是常考陷阱

### Q：自动装箱与拆箱是什么？Integer a=127 与 Integer b=127 相等吗？
- 装箱：基本类型 -> 包装类型，编译期插入 `Integer.valueOf()`
- 拆箱：包装类型 -> 基本类型，编译期插入 `Integer.intValue()`
- `Integer a = 127; Integer b = 127;` 时 `a == b` 为 **true**（命中缓存）
- `Integer a = 128; Integer b = 128;` 时 `a == b` 为 **false**（超范围 new 新对象）
- `new Integer(127)` 永远创建新对象，不经过缓存
> ⚠️ 勘误：资料称缓存对象"直接引用常量池中的 Integer 对象"，不准确。Integer 缓存是内部私有静态内部类 `IntegerCache`（一个数组），**不是字符串常量池**。默认范围 [-128, 127] 闭区间，上限可由 `-XX:AutoBoxCacheMax=n` 调整。Byte/Short/Long/Character 也有类似缓存

### Q：short s1=1; s1=s1+1 有错吗？s1+=1 呢？float f=3.4 正确吗？Math.round(11.5) 和 Math.round(-11.5) 各等于多少？
- `s1 = s1 + 1`：**有错**。s1+1 运算时 s1 提升为 int，赋给 short 需强转
- `s1 += 1`：**没错**。复合赋值隐含强转，等价 `s1 = (short)(s1 + 1)`
- `float f = 3.4`：**不正确**。3.4 默认是 double 字面量，需写 `3.4f` 或 `(float)3.4`
- `Math.round(11.5)` = 12，`Math.round(-11.5)` = -11。原理 `floor(x + 0.5)`，正数四舍五入，负数 .5 向大取整
> ⚠️ 易错点：Math.round(-11.5) = -11 而非 -12，因为 floor(-11.0) = -11

### Q：final、finally、finalize 有什么区别？
- **final**：修饰符。修饰类->不可继承；修饰方法->不可重写；修饰变量->引用不可变（指向的对象内容仍可变）
- **finally**：异常处理块，无论 try/catch 是否抛异常都执行，用于资源释放（JDK 7+ 推荐 try-with-resources）
- **finalize**：Object 类方法，GC 回收对象前调用。Java 9 起标记 `@Deprecated`，已计划移除
> ⚠️ 勘误：资料称 finalize 是"对象是否可回收的最后判断"，不准确。它只是 GC 回收前的回调钩子，不可靠且影响性能，官方明确不推荐。final 修饰变量锁的是引用，对象内部字段仍可修改

### Q：static 关键字的意义与用法？静态方法内为何不能调用非静态成员？
- static 修饰的成员属于类而非实例，类加载时分配内存，所有实例共享一份
- 用法：修饰成员变量、成员方法、代码块、内部类、静态导包
- 静态方法内不能调用非静态成员：静态方法在对象创建之前就已存在，调用时可能没有任何实例，非静态成员依赖实例
- 非静态方法可访问静态成员；static 方法不能用 this/super
> ⚠️ 易错点：static 块只在类初次加载时执行一次；静态方法不可被重写（可被隐藏 hiding，但不构成多态）

### Q：this 和 super 的区别与用法？
- this：指向当前对象本身，区分成员变量与局部变量同名、调用本类其他构造器（`this(...)` 必须首行）
- super：访问父类被隐藏的成员、调用父类构造器（`super(...)` 必须首行）
- this() 与 super() 不能同时出现在同一构造器；都不能在 static 环境使用
- 子类构造器未显式调用则编译器自动插入无参 super()
> ⚠️ 易错点：super 不是父类对象的引用，而是编译期关键字，指示 JVM 查找父类成员；子类对象内存中只有一份父类字段，不存在独立的"父类对象"

### Q：& 和 && 的区别？
- `&`：按位与（整型）/逻辑与（布尔），逻辑与**不短路**，两边都求值
- `&&`：短路与，左为 false 时右侧直接跳过
- `|` 和 `||` 类似：`|` 不短路，`||` 短路
> ⚠️ 易错点：短路特性在 `a != null && a.size() > 0` 判空中很关键，用 `&` 会因右侧对 null 解引用抛 NPE

### Q：面向对象三大特性是什么？多态的实现机制是什么？
- 封装：隐藏内部实现，对外提供公共访问方式
- 继承：子类复用父类非 private 成员并可扩展，是运行时多态的前提。Java 类单继承、接口可多继承
- 多态：父类引用指向子类对象，运行时根据实际类型动态绑定方法。必要条件：继承 + 重写 + 向上转型
- 实现机制：基于动态分派，JVM 通过方法表（vtable）在运行期根据对象实际类型确定调用方法版本
> ⚠️ 勘误：资料将重载称为"编译时多态（前绑定）"。严格说重载是**静态分派**，并非 OOP 真正的多态。Java 真正的多态指运行时多态（动态绑定），重写（override）才是其核心

### Q：重载（Overload）和重写（Override）的区别？构造器能否被重写？
- 重载：同一类中，方法名相同、参数列表不同，与返回值和访问修饰符无关，编译期决定（静态分派）
- 重写：父子类间，方法签名相同，返回值/异常更窄、访问修饰符更宽（里氏替换）；private/static/final 方法不可重写
- 构造器**不能被重写**（不可被继承），但**可以被重载**
> ⚠️ 易错点：重载不能仅靠返回值区分；static 方法可被隐藏（hiding）但不是重写，不参与动态绑定

### Q：接口和抽象类有什么区别？
- 定义：抽象类用 abstract 声明，接口用 interface 声明；抽象类可有构造器，接口**不能有构造器**
- 继承：类单继承抽象类，可实现多个接口
- 成员：抽象类字段任意；接口字段默认 `public static final`（接口**无实例变量**）
- 方法：抽象类方法修饰符任意；接口方法默认 public（Java 8+ default/static，Java 9+ private）
- 设计层面：抽象类是模板设计（is-a），接口是行为规范（can-do）
> ⚠️ 勘误：资料称"接口中的实例变量默认 final"，概念错误。接口**不存在实例变量**，所有字段都是隐式 `public static final` 的常量

### Q：== 和 equals 的区别是什么？
- `==`：基本类型比较值，引用类型比较内存地址（是否同一对象）
- `equals`：Object 默认等价 `==`；String/Integer 等已重写为比较内容
- 自定义类若需"内容相等"语义，必须重写 equals
> ⚠️ 易错点：`42 == 42.0` 为 true（基本类型自动类型提升后比值）；`new String("a").equals(new String("a"))` 为 true，但 `==` 为 false

### Q：为什么重写 equals 时必须重写 hashCode？两者有什么关系？
- 约定：equals 相等则 hashCode 必须相等；hashCode 相等则 equals 不一定相等
- HashSet/HashMap 先用 hashCode 定位桶，再用 equals 判断是否相同。只重写 equals 不重写 hashCode，内容相等的对象可能落在不同桶，导致重复存储
- 默认 hashCode 基于内存地址，内容相同 hashCode 也不同
> ⚠️ 易错点：hashCode 相同不代表 equals 相等（哈希冲突）；对象做 HashMap key 时必须同时重写两者

### Q：Java 是值传递还是引用传递？
- Java **只有值传递**，没有引用传递
- 基本类型：传值的副本，方法内修改不影响原变量
- 引用类型：传引用的**副本**（地址值拷贝），通过引用修改对象属性影响原对象，但让引用指向新对象不影响调用方
- 关键反例：方法内交换两个引用参数，调用方引用不变--证明传的是副本
> ⚠️ 勘误："对象引用是按值传递的"是准确说法。C/C++ 的引用传递是传变量地址本身，方法内可改变调用方变量指向，Java 不支持这种语义

### Q：Java 内部类有哪几种？匿名内部类访问局部变量为何要加 final？
- 四种：成员内部类、静态内部类、局部内部类、匿名内部类
- 静态内部类：只访问外部类静态成员，`new Outer.Inner()`
- 成员内部类：可访问外部类所有成员，依赖外部实例，`outer.new Inner()`
- 局部内部类：定义在方法内，作用域限于方法
- 匿名内部类：必须继承一个类或实现接口，不能定义静态成员
- **访问局部变量要 final 的原因**：内部类捕获的是变量的**副本**，为保证副本与原变量一致，变量不可被重新赋值
> ⚠️ 勘误：资料用"生命周期不一致"解释是早期简化说法。Java 8+ 只要求"effectively final"（事实不可变），不必显式写 final。本质是闭包按值捕获

### Q：什么是反射？优缺点和应用场景？获取 Class 的三种方式？
- 反射：运行时动态获取类的信息（属性、方法、构造器）并动态调用
- 优点：灵活性高，框架设计的灵魂
- 缺点：性能较低（安全检查 + JIT 难优化）；破坏封装（可访问 private）
- 应用：Spring IoC（XML 装载 Bean）、JDBC `Class.forName()`、动态代理、注解处理
- 获取 Class 三种方式：①`对象.getClass()` ②`Class.forName("全限定名")` ③`类名.class`
> ⚠️ 易错点：三种方式获取的是同一个 Class（类加载只产生一个 Class 对象）；`Class.forName()` 会触发类初始化，`类名.class` 不会

### Q：Error 和 Exception 的区别？受检异常和非受检异常的区别？
- Throwable 是所有错误/异常的父类，下分 Error 和 Exception
- **Error**：JVM 层面严重问题（OOM、StackOverflowError），程序一般无法恢复，不应捕获
- **Exception**：程序可处理。受检异常（checked，编译期强制处理，如 IOException）和非受检异常（unchecked，即 RuntimeException 及其子类，如 NPE）
- 受检异常必须 try-catch 或 throws；非受检异常编译器不强制
> ⚠️ 易错点：Error 也属于非受检异常（unchecked），编译器不要求处理。受检异常专指 Exception 中除 RuntimeException 外的部分

### Q：try-catch-finally 中，catch 中 return 了，finally 还会执行吗？
- 会执行，且在 return **之前**。JVM 先记录返回值，执行完 finally 后再返回
- finally 中也有 return 会覆盖 catch 的返回值（强烈不推荐）
- finally 修改基本类型返回变量不影响已记录的返回值；修改引用类型对象字段会影响
- try-with-resources（Java 7+）优先用于资源释放
> ⚠️ 易错点：finally 在 `System.exit()`、JVM 崩溃、线程被杀死时不执行；finally 中修改返回值是反模式

### Q：String 为什么不可变？String/StringBuilder/StringBuffer 区别？String s=new String("a") 创建几个对象？
- 不可变原因：①String 类被 final 修饰不可继承 ②底层存储数组被 final 修饰 ③不提供修改内部数组的方法
- 好处：线程安全、hashCode 可缓存（适合做 HashMap key）、字符串常量池可复用
- String：不可变；StringBuilder：可变、线程不安全、单线程性能最好；StringBuffer：可变、方法加 synchronized、线程安全
- `String s = new String("a")`：若常量池无 "a" 创建 **2 个对象**（常量池 + 堆）；若已存在创建 **1 个**（仅堆）
> ⚠️ 勘误：① 资料 String 源码引用 `private final char value[]` 是 Java 8 及之前版本。Java 9+ 改为 `private final byte[] value` + `byte coder`（compact strings，按内容选 Latin-1 或 UTF-16 节省内存）。② 资料称 `new String("xyz")` 一定创建两个对象不准确，应为"1 或 2 个"。③ 字符串常量池 JDK 7 起从方法区（永久代）移到了堆中，资料"静态区"说法已过时

---

## 第二章 集合框架

### Q：Java 集合体系结构？Collection 和 Collections 的区别？
- 两大顶层接口：Collection（单列）和 Map（双列键值对）。Collection 下有 List、Set、Queue
- List 有序可重复（ArrayList/LinkedList/Vector）；Set 无序不可重复（HashSet/TreeSet/LinkedHashSet）；Queue（ArrayDeque/PriorityQueue）
- Map 不继承 Collection，常用 HashMap/TreeMap/LinkedHashMap/ConcurrentHashMap
- Collection 是接口，定义 add/remove/iterator 等方法；Collections 是工具类，提供 sort/shuffle/synchronizedList 等静态方法
> ⚠️ 易错点：Collection（接口）和 Collections（工具类）拼写相近但完全不同

### Q：List、Set、Map 三者的区别？
- List：有序、可重复、可存多个 null、按索引访问
- Set：无序、不可重复、最多一个 null
- Map：键值对，key 唯一（最多一个 null key）、value 可重复
- List/Set 继承 Collection，Map 是独立顶层接口
> ⚠️ 易错点：Map 不属于 Collection 体系

### Q：ArrayList 的底层实现和扩容机制？
- 底层 Object[] 数组，实现 RandomAccess 接口（标记支持随机访问 O(1)）
- JDK1.7 无参构造默认容量 10；JDK1.8 无参构造默认空数组，首次 add 时才初始化为 10
- 扩容：新容量 = 旧容量 + 旧容量 >> 1，即 1.5 倍
- 增删慢：中间插入/删除需数组复制移动元素 O(n)，尾部增删快
- elementData 用 transient 修饰，重写 writeObject 只序列化已存元素
> ⚠️ 勘误：JDK1.8 ArrayList 无参构造默认是空数组（容量 0）不是 10；说"默认 10"需说明是首次 add 后的容量

### Q：ArrayList 和 LinkedList 的区别？
- 数据结构：ArrayList 动态数组；LinkedList 双向链表
- 随机访问：ArrayList O(1)；LinkedList O(n)
- 增删：LinkedList 头尾增删 O(1)；ArrayList 中间增删 O(n)
- 内存：LinkedList 每节点额外存前后指针，更占内存
- 频繁读选 ArrayList，频繁头尾增删选 LinkedList
> ⚠️ 勘误：资料称"LinkedList 增删比 ArrayList 快"不全面--LinkedList 中间增删仍需 O(n) 遍历定位，只有已知节点位置的增删才是 O(1)

### Q：ArrayList 和 Vector 的区别？
- 线程安全：Vector 方法用 synchronized 修饰，线程安全但性能差；ArrayList 非线程安全
- 扩容：ArrayList 1.5 倍；Vector 默认 2 倍（capacityIncrement 未指定时）
- Vector 已不推荐使用；多线程替代方案：`Collections.synchronizedList()` 或 `CopyOnWriteArrayList`
> ⚠️ 易错点：Vector 扩容 2 倍是未指定 capacityIncrement 时的默认；若指定 capacityIncrement > 0 则按增量扩容而非翻倍

### Q：LinkedList 的底层数据结构是什么？
- 双向链表，每个 Node 存储 item、prev、next
- 维护 first 和 last 指针，first.prev = null、last.next = null
- 同时实现 List 和 Deque 接口，可作队列、双端队列、栈
> ⚠️ 勘误：资料称 LinkedList 是"双向循环链表"是错误的，Java 的 LinkedList 是双向链表（非循环）；资料称"ArrayList、LinkedList、Vector 底层都用数组存储"同样错误

### Q：RandomAccess 接口的作用？
- 标记接口（空接口），表示实现类支持快速随机访问，get(i) 平均 O(1)
- ArrayList 实现了，LinkedList 未实现
- 遍历建议：实现 RandomAccess 用 for 循环，未实现用 Iterator/foreach
> ⚠️ 易错点：RandomAccess 只是标记不是能力保证；LinkedList 也能调 get(i)，只是效率差

### Q：HashMap 的底层实现原理？
- JDK1.8：数组 + 链表 + 红黑树（table 是 Node[]，每个桶是链表或红黑树）
- JDK1.7：数组 + 链表，头插法；JDK1.8 改尾插法
- 默认初始容量 16、负载因子 0.75、阈值 threshold = 容量 × 负载因子
- 链表长度 ≥8 且数组容量 ≥64 时转红黑树；节点数降至 6 退化为链表
- 允许 key 和 value 为 null（key 为 null 时 hash 固定 0，放桶 0）
> ⚠️ 易错点：树化需同时满足"链表长度 ≥8"和"数组容量 ≥64"两个条件；数组容量 <64 时即使链表 ≥8 也只触发扩容不树化

### Q：HashMap 的 put 流程？
1. 计算 hash = (h = key.hashCode()) ^ (h >>> 16)（高低 16 位异或扰动）
2. 确定桶下标 index = (n-1) & hash
3. 桶为空，直接放入新节点
4. 桶非空：key 相同（hash 相等且 equals 为 true）则覆盖 value；否则遍历链表/红黑树插入
5. 链表插入后长度达 8，调用 treeifyBin（数组容量 <64 先扩容，≥64 才树化）
6. size++ 后若 size > threshold 则 resize 扩容
> ⚠️ 易错点：判断 key 相同需同时满足 hash 相等且 equals 返回 true，二者缺一不可

### Q：HashMap 的扩容机制？为什么负载因子是 0.75？为什么容量是 2 的幂次方？
- 扩容条件：size > threshold，每次扩容 2 倍
- 负载因子 0.75 是时空折中：过小浪费空间，过大冲突增多查询变慢
- 扩容后元素定位：JDK1.8 用 (e.hash & oldCap) 判断--为 0 留原位置，非 0 移到"原位置 + 旧容量"
- 容量为 2 的幂次方时 (n-1) 二进制全为 1，& 运算等价于取模但更快
> ⚠️ 易错点：JDK1.7 扩容后需重新计算 hash 取模；1.8 优化为用高位 bit 判断（前提是容量为 2 的幂），元素位置要么不变要么 +旧容量

### Q：HashMap 如何解决哈希冲突？hash 扰动是什么？
- 链地址法（拉链法）：冲突元素挂同一桶链表，1.8 链表过长转红黑树
- hash 扰动：hash = hashCode ^ (hashCode >>> 16)，让高 16 位参与低位运算
- 目的：数组较小时 index = (n-1) & hash 只用到 hash 低位，扰动让高位也影响结果，减少碰撞
- JDK1.7 扰动 9 次（4 次位运算 + 5 次异或），1.8 简化为 1 次异或
> ⚠️ 易错点：hash 扰动不改变 hashCode 本身，而是混合高低位让分布更均匀；key 为 null 时 hash 直接返回 0

### Q：HashMap 为什么引入红黑树？为什么阈值是 6 和 8？
- 链表过长查询退化为 O(n)，红黑树保证 O(log n)
- 树化条件：链表长度 ≥8 且数组容量 ≥64；退化条件：节点数 ≤6
- 用 8 和 6 而非同一个值：避免在边界频繁树化/退化来回切换
- 负载因子 0.75 下桶内节点超 8 的概率极低（泊松分布约 0.00000006），树化是极端情况兜底
> ⚠️ 易错点：退化阈值是 6 不是 8，中间留 7 作缓冲避免抖动

### Q：HashMap 的并发问题？JDK1.7 死循环和 JDK1.8 数据覆盖？
- JDK1.7：扩容时头插法 + 多线程并发，链表节点指针反转后可能形成环形链表，get 时死循环（CPU 100%）
- JDK1.8：改尾插法解决了死循环，但 put 仍非线程安全
- 1.8 并发问题：多线程同时 put 数据被覆盖丢失、size 计数不准、同时触发扩容可能丢数据
- 解决方案：用 ConcurrentHashMap 替代
> ⚠️ 易错点：JDK1.8 尾插法只解决死循环，不是让 HashMap 线程安全；并发 put 仍会数据覆盖丢失

### Q：ConcurrentHashMap 在 JDK1.7 和 1.8 中的实现原理？
- JDK1.7：Segment[]（默认 16 段）+ HashEntry[]，每个 Segment 继承 ReentrantLock，锁粒度为段
- JDK1.8：Node[] 数组 + 链表 + 红黑树，锁粒度为桶节点（Node 头节点）
- 1.8 加锁：空桶用 CAS 插入；非空桶用 synchronized 锁住头节点
- 1.8 用 volatile 保证 Node 的 val/next 可见性
- 1.8 保留 Segment 类仅为序列化兼容，已无结构作用
> ⚠️ 易错点：1.8 的锁粒度是桶节点（Node），不是 Segment；CAS 只用于空桶首次插入，非空桶用 synchronized

### Q：ConcurrentHashMap 为什么 JDK1.8 弃用分段锁？
- 分段锁粒度仍较粗（默认 16 段），并发度受限于段数，最多 16 线程同时写
- 桶节点级锁粒度更细，并发度等于数组长度，冲突概率更低
- 去掉 Segment 层简化数据结构，减少两次 hash 定位开销
- 引入红黑树解决长链表查询慢
- synchronized 在 JDK1.6 后经锁升级优化，性能不逊 ReentrantLock 且内存占用更少
> ⚠️ 易错点：弃用分段锁不是因为它不好，而是桶节点级锁更细粒度；选 synchronized 是因 JVM 内置锁优化后足够高效且无需手动释放

### Q：为什么 ConcurrentHashMap 不允许 null 作为 key 或 value？
- 多线程下 get(key) 返回 null 有歧义：无法判断是"key 不存在"还是"value 就是 null"
- HashMap 单线程可用 containsKey(key) 区分，但 ConcurrentHashMap 多线程下 containsKey 与 get 之间可能被其他线程修改，结果不可靠
- Doug Lea 设计原则：null 在并发 Map 中是"潜伏的错误"，歧义无法消除，直接禁止
- Hashtable 也不允许 null，抛 NullPointerException；HashMap 允许
> ⚠️ 易错点：是 ConcurrentHashMap 和 Hashtable 禁止 null，HashMap 允许；原因是并发下 containsKey 检查无法消除歧义

### Q：HashSet 的实现原理？
- 基于 HashMap 实现，内部维护一个 HashMap<E, Object>
- 元素存为 HashMap 的 key，value 统一用静态常量 PRESENT
- add(e) -> map.put(e, PRESENT)，返回 null 表示插入成功（无重复）
- 去重机制：先比 hashCode，再比 equals；两者都相等才视为重复
> ⚠️ 易错点：HashSet 去重不是只靠 equals，必须同时重写 hashCode

### Q：TreeMap 和 LinkedHashMap 的特点？
- TreeMap：底层红黑树，key 按 Comparable 自然顺序或 Comparator 排序，增删查 O(log n)
- LinkedHashMap：HashMap 子类，额外维护双向链表记录插入/访问顺序
- LinkedHashMap 可按插入顺序（默认）或访问顺序（accessOrder=true）遍历，常用于 LRU 缓存
- TreeSet/LinkedHashSet 底层分别基于 TreeMap/LinkedHashMap
> ⚠️ 易错点：TreeMap 的 key 必须实现 Comparable 或传入 Comparator，否则 ClassCastException

### Q：什么是 fail-fast 机制？
- 集合错误检测机制，遍历中若结构被修改（add/remove）抛 ConcurrentModificationException
- 实现原理：迭代器持有 expectedModCount，每次 next() 前检查 modCount == expectedModCount，不等则抛异常
- 触发场景：单线程 foreach 遍历时直接 list.remove()，或多线程并发修改
- 解决：用 Iterator.remove() 遍历删除，或用 CopyOnWriteArrayList（fail-safe）
> ⚠️ 易错点：foreach 本质是迭代器遍历，遍历时用集合自身的 remove 会触发 fail-fast；必须用迭代器的 remove()

### Q：为什么 String 适合做 HashMap 的 key？
- String 不可变（final 类，内部数组 final 修饰），创建后 hashCode 不变，可安全缓存
- String 重写了 hashCode()，首次计算后缓存到 hash 字段，后续直接返回，避免重复计算
- 不可变保证 key 在 map 中不被意外修改（否则 hashCode 变化找不到对应桶）
- String 的 equals() 实现正确
> ⚠️ 易错点：自定义对象做 key 必须同时重写 hashCode 和 equals，且应为不可变对象；否则属性改变后 hashCode 变化会导致 key 丢失

---

## 第三章 并发编程

### Q：synchronized 的三种用法分别锁的是什么？
- 修饰实例方法：锁当前对象实例 `this`，同一实例多线程串行，不同实例不互斥
- 修饰静态方法：锁当前类的 `Class` 对象，全局锁，所有线程访问该类静态同步方法都互斥
- 修饰代码块 `synchronized(obj)`：锁括号里的对象，锁粒度更可控
> ⚠️ 易错点：修饰静态方法锁的是 Class 对象而非实例；普通方法与静态同步方法之间不会互斥，锁对象不同

### Q：synchronized 的底层原理是什么？
- 基于 JVM 的 monitor（管程）对象实现，对象头中 Mark Word 指向 monitor
- 修饰代码块：编译后通过 `monitorenter` 和 `monitorexit` 两条字节码指令完成加锁/释放；`monitorexit` 出现两次（正常 + 异常释放），保证异常时也释放
- 修饰方法：通过 ACC_SYNCHRONIZED 标志位标识，JVM 调用时自动加锁/释放
- monitor 内部维护计数器 `_count` 与持有线程 `_owner`，重入时 +1，释放时 -1

### Q：synchronized 的锁升级过程是怎样的？
- 无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁，是 JDK 1.6 引入的优化，锁只能升级不能降级
- 偏向锁：假设单线程访问，首次进入时 Mark Word 记录线程 ID，后续同线程无需 CAS
- 轻量级锁：出现竞争时，线程通过 CAS 自旋尝试获取，避免阻塞（适合持锁时间短）
- 重量级锁：竞争激烈/自旋失败时升级，未获取锁的线程进入 monitor 的 EntryList 阻塞
> ⚠️ 易错点：锁升级是 JDK 1.6 后才引入的优化，并非一开始就有；JDK 15 后偏向锁被废弃

### Q：JDK 1.6 对 synchronized 做了哪些优化？
- 锁升级：偏向锁、轻量级锁、重量级锁
- 锁消除：JIT 通过逃逸分析，发现对象不会逃逸出方法被其他线程访问时，自动消除无用同步
- 锁粗化：对同一对象连续多次加锁/解锁，JIT 合并为一个大同步块，减少锁开销
- 自旋自适应：根据上次自旋成功与否动态调整自旋次数

### Q：volatile 关键字的两大作用是什么？
- 保证可见性：被 volatile 修饰的变量，写操作立即刷新回主内存，并使其他线程工作内存副本失效，强制重新从主内存读取
- 禁止指令重排（保证有序性）：通过插入内存屏障，防止编译器/CPU 重排序
- 可见性底层：JVM 对 volatile 写生成 `lock` 前缀汇编指令，触发总线锁或缓存锁（MESI 协议）
> ⚠️ 易错点：volatile 被称为"轻量级 synchronized"，但不保证原子性，不能替代 synchronized

### Q：volatile 为什么不保证原子性？以 i++ 为例
- `i++` 不是原子操作，拆分为三步：读取 i、计算 i+1、写回 i
- volatile 仅保证每次读取从主内存刷新、写回立即同步，但无法保证"读-改-写"三步间的原子性
- 线程 A 读取 i 后尚未写回时，线程 B 也读取了相同旧值，最终两次自增只生效一次，丢失更新
- 要保证原子性，需用 `synchronized`、`AtomicInteger`（基于 CAS）或 `Lock`

### Q：volatile 的内存屏障原理？
- JSR-133 规定，volatile 写前插入 StoreStore 屏障（禁止前面普通写与 volatile 写重排）
- volatile 写后插入 StoreLoad 屏障（禁止后续读/写与 volatile 写重排，并强制刷主内存，开销最大）
- volatile 读前插入 LoadLoad 屏障（禁止后面 volatile 读与前面普通读重排）
- volatile 读后插入 LoadStore 屏障（禁止后面普通写与 volatile 读重排）
> ⚠️ 勘误：资料仅笼统说"写前后各加两个屏障"，实际是 StoreStore/StoreLoad（写）+ LoadLoad/LoadStore（读）四类屏障组合；StoreLoad 是开销最大的全功能屏障

### Q：volatile 与 happens-before 的关系？典型应用场景？
- happens-before：对一个 volatile 变量的写，happens-before 后续任意线程对它的读
- 应用一：DCL（双重检查锁）单例，`instance` 必须 volatile 修饰，防止 new 对象时指令重排（分配内存->赋值引用->初始化，可能重排为先赋值引用再初始化，导致其他线程拿到未初始化对象）
- 应用二：状态标志位（`volatile boolean running`），一写多读无需加锁即可保证可见性
- 应用三：DCL 中 volatile 主要作用是禁止重排而非可见性（synchronized 已保证可见性）

### Q：synchronized 和 volatile 有什么区别？
- 原子性：synchronized 保证，volatile 不保证
- 可见性：两者都保证
- 有序性：synchronized 互斥串行保证，volatile 通过禁止重排保证
- 阻塞：synchronized 会阻塞，volatile 不会
- 适用场景：volatile 适合一写多读状态标志；synchronized 适合复合操作（如 i++、临界区）

### Q：synchronized 和 ReentrantLock 的区别？
- 实现层面：synchronized 是 JVM 关键字（monitorenter/monitorexit）；ReentrantLock 是 J.U.C 包的类（基于 AQS）
- 锁释放：synchronized 自动释放；ReentrantLock 需手动 `unlock()`，必须放 finally
- 响应中断：synchronized 不响应；ReentrantLock 支持 `lockInterruptibly()`
- 超时获取：synchronized 不支持；ReentrantLock 支持 `tryLock(timeout)`
- 非阻塞尝试：synchronized 不支持；ReentrantLock 支持 `tryLock()`
- 公平性：synchronized 仅非公平；ReentrantLock 可选公平/非公平
- 条件变量：synchronized 仅一个等待队列（wait/notify）；ReentrantLock 支持多个 Condition 分组唤醒
- 性能：JDK 1.6 优化后两者差距很小，复杂同步场景选 ReentrantLock
> ⚠️ 勘误：资料说"synchronized 是悲观锁、Lock 是乐观锁"不准确。ReentrantLock 本质仍是独占式悲观锁，只是用 CAS（乐观技术）做锁获取的快速尝试；真正的乐观锁是 CAS 类（Atomic）或 StampedLock 乐观读

### Q：ReentrantLock 的实现原理是什么？
- 基于 AQS（AbstractQueuedSynchronizer），核心三要素：`state`（volatile int 同步状态）+ CLH 变种双向队列 + CAS
- 加锁时通过 CAS 将 state 从 0 改为 1，成功则记录当前线程为持有者
- 可重入：持有锁的线程再次获取锁时 state 递增（+1），释放时递减，减到 0 才真正释放
- 未获取锁的线程被包装为 Node 加入 AQS 双向队列尾部阻塞；锁释放时从队头唤醒后继
- 公平锁：获取前先 `hasQueuedPredecessors()` 判断队列是否有前驱，有则排队；非公平锁直接 CAS 抢锁
> ⚠️ 易错点：AQS 用的是 CLH 变种的"双向"链表，不是单向链表

### Q：ReentrantLock 的公平锁和非公平锁有什么区别？
- 公平锁：获取锁时先检查 AQS 等待队列中是否有更早等待的线程，有则排在后面，FIFO，无饥饿
- 非公平锁：直接尝试 CAS 抢锁，抢不到再入队，允许插队，吞吐量更高
- 默认是非公平锁（`new ReentrantLock()`），减少线程切换开销
> ⚠️ 易错点：公平锁吞吐量低于非公平锁（额外检查队列 + 更多线程切换）；非公平锁可能让某些线程长时间拿不到锁（饥饿）

### Q：AQS 是什么？为什么用双向链表？
- AbstractQueuedSynchronizer，J.U.C 并发包基石，ReentrantLock/Semaphore/CountDownLatch 等都基于它
- 核心：一个 volatile int `state` 表示同步状态；一个 CLH 变种双向队列管理等待线程；CAS 修改 state 保证原子性
- 独占模式（ReentrantLock）与共享模式（Semaphore/CountDownLatch）两种获取方式
- 模板方法模式：AQS 提供入队、阻塞、唤醒骨架，将"如何判断可获取"交给子类实现 tryAcquire/tryRelease
- **双向链表原因**：节点取消等待时需找前驱把后继接上，单向链表无法回溯；唤醒后继时若后继取消需从尾部向前遍历找有效后继
> ⚠️ 易错点：AQS 是双向链表（CLH 变种），不是单向；这是为了支持节点取消和唤醒后继的回溯

### Q：什么是悲观锁和乐观锁？
- 悲观锁：假设一定竞争，先加锁再操作（synchronized、ReentrantLock、数据库行锁），写多读少适用
- 乐观锁：假设无竞争，操作时不加锁，提交时通过版本号/CAS 校验是否被修改，失败则重试（Atomic 原子类、版本号机制），读多写少适用
- 乐观锁典型实现是 CAS；竞争激烈时重试频繁，开销反而更大

### Q：什么是 CAS？存在什么问题？
- Compare And Swap，三操作数：内存值 V、预期旧值 A、新值 B；仅当 V==A 时将 V 改为 B，否则重试（自旋）
- 无锁编程基础，CPU 层面通过 `cmpxchg` 指令保证原子性（多核加 lock 前缀）
- 问题一：ABA 问题（见下题）
- 问题二：自旋开销大，竞争激烈时 CPU 空转
- 问题三：只能保证单变量原子性，多变量需 AtomicReference 封装

### Q：什么是 ABA 问题？如何解决？
- 线程 1 读取值为 A，线程 2 将 A 改为 B 又改回 A，线程 1 CAS 时发现值仍是 A，误以为没被修改过，CAS 成功
- 危害：如栈顶指针被中途改动又恢复，CAS 仍可能操作到已释放的节点
- 解决：`AtomicStampedReference`（加版本号 stamp）或 `AtomicMarkableReference`（加 boolean 标记），每次修改版本号递增，CAS 同时比较值和版本号

### Q：常见的锁分类有哪些？
- 悲观锁 vs 乐观锁：是否预先加锁
- 公平锁 vs 非公平锁：是否按 FIFO 排队
- 可重入锁 vs 不可重入锁：同一线程能否重复获取同一把锁（ReentrantLock/synchronized 可重入）
- 独占锁 vs 共享锁：独占锁同一时刻只能一个线程持有（ReentrantLock），共享锁可多个（读写锁读锁、Semaphore）
- 自旋锁：获取失败不阻塞，循环重试（轻量级锁、CAS）
- 偏向锁/轻量级锁/重量级锁：synchronized 锁升级三个层级

### Q：synchronized 和 ReentrantLock 如何选择？
- 简单同步、不需要高级功能：优先 synchronized，代码简洁、JVM 自动释放不会死锁
- 需要公平锁、响应中断、超时获取、tryLock 非阻塞、多个条件变量：用 ReentrantLock
- 锁竞争激烈且持锁长：考虑 ReentrantReadWriteLock（读多写少）或 StampedLock 乐观读
- JDK 1.6+ synchronized 性能大幅优化，日常优先 synchronized
> ⚠️ 易错点：选 ReentrantLock 时务必 finally 释放锁，否则异常将永久持有导致死锁；不要误以为"Lock 性能一定更好"

### Q：线程池的七大核心参数分别是什么？
- `corePoolSize`：核心线程数，即使空闲也会保留（除非 `allowCoreThreadTimeOut=true`）
- `maximumPoolSize`：线程池能创建的最大线程数
- `keepAliveTime`：非核心线程空闲存活时间
- `unit`：存活时间单位
- `workQueue`：存放待执行任务的阻塞队列
- `threadFactory`：线程创建工厂，可设置有意义的线程名便于排查
- `handler`：饱和拒绝策略
> ⚠️ 易错点：核心线程与非核心线程本身无区别，只是数量上限的判断阈值不同；默认核心线程不被回收

### Q：线程池的工作流程是怎样的？
- 提交任务后，若当前线程数 < `corePoolSize`，创建核心线程执行
- 若线程数已达 `corePoolSize`，任务进入 `workQueue` 排队
- 队列也满后，若线程数 < `maximumPoolSize`，创建非核心线程执行
- 队列满且线程数已达 `maximumPoolSize`，触发拒绝策略
- 非核心线程空闲超过 `keepAliveTime` 被回收
> ⚠️ 易错点：顺序是"核心线程 -> 队列 -> 非核心线程 -> 拒绝策略"，不是先创建到最大线程再入队

### Q：线程池有哪四种拒绝策略？
- `AbortPolicy`（默认）：抛出 `RejectedExecutionException`
- `CallerRunsPolicy`：由提交任务的线程（调用者）自己执行该任务
- `DiscardPolicy`：直接静默丢弃新任务
- `DiscardOldestPolicy`：丢弃队列头部最旧的任务，再次尝试提交新任务
> ⚠️ 易错点：`CallerRunsPolicy` 不会丢任务，还能反压调用方降低提交速度；可自定义实现 `RejectedExecutionHandler`

### Q：常见线程池有哪些？阿里规约为何禁止使用 Executors？
- `newFixedThreadPool`：核心=最大线程数，队列无界（`LinkedBlockingQueue`）
- `newCachedThreadPool`：核心 0、最大 `Integer.MAX_VALUE`、存活 60s，用 `SynchronousQueue`
- `newSingleThreadExecutor`：单线程串行执行，保证 FIFO
- `newScheduledThreadPool`：支持定时/周期任务
- `newWorkStealingPool`（Java 8）：底层 `ForkJoinPool`，工作窃取
- **阿里规约禁止 Executors 原因**：`newFixedThreadPool`/`newSingleThreadExecutor` 用无界队列堆积任务导致 OOM；`newCachedThreadPool`/`newScheduledThreadPool` 最大线程数为 `Integer.MAX_VALUE` 创建过多线程导致 OOM
> ⚠️ 易错点：两类 OOM 根源不同--前者是队列无界，后者是线程数无界；应使用 `ThreadPoolExecutor` 显式传参

### Q：线程数应该如何设置？
- CPU 密集型：线程数 ≈ CPU 核数 N（或 N+1），减少线程切换
- IO 密集型：线程数 ≈ 2N，或 `N × (1 + 等待时间/计算时间)`（Brian Goetz 公式）
- 实际取值需结合任务量、IO/CPU 占比、机器配置，并通过压测调优
> ⚠️ 易错点：上述公式是经验值而非绝对，混合型任务需压测确定最佳值

### Q：线程池有哪几种状态？如何优雅关闭？
- `RUNNING`：接收新任务并处理队列任务
- `SHUTDOWN`：`shutdown()` 触发，不收新任务但处理完队列任务
- `STOP`：`shutdownNow()` 触发，不收新任务、不处理队列、中断正在执行任务
- `TIDYING`：所有任务终止，任务数为 0，执行 `terminated()`
- `TERMINATED`：`terminated()` 执行完毕
- 优雅关闭：`shutdown()` 停止接收新任务继续执行完队列任务；`shutdownNow()` 中断正在执行的任务并返回未执行任务列表；常配合 `awaitTermination(timeout)` 阻塞等待
> ⚠️ 易错点：`shutdownNow` 调用线程的 `interrupt()`，任务需响应中断才会真正停止

### Q：线程有哪几种状态？
- `NEW`：已创建未 `start()`
- `RUNNABLE`：已在 JVM 中执行（可能正在等待 OS 调度）
- `BLOCKED`：等待 monitor 锁（synchronized）
- `WAITING`：无限期等待（`wait()`/`join()`/`LockSupport.park()`）
- `TIMED_WAITING`：限时等待（`sleep(n)`/`wait(n)`/`join(n)`）
- `TERMINATED`：线程执行结束
> ⚠️ 易错点：调用阻塞 IO 时线程状态仍是 RUNNABLE（Java 层面），不要把 IO 阻塞等同于 BLOCKED

### Q：start() 和 run() 的区别？sleep() 与 wait() 的区别？yield()、join() 呢？
- `start()`：启动新线程，由 JVM 调用该线程的 `run()`；`run()` 直接调用会在当前线程同步执行，不新建线程。一个线程对象只能 `start()` 一次
- `sleep`：Thread 静态方法，不释放锁，到时自动唤醒
- `wait`：Object 方法，必须持有 monitor，调用后释放锁，需 `notify` 唤醒
- `yield`：提示调度器让出 CPU，可能立即又获得，不释放锁
- `join`：等待目标线程结束（内部用 `wait` 实现），释放锁
> ⚠️ 易错点：直接调 `run()` 不创建线程，常见面试陷阱；sleep 不释放锁、wait 释放锁是高频考点

### Q：ThreadLocal 的原理是什么？set/get 如何实现？
- 每个 `Thread` 持有一个 `ThreadLocal.ThreadLocalMap` 成员变量 `threadLocals`
- `ThreadLocalMap` 维护 `Entry[]`，`Entry` 的 key 是 `ThreadLocal` 本身，value 是泛型值
- 各线程读写只操作自己的 `ThreadLocalMap`，实现线程隔离（空间换时间）
- `set`：取当前线程 map，以 `this`（ThreadLocal）为 key 存入；`get`：以 `this` 查 Entry
- hash 冲突用开放寻址（线性探测）解决
> ⚠️ 易错点：是 Thread 持有 Map，不是 ThreadLocal 持有 Map；ThreadLocal 是 key；冲突解决是线性探测而非链地址法

### Q：ThreadLocal 为什么会内存泄漏？key 为什么用弱引用？如何避免？
- `Entry` 的 key 是 `ThreadLocal` 的**弱引用**，外部强引用断开后 key 被 GC 回收为 null
- 但 value 是**强引用**，key=null 后 value 仍可达无法回收 -> 泄漏
- 线程池中线程生命周期长，泄漏的 value 不断累积，危害放大
- **key 用弱引用的原因**：若 key 为强引用，外部断开 ThreadLocal 后仍被 Entry 持有无法回收；弱引用使 ThreadLocal 在无外部强引用时被回收，配合 set/get/remove 时探测清理缓解泄漏
- **避免**：使用完毕（尤其线程池场景）在 `finally` 中调用 `remove()`；把 ThreadLocal 声明为 `static final`
> ⚠️ 易错点：泄漏的是 value（强引用），不是 key；key 用弱引用恰恰是为减小泄漏，但仍不彻底，必须手动 `remove()`

### Q：ThreadLocal 有哪些应用场景？InheritableThreadLocal 呢？
- 数据库连接/事务隔离：每线程独占 Connection
- 会话隔离：用户登录上下文、TraceId 传递
- `SimpleDateFormat` 线程安全包装（其本身非线程安全）
- `InheritableThreadLocal`：子线程可继承父线程的值
> ⚠️ 易错点：线程池中线程复用，`InheritableThreadLocal` 在"提交任务时"的父线程上下文会丢失，需用 `TransmittableThreadLocal`（阿里 TTL）

### Q：什么是死锁？产生的四个必要条件？如何排查和预防？
- 死锁：两个或以上线程互相等待对方持有的资源，无外力干预无法继续
- 四个必要条件（缺一不可）：互斥、占有并等待、不可剥夺、循环等待
- 排查：`jstack <pid>` 打印线程栈，末尾检测并输出 "Found one Java-level deadlock"；或 `jconsole`/`VisualVM`；或 `ThreadMXBean.findDeadlockedThreads()` 编程检测
- 预防：破坏占有等待（一次性申请全部资源）、破坏不可剥夺（`tryLock(timeout)`）、破坏循环等待（按固定顺序加锁）
> ⚠️ 易错点：互斥条件无法破坏（互斥锁基本约束）；`tryLock` 超时是工程上最实用手段

### Q：死锁、活锁、饥饿的区别？
- 死锁：线程互相等待，永久阻塞
- 活锁：线程不阻塞但反复重试同一操作，互相"让步"导致都无法前进
- 饥饿：线程长期得不到所需资源（如优先级太低或锁被持续抢占），无法执行
> ⚠️ 易错点：活锁线程是"运行态"但仍无进展，随机退避可缓解；饥饿可通过公平锁解决

### Q：JMM 的主内存与工作内存是什么？并发的三大特性？
- 主内存：所有共享变量存储处；工作内存：每线程私有变量副本（对应 CPU 缓存/寄存器）
- 线程对变量操作须：主内存 -> 读到工作内存 -> 修改 -> 写回主内存
- 三大特性：原子性（synchronized/Lock/Atomic）、可见性（volatile/synchronized/final）、有序性（volatile/happens-before）
> ⚠️ 易错点：工作内存是抽象概念并非真实内存区域；`volatile` 保证可见性与有序性，但不保证原子性

### Q：happens-before 有哪些规则？
- 程序顺序规则：同一线程内前操作 happens-before 后操作
- 监视器锁规则：unlock happens-before 后续对该锁的 lock
- volatile 规则：写 happens-before 后续读
- 线程启动规则：`start()` happens-before 该线程内动作
- 线程终止规则：线程内动作 happens-before `join()` 返回
- 线程中断规则：`interrupt()` happens-before 检测中断
- 对象终结规则：构造方法结束 happens-before finalizer
- 传递性：A hb B 且 B hb C，则 A hb C
> ⚠️ 易错点：共 8 条（含传递性）；happens-before 不是说"前操作必须先执行完"，而是前操作结果对后可见

### Q：什么是指令重排？如何禁止？
- 编译器、CPU、缓存为优化会打乱无依赖指令的执行顺序
- 重排在单线程下不影响结果（as-if-serial），多线程下破坏语义
- 内存屏障禁止重排：LoadLoad、StoreStore、LoadStore、StoreLoad（最强）
- `volatile` 写前插入 StoreStore、写后插入 StoreLoad；读后插入 LoadLoad/LoadStore
> ⚠️ 易错点：StoreLoad 屏障开销最大，是 volatile 写代价的主要来源

### Q：CountDownLatch、CyclicBarrier、Semaphore 的区别？
- `CountDownLatch`：一/多线程等待其他线程完成；`countDown()` 减 1 到 0 后放行；**不可重置**
- `CyclicBarrier`：多线程互等到同步点后一起继续；**可 `reset()` 重置复用**；可指定到达屏障后的动作
- `Semaphore`：许可信号量，`acquire` 获取/`release` 释放，用于限流
> ⚠️ 易错点：CountDownLatch 不可重置（一次性），CyclicBarrier 可重置（周期性）；后者异常会破坏屏障

---

## 第四章 JVM

### Q：JVM 运行时数据区包括哪些？哪些线程共享，哪些线程私有？
- 线程私有：程序计数器、虚拟机栈、本地方法栈
- 线程共享：堆、方法区
- 程序计数器记录当前线程执行字节码行号；虚拟机栈存储栈帧（局部变量表、操作数栈、动态链接、方法出口）；本地方法栈服务于 native 方法
- 堆存放对象实例和数组，是 GC 主战场；方法区存放类元信息、常量、静态变量、JIT 代码
> ⚠️ 勘误：资料把"方法区"与"永久代"等同是旧表述。方法区是 JVM 规范；永久代（JDK7 前）与元空间（JDK8+）是 HotSpot 的两种实现。JDK8 后元空间在本地内存中，不在堆内

### Q：程序计数器为什么是唯一不会 OOM 的区域？
- 程序计数器占用内存极小，仅记录字节码指令地址
- 线程私有，随线程创建而分配，线程结束即回收
- JVM 规范中唯一没有规定任何 OOM 情况的区域
- 执行 native 方法时 PC 值为空（undefined）

### Q：虚拟机栈的栈帧包含哪些内容？什么情况下会栈溢出？
- 栈帧包含：局部变量表、操作数栈、动态链接、方法返回地址
- 局部变量表存放基本类型、对象引用、returnAddress；容量以 Slot 为单位
- 栈深度超过 -Xss 限制抛 StackOverflowError（递归调用常见）
- 栈动态扩展时无法申请足够内存抛 OutOfMemoryError（线程创建过多）
- 可用 -Xss 调整栈大小

### Q：方法区、永久代、元空间三者是什么关系？
- 方法区是 JVM 规范，定义存储类元信息、常量池、静态变量等
- 永久代（PermGen）是 JDK7 及之前 HotSpot 对方法区的实现，位于堆中，易 OOM
- 元空间（Metaspace）是 JDK8 起 HotSpot 的新实现，使用本地内存，大小受物理内存限制
- JDK7 开始把字符串常量池、静态变量从永久代移到堆；JDK8 彻底移除永久代
> ⚠️ 勘误：资料表述"持久带=方法区+其他"是 JDK7 前模型，已过时。元空间不在堆里，在本地内存中；字符串常量池在 JDK7 就从永久代移到了堆中

### Q：直接内存是什么？和 JVM 堆有什么区别？
- 直接内存不是 JVM 运行时数据区的一部分，不受堆大小 -Xmx 限制
- NIO 通过 `ByteBuffer.allocateDirect()` 分配，避免 Java 堆与 native 堆之间数据拷贝，提升 IO 性能
- 受物理内存限制，分配回收成本较高
- 可通过 -XX:MaxDirectMemorySize 限制大小
- 释放靠 Cleaner（PhantomReference + 引用队列），不立即回收；默认约等于 -Xmx，并非无限

### Q：JVM 堆为什么要分新生代和老年代？新生代为什么分 Eden 和两个 Survivor？
- 不同生命周期对象适用不同 GC 算法，提升回收效率
- 新生代对象朝生夕灭用复制算法；老年代存活率高用标记-清除或标记-整理
- 默认新生代:老年代 = 1:2（-XX:NewRatio=2）
- 新生代 = Eden + Survivor0 + Survivor1，默认 Eden:Survivor = 8:1:1（-XX:SurvivorRatio=8）
- 两个 Survivor 交替使用：GC 时 Eden 和正在使用的 Survivor 中存活对象复制到另一个空 Survivor，保证内存连续避免碎片化
- 8:1:1 基于"98% 对象朝生夕灭"的统计，浪费仅 10% 空间换得高效回收
> ⚠️ 勘误：资料个别处说"经历 16 次 Minor GC 才晋升老年代"不准确。默认 -XX:MaxTenuringThreshold=15，即年龄达到 15 晋升

### Q：Minor GC、Major GC、Full GC 有什么区别？
- Minor GC：只回收新生代，频繁但速度快，Eden 满触发，会引发 STW
- Major GC：回收老年代，常伴随至少一次 Minor GC，速度比 Minor GC 慢 10 倍以上
- Full GC：回收整个堆（新生代+老年代）及方法区，STW 时间长，应尽量避免
- Full GC 触发条件：老年代空间不足、空间分配担保失败、方法区/元空间不足、`System.gc()` 调用（建议非强制）、CMS Concurrent Mode Failure

### Q：TLAB 是什么？有什么作用？
- TLAB（Thread Local Allocation Buffer）：线程本地分配缓存区，是堆中 Eden 区的一小块区域
- 对象分配时优先在当前线程的 TLAB 分配，避免多线程分配时的指针碰撞竞争
- TLAB 耗尽才用 CAS 分配新的 TLAB 或直接在堆上分配
- 由 -XX:+UseTLAB 开启（默认开启），提升对象分配效率

### Q：对象的创建过程是怎样的？内存布局如何？
1. 类加载检查：常量池中定位类的符号引用，检查是否已加载解析
2. 分配内存：根据堆是否规整用指针碰撞或空闲列表；线程安全靠 CAS 或 TLAB
3. 初始化零值：将分配的内存空间（除对象头）置零
4. 设置对象头：设置 MarkWord（哈希码、GC 分代年龄、锁状态等）和类型指针
5. 执行构造方法：执行 `<init>`，完成程序员定义的初始化
- **内存布局**：对象头（Mark Word + 类型指针 + 数组长度）+ 实例数据 + 对齐填充（8 字节整数倍）

### Q：对象访问定位有哪两种方式？
- 句柄访问：reference 指向句柄池，句柄分别指向对象实例和类型数据。GC 移动对象时只改句柄指针，reference 不变；缺点多一次寻址
- 直接指针（HotSpot 采用）：reference 直接指向对象。访问快少一次寻址；GC 移动对象时需更新所有 reference
- 对象头中已包含类型指针，直接指针方式也能快速定位类元数据

### Q：判断对象死亡有哪些方法？哪些对象可以作为 GC Roots？
- 引用计数法：对象被引用 +1，引用失效 -1，为 0 即回收。缺点：无法解决循环引用
- 可达性分析：从 GC Roots 向下搜索，路径为引用链；不可达的对象为可回收对象。JVM 主流采用
- **GC Roots**：虚拟机栈中局部变量表引用的对象、本地方法栈中 JNI 引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、同步锁 synchronized 持有的对象、JVM 内部引用（基本类型 Class 对象、常驻异常对象、类加载器等）
> ⚠️ 勘误：资料把 GC Roots 列得不够全。补充：被 synchronized 持有的对象、JVM 内部引用也是 GC Roots；方法区本身不是 GC Roots，是其中静态属性/常量引用的对象才是

### Q：三色标记法是什么？如何解决并发标记的漏标问题？
- 白色：未被标记；灰色：自身已标记，引用未扫描；黑色：自身及引用都已标记
- 漏标问题：并发标记时，黑色对象新增指向白色对象的引用，同时灰色对象指向该白色对象的引用断开，导致白色对象被误回收
- 解决方案：读写屏障
  - CMS 用写屏障 + 增量更新：记录新增引用，重新标记
  - G1 用写屏障 + SATB（原始快照）：记录断开引用，重新标记
  - ZGC 用读屏障：读取引用时动态处理

### Q：什么是安全点和安全区域？
- 安全点：程序执行到特定位置（方法调用、循环跳转、异常跳转）才可进行 GC，保证引用关系稳定
- GC 时需让所有线程跑到最近安全点停顿（主动式中断：设置中断标志，线程轮询）
- 安全区域：线程处于 Sleep 或 Blocked 状态时无法走到安全点，该区域引用不变，GC 时可安全标记
- 解决了"长时间运行的线程无法响应 GC"问题

### Q：强、软、弱、虚四种引用有什么区别？
- 强引用（Strong）：new 出来的对象，只要强引用存在永不回收，即使 OOM 也不回收
- 软引用（Soft）：SoftReference，内存不足时才回收，适合做缓存
- 弱引用（Weak）：WeakReference，下一次 GC 就被回收（无论内存是否充足）。ThreadLocalMap 的 key 是弱引用
- 虚引用（Phantom）：PhantomReference，get() 总返回 null，仅用于跟踪对象被回收的时间，必须配合 ReferenceQueue 使用

### Q：标记-清除、复制、标记-整理三种 GC 算法？为什么新生代用复制算法？
- 标记-清除：标记存活对象后清除死亡对象。缺点：产生内存碎片，大对象难分配
- 复制：将内存分两块，存活对象复制到另一块后清空原块。缺点：可用空间减半。优点：无碎片、效率高
- 标记-整理：标记后将存活对象向一端移动整理。缺点：移动开销大、STW。优点：无碎片
- 新生代用复制算法：因为新生代 98% 对象朝生夕灭，存活少复制开销小，配合 Eden:Survivor=8:1:1 只浪费 10% 空间
- 老年代用标记-整理（或标记-清除）：存活率高，复制浪费大

### Q：常见的垃圾收集器有哪些？搭配关系怎样？
- Serial / Serial Old：单线程，STW，复制/标记整理。适合 Client 模式
- ParNew：Serial 的多线程版，新生代复制算法，可与 CMS 搭配
- Parallel Scavenge / Parallel Old：吞吐量优先，多线程，JDK8 默认组合
- CMS：老年代，标记-清除，低停顿，可与 ParNew 搭配
- G1：JDK9 默认，整堆 Region 化，可预测停顿
- 搭配：新生代 ParNew + 老年代 CMS；Parallel Scavenge + Parallel Old；G1 单独使用不分代搭配

### Q：CMS 收集器的原理、流程和缺点？
- 目标：获取最短回收停顿时间，老年代收集器，标记-清除算法
- 四阶段：①初始标记（STW，标 GC Roots 直接关联）②并发标记 ③重新标记（STW，修正并发标记期间变动）④并发清除
- 缺点：
  - 标记-清除产生内存碎片，大对象分配可能触发 Full GC
  - Concurrent Mode Failure：并发时老年代不足导致退化为 Serial Old 做 Full GC，STW 长
  - 浮动垃圾：并发标记期间产生的新垃圾本次无法回收
  - 对 CPU 资源敏感，占用业务线程
- 可用 -XX:+UseCMSCompactAtFullCollection 在 Full GC 时做碎片整理

### Q：G1 收集器的原理和特点？与 CMS 有什么区别？
- 堆划分为多个大小相等的 Region（1~32MB），逻辑分代（Eden/Survivor/Old/Humongous）但物理不连续
- 可预测停顿：通过 -XX:MaxGCPauseMillis 设定目标停顿时间，优先回收价值最大的 Region
- 混合回收：回收新生代 + 部分老年代 Region
- 算法：Region 间用复制算法，整体看是标记-整理，不产生碎片
- 与 CMS 区别：G1 整堆回收不分代搭配，CMS 只回收老年代；G1 用复制+整理无碎片，CMS 用标记-清除有碎片；G1 可预测停顿
> ⚠️ 勘误：资料说"G1 使用标记-整理算法"过于简化。准确说法：G1 Region 间复制 + Region 内整理，整体看是复制+标记整理，这也是它相对 CMS 无碎片的原因

### Q：ZGC 有什么特点？
- JDK11+ 引入的低延迟收集器，停顿时间 <10ms（JDK16 后达到亚毫秒级）
- 核心技术：染色指针（Colored Pointers）+ 读屏障（Read Barrier）
- Region 化内存布局，支持堆容量从 MB 级到 TB 级
- 并发标记、并发转移、并发重定位，几乎所有阶段都与应用线程并发
- JDK21 前不分代（JDK21 开始支持分代 ZGC）

### Q：对象内存分配有哪些策略？
- 对象优先在 Eden 分配：Eden 空间不足触发 Minor GC
- 大对象直接进老年代：大对象需要连续空间，-XX:PretenureSizeThreshold 控制阈值，避免在 Eden 和 Survivor 间来回复制
- 长期存活对象进老年代：每熬过一次 Minor GC 年龄 +1，达到 -XX:MaxTenuringThreshold（默认 15）晋升
- 动态年龄判断：Survivor 中相同年龄对象大小总和超过 Survivor 空间一半，该年龄及以上对象直接进老年代
- 空间分配担保：Minor GC 前检查老年代最大可用连续空间是否 > 新生代所有对象总空间，不足则检查是否允许担保失败，允许则尝试 Minor GC，仍失败则 Full GC

### Q：简述类加载的 5 个阶段
- 加载：通过全限定名找到 class 文件，读取字节流，将类型信息存入方法区，并在堆中生成对应的 Class 对象
- 验证：校验文件格式、元数据、字节码、符号引用的正确性
- 准备：为静态变量在方法区分配内存并赋"零值"（int=0、引用=null）
- 解析：将常量池中的符号引用替换为直接引用
- 初始化：执行 `<clinit>`，按顺序执行静态变量赋值语句和静态代码块
> ⚠️ 易错点：准备阶段赋的是"零值"而非代码中写的初始值；被 `final static` 修饰且编译期可确定常量的，才在准备阶段直接赋初值；赋值语句和静态块在初始化阶段才执行

### Q：JVM 有哪几种类加载器？什么是双亲委派机制？
- Bootstrap（启动类加载器）：加载 JDK 核心库，由 C++ 实现，无 Java 对象，获取它返回 null
- Extension/Platform：JDK8 为 Extension 加载 `lib/ext`；JDK9+ 改称 Platform
- Application/System：加载用户 classpath 下的类
- 自定义类加载器：继承 ClassLoader 重写 `findClass`
- **双亲委派**：类加载器收到加载请求时，先委派给父加载器去加载，父加载器继续向上委派直到 Bootstrap；父加载器能加载则成功返回，无法加载时子加载器才尝试自己加载。"父加载器"是组合关系（含 parent 引用），并非继承
> ⚠️ 易错点：双亲委派的"父"指父加载器不是父类；获取 Bootstrap 返回 null；自定义加载器应重写 `findClass` 而非 `loadClass` 以遵守该机制

### Q：双亲委派模型有什么好处？哪些场景破坏了它？
- 好处：安全（核心类只能由 Bootstrap 加载，防止伪造核心类篡改）、避免重复加载（同一类只加载一次）、保证类型一致性
- 破坏场景：
  - SPI/JDBC：核心接口由 Bootstrap 加载，实现类在 classpath，通过**线程上下文类加载器**加载实现类
  - Tomcat：每个 webapp 用独立 WebappClassLoader，优先加载自己目录的类，实现应用隔离
  - OSGi/热部署：网状类加载结构，支持模块化与类替换
> ⚠️ 易错点：JDBC 破坏双亲委派用的是"线程上下文类加载器"（`Thread.currentThread().getContextClassLoader()`），而非直接用 AppClassLoader

### Q：哪些情况会触发类的初始化（主动引用）？
- 主动引用（触发初始化）：new 对象、访问/修改静态字段（非 final 常量）、调用静态方法、反射调用（Class.forName）、初始化子类时父类若未初始化则先初始化父类、JVM 启动主类
- 被动引用（不触发）：通过子类访问父类静态字段、定义数组（`Foo[] arr`）、调用编译期常量
> ⚠️ 易错点：`final static` 编译期常量在准备阶段已赋值，访问它不触发初始化

### Q：常见的 JVM 调优参数有哪些？
- 堆：`-Xms`（初始堆）、`-Xmx`（最大堆），建议相等避免扩容抖动
- 新生代：`-Xmn` 或 `-XX:NewRatio`（老年代:新生代）；`-XX:SurvivorRatio` 设 Eden:Survivor
- 元空间：`-XX:MetaspaceSize`（达到该值触发 Full GC 并首次回收元空间）、`-XX:MaxMetaspaceSize`（上限）
- 收集器：`-XX:+UseG1GC`、`-XX:+UseParallelGC`、`-XX:+UseConcMarkSweepGC`
- 其他：`-XX:MaxTenuringThreshold`（晋升年龄）、`-Xss`（线程栈大小）
> ⚠️ 勘误：`-XX:MetaspaceSize` 不是元空间初始大小，而是达到该值触发 Full GC 并首次回收元空间；`-XX:NewRatio=4` 表示老年代:新生代=4:1；`-XX:MaxPermSize`（设置永久代）在 JDK8 已废弃无效，应用 `-XX:MaxMetaspaceSize`

### Q：常见的 OOM 类型有哪些？如何排查？
- Java heap space：堆内存溢出，最常见，多为内存泄漏或大对象
- Metaspace：元空间溢出，常因动态生成大量类（CGLIB、频繁热部署）
- StackOverflowError：栈深度超限，多见于递归未终止
- unable to create new native thread：无法创建新线程（线程数或系统资源超限）
- Direct buffer memory：直接内存溢出，NIO 堆外内存未释放
- **排查**：启动加 `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump（或 `jmap -dump` 手动导出）；用 MAT 分析 dump 找最大对象和 GC Roots 引用链；`jstat -gc` 观察各区域变化；`jstack` 查看线程栈
> ⚠️ 易错点：StackOverflowError 属于 Error 但归类为栈溢出而非 OOM；`GC overhead limit exceeded` 是 OOM 的前置预警；jmap dump 在生产环境慎用会 STW，优先用 HeapDumpOnOutOfMemoryError 自动 dump

### Q：内存泄漏和内存溢出有什么区别？
- 内存泄漏：对象不再使用但仍被引用无法回收，逐渐占用内存
- 内存溢出（OOM）：内存不够分配，程序无法继续运行
- 关系：长期泄漏最终导致溢出；溢出也可能是瞬时分配过大对象
- 泄漏常见场景：静态集合不断 put、未关闭资源、ThreadLocal 未 remove、监听器未注销
> ⚠️ 易错点：Java 中内存泄漏指"无意识的对象保留"，因 GC 自动回收故不像 C 那样直接丢失指针

### Q：Full GC 频繁如何排查？线上 CPU 飙高如何排查？
- **Full GC 频繁**：用 `jstat -gc` 观察老年代/元空间增长速率和 Full GC 次数；老年代持续膨胀（大对象过多、内存泄漏、Survivor 太小导致过早晋升）；元空间超限（调大 MaxMetaspaceSize 或排查类加载泄漏）；是否显式调用 `System.gc()`（可加 `-XX:+DisableExplicitGC` 禁用）
- **CPU 飙高**：
  1. `top` 定位占用 CPU 最高的 Java 进程 pid
  2. `top -Hp pid` 定位该进程中占用 CPU 最高的线程 id（十进制）
  3. `printf "%x\n" 线程id` 转为十六进制
  4. `jstack pid | grep 十六进制nid -A 30` 查看该线程堆栈，定位代码位置
- 常见原因：死循环、频繁 GC、大量计算、锁竞争
> ⚠️ 易错点：CPU 高不一定是业务问题，可能是频繁 GC 导致（用 jstat 看 GC 频率）；线程 id 必须转十六进制与 jstack 的 nid 匹配；`System.gc()` 会触发 Full GC 但非必须执行

### Q：jinfo、jmap、jstack、jstat 各有什么用途？如何查看 GC 日志？
- jinfo：实时查看/修改 JVM 参数（`jinfo -flag 参数 pid`）
- jmap：堆内存信息，`jmap -heap` 看堆配置、`jmap -dump` 导出 dump 文件
- jstack：线程栈信息，排查死锁、CPU 高、线程阻塞（`jstack -l` 可检测死锁）
- jstat：GC 统计，`jstat -gc` 看各区域容量与使用量变化
- 图形化：jconsole、jvisualvm、Arthas（在线诊断，功能最强）
- **GC 日志**：JDK8 启动加 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log`；JDK9+ 用 `-Xlog:gc*=info:file=gc.log` 统一日志框架；工具 GCViewer、GCEasy 分析
> ⚠️ 易错点：jinfo 只能修改部分 Manageable 参数；jmap dump 在生产环境会 STW；JDK9 前后 GC 日志格式完全不同，旧版 `-XX:+PrintGCDetails` 在 JDK9+ 失效

### Q：什么是逃逸分析？对象一定分配在堆上吗？
- 逃逸分析分析对象作用域是否逃出方法或线程
- 方法逃逸：对象被外部引用（赋静态字段、返回、传参）；线程逃逸：被其他线程访问
- 未逃逸对象可优化：标量替换、栈上分配、同步消除
> ⚠️ 易错点：对象并非一定在堆上分配；HotSpot 目前实际未直接做栈上分配，而是用"标量替换"将对象拆成基本类型分散在栈上，达到等效效果

### Q：什么是 JIT 即时编译？
- JVM 混合执行：解释器快速启动 + JIT 编译器（C1/C2）编译热点代码提升性能
- 热点探测：基于方法调用计数器与回边计数器，超过阈值触发编译
- C1（Client）：快速编译，简单优化；C2（Server）：深度优化，性能更好
- 分层编译（Tiered）：先用 C1，热点用 C2，兼顾启动速度与峰值性能
> ⚠️ 易错点：JIT 编译存在开销，需达到编译阈值才触发（`-XX:CompileThreshold`）；Java"一次编译到处运行"指字节码，JIT 才是运行时优化

---

## 资料勘误与重点提醒

> ⚠️ 两份参考资料（java面试宝典、咕泡大厂高频面试题）中识别出的错误与需澄清的高频考点，集中汇总如下。已在前文各题 `⚠️` 行就地修正，此处再强调。

### 一、Java 基础类勘误
1. **Integer 缓存不是常量池**：资料称缓存对象"引用常量池"，实为 `Integer` 内部 `IntegerCache` 私有静态内部类（数组），范围 [-128,127]，上限可由 `-XX:AutoBoxCacheMax` 调。
2. **String 底层数组**：资料引用 `char[]` 是 JDK8 及之前版本；**Java 9+ 改为 `byte[]` + `byte coder`**（compact strings）。
3. **`new String("a")` 创建对象数**：资料称"一定两个"，正确是"**1 或 2 个**"（取决于字面量是否已在常量池）。
4. **字符串常量池位置**：资料称"静态区"，**JDK7 起已从永久代移到堆中**。
5. **接口无实例变量**：资料称"接口实例变量默认 final"概念错误，接口所有字段都是隐式 `public static final` 常量。
6. **多态**：资料将重载称"编译时多态"，严格说重载是静态分派，非 OOP 真正多态；重写才是运行时多态核心。
7. **finalize 已废弃**：Java 9 起 `@Deprecated`，不可靠不推荐。

### 二、集合类勘误
8. **ArrayList JDK1.8 默认空数组**：无参构造默认容量 0，首次 add 才初始化为 10，不是"默认 10"。
9. **LinkedList 是双向链表非循环**：头节点 prev、尾节点 next 均为 null。
10. **HashMap 树化双条件**：链表长度 ≥8 **且数组容量 ≥64**，缺一不可；退化阈值是 6 不是 8。
11. **HashMap 并发**：1.7 头插法死循环，1.8 尾插法解决死循环但**仍非线程安全**（数据覆盖）。
12. **ConcurrentHashMap 1.8 锁粒度是桶节点（Node）**，不是 Segment；CAS 只用于空桶首次插入。

### 三、并发类勘误
13. **"Lock 是乐观锁"说法错误**：ReentrantLock 本质是独占式**悲观锁**，只是用 CAS（乐观技术）做快速抢锁；真正的乐观锁是 Atomic 类/StampedLock 乐观读。
14. **AQS 是双向链表**（CLH 变种），不是单向。
15. **volatile 内存屏障**：不是笼统"写前后各加两个"，而是 StoreStore/StoreLoad（写）+ LoadLoad/LoadStore（读）四类屏障组合，StoreLoad 开销最大。
16. **ThreadLocal 泄漏的是 value**（强引用），不是 key；key 用弱引用是为减小泄漏但仍不彻底，必须 `remove()`。
17. **synchronized 锁升级是 JDK1.6 引入的优化**，不是一开始就有；JDK15 后偏向锁被废弃。

### 四、JVM 类勘误
18. **方法区 ≠ 永久代**：方法区是规范，永久代（JDK7 前）与元空间（JDK8+）是 HotSpot 两种实现；**元空间在本地内存不在堆**。
19. **元空间 OOM 与永久代**：字符串常量池 JDK7 就从永久代移到堆；`-XX:MaxPermSize` 在 JDK8 已废弃。
20. **`-XX:MetaspaceSize` 不是元空间初始大小**，而是达到该值触发 Full GC 并首次回收元空间。
21. **G1 算法**：Region 间复制 + Region 内整理，整体是复制+标记整理，不是简单"标记-整理"。
22. **类加载准备阶段赋"零值"**，非初始值；`final static` 编译期常量才在此阶段赋初值。
23. **Bootstrap 加载器获取返回 null**（不是 AppClassLoader）；双亲委派的"父"是父加载器不是父类。
24. **JDBC 破坏双亲委派**用的是**线程上下文类加载器**，不是直接用 AppClassLoader。
25. **对象不一定分配在堆上**：逃逸分析 + 标量替换可将对象拆解到栈上。

### 五、重点补充（资料常漏的高频点）
- switch 不支持 long/float/double/boolean；Math.round 对负数 .5 向大取整（-11.5 → -11）。
- HashMap hash 扰动（高低 16 位异或）、容量为何 2 的幂次方（(n-1)&hash 等价取模且更快）。
- ConcurrentHashMap 为何禁止 null（并发下 containsKey 无法消除歧义）。
- 线程池工作流程顺序（核心→队列→非核心→拒绝）、两类 OOM 根源（队列无界 vs 线程数无界）。
- sleep 不释放锁 / wait 释放锁；CountDownLatch 不可重置 vs CyclicBarrier 可重置。
- CPU 飙高排查：top → top -Hp → printf 十六进制 → jstack grep nid。
- ABA 问题与 AtomicStampedReference；CAS 只保证单变量原子性，高竞争用 LongAdder。

> 📌 **使用建议**：本文是速查题集，遇到题目先看要点自答再对照。需深入原理的（如 HashMap 源码、AQS 源码、JVM 调优实战）请到 `01-java基础`、`02-JVM虚拟机` 对应章节。面试时把「要点 + 一个例子 + 易错点」讲清楚即可，不必背整段。
