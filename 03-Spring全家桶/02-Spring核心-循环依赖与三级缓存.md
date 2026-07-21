# Spring核心：循环依赖与三级缓存

> 📌 **一句话理解**：Spring 用「三级缓存 + ObjectFactory 提前暴露」解决单例 Bean 的 Setter/字段注入循环依赖；其中三级缓存的 ObjectFactory 本质是为「AOP 延迟代理」与「循环依赖」共存而设计的折中——没有 AOP，两级缓存就够了。

---

## 核心概念

### 一、什么是循环依赖 ⭐

循环依赖（Circular Dependency）：两个或多个 Bean 在初始化时互相持有对方的引用，形成依赖环。

最典型的形式：

```java
@Component
class A {
    @Autowired
    private B b;   // A 依赖 B
}

@Component
class B {
    @Autowired
    private A a;   // B 依赖 A
}
```

执行流程：创建 A → A 发现需要 B → 去创建 B → B 发现需要 A → A 还在创建中…死循环。

链式更长的循环依赖也常见：`A → B → C → A`。

**与「正常依赖」对比**：

| 类型 | 依赖关系 | 创建顺序 |
|------|---------|---------|
| 正常依赖 | A → B → C | 创建 A 时自动依次创建 B、C，无环 |
| 循环依赖 | A → B → A | 形成「环」，无明确起点，无三级缓存则死循环 |

**类比理解**：正常依赖像「生产汽车需要轮胎、轮胎需要橡胶」——单向供应链；循环依赖像「A 厂要 B 厂的零件、B 厂要 A 厂的零件」——两边互相等对方先交货，谁也动不了。

---

### 二、Spring 能否解决各类循环依赖 ⭐⭐

Spring 并非能解决所有循环依赖。准确结论如下：

| 场景 | 能否解决 | 原因 |
|------|---------|------|
| 单例 Setter 注入（`@Autowired` 写在 setter 上） | ✅ 能 | 实例化阶段可先 new 出裸对象放入三级缓存，属性填充时再注入 |
| 单例 字段注入（`@Autowired` 写在字段上） | ✅ 能 | 原理同 Setter，先实例化再填充字段，三级缓存可提前暴露引用 |
| 构造器注入（`@Autowired` 写在构造器上） | ❌ 不能 | 实例化阶段尚未完成，连裸对象都没有，无可提前暴露之物 |
| prototype 作用域（多例） | ❌ 不能 | 每次都新建，不入缓存，无法提前暴露 |
| 单例 + 构造器注入 + `@Lazy` | ✅ 能 | `@Lazy` 注入代理，首次调用才真正 `getBean` |
| `@Async` + 循环依赖 | ⚠️ 有坑 | `AsyncAnnotationBeanPostProcessor` 不实现 `getEarlyBeanReference`，详见文末勘误 |

> ⚠️ **易错点**：很多旧资料说「字段注入不能解决循环依赖」——这是错的。Spring 5.0+ 字段注入的循环依赖同样能解，原理与 Setter 完全一致，都依赖三级缓存。

---

### 三、三级缓存详解 ⭐⭐⭐

源码位于 `DefaultSingletonBeanRegistry`，核心是三个 Map：

```java
public class DefaultSingletonBeanRegistry {
    // 一级缓存：完整成品单例 Bean（已完成属性填充 + 初始化）
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 二级缓存：提前暴露的「半成品」Bean（已实例化，可能已 AOP 代理，但未完成属性填充/初始化）
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    // 三级缓存：ObjectFactory 工厂（lambda，调用 getObject() 触发 getEarlyBeanReference，可能生成代理）
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
}
```

**类比理解**：

