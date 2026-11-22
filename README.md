# zone-coding-standards

Java 编码规范基线（Agent Skill）：AI 执行任何 Java 编码任务（编写、修改、重构、评审、解释代码，设计类/组件/服务）时默认装载遵守，无需用户点名。规范在**落笔时**生效，而不是评审时才补救。

## 规范裁决链（固定）

1. **项目自身规范**（若有）：团队规范最优先；
2. **阿里巴巴 Java 开发手册**：【强制】条款已内置自写提炼速查；
3. **本 skill 自定义规则**：命名规范五条 + 组件规范四条；
4. 条款冲突取**更严格**者，裁决不了问用户。

## 内容结构

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 骨架：触发时机、裁决链、规则清单（命名五条 + 组件四条 + 分层两条 + 工程惯例两条）+ 精简正反例与判定问句 |
| `references/naming-guide.md` | 命名规范：通用五条 + **类/方法/变量分类细则**；宽泛词禁令的「语义落点」豁免原则 |
| `references/component-guide.md` | 组件规范：通用沉淀、单一职责、返回值收敛、中间逻辑私有化（含 Spring Boot 写法） |
| `references/layering-guide.md` | 分层规范：Controller 层两条——依赖边界（只依赖 Service、禁持久层）、职责边界（不写业务逻辑） |
| `references/springboot-guide.md` | 工程惯例：Lombok 优先（样板注解化 + 典型坑）、字段注入为主（@Resource/@Autowired） |
| `references/alibaba-core.md` | 阿里手册【强制】条款速查（泰山版自写提炼，以原文为准） |

每条规则展开必带 **❌ 反例（错在哪）+ ✅ 正例（对在哪）+ 判定问句**；表述业务领域中立，示例 Java 为主、必要处含 Spring Boot 写法，可跨项目复用。

## 亮点设计

- **分类细则**：类名名词性、方法名动宾结构带查询语义分级（get/find/list/count）、变量名描述数据状态——不同标识符类型分开约束；
- **豁免不搞一刀切**：process/handle 等通用动词按「语义落点」原则判定——语义在契约（策略接口）、框架覆写、泛型负载、领域术语中有唯一明确落点的判合法，落点含糊即偷懒命名；
- **AI 高频坏味道专组**：针对 AI 生成代码的高频命名问题（空壳动词、万金油返回值、尾缀数字复用等）设专项自检；
- **分层边界**：Controller 只依赖 Service 业务层、禁止直连持久层、不写业务逻辑——依赖禁令按「是否触达数据与业务」判定而非穷举清单；
- **工程惯例**：样板代码一律 Lombok 注解生成（@Data/@Slf4j/@Builder），Spring Bean 依赖用 @Resource/@Autowired 字段注入、不用构造器注入。

## 安装（ZCode / 兼容 SKILL.md 的 Agent 环境）

```bash
git clone https://github.com/shoanjen/zone-coding-standards.git
ln -s "$(pwd)/zone-coding-standards" ~/.zcode/skills/zone-coding-standards
```

其他 Agent 工具按各自的 skill 目录约定链接或复制本目录即可（入口为 `SKILL.md`）。
