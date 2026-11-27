# Spring Boot 工程惯例详述（Lombok、依赖注入与事务）

三条本 skill 自定义的工程惯例：一条砍样板代码，一条定注入风格，一条定事务写法。手册对这些没有强制条款，按裁决链本规范即为约定。示例 Java + Spring Boot，跨电商、内容、支付、工具等多个业务场景。

---

## 规则一：Lombok 优先——样板代码用注解生成

能被 Lombok 注解替代的样板代码，**禁止手写**。对照表：

| 场景 | 手写（禁止） | Lombok（使用） |
|---|---|---|
| POJO 存取器 | 手写 N 对 getter/setter | `@Getter` / `@Setter`，全套用 `@Data` |
| toString | 手写拼接 | `@ToString`（敏感字段 `@ToString.Exclude`） |
| equals/hashCode | 手写 | `@EqualsAndHashCode` |
| POJO 全套样板 | 四样手写 | `@Data` |
| 构造器 | 手写重载 | `@AllArgsConstructor` / `@NoArgsConstructor` / `@RequiredArgsConstructor` |
| 日志对象 | `LoggerFactory.getLogger(...)` 声明 | `@Slf4j` |
| 多参数构建 | 重叠构造器 / 手写 Builder | `@Builder` |

### 反例

```java
// ❌ 样板代码手写——30 行 getter/setter、手拼 toString、Logger 声明
public class GoodsVO {
    private Long goodsId;
    private String goodsName;
    private BigDecimal salePrice;
    private Integer stockCount;

    private static final Logger log = LoggerFactory.getLogger(OrderQueryService.class);

    public Long getGoodsId() { return goodsId; }
    public void setGoodsId(Long goodsId) { this.goodsId = goodsId; }
    // …… getter/setter 继续手写 20 行 ……

    @Override
    public String toString() {
        return "GoodsVO{goodsId=" + goodsId + ", goodsName=" + goodsName + "}";
    }
}
```

错在哪：这些代码没有信息增量，全靠复制粘贴改名——复制即分叉，漏改一个字段 toString 就说谎；Logger 声明每个类抄一遍。

### 正例

```java
// ✅ 注解生成样板
@Slf4j
@Data
public class GoodsVO {
    private Long goodsId;
    private String goodsName;
    private BigDecimal salePrice;
    private Integer stockCount;
    // getter/setter/toString/equals/hashCode 全部由注解生成
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CommentStat {
    private Long commentId;
    private Long likeCount;
    private String topReply;
    // 多参数构建用 builder；需要反序列化所以补无参构造（见坑 2）
}
```

对在哪：字段即全部内容——新增字段零样板维护；`@Slf4j` 统一日志门面（呼应手册：禁止 System.out，统一 SLF4J）。

**判定问句**：这段 getter/setter/toString/logger 是不是手写的？是 → 换注解。

### 必须规避的坑

1. **继承场景**：子类 `@Data` 必须加 `@EqualsAndHashCode(callSuper = true)`，否则 equals 不含父类字段（父类字段全等、子类判等失真），且编译器告警。
2. **@Builder + 反序列化**：`@Builder` 会隐藏无参构造，Jackson 等反序列化框架要求无参构造——需要反序列化的类补 `@NoArgsConstructor` + `@AllArgsConstructor`。
3. **敏感字段**：手机号、密钥、证件号用 `@ToString.Exclude` 排除出 toString，防日志与报文泄漏（呼应手册日志脱敏条款）。
4. **POJO 布尔属性不加 is 前缀**：Lombok 对 `isXxx` 字段生成 `isXxx()`/`setXxx()`，序列化框架解析错位属性丢失——与命名规范同一个坑（见 naming-guide.md 规则五）。
5. **@AllArgsConstructor 依赖字段声明顺序**：字段顺序调整即构造器签名变化，会被外部手工 `new` 的类谨慎使用，优先 Builder。

---

## 规则二：字段注入为主——Spring Bean 依赖不用构造器注入

Spring 管理的 Bean 之间的依赖，默认用**字段注入**：`@Resource`（推荐，按名装配）或 `@Autowired`（按类型装配，多实现配合 `@Qualifier`）；**尽量不使用构造器注入**（`private final` 字段 + 构造器参数的形式）。

### 反例