| 缓存 | 类比 | 作用 |
|------|------|------|
| 一级 `singletonObjects` | 成品仓库 | 存放「质检合格、包装完成」的最终 Bean，可直接发货 |
| 二级 `earlySingletonObjects` | 半成品货架 | 存放「已成型但未组装完成」的提前暴露对象，仅供「循环依赖时被对方引用」 |
| 三级 `singletonFactories` | 生产工单/图纸 | 存放「对象工厂 lambda」，需要时按图纸现场生产（可能含 AOP 代理），生产后搬到二级货架 |

**三级缓存结构图**：

```
                DefaultSingletonBeanRegistry
   ┌──────────────────────────────────────────────────────────┐
   │                                                          │
   │  一级 singletonObjects（成品仓库）                          │
   │  ┌────────────────────────────────────────────┐           │
   │  │  "userDao"     -> UserDaoImpl@成品代理       │           │
   │  │  "orderService"-> OrderServiceImpl@成品      │           │
   │  └────────────────────────────────────────────┘           │
   │                                                          │
   │  二级 earlySingletonObjects（半成品货架）                   │
   │  ┌────────────────────────────────────────────┐           │
   │  │  "A" -> A 的提前暴露对象（可能已 AOP 代理）     │           │
   │  └────────────────────────────────────────────┘           │
   │                                                          │
   │  三级 singletonFactories（生产工单/图纸）                    │
   │  ┌────────────────────────────────────────────┐           │
   │  │  "A" -> ObjectFactory（lambda）              │           │
   │  │       调用 getObject()                      │           │
   │  │         -> getEarlyBeanReference(A裸对象)    │           │
   │  │         -> 可能生成 A 的代理对象               │           │
   │  └────────────────────────────────────────────┘           │
   │                                                          │
   │  + singletonsCurrentlyInCreation（正在创建集合）             │
   │    {"A", "B"}                                            │
   │                                                          │
   └──────────────────────────────────────────────────────────┘
```

三个缓存的「生命周期」与「AOP 时机」密切相关，下面用流程图说清。

---

### 四、三级缓存解决循环依赖完整流程 ⭐⭐⭐

以 `A -> B -> A`，且 A 需要 AOP 代理为例（这是面试官最想听到的完整流程）：

```
 调用方              Spring 容器                       缓存状态
   │                                                    
   │── getBean("A") ──>│                                
   │                    │ 1. 一级查无、二三级查无 A         
   │                    │ 2. 标记 A "正在创建"（加入       
   │                    │    singletonsCurrentlyInCreation）
   │                    │ 3. 实例化 A（new 出裸对象，未代理）
   │                    │ 4. 把 A 的 ObjectFactory 入三级缓存
   │                    │    singletonFactories.put("A",
   │                    │      () -> getEarlyBeanReference(A裸对象))
   │                    │                                
   │                    │── populateBean(A)              
   │                    │  发现依赖 B，递归调用          
   │                    │                                
   │                    │── getBean("B") ───────────┐   
   │                    │                            │   
   │                    │ 5. 一级查无 B、二三级查无 B  
   │                    │ 6. 标记 B "正在创建"        
   │                    │ 7. 实例化 B（裸对象）        
   │                    │ 8. B 的 ObjectFactory 入三级缓存
   │                    │                                
   │                    │── populateBean(B)              
   │                    │  发现依赖 A，递归调用          
   │                    │                                
   │                    │── getBean("A") ──────────┐   
   │                    │                          │   
   │                    │ 9. 一级查无 A、二级查无 A  
   │                    │    从三级缓存取 A 的 ObjectFactory
   │                    │    执行 getObject()        
   │                    │      -> 触发 SmartInstantiationAware
   │                    │         BeanPostProcessor 
   │                    │         .getEarlyBeanReference()
   │                    │      -> 生成 A 的代理对象   
   │                    │ 10. 代理 A 放入二级缓存     
   │                    │     earlySingletonObjects.put("A", 代理A)
   │                    │     三级缓存删除 A          
   │                    │                                
   │                    │<── 返回 代理 A              
   │                    │                                
   │                    │ B 拿到 A（代理 A），完成属性填充
   │                    │ B 完成初始化                
   │                    │ 11. B 进入一级缓存          
   │                    │     singletonObjects.put("B", B)
   │                    │     删除 B 的二三级缓存      
   │                    │                                
   │                    │<── 返回 B                  
   │                    │                                
   │ A 拿到 B，完成属性填充                                  
   │ A 完成初始化                                          
   │   (postProcessAfterInitialization 阶段               
   │    AbstractAutoProxyCreator 检测 wrappedBean 已存在,   
   │    复用已生成的代理 A，避免重复代理)                     
   │ 12. A 进入一级缓存                                    
   │     singletonObjects.put("A", 代理A)                  
   │     删除 A 的二三级缓存                                
   │                                                        
   │<── 返回 代理 A                                         
```

