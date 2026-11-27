# 命名规范详述（通用五条 + 分类细则）

组织方式：第一部分是**一切标识符通用**的五条规则；第二部分按**类 / 方法 / 变量**分类给出各自的构词细则——不同标识符类型的规则本不相同。每条规则与每组分类细则都带正反例与判定问句。示例 Java 为主，跨电商、内容、支付、工具等多个业务场景。

| 你正在命名什么 | 先看 | 再对照 |
|---|---|---|
| 类 / 接口 / 枚举 / 抽象类 | 第二部分 · 类命名 | 通用五条 |
| 方法 | 第二部分 · 方法命名 | 通用五条 |
| 变量 / 常量 / 参数 | 第二部分 · 变量命名 | 通用五条 |

---

# 第一部分：通用五条（一切标识符）

## 规则一：业务贴合——命名还原业务动作与业务状态

命名必须能还原当前业务场景里的动作或状态，禁止与业务脱节的「万能名」。读名字不看实现，就该知道这段代码在业务里干什么。

### 反例

```java
// ❌ 说不出处理什么、做什么、属于哪个业务
public void process(Map<String, Object> data) { ... }   // 处理什么？做什么？
public boolean handle(Order order) { ... }               // 校验？支付？取消？
private boolean flag;                                     // 什么状态？
private int count;                                        // 什么数量？
```

错在哪：`process`/`handle` 是空壳动词，`data` 是空壳名词，`flag`/`count` 与具体业务状态无关——读的人必须进方法体才知道在干什么。

### 正例

```java
// ✅ 动词具体 + 对象具体，业务动作一目了然
public void calcCouponDiscount(Coupon coupon) { ... }       // 算券优惠
public boolean sendPaymentReminder(Order order) { ... }     // 发催付提醒
private boolean membershipExpired;                            // 会员是否已过期
private int pendingReviewCount;                               // 待审核数量
```

对在哪：名字即业务——读 `sendPaymentReminder(order)` 就知道给订单发催付提醒，不需要点进实现。

**判定问句**：只看名字不看实现，能否说出它在业务里干什么？说不出来就重命名。

### 边界情况

- 业务贴合 ≠ 冗长堆词：`getUserByIdAndTypeAndStatusAndPage(...)` 是另一种病，保留关键维度即可。
- 纯技术动作允许技术化命名：`parseJson`、`serialize`、`retryWithBackoff` 本身就是具体动作，不算宽泛。

---

## 规则二：禁宽泛词——万金油词白名单制

`data`、`info`、`result`、`value`、`param`、`item`、`obj`、`res`、`ret`、`temp`、`handle`、`process`、`do`、`execute`、`manage`、`operate`、`manager`、`helper`、`util`、`common` 这类词，在具体业务实现类中禁止使用；唯一豁免是该元素确实是跨业务复用的通用工具（如 `StringUtils`、`DateUtils`），且工具内部方法自身满足业务贴合。

### 豁免：通用动词的「语义落点」原则

禁令针对的是**有具体语义却不写**的偷懒；豁免面向的是**该层本来就不该写具体语义**的抽象。豁免不按场景清单判定（清单永远穷举不完），只看一条统一原则：

> 通用动词（process/handle/execute/...）合法，**当且仅当它的具体语义在系统中有唯一明确的落点**。

语义可能落在哪里？四类典型落点：

| 落点 | 场景 | 例子 |
|---|---|---|
| **契约落点** | 多实现策略/抽象/函数式接口的统一入口，接口只负责统一调度，语义由实现类承载 | `PayChannelStrategy.process(...)` → `WechatPayStrategy`；JDK `Runnable.run`、`Function.apply`、`Comparator.compare`、Spring `MessageHandler.handle` |
| **框架落点** | 覆写框架定义的方法——名字是框架契约，本就不可改名 | `Filter.doFilter`、`Servlet.service`、Spring `ApplicationRunner.run`、各框架 `@Override` 钩子 |
| **负载落点** | 泛型引擎/中间层处理语义未知的负载，语义随负载类型走 | 事件总线 `handle(MessageEvent)`、任务调度 `execute(Task<T>)`、重试模板 `execute(RetryCallback)`、管道 `process(PipelineStage)` |
| **领域落点** | 业务领域语言本身就用这个通用动词 | 工单流转的「受理/处理/办结」是领域术语，`WorkOrderService.process(workOrder)` 按项目术语表对齐 |

