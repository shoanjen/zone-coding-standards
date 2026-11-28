# 中间件二方包使用规约（zone-middleware-support 五条）

技术底座基准：统一返回体、业务异常、日志、Redis、HTTP 客户端、JSON、令牌与配置键，默认以二方包 **`com.zone.middleware:zone-middleware-support`** 为准（本文类名、方法签名、配置键均对照 2026-11-28 源码与《接入手册》核实）；工程骨架基准为 zone-boot-template（分层落位见文末，分层规则见 [ddd-layering-guide.md](ddd-layering-guide.md)）。项目自身规范按裁决链仍最优先。

**基线事实卡**（写代码前对一眼，防凭旧印象落笔）：

- 坐标：groupId `com.zone.middleware`；能力模块 `zone-middleware-{common,log,redis,database,http,security,web}`；全量套件 `zone-middleware-suite`（packaging=pom）
- 基线：JDK 21、Spring Boot 3.5.6（jakarta 命名空间）；消费方需 Boot 3.1+，不兼容 Boot 2.7
- 全部能力走 starter 自动装配（`AutoConfiguration.imports`），**引入即生效，无需 @EnableXXX**
- **能力边界**：无 MQ、无分布式锁门面（仅 `setIfAbsent` 简易互斥）、无分布式 ID、无幂等/限流组件——这些需求直接用官方客户端或另立方案，**禁止臆造 `ZoneXxx` 门面**

---

## 规则一：接入坐标纪律——按需引入，套件必须 type=pom

接入方式二选一：**按需引入（推荐）**——properties 锁版本、只依赖用到的能力模块（除 common 外任何能力模块自动传递 log 与 common）；**套件引入**——`zone-middleware-suite` 单坐标全量，**必须声明 `<type>pom</type>`**。`com.zone.middleware:zone-middleware` 是构建聚合父 POM（packaging=pom，只传递 common 与 spring-boot-starter），**不是接入坐标**。

### 反例

```xml
<!-- ❌ 三种错法 -->
<dependency>                                        <!-- 错 1：把聚合父 POM 当接入坐标 -->
    <groupId>com.zone.middleware</groupId>
    <artifactId>zone-middleware</artifactId>        <!-- 只会带进 common，Redis/HTTP 等能力一个都没有 -->
    <version>1.0.0-SNAPSHOT</version>
</dependency>

<dependency>                                        <!-- 错 2：suite 不写 type=pom -->
    <groupId>com.zone.middleware</groupId>
    <artifactId>zone-middleware-suite</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>                                        <!-- 依赖解析即报 Could not find artifact ...:jar:... -->

<!-- 错 3：版本号散落各模块 pom 手抄，升级漏改一处就版本漂移 -->
```

错在哪：聚合父 POM 的 `<dependencies>` 是给六个子模块继承用的公共依赖，不是给上游用的菜单——依赖它等于什么能力都没接入；suite 是 pom 聚合坐标不产出 jar，缺省按 jar 解析必然失败；版本手抄在多模块工程里必然漂移。

### 正例

```xml
<!-- ✅ 按需引入（推荐）：properties 锁版本，只带用到的能力 -->
<properties>
    <zone-middleware.version>1.0.0-SNAPSHOT</zone-middleware.version>
</properties>
<dependency>
    <groupId>com.zone.middleware</groupId>
    <artifactId>zone-middleware-redis</artifactId>   <!-- 自动传递 log + common -->
    <version>${zone-middleware.version}</version>
</dependency>

<!-- ✅ 新应用一步到位：suite 全量，type=pom 必带 -->
<dependency>
    <groupId>com.zone.middleware</groupId>
    <artifactId>zone-middleware-suite</artifactId>
    <version>${zone-middleware.version}</version>
    <type>pom</type>
</dependency>
```

对在哪：版本单一源；能力模块清单即依赖清单，一眼看清这个工程用了哪些中间件能力；suite 声明 `type=pom` 与 Maven 解析语义对齐。

**判定问句**：这个坐标真能把要用的能力带进来吗？suite 带了 `<type>pom</type>` 吗？版本是 properties 锁的吗？

### 边界情况

