# 阿里巴巴 Java 开发手册【强制】条款速查

> 依据《阿里巴巴 Java 开发手册》泰山版（2020 年公开发布版）的【强制】级条款**用自己的话提炼**，按 AI 编码最易踩的顺序分组；非手册原文，如与原文冲突以原文为准。手册还有大量推荐/参考级条款与安全规约，此处只收编码时最高频的强制项——拿不准时按裁决链查手册原文。

## 命名

- 标识符不得以下划线 `_` 或美元符 `$` 开始、结束。
- 禁止纯拼音命名、拼音英文混合命名（业界通用的中文拼音名词如 `taobao`、`hangzhou` 等除外）。
- 类名 UpperCamelCase；领域模型按约定后缀：数据对象 `xxxDO`、传输对象 `xxxDTO`、展示对象 `xxxVO`、业务对象 `xxxBO`、查询对象 `xxxQuery`、异常类 `xxxException`。
- 方法名、变量名、参数名 lowerCamelCase。
- 常量全大写、下划线分隔、语义完整，禁止一个字母或无语义字母组合（如 `A`、`STATUS_1`）。
- 抽象类 `Abstract` 或 `Base` 开头；测试类以测试的类名开头、`Test` 结尾。
- 枚举成员名全大写，单词间下划线分隔。
- POJO 类布尔属性**禁止** `is` 前缀（部分框架反序列化会丢 `is`，导致属性映射失败）；局部变量与查询方法（`isXxx()`）不受此限。
- Service/DAO 层实现类以 `Impl` 结尾。
- 杜绝不规范的缩写，避免望文不知义（对应自定义规则四）；代码中的单词拼写必须正确。

## 常量与变量

- 禁止魔法值：未经预定义的常量值不得直接出现在代码里，先定义常量再使用。
- `long` 或 `Long` 赋值用大写 `L`，小写 `l` 与 `1` 难分辨。
- 浮点数之间的等值判断不用 `==`（精度陷阱），基本类型用误差范围比较，包装类型用 `compareTo`/`BigDecimal`。

## OOP

- 不使用/不复制已标 `@Deprecated` 的方法与类，迁移到替代实现。
- `equals` 比较时，常量或确定非空的对象写在左侧（`"abc".equals(variable)`），避免 NPE。
- 所有 POJO 类必须重写 `toString()`（继承父类的要调 `super.toString()`），便于排查问题时看清字段值。
- 重写 `equals` 必须重写 `hashCode`，且两者判定标准一致，否则集合（HashSet/HashMap）行为错乱。
- 包装类之间的值比较**一律用 `equals`**，不用 `==`（`Integer` 在 -128~127 之外是不同对象，`==` 比较的是引用）。
- POJO 类属性使用包装类型；RPC 方法的参数与返回值必须用包装类型；局部变量推荐基本类型。
- 定义 DO/DTO/VO 等 POJO 类时禁设定默认值（默认值会掩盖「未赋值」的真实状态）。
- 构造方法里禁止复杂业务逻辑，复杂初始化抽到 `init` 方法；构造方法禁注入逻辑后仍被重载混淆。
- 类成员与方法访问粒度从严：能用 private 不用 protected，能模块内不用跨模块（对应自定义规则四）。

## 集合处理

- `ArrayList.subList()` 的结果不可强转 `ArrayList`（是内部视图类），强转会抛 `ClassCastException`。
- 集合转数组必须用 `toArray(T[] array)` 传入类型、大小完全一致的数组；直接 `toArray()` 无参转出的 `Object[]` 不能强转。
- `Arrays.asList()` 返回的 List 不可做 `add`/`remove`/`clear`（`UnsupportedOperationException`）；要可变列表就再包一层 `new ArrayList<>(...)`。
- 不要在 `foreach` 循环里对集合做 `remove`/`add`；`remove` 用 `Iterator` 方式，并发场景需加锁（否则 `ConcurrentModificationException` 或数据错乱）。
- `Comparator` 实现必须满足自反性、传递性、对称性，否则排序结果不可预期甚至抛异常。
- 集合泛型要用通配符区分读写场景（`? extends` 读、`? super` 写），减少运行期强转。

## 并发处理