类名角色后缀同理：`RefundMessageHandler` 的业务语义落在前缀（RefundMessage）、角色语义落在后缀（Handler），两处都明确——合法；`Manager`/`Helper` 前后缀都读不出语义——不合法。

**配套义务（豁免不是免费的）**：使用豁免必须让落点**可发现**——契约落点要有接口 javadoc 与具体实现类名；框架落点保持 `@Override`；负载落点的负载类型名要具体；领域落点要与项目术语表对齐。落点含糊、多处落点或无处落点的，视为偷懒命名，必须换具体动词。

```java
// ✅ 落点明确
public interface PayChannelStrategy {
    PayResult process(PayRequest request);    // 契约落点：语义在 WechatPayStrategy 等实现类
}
public class RetryTemplate {
    public <T> T execute(RetryCallback<T> callback) { ... }   // 负载落点：语义在 callback 里
}

// ❌ 落点含糊——语义无处定位
public class CouponService {
    public void process(Coupon coupon) { ... }   // 不是契约、不是框架覆写、没有负载抽象、领域语言也不叫"处理"
}
```

**判定问句（豁免判定）**：这个通用动词的具体语义，在系统里有没有唯一明确的落点？有 → 合法，并落实配套义务；没有 → 必须换具体动词。

### 反例

```java
// ❌ 万金油词堆出来的「什么都像、什么都不像」（业务实现类中）
public Result handle(UserInfo info) { ... }
List<Map<String, Object>> list = query();     // list 是什么列表？
userManager.manage(data);                      // 管什么？
```

错在哪：`Result`/`UserInfo`/`handle`/`manage`/`data`/`list` 全部没有增量信息，等于没命名。

### 正例

```java
// ✅ 每个词都携带信息
public PaymentResponse pay(PayRequest payRequest) { ... }
List<Comment> hotComments = queryHotComments();   // 热评列表
public RefundOrder createRefundOrder(AfterSaleOrder afterSaleOrder) { ... }
```

对在哪：换成「具体动词 + 具体对象」后，不看签名细节也能推断行为与返回内容。

**判定问句**：把这个词换成更具体的说法，代码是否更清楚？是 → 必须换。

### 边界情况

- 语言惯例不在管辖范围：泛型参数 `T`/`E`/`K`/`V`、异常变量 `ex`、循环变量 `i`/`j`。
- `util`/`Helper` 的豁免仅限纯技术通用工具；一旦开始装业务逻辑，必须改名并拆职责（组件规范见 component-guide.md）。

---

## 规则三：禁多义词——一个概念一个词，一个词一个概念

同一概念在系统中只允许一个词（全项目统一术语），同一个词在系统中只允许一个概念。这是团队沟通与全局搜索的基础。

### 反例

```java
// ❌ 「客户」三个词并存——读代码的人以为是三种东西
class CustomerService { ... }
class ClientMapper { ... }        // client 又是谁？
class UserLevelCalculator { ... } // user 指客户还是登录账号？

// ❌ 裸 status 一词多义
private Integer status;           // 订单状态？审核状态？支付状态？
```

错在哪：概念与词语没有一一对应，沟通中「改一下 client 那块」会产生歧义；`status` 裸用后，全局搜索无法定位真正的目标。

### 正例

```java
// ✅ 全局术语一致，状态带上主体
class CustomerService { ... }               // 「客户」全系统只叫 customer
class CustomerProfileMapper { ... }
private OrderStatus orderStatus;            // 主体 + 状态，不再有裸 status
private AuditStatus auditStatus;
```

