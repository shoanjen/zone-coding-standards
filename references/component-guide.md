# 组件规范详述（四条）

「组件」指一切可复用的封装单元：类、Service、Manager、工具类、方法、配置化的处理器等，不绑定特定框架形态。每条规则按「规则陈述 → 反例（错在哪）→ 正例（对在哪）→ 判定问句 → 边界情况」展开。涉及组件沉淀与返回值收敛处给 Spring Boot 写法。

---

## 规则一：沉淀通用组件——重复逻辑参数化沉淀

相同或相似的逻辑操作/操作集合出现 **≥2 次**，必须把差异点参数化，沉淀为通用组件；调用方通过参数注入差异，不再复制粘贴。

### 反例

```java
// ❌ 同一段手机号校验在三个 Service 里复制了三份
// OrderService.java
private boolean validPhone(String phone) {
    return phone != null && phone.matches("^1\\d{10}$");
}
// MemberService.java —— 一模一样又写了一遍
// RefundService.java —— 第三遍
```

错在哪：三份副本将来改校验规则要改三处，漏一处就是线上事故；复制即分叉的开始。

### 正例

```java
// ✅ 沉淀为通用组件（Spring Boot 场景），调用方注入使用
@Component
public class PhoneValidator {
    private static final Pattern CN_MOBILE = Pattern.compile("^1\\d{10}$");

    public boolean isValid(String phone) {
        return phone != null && CN_MOBILE.matcher(phone).matches();
    }
}

@Service
public class OrderService {
    @Resource
    private PhoneValidator phoneValidator;   // 字段注入（见 springboot-guide.md）
    // ... 使用 phoneValidator.isValid(...)
}
```

对在哪：校验逻辑只有一份真相；正则预编译；差异点（如是否支持座机）可再参数化。

**判定问句**：这段逻辑第三次出现时，你是复制还是引用？复制就是违规信号。

### 边界情况

- 不过早抽象：只出现一次的逻辑不强制沉淀；出现两次但明显会分叉（各场景走向不同）的，也不强行合并——参数越堆越多说明它不该是一个组件。
- 沉淀时必须去业务特化：把写死的阈值、格式、文案改为参数、策略或配置，否则「通用组件」只是换了个地方的复制。
- 沉淀粒度自下而上判断：单方法 → 工具方法；一组相关操作 → 组件（@Component/@Service）；跨项目复用 → 独立模块。

---

## 规则二：单一职责——一个组件只回答一个问题

一个组件只负责一件事，做到高内聚（内部数据与方法紧密相关）低耦合（组件间只通过明确的方法签名交互）。**描述职责时说出了「并且」，就是拆分信号。**

### 反例

```java
// ❌ 一个 Service 同时承担计算与导出
@Service
public class OrderService {
    public OrderCalcResult calc(Order order) { ... }        // 职责 1：金额计算
    public byte[] exportExcel(List<Order> orders) { ... }   // 职责 2：Excel 导出
}
```

错在哪：职责描述必须用「并且」才能说完；计算逻辑改动和导出格式改动都要动同一个类，互相牵连。

### 正例

```java
// ✅ 按职责拆开，各自只回答一个问题
@Service
public class OrderCalcService { ... }      // 只回答：这个订单金额怎么算

@Service
public class OrderExportService { ... }    // 只回答：订单怎么导出成 Excel
```

对在哪：每个组件只有一个变化理由；导出格式调整不影响金额计算。

低耦合的落点：组件之间只通过方法签名交互——不互相摸内部状态、不调用对方 private 逻辑、不共享可变静态变量；共享数据走参数、返回值或事件。

**判定问句**：用一句话描述这个组件的职责，需要「并且」吗？需要 → 拆。

### 边界情况

- 职责相近的私有辅助方法可以共存于一个组件内（只要对外仍是一个问题），拆分针对的是对外职责。
- 「Facade/门面」组合多个组件对外提供流程编排，职责是「编排流程」，不算违反单一职责；但编排内部不许混入具体业务计算。

---

## 规则三：返回值收敛——返回值只与功能相关

组件的返回值只承载其职责范围内的结果，禁止夹带与功能无关的副产物：内部实现细节（缓存命中与否）、调试信息（耗时字符串）、环境信息（当前操作人）等。副产物走日志、监控、事件或独立方法。

### 反例

```java
// ❌ 查询组件的返回值什么都塞
public Map<String, Object> queryOrders(OrderQuery query) {
    Map<String, Object> result = new HashMap<>();
    result.put("orders", orders);
    result.put("cacheHit", cacheHit);          // 内部实现细节
    result.put("elapsed", elapsedMs + "ms");   // 调试信息
    result.put("operator", currentUser);       // 与查询功能无关
    return result;
}
```

错在哪：调用方被迫解析一锅大杂烩；`Map<String, Object>` 让编译器与 IDE 全部失明；夹带的字段暗示组件职责不清（查询的顺带干起了统计）。