**关键点说明**：

1. **三级缓存存的是 ObjectFactory（lambda），不是 Bean 本身**：只有真正发生循环依赖、别的 Bean 来取 A 时，lambda 才执行 `getEarlyBeanReference` 提前生成代理。
2. **生成代理后立刻「升级」到二级缓存**：从三级缓存删除，避免后续重复执行 lambda 生成多个代理。
3. **B 拿到的 A 与最终 singletonObjects 中的 A 是同一个代理对象**：避免「B 用裸 A、其他地方用代理 A」的不一致问题。
4. **A 完成 `postProcessAfterInitialization` 时复用已生成的代理**：`AbstractAutoProxyCreator` 内部判断 `wrappedBean` 是否已存在（即被 `getEarlyBeanReference` 提前暴露过），若已生成则直接复用，不重复代理。

---

### 五、为什么是三级缓存而不是二级缓存？⭐⭐⭐

**这是面试官最常追问的问题。标准答案要点如下**：

#### 1. 没有 AOP 时，二级缓存就够了

如果完全不考虑 AOP，理论上只用两级缓存即可：

| 缓存层级 | 存什么 |
|---------|--------|
| 一级 `singletonObjects` | 完整成品 Bean |
| 二级 `earlySingletonObjects` | 提前暴露的半成品 Bean（裸对象） |

流程：实例化 A → 把 A 裸对象直接放入二级缓存 → 填充属性时去取 B → B 同样操作 → B 取 A 时从二级缓存拿到裸 A → B 完成进入一级 → A 完成进入一级。整个流程没有「代理」的参与，两级缓存足够。

#### 2. 三级缓存的 ObjectFactory 是为「AOP 延迟代理」而存在

Spring AOP 的设计原则是：**代理对象延迟到 `postProcessAfterInitialization` 阶段才生成**（即 Bean 生命周期最后一步）。这是 Spring 一贯的「懒代理」理念。

如果用二级缓存同时解决循环依赖 + AOP，必须在实例化后立刻生成代理放入二级缓存，这会带来两个问题：

| 问题 | 说明 |
|------|------|
| 违反懒代理设计 | 所有 Bean 一实例化就被强制代理，即便 99% 的 Bean 没有循环依赖也要提前代理 |
| 性能损耗 | 提前代理意味着多创建代理对象，对无循环依赖的 Bean 是无谓开销 |

#### 3. 三级缓存的本质：循环依赖时才「破例」提前暴露代理

`singletonFactories` 里存的 `ObjectFactory`（lambda）相当于「生产图纸」：

- **正常情况**：三级缓存的 ObjectFactory 永远不会被调用，Bean 正常走完生命周期，最后在 `postProcessAfterInitialization` 才生成代理。
- **发生循环依赖时**：别的 Bean 来取 A，三级缓存的 ObjectFactory 被触发，**临时**通过 `getEarlyBeanReference` 提前生成代理，并「升级」到二级缓存。

> ⚠️ **易错点**：千万不要说「二级缓存存未代理对象、三级缓存存代理对象」。正确说法：二级缓存 `earlySingletonObjects` 存的是「**可能已代理**」的提前暴露对象——是否代理取决于是否触发过 `getEarlyBeanReference`。