对在哪：一个概念一个词后，全局搜索即全量定位；状态字段带主体，多状态共存不打架。

**判定问句**：全局搜索这个词，出现的含义是否唯一？

### 边界情况

- 相似但不同的概念必须显式区分：`createTime`（下单时间）与 `payTime`（支付时间）不能都叫 `time`。
- 项目级术语表是**建议项**而非硬性要求；小项目用「判定问句」抽检即可。

---

## 规则四：禁无脑缩写——白名单之外一律写全

只允许广泛共识的缩写：`id`、`url`、`uri`、`api`、`ip`、`db`、`sql`、`http(s)`、`json`、`xml`、`html`、`ui`、`io`、`cpu` 等。判定标准：**团队新人不查文档能猜对含义才可用**。手册关联：【强制】杜绝完全不规范的缩写，避免望文不知义；禁止拼音与英文混合、禁止纯拼音命名。

### 反例

```java
// ❌ 自造缩写，新人全靠猜
String usrNm;                 // user name?
int ordCnt;                   // order count?
List<Order> ordLst;           // order list?
BigDecimal dscntAmt;          // discount amount?
String yhMc;                  // 优惠名称？拼音+缩写双重灾难
```

错在哪：缩写只省了打字，把理解成本转嫁给所有后续读者；`yhMc` 这类拼音缩写连猜都没有线索。

### 正例

```java
// ✅ 写全；公认缩写保留
String userName;
int orderCount;
List<Order> orderList;
BigDecimal discountAmount;
Long userId;                  // id 属公认缩写
String apiUrl;                // url/api 属公认缩写
```

对在哪：读即懂，不依赖口头传承；公认缩写不增加理解成本，保留无妨。

**判定问句**：新人不查文档，能猜对这个缩写吗？

### 边界情况

- 行业专有缩写（如电商的 `sku`、支付的 `otp`）若在目标领域属公认，可用，但团队内要有一次书面约定。
- 首字母缩略的类名（如 `Http`、`Sql`）遵循语言惯例大小写即可。

---

## 规则五：命名匹配数据状态——名字描述「此刻装的是什么」

变量名必须与它**此刻**装的内容一致：`count` 只能装数量；布尔用 is/has/can/should 前缀；计量单位写进名字；数据经过处理后名字要跟着变（或另起新名），禁止「变量漂移」。

### 反例

```java
// ❌ 名实不符三连
int count = order.getTotalAmount();          // count 装了金额
List<Order> orderList = pageData.getList();  // 装的是「一页」数据，不是全量列表
boolean vip = user.getVipExpireTime();       // 装的是时间，却起了布尔名

// ❌ 变量漂移：一个变量先后装两种东西
List<Order> orders = orderMapper.queryAll();
orders = orders.stream().filter(o -> o.isValid()).collect(toList());  // orders 含义已变
```

错在哪：名字停留在上一个状态，读 `orders` 以为是全量，实际是过滤结果；改 bug 的人会在这里埋雷。

### 正例

```java
// ✅ 名字与内容时刻一致
BigDecimal payableAmount = order.getTotalAmount();      // 应付金额
List<Order> orderPage = pageData.getList();             // 一页数据
boolean hasPaid = order.getPaidTime() != null;
long retryIntervalMs = 3000;                             // 单位入名
BigDecimal priceYuan = ...;                              // 元/分等币种单位同理（priceCent）

// ✅ 处理后另起新名，不漂移
List<Order> allOrders = orderMapper.queryAll();
List<Order> validOrders = allOrders.stream().filter(Order::isValid).collect(toList());
```

对在哪：每个名字描述当前数据状态；过滤、分页、转换后的新数据有新名字，读者不会错意。

**判定问句**：变量的名字和它此刻装的东西，是同一件事吗？

### 边界情况：POJO 布尔属性的 is 前缀豁免（手册【强制】）