- 多模块 DDD 工程的落位（能力就近、domain 纯净，详见文末落位表）：套件坐标只落**装配根**一处，能力模块按层各落各家，pom 不散落；
- 必配项缺失启动即快速失败：引入 database 必配 `spring.datasource.url`；security 的 `sign.enabled` 默认 `true`，启用签名校验必配 `zone.middleware.security.sign.secret`（不需要时显式设 `false`）；
- 版本升级属二方包 SNAPSHOT 期，破坏性变更可能发生（如 2026-11-27 成功码改 `"200"`、http 熔断移除、包路径迁移）——升级后以当前源码为准，不凭记忆写 API。

---

## 规则二：能力一律走门面——不绕过门面直用底层客户端

二方包封装的中间件能力，统一经门面类使用；门面没有的能力才直接用官方客户端。DDD 工程中门面调用的**落点**是基础设施层（仓储实现、gateway 实现内部），应用层只见领域接口（见 [ddd-layering-guide.md](ddd-layering-guide.md) 规则三）。

**门面速查**（类名 = 全限定名，签名已核对源码）：

| 能力 | 门面 | 典型签名 |
|---|---|---|
| 日志 | `com.zone.middleware.log.ZoneLog`（静态） | `info(String className, String methodName, String message, Object... args)`；`error(..., Throwable, Object...)` |
| 链路上下文 | `com.zone.middleware.log.ZoneTraceContext`（静态） | `bind(traceId, reqId)` / `getTraceId()` / `getReqId()` / `clear()` |
| Redis | `com.zone.middleware.redis.service.ZoneRedisTemplate` | `set(String key, Object value, Duration ttl)` / `<T> T get(String key, Class<T> type)` / `setIfAbsent(...)` / hash/list/set/zset/计数全套 |
| HTTP | `com.zone.middleware.http.client.ZoneHttpTemplate` | `<T> T get(String url, Class<T> type)` / `<T> T post(String url, Object body, Class<T> type)` / TypeReference 泛型重载 |
| JSON | `com.zone.middleware.common.util.JsonUtil`（静态，fastjson2 收口） | `toJson(Object)` / `fromJson(String, Class<T>)` / `fromJson(String, TypeReference<T>)` |
| 令牌 | `com.zone.middleware.security.token.ZoneTokenService` | `issueToken(userId, userInfo)` / `<T> T parseToken(token, Class<T>)` / `revokeToken` / `revokeUserTokens` |
| 登录上下文 | `com.zone.middleware.security.context.ZoneSecurityContext`（静态） | `currentUser()` / `currentUserId()` / `currentToken()` |

### 反例

```java
// ❌ 绕过门面，各写各的
@Service
public class StockAppService {
    private final StringRedisTemplate stringRedisTemplate;   // 直用底层客户端
    private final RestTemplate restTemplate;                 // 自建 HTTP 客户端
    private static final Logger log = LoggerFactory.getLogger(StockAppService.class);  // 手拼 Logger

    public void deduct(String skuId) {
        log.info("扣减库存 skuId={}", skuId);                // 无 traceId/reqId，链路断片
        stringRedisTemplate.opsForValue().set("stock:" + skuId, "5");
        restTemplate.postForEntity("/erp-api", body, String.class);  // 超时/重试/错误语义各自为政
    }
}
```

错在哪：超时、重试、错误码翻译、traceId 织入这些横切语义全部失效或各自为政——ZoneLog 统一补 traceId/reqId、ZoneHttpTemplate 统一超时与重试并抛标准异常、ZoneRedisTemplate 统一 fastjson2 序列化；绕过门面等于放弃这套一致性，出问题排查时各查各的。

### 正例

```java
// ✅ 门面收口；DDD 工程里落点在 infra 的 gateway 实现内
@Component
public class ErpGatewayImpl implements ErpGateway {          // ErpGateway 接口在 domain
    private final ZoneHttpTemplate zoneHttpTemplate;

    public ErpGatewayImpl(ZoneHttpTemplate zoneHttpTemplate) { this.zoneHttpTemplate = zoneHttpTemplate; }

    @Override
    public ErpResult deduct(StockDeduction deduction) {
        ErpResponse resp = zoneHttpTemplate.post("/erp-api", deduction, ErpResponse.class);  // 超时/重试/异常统一
        return toDomainResult(resp);                          // 外部模型在此翻译为领域概念，禁止外泄
    }
}
```

