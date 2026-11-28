# 分层规范详述 · Controller 层

「分层」约束回答的不是「组件怎么写」（那是组件规范的职责），而是「组件放在哪一层、允许依赖谁」，以及接入层对外的协议契约。当前展开适配层（Controller）三条；应用层（Service）/基础设施层条款按需扩展。

Maven 多模块 DDD 工程（domain / infrastructure / service / web / bootstrap / common 六模块）的完整分层规范见 [ddd-layering-guide.md](ddd-layering-guide.md)——本篇 Controller 条款是 DDD「适配层」规范在单模块工程的表达（依赖/职责两条与 DDD 规则四同源，路径规范对两种架构同样生效），冲突时按 SKILL.md 裁决链以更严格者为准。示例中的统一返回体与业务异常以中间件二方包为默认基准（`ZoneResult` / `ZoneBusinessException`，详见 [zone-middleware-guide.md](zone-middleware-guide.md)）；项目另有约定按裁决链以项目规范优先。

分层依赖方向按 DDD 分层表达，固定单向、禁止回环、禁止跨层（与 [ddd-layering-guide.md](ddd-layering-guide.md) 同图同源）：

```
Controller/web（适配层） ──► service（应用层） ──► domain（领域层） ◄──── infrastructure（基础设施层）
                                                        ▲
                                    bootstrap（装配根：运行期注入实现）
```

单体三层工程不分 Maven 模块，角色对应：Controller ≈ 适配层、Service ≈ 应用层、DAO/Repository/Mapper ≈ 基础设施层（实现领域接口，由容器/装配根注入）——依赖箭头依旧朝领域收口：Controller 永远只依赖应用层，DAO/Mapper 不出现在应用层之上，数据库访问被应用层与基础设施层的接口-实现边界隔开。

---

## 规则一：依赖边界——Controller 只依赖应用层

Controller 只能引入**应用层组件**（Service，业务逻辑的唯一入口）；**禁止引入基础设施层（持久层）组件**：DAO、Repository、Mapper、JPA Repository、MyBatis Mapper、JdbcTemplate 等一律不得出现在 Controller 的字段、构造参数与方法体内。

### 反例

```java
// ❌ Controller 直连持久层——分层被打穿
@RestController
public class OrderController {
    @Resource
    private OrderMapper orderMapper;      // 禁止：持久层组件
    @Resource
    private JdbcTemplate jdbcTemplate;    // 禁止：裸 SQL 直查

    @GetMapping("/api/v1/orders/{id}")
    public ZoneResult<OrderVO> detail(@PathVariable Long id) {
        Order order = orderMapper.selectById(id);   // 绕过 Service 直查库
        ...
    }
}
```

错在哪：

- 绕过 Service 即绕过事务边界——该有的事务没有，绕过业务规则——缓存、权限、审计、防重全被跳过；
- 同一份数据访问逻辑在 Controller 和 Service 各写一遍，分叉开始；
- 依赖方向被打穿：上层可跨层直触最底层，分层只剩目录名。

### 正例

```java
// ✅ 只引入 Service，数据访问一律经应用层
@RestController
public class OrderController {
    @Resource
    private OrderQueryService orderQueryService;

    @GetMapping("/api/v1/orders/{id}")
    public ZoneResult<OrderVO> detail(@PathVariable Long id) {
        return ZoneResult.ok(orderQueryService.getOrderDetail(id));
    }
}
```

对在哪：Controller 的依赖面收敛到 Service 接口，事务、缓存、规则校验在 Service 内统一生效；换协议（HTTP 改 RPC、加消息入口）应用层零改动。

**判定问句**：Controller 的依赖字段里，有没有 Service 以外的业务组件？有持久层组件 → 立即改道 Service。

### 边界情况

- **接入层自身设施不算越界**：`HttpServletRequest`、`@Valid` / `@Validated` 触发的 Bean Validation、MVC 类型转换、接口文档注解——这些是协议适配的原材料，不是业务依赖。
- **Service 以外的灰色组件按「是否触达数据与业务」判定**（与命名豁免的落点思想一致）：只做协议适配的（如统一返回体包装器、常量）接入层可用；触达数据与业务的（缓存客户端、MQ 模板、搜索引擎客户端）与持久层组件同等对待，一律下沉 Service。禁令不靠穷举组件清单——清单永远写不全，判据是它碰不碰数据与业务。

---

## 规则二：职责边界——Controller 不写业务逻辑

Controller 只做三件事：**接参数**（入参 >= 2 抽成 Req 对象——XxxReq，含触发参数格式校验，见 naming-guide「入参对象化」）、**调 Service**（调用应用层）、**组返回**（包装统一响应体、表达 HTTP 语义与状态码）。业务规则判断、金额计算、复杂数据组装一律写在 Service。

### 反例