```java
// ❌ POJO 布尔属性加 is 前缀——部分框架反序列化时会把 isDeleted 解析成 deleted，属性丢失
public class UserPO {
    private Boolean isDeleted;      // 危险
}

// ✅ POJO 布尔属性不加 is 前缀；数据库字段仍可用 is_deleted，靠映射层转换
public class UserPO {
    private Boolean deleted;
}
```

- 局部变量与**方法名**不受此限：`isVip()`、`hasPaid()` 仍推荐 is/has 前缀（手册亦如此区分）。
- 查询方法的「必有/可能无」也要名实一致：`getXxx` 表达必得（查不到是异常），`findXxx`/`queryXxx` 表达可能为空，配合 `Optional` 或项目判空约定。

---

# 第二部分：分类细则

## 类命名（类、接口、枚举、抽象类）

构词细则：

1. UpperCamelCase，**名词性命名**——类名回答「它是什么业务概念」：`Order`、`CouponRule`、`UserAccount`。
2. 职责类型用**角色后缀**表达：`XxxService`（业务编排）、`XxxController`（接入）、`XxxRepository`/`XxxDAO`（存储）、`XxxValidator`（校验）、`XxxConverter`（转换）、`XxxListener`（事件监听）、`XxxStrategy`（策略）、`XxxTemplate`（流程模板）。后缀必须与真实职责一致——叫 Service 就只做编排，别在里面堆计算（对应组件规范单一职责）。
3. 领域模型按手册约定后缀：`xxxDO`/`xxxDTO`/`xxxVO`/`xxxBO`/`xxxQuery`；抽象类 `Abstract`/`Base` 开头；异常类 `xxxException`；测试类 `xxxTest`；枚举成员全大写。
4. 禁止万金油类名与无语义后缀：`OrderManager`（管什么？）、`Data`、`Info`、把业务逻辑塞进 `XxxUtil`。

```java
// ❌ 类名说不出概念与职责
class OrderManager { ... }          // 管什么？算钱？发货？导出？
class DataProcessor { ... }         // 什么数据？处理什么？
class CouponUtil { ... }            // 优惠券业务逻辑混进 util

// ✅ 业务概念 + 职责后缀（或角色原型），一眼定位
class OrderCalcService { ... }          // 订单金额计算
class CouponRuleValidator { ... }       // 优惠券规则校验
class PayChannelStrategy { ... }        // 支付渠道策略（角色原型，多实现）
class RefundMessageHandler { ... }      // 退款消息处理器（角色原型）
```

错在哪/对在哪：❌ 组的 `Manager`/`Data`/`Util` 后缀没有回答「概念是什么、职责是什么」；✅ 组每个类名都能读出「业务概念 + 职责类型」，配合接口与实现一眼可见（`PayChannelStrategy` → `WechatPayStrategy`）。

**判定问句**：只看类名，能说出它是什么业务概念、承担什么职责类型吗？

### 边界情况

- 角色后缀（Handler/Processor/Strategy/Listener）是架构原型词汇，不算宽泛词（见规则二豁免）；但后缀要与真实职责一致，叫 Handler 就真的只做消息处理。
- 概念本身就通用的技术类允许中性名（`StringBuilder` 风格），前提是它真的不含业务逻辑。

## 方法命名

构词细则：

1. lowerCamelCase，**动词或动宾结构**开头，回答「做什么、对谁做」：`calcCouponDiscount(coupon)`、`sendPaymentReminder(order)`。
2. **查询语义分级**——动词前缀约定与返回语义必须一致：
   - `getXxx`：必得，查不到即异常；
   - `findXxx`：可能无，返回 `Optional` 或可空，调用方必须判空；
   - `listXxx` / `queryXxx`：集合查询，如 `listValidCoupons`；
   - `countXxx`：计数；`existsXxx` / `isXxx` / `hasXxx`：布尔判断；
   - `toXxx` / `convertXxx`：类型转换，如 `toOrderVO`。