### 正例

```java
// ✅ 返回值只与查询功能相关，类型明确
public List<Order> queryOrders(OrderQuery query) { ... }

// 副产物各走各路：
// 耗时/缓存命中 → AOP 切面、日志、MDC、监控埋点
// 确有调用方需要缓存命中状态 → 单独提供方法，如 queryOrdersWithCacheInfo(query)
```

对在哪：返回类型即契约；谁需要什么，走独立的显式通道，而不是都挤进一个返回值。

**Spring Boot 统一返回体场景**：`Result<T>` 的 `T` 只装业务数据；统计信息走 AOP/MDC/埋点，不塞进 `T`。业务组件自身返回纯业务对象，包装成 `Result` 是接入层（Controller）的事。

**判定问句**：返回值里有没有调用方根本不需要的字段？调用方拿返回值只做本组件职责内的事，够吗？

### 边界情况

- 分页查询返回 `Page<Order>`（数据 + 分页元信息）不算夹带——分页元信息是查询功能的一部分。
- 事件/回调组件返回处理是否成功等状态信息是功能的一部分，不是夹带。

---

## 规则四：中间逻辑私有化——「步骤」必须 private

组件中的中间逻辑方法必须 private，不能跨包或跨类使用。两类典型场景：

**场景一（组件内部）**：通用组件的某个方法片段太长，需要抽取中间逻辑作为单独的方法——抽取出的方法是实现步骤，必须 private。

```java
// ❌ 300 行 public 方法，所有中间逻辑裸露在一个方法体里
public RefundResult refund(RefundRequest request) {
    // ……100 行参数与资格校验……
    // ……100 行退款金额计算……
    // ……100 行结果组装与落库……
}

// ✅ 主方法只留骨架，中间逻辑抽成 private（各方法入参均 ≤1，符合入参对象化）
public RefundResult refund(RefundRequest request) {
    validateRefundable(request);
    return assembleRefundResult(request);
}
private void validateRefundable(RefundRequest request) { ... }
private BigDecimal calcRefundAmount(RefundRequest request) { ... }
private RefundResult assembleRefundResult(RefundRequest request) {
    BigDecimal refundAmount = calcRefundAmount(request);   // 步骤内调用计算
    // ……组装与落库……
}
```

错在哪：裸 300 行读不动、测不了；抽出的中间方法若设为 public，会变成绕过主流程的「后门」调用路径。

对在哪：public 方法即组件能力清单（一眼看清这个组件会什么）；步骤方法 private，内部自由重构不惊动调用方。手册关联：单个方法推荐不超过 80 行。

**场景二（组件间）**：A 组件需要使用 B 组件中的某个方法的结果做逻辑处理，且处理逻辑过长，需要在 A 组件中单独作为一个方法——这个后处理方法是 A 的步骤，写成 A 的 private 方法；不回塞给 B，不散落在 A 的调用现场。

```java
// ❌ 两种错误走向
public class PayService {                     // B 组件
    public PayResponse pay(Order order) { ... }
    public PayRecord toPayRecord(PayResponse response) { ... }  // 错 1：为 A 特意暴露的中间方法
}
public class PayFacade {                      // A 组件
    public PayVO payAndShow(Order order) {
        PayResponse response = payService.pay(order);
        // 错 2：50 行后处理直接摊在调用现场
        ...
    }
}

// ✅ 后处理归 A，是 A 的 private 步骤
public class PayService {                     // B 组件保持纯净
    public PayResponse pay(Order order) { ... }
}
public class PayFacade {                      // A 组件
    public PayVO payAndShow(Order order) {
        PayResponse response = payService.pay(order);
        return assemblePayVO(response);       // 骨架一行，细节下沉
    }
    private PayVO assemblePayVO(PayResponse response) { ...50 行后处理... }
}
```

错在哪：B 为特定调用方暴露中间方法，等于把 A 的实现细节寄生进 B，B 的接口从此不稳定；后处理摊在调用现场则让流程读不成句。

对在哪：B 不为 A 改接口；A 的 public 方法保持流程骨架，长后处理下沉为 A 的 private 步骤。

**收敛判定**：private 中间方法被外部调用了，只有两种出路——
1. 它其实是组件的**公开能力**：升格——按能力重命名、进接口、写 javadoc，成为正式 API；
2. 它只是**步骤**：收回 private，调用方自己组织自己的步骤。

**判定问句**：这个方法是组件的「能力」（对外承诺）还是「步骤」（内部实现）？步骤必须 private。

### 边界情况

- Spring 生态的入口方法由框架调用（如 `@PostConstruct`、`ApplicationRunner.run`），可见性遵循框架约定，不算「跨类暴露」违规。
- 设计给子类扩展的模板方法骨架（protected 钩子方法）是刻意的扩展点契约，不属于中间逻辑泄漏；但钩子必须有 javadoc 说明扩展契约。