#### 4. 一句话总结

**三级缓存本质是为「AOP 延迟代理 + 循环依赖」共存而设计的折中方案**：没有 AOP 时二级缓存即可；有了 AOP 后，为了不破坏懒代理理念、又不让所有 Bean 都提前代理，于是引入 ObjectFactory 实现真正的「按需代理」。

---

### 六、构造器注入为何不能解决 ⭐⭐

回顾 Bean 创建三阶段：

```
1. 实例化（createBeanInstance）：调用构造器 new 出裸对象
2. 属性填充（populateBean）：处理 @Autowired/@Resource
3. 初始化（initializeBean）：调用 BeanPostProcessor 前后置、init-method
```

三级缓存解决循环依赖的前提是：**在「实例化」之后、能拿到裸对象，才能把 ObjectFactory 放入三级缓存**。

构造器注入的问题：**A 的构造器需要 B 作为参数，但此时 A 还在执行构造器（步骤 1 尚未完成），连「裸对象」都没诞生，无东西可暴露**：

```java
@Component
public class A {
    private final B b;

    public A(B b) {   // 此处需要 B，但 A 还在实例化中
        this.b = b;
    }
}
// A 的引用还不存在 → 无法放入三级缓存 → B 反过来需要 A 时拿不到
// → 抛 BeanCurrentlyInCreationException
```

#### 解决办法

| 方案 | 做法 | 原理 |
|------|------|------|
| 改用 Setter/字段注入 | 把构造器注入改为 `@Autowired` 字段或 setter | 走三级缓存正常流程 |
| `@Lazy` 注入 | `public A(@Lazy B b)` | 注入 B 的代理，首次调用代理方法时才真正 `getBean("B")`，此时 A 已实例化 |
| `@PostConstruct` 手动注入 | 在 `@PostConstruct` 方法中通过 `ApplicationContext.getBean(B.class)` 手动拿 | `@PostConstruct` 在初始化阶段执行，此时 A 已实例化 |
| 重新设计 | 抽出公共逻辑到第三个 Bean，让 A、B 都依赖 C，而非互相依赖 | 从源头消除环 |

**`@Lazy` 解决构造器循环依赖示例**：

```java
@Component
public class A {
    private final B b;

    // @Lazy 让 Spring 注入 B 的代理，而非真实 B
    public A(@Lazy B b) {
        this.b = b;
    }
}
```

注入的 `b` 是 CGLIB 代理，所有方法调用都被拦截，首次调用时才触发 `getBean("B")`，此时 A 早已实例化完成，能正常走三级缓存流程。

---

### 七、Spring 6 / Boot 3.x 变化 ⭐

> ⚠️ **重要变更**：从 **Spring Boot 2.6**（基于 Spring Framework 5.3.x）起，`spring.main.allow-circular-references` 默认值改为 `false`，默认禁止循环依赖；启动时若检测到循环依赖直接报错。此变更延续到 Spring Boot 3.x（Spring Framework 6.0）。

#### 配置项

```yaml
spring:
  main:
    allow-circular-references: true   # 显式开启，不推荐
```

底层对应 `AbstractAutowireCapableBeanFactory` 的 `allowCircularReferences` 字段，关闭后整个三级缓存机制不再生效。

#### 为什么默认禁止？

官方认为循环依赖是「设计坏味道」，往往意味着：
- 职责划分不清，组件边界混乱
- 应该重构为单向依赖，引入中间层
- 隐藏的初始化时序问题，难以排查

#### 兼容旧项目

迁移老项目时若启动报 `BeanCurrentlyInCreationException`，可临时设为 `true` 维持运行，但应尽快重构消除循环依赖。

> ⚠️ **修正说明**：部分资料称「Spring Framework 6.0 / Spring Boot 3.0 起默认禁循环依赖」。准确说法：**Spring Boot 2.6（基于 Spring Framework 5.3.x）**就已默认禁用，Spring 6 / Boot 3 只是延续此行为。