3. 事件回调用 `on + 事件名`：`onOrderPaid`、`onRefundCompleted`。
4. 同一动作全系统同一动词（规则三在方法维度的落点）：别一处 `check`、一处 `validate`、一处 `verify` 指同一件事。
5. 通用动词豁免见规则二「语义落点原则」：语义有唯一明确落点（契约/框架/负载/领域）的 `process`/`handle` 合法；具体业务实现类里落点含糊的自有方法不合法。
6. **入参对象化**——请求参数 >= 2 个禁止平铺，抽成单一对象（Controller→XxxReq，其他层→语义对象）；展开见下方「入参对象化」小节。

```java
// ❌
public void process(Coupon coupon) { ... }          // 具体业务类里用万能动词
public User getUser(Long id) { ... }                 // 实际可能返回 null——语义分级错误
public void check(Order order) { ... }               // 校验？还是检查状态？检查什么？

// ✅
public void issueCoupon(Coupon coupon) { ... }       // 发券
public Optional<User> findUserById(Long id) { ... }  // 可能无——Optional 表达语义
public void validateCouponUsable(Coupon coupon) { ... } // 校验券可用性
public void onOrderPaid(Order order) { ... }         // 订单已付回调
```

错在哪/对在哪：❌ 组的 `process` 没有契约语境兜底、`getUser` 名义必得实际可空、`check` 不知道检查什么；✅ 组动词具体、语义分级与返回类型一致。

**判定问句**：只看方法名，能说出「做什么、对谁做、返回什么语义」吗？

### 边界情况

- 测试方法允许描述式长名（`refundShouldFailWhenBalanceInsufficient`），因为它本身就是断言描述。
- 重载的同名方法靠参数区分语义，不算多义词（如 `createOrder(OrderCmd)` 与 `createOrder(Cart)`）。

### 入参对象化（参数结构）

**所有方法**的请求参数 **>= 2 个**时，禁止平铺所有参数，必须抽成单一对象：**Controller 层抽 Req 对象**（XxxReq，GET 经参数绑定、写请求经 @RequestBody 接收），**其他层抽语义对应的对象**（查询→XxxQuery、命令→XxxCmd、导出→XxxExportQuery 等），对象字段命名按变量细则。private 中间方法一视同仁。

#### 反例

```java
// ❌ 平铺散参——调用点读不出语义，加参数要改所有调用方
@PostMapping("/orders")
public Result<OrderVO> create(@RequestParam Long skuId,
                              @RequestParam Integer quantity,
                              @RequestParam String couponCode) { ... }   // Controller 平铺

public RefundResult refund(Long orderId, BigDecimal amount, String reason) { ... }  // Service 平铺

private void markRead(List<Long> msgIds, Long userId, boolean all) { ... }          // 内部方法同样平铺
```

错在哪：调用点 `create(1001L, 2, "NEW5")` 读不出每个散参是什么；新增一个参数（如渠道来源）要改动所有调用方签名；参数顺序错位是隐性 bug 温床。

#### 正例

```java
// ✅ Controller：Req 对象接收，字段即契约
@PostMapping("/orders")
public Result<OrderVO> create(@RequestBody @Valid OrderCreateReq req) {
    return Result.ok(orderCreateService.createOrder(req));
}

// ✅ 其他层：语义对应的对象
public RefundResult refund(RefundCmd cmd) { ... }                      // 命令 → Cmd
public List<CommentVO> listHotComments(CommentQuery query) { ... }     // 查询 → Query
private void markRead(MarkReadCmd cmd) { ... }                         // 内部方法同样抽对象
```

对在哪：调用点 `refund(cmd)` 一个名字说清意图；新增字段只动对象内部，调用方零改动；参数错位在类型层面消失；对象字段名即文档。

**判定问句**：方法入参 >= 2 个了吗？是 → 先抽对象再落笔。

#### 边界情况

