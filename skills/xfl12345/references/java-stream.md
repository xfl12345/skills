---
name: java-stream
description: 写、改、审 Java（.java）的集合加工 / 数据流水线时套用：优先 Stream API 表达「声明式数据流」，何时该用、何时禁用、常见误用与性能红线。凡 .java 里出现集合遍历加工、filter/map/collect、并行流、Stream 相关 API 就触发，即使用户没提规范。
---

# Java Stream

## 心法

Stream 是声明式流水线：**描述“做什么”，不写“怎么做”**。中间操作惰性执行，终端操作一次性驱动整条链。它的价值在「意图一眼可读 + 中间集合零落地」，代价是栈深一层——所以它不是免费的默认值，下面先划边界。

**双重目标缺一不可：降低出错可能性，且提升可读性。** 二者冲突时以可读性优先——嵌套收集器、内联 lambda 塞满参数这类「看似高级」的写法让读者做脑内类型展开，是开倒车；拆成具名局部变量（中间结果、比较器各占一名），每步一个动作，永远好过一行炫技。

落到可执行判据：

- **行尾判据**：每个语句读到行尾都不需要回头做脑内展开；
- **逐层变平**：外层拆干净了还不够，参数位、lambda 体里塞着完整子流水线照样算复合表达式——拆到每一层都平；
- **同源保持流形态**：若整段本是一条同源数据流，让它以流的形态走完，让结果容器作为流水线的产物诞生；中途 new 空壳再逐个回填，是「怎么做」倒灌；
- **顺序即语义**：中间操作的先后本身就是能力——能用上游操作的语义免费拿到的性质，不要靠下游加工补

## 什么时候用 Stream

满足以下任一形态时优先 Stream：

- **过滤 → 变换 → 收集 / 消费** 的数据加工（最典型场景；终点要单一容器用收集器，要逐元素动作或多集合分拣用 `forEach` 收尾，见第 5 节）；
- 多级加工需要**中间结果复用**（`toList()` 落地一次，后续继续流）；
- 查找类语义（`findFirst` / `anyMatch` / `noneMatch`），比手写循环 + break 意图清晰

```java
// ✅ GOOD: 声明式三段论——过滤、映射、收集，一行讲完意图
List<String> activeNames = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .toList();

// ✅ GOOD: 中间落地复用
List<Order> paidOrders = orders.stream().filter(Order::isPaid).toList();
Set<Long> userIds = paidOrders.stream().map(Order::getUserId).collect(toSet());
```

## 什么时候禁用

- **中间操作改写本地共享状态、干扰流源头** → 禁止，无干扰是 Stream 正确性的地基。远程调用不在此列：为拿返回值继续加工的留 `map`（见第 6 节），为产生效果、不回收结果的走末端 `forEach`（见第 5 节）
- **需要 `break` / 早退的遍历** → 用查找类短路操作（`anyMatch` 等）；都不匹配就老老实实 for
- **异常抛出密集的逻辑**（受检异常处理）→ for 更直白；Stream 里 try/catch 会把主逻辑淹没在样板里
- **单层简单遍历**（只是打印或逐个调用 setter）→ 只读场景 `Iterable.forEach` 直接消费（`users.forEach(System.out::println)`），套 `.stream()` 纯属仪式感；普通 for 只留给「遍历中修改集合」这类 forEach 接不住的活

## 高频正反例

> 示例中裸写的 `groupingBy` / `toMap` / `toSet` / `toCollection` / `identity` / `collectingAndThen` 等均为 `java.util.stream.Collectors` 的静态工厂方法，`comparing` 即 `Comparator.comparing`、`mapConcurrent` 即 `Gatherers.mapConcurrent`（Gatherer 体系 Java 24 转正）——为保流水线干净可读，业务代码对它们做静态导入：

```java
import static java.util.Comparator.comparing;
import static java.util.stream.Collectors.*;
import static java.util.stream.Gatherers.mapConcurrent;
```

> `.toList()` 有两个身份，按形态自辨：跟在链尾的无参 `.toList()` 是 Stream 的实例方法（Java 16+）；出现在 `collect(...)` 参数位的收集器工厂则来自上面的通配静态导入（见第 2 节 BAD 1）。

### 1. collect vs toList()

Java 16+ 优先 `.toList()`（不可变、天然防下游篡改）；需要 `toMap` / `groupingBy` / `partitioningBy` 或指定容器时才用 `Collectors.*`：

```java
// ✅ 默认收集
List<User> adults = people.stream().filter(u -> u.age() >= 18).toList();

// ✅ 需要键值结构 / 分组时才请 Collectors 出场
Map<Long, List<Order>> byUser = orders.stream().collect(groupingBy(Order::getUserId));

// ❌ BAD: 只是收个 List 却动用 Collectors.toList()
List<String> bad = names.stream().collect(Collectors.toList());
```

### 2. 桶内排序：sorted 放在 groupingBy 前

```java
// 前置：两种 BAD 共用的中间态
Comparator<Order> byCreatedAtDesc = comparing(Order::getCreatedAt).reversed();
Map<Long, List<Order>> grouped = orders.stream().collect(groupingBy(Order::getUserId));

// ❌ BAD 1: collectingAndThen 嵌套——分组与排序挤进一个 collect，嵌套 lambda 全靠脑内展开
Map<Long, List<Order>> badNested = orders.stream()
    .collect(groupingBy(Order::getUserId,
        collectingAndThen(
            toList(),
            bucket -> bucket.stream().sorted(byCreatedAtDesc).toList())));

// ❌ BAD 2: entrySet 重流二次 collect(toMap)——为改桶内顺序多出一整趟搬运
Map<Long, List<Order>> badReflow = grouped.entrySet().stream()
    .collect(toMap(Map.Entry::getKey, e -> {
        return e.getValue()
            .stream()
            .sorted(byCreatedAtDesc)
            .toList();
    }));

// ❌ BAD 3: 只是想桶内有序，但是破坏了闭包纯净。损失了并行安全性和产物只读性。
Map<Long, List<Order>> badBackfill = new HashMap<>();
grouped.forEach((userId, bucket) -> {
    var sorted = bucket.stream().sorted(byCreatedAtDesc).toList();
    badBackfill.put(userId, sorted);
});
```