对在哪：横切语义（traceId、超时、重试、序列化、错误翻译）由门面一处保证；单测 mock 门面即可；换底层实现不动业务代码。

**判定问句**：这个中间件操作，门面有对应方法吗？有 → 走门面；没有 → 用官方客户端并说明原因。

### 边界情况

- 门面允许整体接管：Redis 装配是 `@ConditionalOnMissingBean`，需要深度定制时自己装配 `StringRedisTemplate`/`ZoneRedisTemplate` 即可顶替——定制归定制，其余调用点仍走门面；
- 出站 HTTP 需要熔断的，用资源规则把 `ZoneHttpTemplate` 调用包进 Alibaba Sentinel（二方包自研熔断已移除，不等待回归）；
- HTTP 鉴权透传实现 SPI `com.zone.middleware.http.interceptor.ZoneAuthInterceptor` 注册为 Bean 即自动应用，多个按 `@Order` 生效；
- 非业务静态工具（`ZoneIpUtils`/`ZoneBrowserUtils`/`ZoneRandomUtils`）直接静态调用，不算绕过——它们本身就是门面。

---

## 规则三：返回体与业务异常统一——ZoneResult / ZoneBusinessException

出参统一 `ZoneResult<T>` 包装（包装动作归接入层 Controller），业务失败抛 `ZoneBusinessException(ZoneError, message)` 交全局异常处理器翻译；自定义错误码实现 `ZoneError` 接口。**码段规约**：`SYS-` 段二方包独占；`BIZ-4001–4099` 为二方包内置保留段（上游 MUST NOT 侵占）；业务自定义码用未占段 `BIZ-7xxx` / `BIZ-9xxx`。判成功调 `isSuccess()`，不比对 `"200"` 字面量。

### 反例

```java
// ❌ 各造各的返回体、各抛各的异常
public class OrderCreateController {
    @PostMapping("/api/v1/orders")
    public ApiResult<OrderVO> create(...) { ... }            // 自造 ApiResult，全局口径分裂

    public String createOrder(...) {
        if (库存不足) return "STOCK_NOT_ENOUGH";             // 字符串当错误码，调用方猜谜
        throw new RuntimeException("已取消");                 // 裸异常，全局处理器接不住语义
    }
}
```

错在哪：返回体一多，前端/网关就得适配 N 套结构；裸异常与魔法字符串让全局异常处理器无法翻译成标准错误码，监控告警按码聚合的链路直接断。

### 正例

```java
// ✅ 返回体与异常全部走二方包体系
@RestController
@RequestMapping("/api/v1/orders")
public class OrderCommandController {
    @PostMapping
    public ZoneResult<OrderCreateRes> create(@RequestBody @Valid OrderCreateReq req) {
        return ZoneResult.ok(orderCommandAppService.createOrder(req));   // 包装归 Controller
    }
}

// 应用服务里：业务失败抛标准业务异常（错误码段 BIZ-）
if (!userGateway.exists(userId)) {
    throw new ZoneBusinessException(BusinessError.DATA_NOT_FOUND, "用户不存在：" + userId);
}
if (order.isPaid()) {
    throw new ZoneBusinessException(BusinessError.DATA_CONFLICT, "订单已支付，不可重复支付");
}
```

对在哪：`ZoneResult`（success/code/message/data/timestamp）与 `ZoneBusinessException` 全局唯一；`SystemError`（SYS-1001 配置 / SYS-2001 DB / SYS-2002 Redis / SYS-3001 HTTP / SYS-9999 兜底）与 `BusinessError`（BIZ-4001 参数 / BIZ-4002 未授权 / BIZ-4003 禁止 / BIZ-4004 不存在 / BIZ-4005 冲突 / BIZ-5001 业务失败）覆盖高频语义；自定义码进未占段，监控按码聚合不断链。

**判定问句**：这个失败，会被统一异常体系翻译成标准错误码吗？这个返回值，是全局唯一的那个返回体吗？

### 边界情况

