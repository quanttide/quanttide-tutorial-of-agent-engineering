# 约束工程

约束工程（Constraint Engineering）是不依赖模型的自觉、把约束写进系统的工程方法。沙箱、审批、工具门禁、提示词组装等结构性手段共同划定智能体的行为边界，使约束成为系统的属性，而不是模型的选择。

## 为什么需要约束

智能体的自由度与风险成正比：它能读写文件、执行命令、访问网络、委派子 agent。若行为边界只靠提示词中的口头要求维持，会产生以下问题：

- 措辞冲突时，模型可能自行取舍，口头约束随时被打破。
- 升级模型或更换供应商后，约束效果可能漂移。
- 模型判断失误时，越界动作没有第二道防线。

约束工程把「期望」变成「结构」：模型在约束之内自由行动，约束之外无从发生。

## 约束的分层

约束按硬度分层，越靠外层越硬，越靠内层越软：

- 环境约束：沙箱、文件系统边界、进程隔离，物理上不可达，最硬。
- 执行约束：工具注册表、作用域、权限预设，能力上不存在。
- 决策约束：审批策略、规划模式、人工确认点，动作需授权。
- 上下文约束：系统提示组装、技能、压缩策略，影响判断与取向，最软。

越界路径通常要穿透多层约束才会发生。设计目标是让最危险的路径在最外层就被截断。

## DeepSeek Harness 中的约束面

### 环境约束

沙箱模式由 dsh-bash-sandbox 与 dsh-sandbox-policy 提供，是执行类工具的实际边界：

- read-only：只读，失败安全，默认。
- workspace-write：仅工作区内可写。
- danger-full-access：全权访问，慎用。

环境约束的纪律是默认只读、显式放宽。

### 执行约束

core/tools 维护作用域化的工具注册表。工具按 agent 划分作用域，子 agent 只见父级授予的工具，实现能力最小化。工具 schema 经 core/system-prompt 组装后才进入模型上下文，模型能调用的工具集合由系统决定。替换一个提供方即可整体改变能力面：把文件系统与进程提供方指向远程沙箱，Bash、PTY、LSP 便一起迁移。

### 决策约束

审批策略决定需要授权动作的处理方式：

- ask：需要审批的操作先询问用户，默认，无可用应答者时失败关闭。
- never：确定性拒绝，适用于 CI 与无人值守场景。

ask 是默认的人工把关点，危险动作必须经过人；never 则把可能危险的动作整体排除，换取确定性。

### 上下文约束

系统提示组装（core/system-prompt）把提示词片段按组合包逐层叠加，可在 profile patch 中注入约束性规则，适合表达偏好与流程，不适合表达安全边界。技能（SKILL.md）把规范与检查清单固化为可加载文件。压缩策略（dsh-compaction-basic）默认在上下文窗口 80% 时压缩、保留 16%，防止上下文失控。模型参数 reasoningEffort、thinking、maxTokens 约束思考与输出预算。

### 行为范式约束

规划模式要求先提交计划、经用户批准后再执行，把「先想后做」变成流程约束。目标（goal）把长期意图固化为可追踪的约束，避免目标漂移，支持自主轮次续跑。

## 实践原则

1. 最小权限起步：从 read-only 或 workspace-write 开始，按任务需要显式放宽。
2. 外层优先：能靠环境约束解决的，不要靠审批；能靠审批解决的，不要靠提示词。
3. 失败安全：默认拒绝、显式授权，无法判断时按最保守路径处理。
4. 可审计：会话日志模型可见即已记录，约束的每次拦截都应能从日志还原。
5. 可替换：约束通过 fs、tools、telemetry 等 seam 策略事件挂载，可插拔、可升级。

## 反模式

- 把安全边界写进提示词：模型可违反、可漂移，无第二道防线。
- never 搭配 danger-full-access：审批与沙箱同时失效，约束形同虚设。
- 过度约束：agent 无法完成需要弹性的任务，约束沦为摩擦。
- 约束不可见：用户不知边界所在，误判 agent 的能力范围。

## 典型配置示例

只读审计任务的约束组合：

```yaml
# $DSH_HOME/settings.yaml 或 profile 的 cordis.patch.yml
dsh-sandbox-policy:
  defaultMode: read-only        # 环境约束：物理上不可写
dsh-approval-policy:
  policy: ask                   # 决策约束：异常动作询问用户
dsh-compaction-basic:
  enabled: true                 # 上下文约束：防止失控
```

全权开发任务的约束组合：

```yaml
dsh-sandbox-policy:
  defaultMode: workspace-write  # 环境约束：仅工作区可写
dsh-approval-policy:
  policy: ask                   # 决策约束：保留人工把关点
```

约束工程的核心不是松紧之争，而是每一层都有意识地设置边界，且边界可解释、可审计。

## 延伸阅读

- 库内资料：data/library 子模组的 DeepSeek Harness 索引。
- DSH 官方文档：[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 仓库的 docs 目录，含 architecture、config-catalog、subsystems 等章节。
