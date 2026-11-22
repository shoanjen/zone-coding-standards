# Spring Boot 工程惯例详述（Lombok 与依赖注入）

两条本 skill 自定义的工程惯例：一条砍样板代码，一条定注入风格。手册对这两项没有强制条款，按裁决链本规范即为约定。示例 Java + Spring Boot，跨电商、内容、支付、工具等多个业务场景。

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