---

## 常见面试题

### 1. Spring 如何解决循环依赖？

**回答思路：** 三级缓存 + ObjectFactory 提前暴露 + 完整流程（以 A→B→A 为例）。

> Spring 通过 `DefaultSingletonBeanRegistry` 的三级缓存解决单例 Bean 的 Setter/字段注入循环依赖：
> - 一级 `singletonObjects`：成品 Bean
> - 二级 `earlySingletonObjects`：提前暴露的半成品（可能已代理）
> - 三级 `singletonFactories`：ObjectFactory 工厂
>
> 以 `A → B → A` 为例：
> 1. `getBean("A")`：三级缓存均无 A，标记 A 正在创建
> 2. 实例化 A（裸对象，未代理）
> 3. 将 A 的 `ObjectFactory` 放入三级缓存（lambda 会触发 `getEarlyBeanReference`）
> 4. `populateBean(A)`：发现依赖 B，递归 `getBean("B")`
> 5. 同样地，B 实例化后将 `ObjectFactory` 放入三级缓存
> 6. `populateBean(B)`：发现依赖 A，递归 `getBean("A")`
> 7. 此时 A 在三级缓存中，执行 `ObjectFactory.getObject()` → 触发 `getEarlyBeanReference` → 生成 A 代理 → 放入二级缓存，删除三级缓存
> 8. B 拿到代理 A，完成初始化，B 进入一级缓存
> 9. A 拿到 B，完成初始化，复用已生成的代理，A 进入一级缓存
>
> 关键点：三级缓存存的是「工厂」而非 Bean，只有真正发生循环依赖才执行工厂方法生成代理，符合 AOP 延迟代理的设计。

### 2. 为什么 Spring 用三级缓存而不是二级缓存？

**回答思路：** 区分「有无 AOP」两种场景，引出 ObjectFactory 的延迟代理作用。

> - 如果不考虑 AOP，两级缓存就够了：一级放成品、二级放半成品裸对象。
> - 但 Spring AOP 的设计原则是「代理延迟到 `postProcessAfterInitialization` 才生成」（懒代理）。
> - 如果用二级缓存解决循环依赖 + AOP，必须实例化后立刻生成代理放入二级缓存，这会破坏懒代理理念，且让所有 Bean（哪怕没循环依赖）都提前代理，性能差。
> - 三级缓存的 `ObjectFactory` 实现「按需代理」：只有发生循环依赖、别的 Bean 来取 A 时才执行 `getEarlyBeanReference` 提前生成代理；正常情况三级缓存不会被调用，Bean 走完正常生命周期才代理。
> - 本质上，三级缓存是为「AOP 延迟代理 + 循环依赖」共存而设计的折中方案。

### 3. 构造器注入的循环依赖能解决吗？为什么？怎么解决？

**回答思路：** 不能。原因：实例化阶段未完成。三种解法。

> 不能解决。
>
> 原因：构造器注入要求在「实例化阶段」就拿到依赖对象。但实例化阶段（步骤 1）尚未完成，A 的裸对象还没诞生，无法放入三级缓存提前暴露。B 反过来需要 A 时拿不到，抛 `BeanCurrentlyInCreationException`。
>
> 解决办法：
> 1. 改用 Setter 或字段注入，走正常三级缓存流程
> 2. `@Lazy` 注入：`public A(@Lazy B b)`，注入 B 的代理，首次调用才真正 `getBean("B")`
> 3. `@PostConstruct` 中通过 `ApplicationContext.getBean(B.class)` 手动拿
> 4. 重构：抽出公共逻辑到第三个 Bean，让 A、B 都依赖它

### 4. @Autowired 字段注入的循环依赖能解决吗？

**回答思路：** 能。原理同 Setter。