```java
// ✅ GOOD: 先排序再分组，一次 collect 完成，桶内天然有序
Map<Long, List<Order>> sortedOrdersByUser = orders.stream()
    .sorted(byCreatedAtDesc)
    .collect(groupingBy(Order::getUserId));
```

> 一行讲完意图。`groupingBy` 的桶按遭遇序收集元素（下游 `toList()` 同样保序），所以 `sorted` 放在前面，桶内即有序（顺序即语义）。代价仅是全局排序替代分桶各自排序，复杂度量级不变。

### 3. toMap 的合并函数不是可选项

重复 key 不给 merge 函数直接 IllegalStateException——这是运行时炸弹，不是风格问题：

```java
// ❌ BAD: key 重复即炸
Map<Long, Order> bad = orders.stream().collect(toMap(Order::getUserId, identity()));

// ✅ GOOD: 显式声明冲突策略
Map<Long, Order> ok = orders.stream().collect(toMap(Order::getUserId, identity(), (a, _) -> a));
```

### 4. 数值流避免装箱

数值聚合走特化流，别让每个元素都装箱成 `Integer` 再拆箱：

```java
// ❌ BAD: boxed 后 reduce，全程装箱
int bad = list.stream().reduce(0, Integer::sum);

// ✅ GOOD: mapToInt 特化流
int total = list.stream().mapToInt(Order::getAmount).sum();
```

### 5. forEach 的地盘：无产物收尾，多集合分拣

forEach 的本职是**复用循环**：流水线滤完转完，末端就地消费，不再落地中转。用不用它，看末端产物：

- **没有产物**（逐元素写库、发送、通知）→ `forEach` 收尾，复用流水线现成的循环
- **单一集合** → 收集器交差（`toList` / `toMap` / `toSet` / `collect` 都行），轮不到 forEach 回填空壳
- **多个集合**（分拣：出口类型各异、各有名字）→ `forEach` 配 `switch` 逐元素归位——闭包引用各出口正是「真的需要修改外部集合」的正当场景；同构分桶找 `groupingBy`（见第 2 节）

```java
// ❌ BAD: 收完即遍历——中转 List 没人读第二遍，落地纯属浪费。而且还徒增一次遍历。
var paidOrders = orders.stream().filter(Order::isPaid).toList();
for (var order : paidOrders) {
    orderRepository.save(order);
}

// ✅ GOOD: 不落地，流水线走完直接 forEach 消费。复用循环，一次遍历。
orders.stream().filter(Order::isPaid).forEach(orderRepository::save);

// ❌ BAD: 产物明明只有一个 List，却 new 空壳靠 forEach 回填——破坏闭包纯净，丢掉产物只读性
List<String> readonlyActiveNames = new ArrayList<>();
users.stream()
    .filter(User::isActive)
    .forEach(u -> readonlyActiveNames.add(u.getName()));

// ✅ GOOD: 只读场景——产物由流水线生，落地即只读
List<String> activeNames = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .toList();

// ✅ GOOD: 可写场景——收集器也能指定可写容器，轮不到 forEach 回填
List<String> writableActiveNames = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .collect(toCollection(ArrayList::new));

// ✅ GOOD: 多集合分拣——switch 按类型归位，各出口就地落地；收集器只产单一容器，装不下类型各异的多出口
events.stream().filter(Event::isValid).forEach(event -> {
    switch (event) {
        case ClickEvent e -> clickEvents.add(e);
        case ViewEvent e -> viewEvents.add(e);
        case ErrorEvent e -> errorEvents.add(e);
    }
});
```

### 6. 远程调用：拿值留 map，扇出用 mapConcurrent

为拿返回值而调远程接口是 `map` 的合法职责（效果型调用走第 5 节 `forEach`），但串行等待是白挨的延迟——虚拟线程让并发无论多少都有收益。要并发也别退回前 Gatherer 时代的手搓扇出：

```java
// ❌ BAD: 传统 CompletableFuture 扇出——执行器开关自理、List 倒腾多趟、全量发射不设限，意图被样板淹没
List<UserProfile> badProfiles;
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<CompletableFuture<UserProfile>> futures = userIds.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> profileClient.get(id), executor))
        .toList();
    CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();
    badProfiles = futures.stream().map(CompletableFuture::join).toList();
}

// ✅ GOOD: mapConcurrent 为阻塞 I/O 扇出而生——虚拟线程承载、限并发、结果保序
List<UserProfile> profiles = userIds.stream().gather(mapConcurrent(8, profileClient::get)).toList();
```

> 想攒批再调（如 100 个 ID 一次 RPC）看 `Gatherers.windowFixed`：先把流切成定长窗口，再逐窗发请求——Gatherer 体系正是自定义中间操作的官方扩展点

## 性能红线

- **小集合别并行**：`parallelStream` 有任务拆分与线程池调度开销，元素少反而更慢；且默认吃公共 ForkJoinPool，Web 容器里容易互相拖累
- **并行流的元素必须是「重」的**：要么量大、要么单元素计算昂贵，否则得不偿失。拿不准就串行——并行永远是需要论证的决定，不是加速开关
- **NIO 文件流记得关**：`Files.list` / `Files.walk` 返回目录句柄流，必须 try-with-resources，否则文件描述符泄漏
