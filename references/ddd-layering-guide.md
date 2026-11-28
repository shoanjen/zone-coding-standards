# 分层规范详述 · DDD 多模块工程

适用形态：按 DDD 划分 Maven 多模块的工程（模块名习惯 `{app}-domain` / `{app}-infrastructure` / `{app}-service` / `{app}-web` / `{app}-bootstrap` / `{app}-common`，示例统一用包前缀 `com.example.app`）。源自工作区 DDD 后端模板脚手架（zone-boot-template，六模块 + ArchUnit 六条，2026-11-28 定稿版）的分层实践，表述已领域中立化；派生/新建工程沿用自身包前缀后，规则字面量随之迁移（派生必换清单见文末）。

单体三层工程（Controller/Service/DAO 不分 Maven 模块）的接入层条款见 [layering-guide.md](layering-guide.md)——与本篇同源：适配层不碰持久层，只是表达粒度不同。

分层依赖方向固定为单向，禁止回环、禁止跨层、禁止从实现翻回上层：

```
              ┌─────────────┐
              │   common    │（任何层可用；不依赖任何业务层）
              └─────────────┘
  web(适配) ──► service(应用) ──► domain(领域) ◄────── infrastructure(基础设施)
                                ▲
                    bootstrap（装配根：运行期注入实现，测试域 ArchUnit 守护）
```

| 模块 | DDD 角色 | 放什么 |
|---|---|---|
| `{app}-common` | 通用层 | 工具类、常量（无方向工具箱） |
| `{app}-domain` | 领域层 | 充血实体、值对象、领域服务、领域事件、仓储接口、防腐网关接口 |
| `{app}-infrastructure` | 基础设施层 | DO、Mapper、仓储实现、DO↔实体翻译器（Translator）、外部网关实现 |
| `{app}-service` | 应用层 | 应用服务接口（service 包）+ 实现（service.impl 子包，用例编排 + 事务边界）、通用业务组件（manager）、Req/Res DTO、转换器（converter） |
| `{app}-web` | 适配层 | Controller、Interceptor、Web 配置 |
| `{app}-bootstrap` | 装配层（组合根） | 启动类、多环境配置、显式聚合 infrastructure、分层守护测试 |

---

## 规则一：领域层框架无关——纯 POJO + 接口

domain 只放业务本质：实体（业务规则写成方法、不变式守卫写进构造器）、值对象（不可变、按值判等）、领域服务、领域事件，以及仓储/防腐网关**接口**。禁止依赖 Spring、MyBatis-Plus、Servlet、中间件二方包等任何框架；实体上禁止持久化注解。

### 反例

```java
// ❌ 领域实体被持久化与框架污染
@TableName("t_order")               // MyBatis-Plus 注解进了领域层
@Service                            // Spring 注解进了领域模型
public class Order {
    @TableField("payable_amount")
    private BigDecimal payableAmount;
    // 没有行为，只有字段——贫血模型 + 框架耦合双输
}
```

错在哪：领域模型与持久化框架绑死——换 ORM、换存储、单测都要拖着整个框架跑；实体没有行为，业务规则散落到 Service 变成过程式脚本。

### 正例

```java
// ✅ 纯 POJO：不变式守卫在构造器，业务规则是实体方法
public class Order {
    private final String orderId;
    private final Money payableAmount;

    private Order(String orderId, Money payableAmount) {
        this.orderId = Objects.requireNonNull(orderId, "orderId 不能为空");
        this.payableAmount = Objects.requireNonNull(payableAmount, "应付金额不能为空");
    }

    /** 领域行为：能否支付（终态单不可支付） */
    public boolean canPay() {
        return !status.isTerminal() && payableAmount.isGreaterThan(Money.zero());
    }
}
```

对在哪：领域层单独编译零框架依赖；业务规则内聚在实体行为里，应用服务只编排不写规则；持久化注解归 infrastructure 的 DO。

**判定问句**：把 domain 模块单独编译、不引任何 starter，能过吗？实体上出现框架注解 = 污染。

### 边界情况

- 领域层可以依赖**任何来源的纯 Java**（JDK、无框架的值类型库）；
- 仓储/网关接口的**方法签名只出现领域对象**（entity/值对象），不出现 DO、Wrapper、分页插件类型——签名出现持久化类型 = 接口定义在 domain、耦合留在 domain，依赖倒置白做。

---

## 规则二：依赖方向单向——适配 → 应用 → 领域 ← 基础设施

方向与访问权：适配层仅可被装配根访问；应用层只被适配层访问；领域层被适配/应用/基础设施/装配根访问；**基础设施层仅被装配根访问**（它的实现由装配根在运行期注入或组件扫描发现），也不得反向依赖应用/适配层；装配根（bootstrap）自由聚合各层，自身不被任何层访问。