- **构造器不受此条约束**：构造器的多字段参数是对象不变式的本义（领域实体构造器、全参构造），对象本身就是参数聚合，不存在「再抽一个对象」；
- **框架契约方法不受控**：覆写的框架方法（如 `doFilter(req, resp, chain)`）与调用的库 API 签名不归我们定，按规则二「框架落点」豁免；
- **单参数不抽**：1 个参数天然已是对象或语义完整，不为形式强拆；
- Req 对象是参数校验的载体：@NotBlank 等校验注解放对象字段上，@Valid 触发——呼应手册「入口层完成参数合法性校验」。

## 变量命名（字段、局部变量、常量、参数）

构词细则：

1. lowerCamelCase，**名词性命名**——变量名回答「它此刻装的是什么」：`orderId`、`payableAmount`、`hotComments`（规则五在本类的核心落点）。
2. 常量全大写、下划线分隔、语义完整：`MAX_RETRY_COUNT`、`DEFAULT_PAGE_SIZE`。
3. 布尔变量（字段与局部变量）用 is/has/can/should 前缀：`isVip`、`hasPaid`；POJO 属性豁免（见规则五）。
4. 单位入名、容器入名：`durationMs`、`timeoutSeconds`、`priceCent`；集合名体现内容（复数）或容器类型（`orderList`、`userMap`）。
5. 禁止变量漂移与尾缀数字复用（`list2`、`str1`）：数据经处理后另起新名。

```java
// ❌
Map<String, List<Sku>> data = ...;     // 什么数据？
boolean flag = ...;                     // 什么状态？
long timeout = 3000;                    // 单位是秒还是毫秒？
List<Order> list2 = ...;                // 尾缀数字，与 list1 什么关系？

// ✅
Map<String, List<Sku>> skusBySpuId = ...;   // 键值语义清楚
boolean membershipExpired = ...;
long timeoutMillis = 3000;
List<Order> refundableOrders = ...;
```

错在哪/对在哪：❌ 组每个名字都要读上下文才能猜；✅ 组从名字直接读出内容、单位与状态。

**判定问句**：只看变量名，能说出它装的是什么（内容、类型、单位、状态）吗？

### 边界情况

- 方法参数名同样按变量细则执行；入参抽成对象后（见上方「入参对象化」），**对象字段名承担契约文档职责**——`RefundCmd` 的 `orderId`、`amountCent` 字段优于单字母字段；单参数方法的参数名按变量细则。
- 循环变量、临时交换变量等语言惯例外已在规则二边界排除，不重复约束。

---

## AI 高频坏味道专组

AI 生成的 Java 代码高频出现以下命名问题，编码自检时优先比对：

| 坏味道 | 典型形态 | 纠正方向 |
|---|---|---|
| 空壳动词三件套 | 业务实现类里的 `process(...)`、`handle(...)`、`manage(...)` | 换具体业务动词（规则一、二）；**豁免按「语义落点」原则判定**（规则二豁免） |
| 万金油返回/入参 | `Result`、`data`、`res`、`ret`、`info` | 换具体名词（规则二） |
| 尾缀数字复用 | `list2`、`map1`、`str3` | 另起新名（规则五、变量命名） |
| 后缀滥用 | `UserInfo`/`UserDetail` 无新增含义 | 不加无信息量的后缀（规则三） |
| 拼写错误 | `Calcuate`、`Recieve` | 交付前 grep 拼写 |
| 复制后不改名 | 复制 A 方法改名 `newMethod` | 按新职责重命名（规则一） |
| 拼音/拼音缩写 | `yhMc`、`dingDanList` | 英文写全（规则四，手册【强制】） |
| 语义分级错误 | `getUser` 实际可能返回 null | `findXxx` + `Optional` 表达可能无（方法命名） |
| 平铺散参 | `refund(orderId, amount, reason)` 散参调用点 | 抽成 Req/Cmd/Query 单对象（入参对象化） |