- 判断成败一律 `result.isSuccess()`，不比对 `"200"` 字面量（成功码历史上从 `SUCCESS` 改为 `"200"`，字面量判等是升级地雷）；
- 二方包错误码不够用时：实现 `ZoneError` 接口建业务错误码枚举（码取 `BIZ-7xxx`/`BIZ-9xxx` 未占段），`ZoneBusinessException` 照常抛——扩而不裂；
- 内部实现细节（SQL 异常、空指针等）不往上抛——全局兜底统一翻译为 `SYS-9999`，不泄露内部细节；
- 全局异常处理器与 Trace Filter 本身由 `zone-middleware-web` 自动装配（`ZoneGlobalExceptionAdvice`/`ZoneTraceInboundFilter`），**工程不自有**，开关 `zone.middleware.web.exception-advice-enabled` / `trace-filter-enabled` 默认关、模板工程显式开。

---

## 规则四：JSON 与配置纪律——fastjson2 收口，官方键优先

**JSON 一律 fastjson2**：全模块依赖树无 jackson 构件，序列化统一走 `JsonUtil`（或 fastjson2 API），**禁止引入与使用 jackson**。**配置官方键优先**：连接、池、超时、迁移等资源配置全走 Spring Boot 官方键（`spring.datasource.*` / `spring.data.redis.*` / `spring.flyway.*` / `spring.http.client.*`）；`zone.middleware.*` 只放行为项（log / security / web / http 并发与重试）。同一配置项勿在 yml 与 properties 重复定义。

### 反例

```java
// ❌ JSON 引 jackson、配置回写方言键
ObjectMapper mapper = new ObjectMapper();                     // 禁：引 jackson 分裂 JSON 口径
String json = mapper.writeValueAsString(order);

// application.yml
zone:
  middleware:
    redis:
      host: redis.example.com                                 // 禁：zone.middleware.redis.* 旧方言键已被拆除，不生效
```

错在哪：jackson 与 fastjson2 双序列化并存，同一对象两种 JSON、两种注解体系，反序列化行为分叉；方言键写入后配置静默失效（去方言化后本库只认官方键），Redis 连回默认地址这类事故只能靠运行时发现。

### 正例

```java
// ✅ fastjson2 收口
String json = JsonUtil.toJson(order);
User user = JsonUtil.fromJson(json, User.class);
```

```yaml
# ✅ 官方键优先；zone.middleware.* 只放行为项
spring:
  datasource:
    url: jdbc:mysql://mysql.example.com:3306/app_db
  data:
    redis:
      host: redis.example.com
      database: 1
  flyway:
    baseline-on-migrate: true
  http:
    client:
      connect-timeout: 5s
      read-timeout: 10s
zone:
  middleware:
    log:
      app-name: order-center
      format: TEXT            # 只接受 TEXT / JSON，非法值启动失败
    web:
      exception-advice-enabled: true
      trace-filter-enabled: true
```

对在哪：JSON 一套真相（fastjson2 + JsonUtil）；连接面跟着 Spring Boot 官方文档走、升级零翻译成本；行为项集中在 `zone.middleware.*`，一眼看清二方包开关。

**判定问句**：这个 JSON 操作走 JsonUtil/fastjson2 了吗？这个配置项是官方键还是已拆除的方言键？

### 边界情况

- 配置快速失败语义是护栏不是障碍：log format 非法、datasource type 非法（只接受缺省 HikariCP 或 `com.alibaba.druid.pool.DruidDataSource`）、web req-id-length 越界（8–32）均启动失败——这是把配置错误挡在启动期，不改语义；
- 生产建议显式配置 Redis 超时/池与 Hikari/Druid 池参数（未配置回归官方默认）；共享中间件必须隔离：MySQL 独立 database、Redis 独立 db index，禁止另起实例占端口；
- 历史工程升级时按二方包手册 §15 对照表迁移旧 `zone.middleware.db/redis/http.*` 键，并用 grep 自查残留；
- Druid 监控页二方包不代注册：应用层自行决策，启用必须配口令 + IP 白名单。

---

## 规则五：横切与线程规约——签名、凭证、链路标识