### 反例

```java
// ❌ 两条都是方向违例
public class RefundAppService {
    private final OrderMapper orderMapper;        // 应用层直触基础设施实现
}
// infra 里反过来 import service 的 DTO
public class OrderRepositoryImpl {
    private OrderCreateReq lastReq;                   // 实现反向认识上层，环即将成形
}
```

错在哪：应用层直触 Mapper 绕过仓储接口——事务、缓存、规则校验全被跳过；基础设施反向依赖上层形成 `domain ← infra → service → domain` 环，任何一层改动都变成全工程改动。

### 正例

```java
// ✅ 应用服务只认识领域接口；实现由装配根注入
@RequiredArgsConstructor
public class RefundAppService {
    private final OrderRepository orderRepository;   // domain 仓储接口
}
public class OrderRepositoryImpl implements OrderRepository { ... }   // infra 实现，@Repository
```

对在哪：依赖箭头全部朝领域收口；换数据源、换 ORM、Mock 掉仓储做单测，应用服务一行不改。

**判定问句**：顺着这个 import 往下走，会不会从应用层走进实现细节、或从底层翻回上层？会 = 方向违例。

### 边界情况

- common 是无方向工具箱，任何层可用，但它自己不得依赖任何业务层；
- 装配根（bootstrap）是唯一例外：它显式聚合 web + service + infrastructure 完成运行期装配——这是「组合根」职责，不是跨层违例。

---

## 规则三：应用层只依赖领域接口——依赖倒置落点

应用服务编排用例（查仓储接口 → 调实体领域行为 → 装配出参）并承载事务边界；数据访问只经仓储接口，外部系统调用只经防腐网关接口。接口定义在 domain、实现在 infrastructure、装配根注入——三件套缺一不可。

### 反例

```java
// ❌ 应用服务里直接驱动基础设施能力
public class CouponAppService {
    private final ZoneHttpTemplate httpTemplate;   // 出站 HTTP 直调
    private final StringRedisTemplate redis;       // Redis 直调
    private final CouponMapper couponMapper;       // Mapper 直调

    public CouponRes calc(...) {
        String rule = httpTemplate.post("/rule-api", ...);   // 外部服务无防腐接口
        ...
    }
}
```

错在哪：外部协议、缓存实现、SQL 细节全渗进应用层——外部接口一改，业务代码跟着改；单测必须起真依赖；中间件能力包被应用层认识后，技术栈替换成本扩散到每个用例。

### 正例

```java
// ✅ domain.gateway 定义防腐接口；infra 用二方包 ZoneHttpTemplate 实现；应用服务依赖接口
public interface RuleGateway {                      // domain：防腐网关接口
    CouponRule loadRule(String couponId);
}
public class RuleGatewayImpl implements RuleGateway { ... }   // infrastructure：ZoneHttpTemplate 调用细节在此

public class CouponAppService {
    private final RuleGateway ruleGateway;             // @RequiredArgsConstructor 构造注入
    public CouponRes calc(CouponCalcReq req) { ... }   // 读路径无需显式事务
}
```

对在哪：应用层对「数据从哪来、外部服务怎么调」完全无感；HTTP 细节、重试熔断、序列化全部沉在 infrastructure。

**判定问句**：换掉数据源或外部服务，应用服务要改吗？要改 → 依赖错了，补接口下沉实现。

### 边界情况

- `@Service` 等 Spring 组件注解在应用层**允许**（「框架无关」约束的是领域层，不是应用层）；但**事务实现禁止 `@Transactional` 注解**——事务边界归应用层，写法必须编程式 `TransactionTemplate`（见 [springboot-guide.md](springboot-guide.md) 规则三）；
- 应用层的日志门面、统一异常允许使用（横切能力），但中间件能力包（HTTP 客户端/Redis/数据源）禁止——判据：它是不是「实现细节的入口」。
- 应用层 `manager` 包（通用业务组件）同样只依赖领域接口：可复用业务能力（业务编号生成、金额分摊、通用校验）沉淀于此；封装第三方协议/出站 HTTP/Redis 的能力一律下沉 infrastructure.gateway——判据与上一条相同。类名业务贴合（如 `OrderIdGenerator`），禁 `XxxManager` 泛称（同命名规范「禁宽泛词」）。
- 领域服务是纯 Java 对象（domain 禁 Spring 注解），由**使用方模块**的 `@Configuration` 注册为 Bean（模板示范：应用层 config 包注册计价领域服务；跨模块装配归装配根）。

---

## 规则四：适配层不碰基础设施——Controller 三件事

