---
name: zone-coding-standards
description: Java 编码规范基线：AI 执行任何 Java 编码任务（编写、修改、重构、评审、解释 Java 代码，设计 Java 类/组件/服务）时默认装载遵守，无需用户点名。规范裁决链固定为「项目自身规范 > 阿里巴巴 Java 开发手册 > 本 skill 自定义规则」，自定义规则含命名规范五条、组件规范四条、分层规范（单模块 Controller 层两条 + DDD 多模块工程六条：领域框架无关/依赖方向/依赖倒置/对象归位/装配根/ArchUnit 守护）与 Spring Boot 工程惯例（Lombok 优先、字段注入为主），每条规则展开必带正反例与判定问句；表述业务领域中立，示例 Java 为主、必要处含 Spring Boot 写法。触发时机：用户要求实现/修改/重构/评审 Java 代码、解决命名、组件设计、分层放置、模块划分或工程惯例问题，或在 DDD 多模块工程中新建/迁移模块与代码时，或 AI 自检 Java 代码质量；落笔前装载，而非评审时才补救。
---

# Java 编码规范基线

写 Java 代码前先过本规范：规范在**落笔时**生效，不是评审时才补救。规则骨架在本文件，完整展开与边界情况按需读 `references/`。

## 规范裁决链（固定，自上而下）

1. **项目自身规范**（若有）：团队规范、项目 README 中的约定，最优先。
2. **阿里巴巴 Java 开发手册**：Java 项目直接适用；【强制】条款速查见 [references/alibaba-core.md](references/alibaba-core.md)。
3. **本 skill 自定义规则**：命名规范五条 + 组件规范四条 + 分层规范（单模块 Controller 两条 / DDD 多模块六条），补手册未覆盖或需强化处。
4. 条款冲突取**更严格**者；裁决不了问用户。

## 硬约束

- Java 编码任务默认装载本规范，无需用户点名；用户点名其他规范时按裁决链处理。
- 每条规则展开（无论口头解释还是写进文档）必须带正反例：❌ 反例说明**错在哪**，✅ 正例说明**对在哪**，并附一句判定问句。
- 表述与示例业务领域中立：示例至少跨三个业务场景，不出现任何特定项目字眼。
- 详述文档按需读，不要求每次全读；但拿不准的规则必须先读对应详述再落笔。

## 命名规范五条（一切标识符通用）+ 分类细则（类 / 方法 / 变量）

详述与边界情况见 [references/naming-guide.md](references/naming-guide.md)——通用五条之外，按类/方法/变量分类给出各自的构词细则，命名前先对号入座。

1. **业务贴合**：命名还原当前业务动作与状态，禁止与业务脱节。
   ❌ `process(data)`（说不出干什么） ✅ `calcCouponDiscount(coupon)`
   判定问句：只看名字不看实现，能否说出它在业务里干什么？
2. **禁宽泛词**：data、info、result、handle、manager 等万金油词白名单制，仅限真正的通用工具；**豁免按「语义落点」原则判定**——通用动词合法当且仅当语义有唯一明确落点（契约 / 框架覆写 / 泛型负载 / 领域术语四类，详见详述）。
   ❌ 具体业务类里的 `handle(msg)`（语义无处落） ✅ `sendPaymentReminder(order)`；✅ `PayChannelStrategy.process(...)`（契约落点）
   判定问句：这个通用词的具体语义，在系统里有没有唯一明确的落点？没有 → 具体业务里就是偷懒。
3. **禁多义词**：一个概念全系统只用一个词，一个词只表达一个概念。
   ❌ 「客户」混叫 customer/client/user ✅ 全局一致一个词
   判定问句：全局搜索这个词，出现的含义是否唯一？
4. **禁无脑缩写**：只保留公认缩写（id、url、api 等），其余写全。
   ❌ `usrNm`、`calcDscnt` ✅ `userName`、`calcDiscount`
   判定问句：新人不查文档，能猜对这个缩写吗？