> 能解决。
>
> `@Autowired` 写在字段还是 setter 上，本质上都属于「属性填充」阶段（`populateBean`），在实例化之后才执行。三级缓存解决循环依赖的前提就是「实例化完成后能拿到裸对象」，所以字段注入同样适用。
>
> 注意：很多旧资料说「字段注入不能解循环依赖」，这是错的，Spring 5.0+ 字段注入的循环依赖完全能解。

### 5. 多例 prototype 作用域的循环依赖能解决吗？

**回答思路：** 不能。原因：不缓存。

> 不能。
>
> prototype 作用域每次 `getBean` 都新建一个实例，**不入缓存**。三级缓存机制完全依赖缓存提前暴露半成品对象，prototype 不缓存就无法提前暴露，因此无法解决。
>
> 解决办法：改用单例，或用 `@Lazy`。

### 6. Spring 6 默认还支持循环依赖吗？allow-circular-references 是干什么的？

**回答思路：** 默认不支持。配置项作用。

> 从 **Spring Boot 2.6** 起默认不支持循环依赖。`spring.main.allow-circular-references` 默认 `false`，启动时若检测到循环依赖直接抛 `BeanCurrentlyInCreationException`。
>
> 可显式设为 `true` 开启（兼容老项目），但官方不推荐——循环依赖是设计坏味道，应重构消除。
>
> Spring 6 / Boot 3 沿用此默认行为。

### 7. @Lazy 如何解决循环依赖？

**回答思路：** 注入代理，延迟 getBean。

> `@Lazy` 作用于注入点（字段或构造器参数）时，Spring 会注入一个 CGLIB 代理对象，而非真实 Bean。代理内部拦截所有方法调用，首次调用时才真正触发 `getBean("xxx")`。
>
> 此时被注入方（如 A）已完成实例化，能正常走三级缓存流程，循环依赖被「打破」。
>
> 适用场景：构造器注入的循环依赖（必加 `@Lazy`）、`@Async` 引起的循环依赖问题。

### 8. 三级缓存分别存什么？

**回答思路：** 三个 Map 的内容。

> - **一级 `singletonObjects`**：完整成品单例 Bean（已完成属性填充 + 初始化），可直接对外使用
> - **二级 `earlySingletonObjects`**：提前暴露的「半成品」Bean（已实例化，可能已 AOP 代理，但未完成属性填充/初始化），仅供循环依赖时被对方引用
> - **三级 `singletonFactories`**：`ObjectFactory<?>` 工厂（lambda），key=beanName，调用 `getObject()` 触发 `SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference`，可能生成代理

---

## 本章学习建议

1. **先理解 Bean 创建三阶段**：实例化 → 属性填充 → 初始化。三级缓存的「提前暴露」发生在实例化之后、属性填充之前，这是整个机制的时间窗口。
2. **背诵三个 Map 的定义**：一级成品、二级半成品（可能已代理）、三级 ObjectFactory 工厂。注意二级「可能已代理」，不要说成「未代理对象」。
3. **手画一遍 A→B→A 的完整时序图**：能默写出来基本就过面试。
4. **重点理解「为什么三级不是二级」**：AOP 延迟代理是核心动机，没有 AOP 二级就够。这是最常被追问的点。
5. **构造器注入不能解的原因**：实例化阶段未完成、裸对象不存在。配合 `@Lazy` 注入代理的解法一起记。
6. **Spring 6 / Boot 2.6+ 默认禁循环依赖**：现代项目应重构消除循环依赖，而非依赖三级缓存兜底。
7. **警惕 `@Async` + 循环依赖**：`AsyncAnnotationBeanPostProcessor` 不实现 `getEarlyBeanReference`，需 `@Lazy` 解。
8. **源码入口**：`AbstractBeanFactory.doGetBean` → `DefaultSingletonBeanRegistry.getSingleton` → 三级缓存查找逻辑；`SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference` 是提前代理的钩子。