- 线程池**禁止**用 `Executors` 快捷方法创建（无界队列/无限线程有 OOM 风险），必须手工 `new ThreadPoolExecutor(...)` 明确核心参数与有界队列。
- `ThreadLocal` 用完必须 `remove()`，尤其线程池复用线程的场景，否则内存泄漏与脏数据。
- `SimpleDateFormat` 线程不安全，禁止静态共享定义；用 `ThreadLocal` 隔离或改用 Java 8 的 `DateTimeFormatter`。
- 加锁目标要明确：多资源并发修改时，保证加锁对象与被保护资源生命周期一致，防止「锁了 A 改了 B」的锁失效。
- 使用 `CountDownLatch` 等协调工具时，异常路径也要保证 `countDown()` 被执行，否则等待方永久阻塞。
- 避免多个 `Random` 实例竞争（种子碰撞），共享场景用 `ThreadLocalRandom`。
- 双重检查锁（单例懒加载）必须给实例加 `volatile` 修饰，防止指令重排返回半初始化对象。

## 控制语句

- `switch` 必须有 `default`（哪怕只做记录或break）；每个 case 必须 `break`/`return` 或显式注释穿透意图。
- 条件表达式（三目、条件分支）注意自动拆箱 NPE：包装类型参与比较/拆箱前先判空。
- 禁止超过 3 层的 `if-else` 嵌套：用卫语句提前返回（先处理异常/为空/越界分支再走主逻辑）。
- 参数校验放在入口层（Controller/对外方法），业务规则校验放业务层，不混做一层。

## 异常处理

- 禁止捕获后不处理：空 `catch` 块、只打印不分类处理都算吞异常；要么处理恢复，要么记日志后向上抛。
- 不要捕获大段代码包一个 `try`，只包裹可能抛异常的语句，便于定位与阅读。
- `finally` 块中禁止 `return`（会吞掉 try/catch 的异常与返回值）。
- 资源（流、连接、锁）必须关闭：优先 `try-with-resources`，否则 `finally` 中关闭。
- 禁止用 `Exception`/`Throwable` 一把兜底捕获，捕获最具体的异常类型，便于分层处理。
- 对外方法的运行时异常、受检异常必须在 javadoc 用 `@throws` 说明抛出条件。
- 防止 NPE 的返回约定：返回集合/对象的方法**返回空集合/空对象而不是 null**；查询数据库的结果可能为 null 时调用方必须判空。
- 不要让「用户输入不合法」走到运行时异常才暴露，入口层完成参数合法性校验。

## 日志

- 禁止 `System.out`/`System.err`/`e.printStackTrace()`，统一日志框架（SLF4J 门面）。
- 日志用占位符 `log.info("order paid, orderId: {}", orderId)`，禁止字符串拼接（性能与可读性）。
- 日志内容禁止携带用户敏感信息（手机号、身份证、密码、密钥），必须脱敏；对外接口日志同样脱敏。
- 异常日志要带上下文与异常堆栈（两参/三参形式 `log.error("msg: {}", param, e)`），只记一次，避免上下游重复打印同一异常。

## MySQL

- 表名、字段名禁用数据库保留字，全部小写下划线风格。
- 唯一索引 `uk_` 前缀，普通索引 `idx_` 前缀。
- 表必备三字段：`id`（bigint unsigned 自增主键）、`gmt_create`、`gmt_modified`。
- 表达「是/否」概念的字段用 `is_xxx`（tinyint，1 是 0 否），POJO 属性对应**不带 is 前缀**的布尔（见命名节）。
- 禁止 `select *`：按需取列，防全表扫描放大与 schema 变更脆弱。
- 超过三个表禁止 join：需要的数据在应用层组装或冗余设计，避免大结果集笛卡尔放大。
- 深分页（offset 极大）有性能悬崖：用游标/条件定位替代大 offset。
- `count(*)` 统计行数（`count(列)` 有 NULL 语义差异，`count(distinct col)` 另算）；空结果返回 0 不返回 null。

## 安全（编码最高频项）

- 任何用户可控输入必须做合法性校验（长度、类型、范围、格式），入口层完成。
- SQL 一律参数化（`PreparedStatement`/框架参数绑定），禁止字符串拼接 SQL。
- 用户敏感数据展示与传输必须脱敏/加密，日志与返回报文同等对待。
- 高并发写场景必须防重复提交（幂等），防刷有频控。