5. **命名匹配数据状态**：名字描述「此刻装的是什么」——count 只装数量、布尔带 is/has 前缀（POJO 属性按手册豁免）、单位入名。
   ❌ 分页单页数据叫 `orderList` ✅ `orderPage`；❌ `long count = order.getTotalAmount()` ✅ `payableAmount`
   判定问句：变量的名字和它此刻装的东西，是同一件事吗？

## 组件规范四条（类、Service、方法等可复用封装单元）

详述与 Spring Boot 写法见 [references/component-guide.md](references/component-guide.md)。

1. **沉淀通用组件**：相同/相似逻辑出现 ≥2 次，参数化差异点沉淀为通用组件。
   ❌ 三个 Service 各复制一份手机号校验 ✅ 抽出 `PhoneValidator` 组件
   判定问句：这段逻辑第三次出现时，你是复制还是引用？
2. **单一职责**：一个组件只回答一个问题，高内聚低耦合。
   ❌ `OrderService` 又算又导出 ✅ 拆 `OrderCalcService` / `OrderExportService`
   判定问句：一句话描述这个组件的职责，需要「并且」吗？
3. **返回值收敛**：返回值只与组件功能相关，不夹带无关副产物。
   ❌ 查询返回值夹带缓存命中数、耗时字符串 ✅ 副产物走日志/事件/独立方法
   判定问句：返回值里有没有调用方根本不需要的字段？
4. **中间逻辑私有化**：两类场景必须收 private——
   - 组件内部：方法片段过长，抽取中间逻辑为 private 方法；
   - 组件间：A 组件用 B 组件的结果做长逻辑处理，该处理是 A 的 private 方法——不回塞 B、不散落在调用现场。
   ❌ B 为 A 暴露 public 中间方法 ✅ A 内 `assembleXxx()` private
   判定问句：这个方法是组件的「能力」还是「步骤」？步骤必须 private。

## 分层规范 · Controller 层（两条）

详述与边界情况见 [references/layering-guide.md](references/layering-guide.md)。

1. **依赖边界**：Controller 只能引入 Service 业务层组件；禁止引入持久层组件（DAO/Repository/Mapper 等），触达数据与业务的基础设施同理下沉。
   ❌ Controller 直注 `OrderMapper` 绕过 Service 查库 ✅ 只注入 `OrderQueryService`
   判定问句：Controller 的依赖字段里，有没有 Service 以外的业务组件？
2. **职责边界**：Controller 不写业务逻辑、不做对象转换（DTO→VO 归 Service，Service 直接返回 VO），只做三件事——接参数（含触发参数校验）、调 Service、组返回（统一响应体 + HTTP 语义）。
   ❌ 在 Controller 里判库存、算金额、调 Converter 转 VO ✅ `Result.ok(orderCreateService.createOrder(cmd))`（Service 直接给 VO）
   判定问句：Controller 里除了接参数、调 Service、包返回，还剩什么？剩了就越界。

## 分层规范 · DDD 多模块工程（六条）

适用 Maven 多模块 DDD 工程（模块划分 domain / infrastructure / service / web / bootstrap / common）；单体三层工程的 Controller 条款即上一节。详述、正反例与派生迁移指引见 [references/ddd-layering-guide.md](references/ddd-layering-guide.md)。

依赖方向固定单向：`web(适配) → service(应用) → domain(领域) ← infrastructure(实现)`，装配根（bootstrap）运行期注入实现并用 ArchUnit 守护。

1. **领域层框架无关**：domain 只放纯 POJO（实体/值对象/领域服务/事件）与接口（仓储/防腐网关），禁任何框架与中间件依赖；实体禁持久化注解；接口签名只出现领域对象。
   ❌ 实体标 `@TableName`、domain 引 Spring/MyBatis ✅ 持久化注解归 infra 的 DO，业务规则写成实体方法
   判定问句：domain 模块单独编译、不引任何 starter，能过吗？