> 💡 **学习心法**：三级缓存不是「为解决循环依赖而生的奇技淫巧」，而是 Spring 在「AOP 懒代理」与「循环依赖兼容性」之间做出的折中设计。理解了「ObjectFactory 延迟代理」这个动机，整个机制就豁然开朗。

---

## 资料勘误与重点提醒

### 一、易错点与勘误

1. **「字段注入不能解循环依赖」——错**。Spring 5.0+ 字段注入与 Setter 注入原理一致，都能被三级缓存解决。

2. **「二级缓存存未代理对象」——错**。二级缓存 `earlySingletonObjects` 存的是「可能已代理」的提前暴露对象。是否代理取决于是否触发过 `getEarlyBeanReference`。

3. **「Spring 6 / Boot 3.0 起默认禁循环依赖」——表述不严谨**。准确说法是 **Spring Boot 2.6**（基于 Spring Framework 5.3.x）起默认禁用，Spring 6 / Boot 3 只是延续此行为。

4. **「@Async 配合循环依赖，AsyncAnnotationBeanPostProcessor 实现了 getEarlyBeanReference 才解决」——错**。`AsyncAnnotationBeanPostProcessor` 继承自 `AbstractAdvisingBeanPostProcessor`，**并未**实现 `SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference`，它只在 `postProcessAfterInitialization` 阶段生成 @Async 代理。这意味着：
   - 若 A 同时有 AOP + @Async 且发生循环依赖：B 拿到的早期引用只有 AOP 代理（无 @Async 代理），而最终 singletonObjects 中的 A 是「AOP + @Async」嵌套代理——B 持有的引用与最终 Bean 不一致，异步行为丢失。
   - 实际项目中 `@Async` + 循环依赖通常启动就报错或行为异常，解法：用 `@Lazy` 延迟注入，或重构消除依赖环。
   - 对比：Spring AOP 的 `AbstractAutoProxyCreator` **实现了** `getEarlyBeanReference`，所以 AOP 代理能正确提前暴露，循环依赖才能配合 AOP 工作。

### 二、面试高频重点补充

1. **「半成品」的边界**：半成品 = 已实例化但未完成属性填充 + 初始化。如果半成品被 AOP 代理，那么这个代理对象本身是「半成品代理」，但代理内部的目标对象（裸 Bean）也还没填充完属性。

2. **`SmartInstantiationAwareBeanPostProcessor` 是关键扩展点**：
   - `getEarlyBeanReference(bean, beanName)`：循环依赖时提前暴露引用，可生成代理
   - `AbstractAutoProxyCreator` 实现了它，所以 Spring AOP 能配合三级缓存
   - 自定义 BPP 若想在循环依赖时正确暴露代理，也必须实现此接口

3. **`singletonsCurrentlyInCreation` 集合**：标记「正在创建中」的 Bean，与三级缓存配合检测循环依赖。构造器注入抛 `BeanCurrentlyInCreationException` 就是基于此集合判断。

4. **Spring 5.3 的 `allowCircularReferences` 字段**：在 `AbstractAutowireCapableBeanFactory` 中有一个 `allowCircularReferences` 开关，Spring Boot 2.6+ 通过 `spring.main.allow-circular-references` 配置它，默认 false 时关闭整个三级缓存机制。

5. **`@Configuration` 类的 CGLIB 代理**：full 模式的 `@Configuration` 类通过 `ConfigurationClassPostProcessor` 在 `BeanFactoryPostProcessor` 阶段就生成 CGLIB 代理，与三级缓存机制不冲突；但 `@Bean` 方法间的循环依赖仍走三级缓存流程。

6. **Spring 5.0 字段注入修复的细节**：早期 Spring（2.x/3.x）字段注入与构造器注入类似，存在循环依赖问题；自 Spring 5.0 起，字段注入通过 `AutowiredFieldElement` 在 `populateBean` 阶段统一处理，与 Setter 一样依赖三级缓存，彻底解决了字段注入的循环依赖。