```java
// ❌ 构造器注入——依赖一多构造器膨胀，改起来牵一发动全身
@Service
public class OrderQueryService {
    private final OrderRepository orderRepository;
    private final CacheClient cacheClient;
    private final MemberService memberService;

    public OrderQueryService(OrderRepository orderRepository,
                             CacheClient cacheClient,
                             MemberService memberService) {
        this.orderRepository = orderRepository;
        this.cacheClient = cacheClient;
        this.memberService = memberService;
    }
}
```

错在哪：每加一个依赖都要动构造器签名与赋值链；本规范约定不采用此形式（这是本 skill 的团队约定，按裁决链高于通用推荐做法）。

### 正例

```java
// ✅ 字段注入，直接声明即用
@Slf4j
@Service
public class OrderQueryService {
    @Resource
    private OrderRepository orderRepository;
    @Resource
    private CacheClient cacheClient;
    @Resource
    private MemberService memberService;
}
```

**多实现区分**（如策略接口有多个实现时）：

```java
// 按名装配（@Resource 自带 name）
@Resource(name = "wechatPayStrategy")
private PayChannelStrategy payChannelStrategy;

// 或按类型 + 限定符
@Autowired
@Qualifier("wechatPayStrategy")
private PayChannelStrategy payChannelStrategy;
```

**判定问句**：这个 Bean 的依赖是构造器塞进来的吗？是 → 改 `@Resource` / `@Autowired` 字段注入。

### 边界情况（构造器传参仍合理之处）

- **非 Spring 管理的普通类**：手工 `new` 的对象、纯工具类内部协作——没有容器参与，就是普通构造器/赋值，与本规则无关。
- **不可变性技术要求**：字段必须 `final` 的场景（如持有不可变配置的 key 生成器），按技术需求走构造器。
- **框架自身的配置类**：遵循框架既有惯例，不强行改造。
- **单元测试**：字段注入的类用 `@InjectMocks` 照常注入 mock；非 Spring 场景直接 `new` 被测对象再赋值。

---

## 规则三：编程式事务——禁止 @Transactional 注解

Spring 事务**禁止使用 `@Transactional` 注解**（声明式），一律使用**编程式事务**：注入 `TransactionTemplate`，在方法体内用 `execute` 包裹真正需要原子性的片段；需要定制传播行为时构造 Template 时显式声明。

### 反例

```java
// ❌ 注解声明事务——边界靠猜，失效场景多
@Service
public class OrderCreateService {

    @Transactional                    // 生效条件藏在注解里：代理？可见性？回滚规则？
    public OrderVO createOrder(OrderCreateReq req) {
        ...
    }

    public void reprocess() {
        createOrder(req);             // 同类内部自调用——@Transactional 静默失效，事务根本没开
    }
}
```

错在哪：注解事务的生效依赖 AOP 代理——同类内部自调用静默失效、默认只回滚 unchecked 异常、要求 public 可见性、传播行为默认值易理解偏差，这些坑都不在代码字面上；「声明在方法头上」的边界与实际执行路径脱节，事务没开或范围不对时极难排查。

### 正例

```java
// ✅ 编程式事务：边界写在调用现场，一眼可见
@Slf4j
@Service
public class OrderCreateService {
    @Resource
    private TransactionTemplate transactionTemplate;

    public OrderVO createOrder(OrderCreateReq req) {
        Order order = transactionTemplate.execute(status -> {   // 事务边界就是这一行
            Order o = assembleOrder(req);
            orderRepository.save(o);
            stockDeductionService.deduct(o.getSkuId(), o.getQuantity());  // 同一事务内原子完成
            return o;
        });
        return toVO(order);
    }
}
```

对在哪：边界在调用现场显式可见；自调用无所谓（没有代理参与）；只包真正需要原子性的写库片段，RPC/HTTP 调用天然在事务外（长事务不占连接）；异常回滚由模板统一处理，不依赖「默认回滚规则」的记忆。

**判定问句**：事务边界是 `@Transactional` 注解声明的吗？是 → 改 `TransactionTemplate.execute`。

### 边界情况

- **只读查询**不需要显式事务；确需时在定制的 Template 上设 `readOnly`，不为查询强行包事务；
- **手动 PlatformTransactionManager**（getTransaction / commit / rollback / finally 回滚）也算编程式，但优先 `TransactionTemplate`——模板统一处理异常回滚，少写样板；
- **事务范围最小化**是本条的隐性收益：只包写库片段，外部 HTTP 调用、消息发送不进事务边界；
- 本条为团队约定（手册未禁止注解事务），按裁决链生效；DDD 多模块工程的应用服务同样适用——事务边界归应用层，实现用编程式（见 ddd-layering-guide.md 工程惯例补充）。