与 [layering-guide.md](layering-guide.md) 的 Controller 规范完全同源：Controller 只能注入应用服务，只做接参数（入参 >= 2 抽 Req 对象，含 @Valid 校验触发）、调应用服务、组返回（统一响应体）；不写业务、不做 DTO→VO 转换、不注入任何持久层与基础设施组件；接口路径同受 `api/v1/xxx` 版本化约束。

### 反例 / 正例速记

```java
// ❌ 注入 Mapper + 手搓转换 + 写业务
@GetMapping("/api/v1/orders/{id}")
public ZoneResult<OrderDetailRes> detail(@PathVariable Long id) {
    OrderDO orderDO = orderMapper.selectById(id);          // 直触基础设施
    if (orderDO.getStatus() == 2)                          // 业务规则
        throw new ZoneBusinessException(BusinessError.DATA_CONFLICT, "订单已取消");
    OrderDetailRes res = new OrderDetailRes(orderDO.getXxx(), ...);  // 手搓转换
    return ZoneResult.ok(res);
}

// ✅ 三件事收口
@GetMapping("/api/v1/orders/{id}")
public ZoneResult<OrderDetailRes> detail(@Valid OrderQueryReq req) {
    return ZoneResult.ok(orderQueryAppService.detail(req));   // Res 由应用层 Converter 转换
}
```

**判定问句**：Controller 方法体里除了「接参数、调应用服务、包返回」还剩什么？剩了就越界。

### 边界情况

- 全局异常处理、链路 Filter 属适配层自身设施，由二方包 `zone-middleware-web` 自动装配提供（`ZoneGlobalExceptionAdvice`/`ZoneTraceInboundFilter`，开关 `zone.middleware.web.*`），**工程不自有、不重复造**；
- Interceptor/CORS/文档配置归适配层的 Web 配置类，不算业务依赖。

---

## 规则五：对象各归其位——转换不跨层代做

| 对象 | 所在层 | 职责 |
|---|---|---|
| entity / valueobject | domain | 领域模型，业务行为内聚 |
| DO（dataobject） | infrastructure | 表映射，持久化注解只在此 |
| Req / Res | service（应用层） | 接口契约；校验注解在 Req |
| Converter | service（应用层） | 领域对象 → Res |
| Translator | infrastructure | DO ↔ 领域实体 |

### 反例

```java
// ❌ 对象越层流转
web 层出现 OrderDO                    // 表模型直接出适配层
service 层 import infrastructure 的 Translator  // 应用层认识实现细节
domain 的 Order 实体上带 @TableName   // 见规则一
```

错在哪：表模型一旦出基础设施层，改列名就是全工程编译事故；转换职责一混，DO↔实体 与 实体→Res 两类加工互相污染。

### 正例

```java
// ✅ 链路上对象逐层受控转换
Controller：OrderQueryReq ──► AppService（编排）
AppService：orderRepository.findById() → Order（实体）→ Converter.toRes(order)
RepositoryImpl：orderMapper.selectById() → OrderDO → Translator.toEntity(do)
```

对在哪：每类对象只在自己的层里出生和转换；DO 永不出 infra，实体可以出 domain 进 service/web（经 Converter 变 Res）。

**判定问句**：这个对象的包路径，和它的职责对得上吗？转换逻辑出现在「非归属层」= 越位。

### 边界情况

- 同一实体按用例裁剪出多个 Res（全量视图 / 列表瘦视图）属正常设计，都放应用层 response 包；
- 读写职责分开：查询侧 Query 服务、写侧 Command 服务分开建，不追求「一个 Service 打天下」。

---

## 规则六：结构由测试守护——ArchUnit 随包前缀迁移

分层规则不是口头约定：用 ArchUnit 写进**装配根模块的测试域**（不随业务代码发布），违规 `mvn test` 直接红。基线六条（与模板守护测试 `DddLayerRulesTest` 对齐）：①领域层框架无关（禁依赖含中间件二方包 `com.zone.middleware..`）；②应用层不触基础设施（禁依赖 infrastructure、MyBatis-Plus、二方包 http/redis/database 能力包）；③适配层不触基础设施；④基础设施不反向依赖；⑤common 中立（不得依赖任何业务层，但可用框架与工具——common 的中立指不依赖业务层，并非框架无关）；⑥layeredArchitecture 依赖方向总纲（适配层仅被装配根访问、应用层仅被适配层与装配根访问、基础设施层仅被装配根访问、装配根不被任何层访问）。

### 反例 / 正例

```java
// ❌ 分层靠 code review 口头把关——人总会漏
// ✅ 规则测试化（示例：规则一，禁依赖清单含中间件二方包构件）
@Test
void 领域层应为框架无关纯领域层() {
    noClasses().that().resideInAPackage("com.example.app.domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "org.springframework..", "com.baomidou..", "jakarta.servlet..",
                    "com.zone.middleware..")
            .because("领域层只描述业务本质；持久化/中间件实现一律下沉 infrastructure（依赖倒置）")
            .check(classes);
}
```