2. **依赖方向单向**：适配层不被任何层访问，应用层只被适配层访问，基础设施层不被任何层访问（实现仅由装配根注入）且不反向依赖上层。
   ❌ service 注入 Mapper、infra 反向 import service ✅ service 注入 domain 仓储接口，infra 实现
   判定问句：顺着 import 走，会不会从应用层走进实现细节、或从底层翻回上层？
3. **应用层只依赖领域接口**：数据访问经仓储接口、外部调用经防腐网关接口；接口在 domain、实现在 infrastructure、装配根注入（依赖倒置三件套缺一不可）。
   ❌ 应用服务直调 HTTP 客户端/Redis/Mapper ✅ `domain.gateway` 定义接口，infra 实现，应用服务依赖接口
   判定问句：换数据源或换外部服务，应用服务要改吗？要 → 依赖错了。
4. **对象各归其位**：Req/Res/Assembler 在应用层，DO/Converter/Mapper 在基础设施层，实体/值对象在领域层；转换不跨层代做。
   ❌ Controller 手搓 VO、DO 出适配层、应用层引 infra 的 Converter ✅ Assembler（实体→Res）在 service，Converter（DO↔实体）在 infra
   判定问句：这个对象的包路径，和它的职责对得上吗？
5. **装配根显式聚合**：只有 bootstrap（启动模块）同时依赖 web+service+infrastructure，负责配置、Mapper 扫描与运行期注入，不写业务。
   ❌ 业务散在启动模块、其他模块依赖启动模块 ✅ bootstrap 只做装配 + 守护测试
   判定问句：启动模块里出现业务代码了吗？出现 = 越界。
6. **结构由测试守护**：分层规则用 ArchUnit 写进装配根测试域（排除测试代码），违规 `mvn test` 红；新增规则同步依赖方向图。
   ❌ 分层靠 review 口头把关 ✅ 六条规则测试化，构建即拦截
   判定问句：这条分层约定，机器在 `mvn test` 里能拦住吗？不能 → 补规则。

## Spring Boot 工程惯例（两条）

详述与坑见 [references/springboot-guide.md](references/springboot-guide.md)。

1. **Lombok 优先**：getter/setter、toString、equals/hashCode、logger 声明、Builder 等样板代码用注解生成，禁止手写。
   ❌ 手写 30 行 getter/setter、每个类声明 Logger ✅ `@Data` / `@Slf4j`
   判定问句：这段样板代码是手写的吗？是 → 换注解。
2. **字段注入为主**：Spring Bean 依赖默认 `@Resource` / `@Autowired` 字段注入，尽量不用构造器注入（`private final` + 构造器参数）；多实现用 name/Qualifier 区分。
   ❌ `private final OrderQueryService service;` + 构造器参数 ✅ `@Resource private OrderQueryService service;`
   判定问句：这个 Bean 的依赖是构造器塞进来的吗？是 → 改字段注入。
   > 例外按裁决链：DDD 多模块工程若项目自身约定 `@RequiredArgsConstructor`（Lombok）+ `private final` 构造器注入（不可变依赖），以项目规范为准（见 [references/ddd-layering-guide.md](references/ddd-layering-guide.md) 工程惯例补充）。

## 编码自检

- **落笔前**：命名过五条；新写组件先想职责边界；接入层代码先想分层边界（只依赖 Service、不写业务）；**DDD 多模块工程先对六条**（领域纯净、依赖方向、依赖倒置、对象归位、装配根、测试守护），新模块/新类先确认落点层。
- **交付前**：组件过四条、Controller 过两条、DDD 工程六条、工程惯例过两条（样板换注解、依赖字段注入或项目约定的构造器注入）；再对照手册【强制】条款速查（[references/alibaba-core.md](references/alibaba-core.md)）扫一遍。