三条横切铁律：**签名拼串禁用 HashMap**（无序导致双端拼串不一致，必须 `LinkedHashMap` / fastjson2 `JSONObject` / POJO）；**凭证不进 body**——登录令牌一律走 `Zone-Auth-Token` 请求头、签名走 `Zone-Access-Token` 请求头、链路标识走 `Zone-Trace-Id`（常量见 `ZoneConstants`）；**MDC 不跨线程池**——异步任务提交前显式复制 `ZoneTraceContext` 双标识（traceId + reqId），任务结束 `clear()`。

### 反例

```java
// ❌ 三条全踩
Map<String, String> params = new HashMap<>();                 // 禁：HashMap 无序，签名双端不一致
params.put("orderId", orderId);
params.put("userId", userId);
String sign = ZoneSignUtils.sign(params, secret);

@PostMapping("/api/v1/pay")
public ZoneResult<PayRes> pay(@RequestBody PayReq req) {      // req 里带 accessToken 字段——凭证进 body
    ...
}

executor.execute(() -> {                                      // 禁：子线程 MDC 空，日志链路断片
    ZoneLog.info("OrderService", "notify", "异步通知开始");     // traceId 丢失，无法串链排查
});
```

错在哪：HashMap 遍历顺序不稳定，同一个签名算法两端算出不同结果，偶现验签失败最难排查；令牌进 body 会被日志、网关、序列化到处复印，泄漏面失控；异步线程丢了 traceId，一条链路的日志在子线程处断成两截。

### 正例

```java
// ✅ 有序容器 + 请求头 + 显式复制上下文
Map<String, String> params = new LinkedHashMap<>();
params.put("orderId", orderId);
params.put("userId", userId);
String sign = ZoneSignUtils.sign(params, secret);

// 凭证：请求头携带（Zone-Auth-Token），服务端经 ZoneSecurityContext 取
String userId = ZoneSecurityContext.currentUserId();

// 异步：提交前复制、结束清场
final String traceId = ZoneTraceContext.getTraceId();
final String reqId = ZoneTraceContext.getReqId();
executor.execute(() -> {
    ZoneTraceContext.bind(traceId, reqId);
    try {
        ZoneLog.info("OrderService", "notify", "异步通知开始");
    } finally {
        ZoneTraceContext.clear();
    }
});
```

对在哪：签名串两端可复现；凭证只在头部流动、日志与报文不复印；子线程日志与主链路 traceId 连续，排查不断链。

**判定问句**：签名参数容器有序吗？凭证是在请求头里吗？这个异步任务结束时上下文 clear 了吗？

### 边界情况

- `ZoneSignCheck` 支持类/方法双粒度（方法优先），`timestampWindowMillis` 控时间窗、`requireLogin` 串接登录态；
- 入站 traceId 由 `zone-middleware-web` 的 `ZoneTraceInboundFilter` 自动透传/生成，出站到下游时把 `Zone-Trace-Id` 头继续带上；
- 登录态经 `ZoneSecurityContext` 门面读取，不直接解析请求头/令牌——令牌格式与续期策略变化时调用点零改动。

---

## 附：DDD 分层落位表（能力就近、domain 纯净）

| zone 模块 | 落位层 | 理由 |
|---|---|---|
| `zone-middleware-suite` | 装配根（bootstrap） | 坐标只落一处，pom 不散落 |
| `zone-middleware-database` | infrastructure | 数据源 / MyBatis-Plus / Flyway 属基础设施 |
| `zone-middleware-http` | infrastructure | 出站客户端与重试属基础设施（熔断走 Sentinel） |
| `zone-middleware-security` | web | 签名过滤器 / 拦截器作用于入站请求 |
| `zone-middleware-web` | web | 全局异常处理器 / Trace Filter 作用于入站请求 |
| `zone-middleware-log` | service / web | ZoneLog 埋点横切业务与接入层 |
| `zone-middleware-common` | service 及以下 | ZoneResult / 异常 / 工具被各层使用 |

> ⚠️ **domain 层禁引任何 zone 构件**：领域模型保持纯净，不依赖返回体 / 异常 / 日志门面（ArchUnit 规则同步约束 `com.zone.middleware..`，见 [ddd-layering-guide.md](ddd-layering-guide.md) 规则六）。