**判定问句**：这条分层约定，机器在 `mvn test` 里能拦住吗？不能 → 补规则；规则改动 → 同步依赖方向图与分层文档。

### 边界情况

- 导入范围排除测试代码（`ImportOption.DoNotIncludeTests`），只约束主代码；
- 新增守护规则时从「真实违例」出发，不为规则而规则。

---

## 工程惯例补充

- **依赖注入风格**：DDD 多模块工程惯例统一 `@RequiredArgsConstructor`（Lombok）+ `private final` 不可变字段注入（模板工程有依赖的组件 100% 此风格）；与单体工程的字段注入惯例并存时，**按裁决链以项目自身规范优先**。
- **应用服务面向接口**：顶层接口放 `service` 包（应用层契约），实现放 `service.impl` 子包（`XxxAppServiceImpl`，用例编排 + 事务边界）；上层（Controller）只依赖接口——mock 单测与实现替换成本低；单实现场景每个用例多一个接口文件是本约定的固定成本（项目约定优先于 YAGNI 一般原则）。
- **Req/Res 契约**：Req 一律继承二方包 `com.zone.middleware.common.request.BaseRequest`（reqId/platform/ip/deviceId/os/version/timestamp 通用契约字段），`@Data` + `@EqualsAndHashCode(callSuper = true)`，jakarta 校验注解带中文 message，接口文档 `@Schema`；**Req 原样传给应用服务**，不做 request→command 二次转换；Res 用 `@Data @Builder`，同实体按用例裁剪多视图（全量 DetailRes / 列表瘦 SummaryRes）。
- **日志门面**：业务组件与应用服务用二方包 `ZoneLog.info("类名", "方法名", "消息 {}", args...)`（类名/方法名显式入参，traceId/reqId 自动织入）；不用 `@Slf4j` 自拼 Logger（详见 [zone-middleware-guide.md](zone-middleware-guide.md) 规则二）。
- **领域模型写法**：值对象与领域事件用 record（紧凑构造器守不变式，如 `Money(long amountFen)`）；实体纯 POJO——手写 getter、静态工厂 `Order.of(...)`、构造器不变式守卫，业务行为内聚成实体方法（`canPay()`/`isTerminal()`），不套 `@Data`；仓储接口签名用集合语义（`Optional<T>`/`List<T>`），不暴露持久化分页类型。
- **事务写法**：应用服务的事务边界禁止 `@Transactional` 注解，一律 `TransactionTemplate` 编程式（见 [springboot-guide.md](springboot-guide.md) 规则三，含「领域事件提交后发布」等铁律），无工程差异例外。
- **二方包落位**：中间件能力按「能力就近、domain 纯净」落层——suite 落装配根、database/http 落 infrastructure、security/web 落 web、log 落 service/web、common 落 service 及以下；完整对照见 [zone-middleware-guide.md](zone-middleware-guide.md) 附表。
- **新业务开发动线**（照此填代码即合规）：domain 建实体/接口 → infrastructure 建 DO/Mapper/Translator 并实现接口 → service 写应用服务（事务在此）+ Req/Res/Converter（通用业务组件放 manager）→ web 写 Controller 三件事。

### 派生新工程（基于模板新建业务工程）

从模板派生时**带走**：六模块分层 + ArchUnit 六条 + 开发动线 + 编码约定（fastjson2 禁 jackson、Lombok 优先、ZoneResult 统一返回体、Controller 三件事、编程式事务、注入风格）+ suite 接入姿势 + 多环境配置骨架。**必换清单**（漏一项就是事故）：

1. 项目名与 Maven 坐标（根 pom + 六模块 pom 的 groupId/artifactId）；
2. 包前缀迁移（目录树、主类 `scanBasePackages`、`@MapperScan`、ArchUnit 测试包字面量、application*.yml 四份的 logging/knife4j 触点、javadoc @author）——迁移后 grep 自查旧前缀残留；
3. **端口**（模板 8212 禁止沿用，派生前 lsof 双查）与库名/Redis db index（共享实例独立库、独立 db，禁止另起实例占端口）；
4. 删除示例业务全链路文件（Controller/AppService/Converter/Translator/DO/Mapper/Flyway 脚本一整纵切）再落自己的业务；
5. 验证收口：`mvn clean package` + `mvn test`（ArchUnit 全绿）+ 启动日志三关键字 + 接口文档冒烟。

- **JSON/异常/返回体**等编码约定默认以中间件二方包为基准（ZoneResult / ZoneBusinessException / fastjson2 / 官方配置键，详见 [zone-middleware-guide.md](zone-middleware-guide.md)）；项目另有约定时按 SKILL.md 裁决链第 1 条以项目规范优先。