```java
// ❌ Controller 干起了 Service 的活
@PostMapping("/api/v1/orders")
public ZoneResult<OrderVO> create(@RequestBody @Valid OrderCreateReq req) {
        if (req.getQuantity() > stockService.getStock(req.getSkuId())) {      // 业务规则判断
            throw new ZoneBusinessException(BusinessError.BUSINESS_FAILED, "库存不足");
        }
    BigDecimal payable = req.getPrice()
            .multiply(BigDecimal.valueOf(req.getQuantity()))              // 金额计算
            .subtract(couponService.calcDiscount(req));
    Order order = orderConvert.toOrder(req, payable);                     // 复杂组装
    orderMapper.insert(order);                                            // 连带出依赖违例
    return ZoneResult.ok(orderConvert.toVO(order));
}
```

错在哪：库存判断与金额计算是业务规则，写死在 HTTP 入口——同一个逻辑到了 RPC、定时任务入口就要再抄一遍；Controller 从「协议适配层」膨胀成「另一个 Service」，单测必须起 Web 环境才能跑。

### 正例

```java
// ✅ 业务全部下沉，Controller 保持三步
@PostMapping("/api/v1/orders")
public ZoneResult<OrderVO> create(@RequestBody @Valid OrderCreateReq req) {
    return ZoneResult.ok(orderCreateService.createOrder(req));
}
```

对在哪：库存判断、金额计算、组装落库都在 `createOrder` 的同一个事务边界内完成；Controller 薄到任何协议都能直接复用 Service；Service 单测不起 Web 容器。

**判定问句**：Controller 方法体里除了「接参数、调 Service、包返回」，还剩什么？剩了任何业务判断或计算，就是越界。

### 边界情况

- **参数校验分两层**：格式与范围校验（非空、长度、区间、格式）在入口层用 Bean Validation 完成——手册亦要求入口层完成参数合法性校验；业务规则校验（库存够不够、用户有没有资格）归 Service。判据：换个协议调用，这条规则还成立吗？成立 → Service。
- **DTO→VO 的展示转换也不在 Controller 做**：转换（DO/DTO→VO）属于数据组装，一律归 Service 层——Service 返回现成的 VO，Controller 拿到即包装统一响应体；Converter 组件由 Service 调用。Controller 不触碰 VO 的字段结构，转换中出现业务计算（按身份过滤字段、拼权限标记）更是早已越界——那是 Service 的业务逻辑。
- **不算业务逻辑的接入层职责**：统一返回体包装、全局异常处理（`@RestControllerAdvice`）、访问日志与链路埋点、HTTP 状态码表达。

---

## 规则三：接口路径规范——api/v1/xxx 必带版本号

Controller 的接口路径必须遵循 **`api/v1/xxx` 格式**：`api` 前缀 + **版本号**（v1、v2…）+ 资源路径。版本号是接口契约的一部分——演进靠版本号，不靠破坏性改路径。类级 `@RequestMapping("/api/v1/orders")` 统一前缀与方法级全路径两种写法等价，团队统一其一即可。

### 反例

```java
// ❌ 三种不合规写法
@GetMapping("/orders/{id}")        // 无版本号
@GetMapping("/api/orders/{id}")    // 有前缀无版本
@GetMapping("/v1/orders/{id}")     // 有版本无 api 前缀
```

错在哪：路径没有版本护栏——后续字段增删、语义变更只能原地改路径，所有调用方（前端、网关、第三方）跟着断；缺 `api` 前缀则业务路径与框架自带路径（actuator、静态资源）混在同一命名空间。

### 正例

```java
// ✅ 类级统一前缀 + 方法级相对路径（推荐写法）
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    @GetMapping("/{id}")
    public ZoneResult<OrderVO> detail(@Valid OrderQueryReq req) { ... }

    @PostMapping
    public ZoneResult<OrderVO> create(@RequestBody @Valid OrderCreateReq req) { ... }
}

// ✅ 方法级全路径（等价合规）
@GetMapping("/api/v1/orders/{id}")
public ZoneResult<OrderVO> detail(@Valid OrderQueryReq req) { ... }
```

对在哪：`/api/v1/orders/{id}` 一眼读出「API、第一版、订单资源」；演进 v2 时 `/api/v2/...` 并行上线、v1 保留兼容；调用方看到的地址契约稳定。

**判定问句**：路径是 `api/v1/xxx` 格式吗？版本号带了吗？不满足 → 补齐再落笔。

### 边界情况

- **版本升级**：破坏性变更建 v2 新路径，v1 保留兼容（或按项目下线策略），禁止原地改 v1 的语义；
- **框架自带路径豁免**：actuator、swagger UI、error 页等框架/中间件自有路径不受此约束；
- **项目另有约定优先**：按裁决链第 1 条，项目若统一 `/xxx/v1`、header 版本等自有约定，以项目规范为准；
- 与入参对象化互补：路径版本化管**地址契约**，Req 对象管**载荷契约**，两者共同构成 Controller 的对外接口。
