# active-jira-workflow

`active-jira-workflow` 是面向 Active 团队的 Jira 与飞书 Agent 工作流仓库。它以本地 `ankitpokhrel/jira-cli` 和官方 `lark-cli` 为执行底座，把通用 Jira 操作、项目规则化报告、飞书资源操作和 OpenClaw 定时提醒拆成四个可独立演进的 Skill，供 Codex、GitHub Copilot 等 Agent 使用。

## 核心能力

- **通用 Jira 操作**：查询 issue、epic、sprint、release、project 与 board；在用户明确授权后创建、编辑、流转、评论、指派、链接、克隆和关注 issue。
- **Active 字段规则**：为 Bug 创建与编辑提供固定字段 ID、必填字段和动态枚举查询辅助脚本，避免猜测自定义字段。
- **长期未处理报告**：完整分页读取 Jira，按 Severity、状态风险、责任人和超期时长排序，生成包含 Highlight、按归属 Team 分组清单及汇总统计的 Markdown 报告。
- **飞书发布**：将 Markdown 创建或更新为飞书云文档，可为指定群授予查看权限并发送文档链接；外部写操作支持 `--dry-run`。
- **OpenClaw 定时提醒**：把自然语言筛选意图固化为可审计的 `query_spec`、`base_jql` 与窗口语义，复用确定性脚本完成查询、去重、详情拉取、interactive 卡片渲染和飞书投递。

## 仓库组成

| 目录 | 职责 |
| --- | --- |
| `active-jira/` | 通用 JiraCLI 原子能力与 Active 字段规则。 |
| `active-jira-report/` | 项目化报告、长期未处理 Jira 分析和规则化建单。 |
| `active-lark/` | 官方 Lark CLI 的通用飞书能力与 Markdown 文档发布。 |
| `active-jira-automation/` | OpenClaw 原生 Jira 定时查询提醒契约和确定性运行时。 |
| `install.sh` | 源码同步、jira-cli 初始化、Skill 部署及本地管理命令安装。 |
| `jira-cli.sh` / `lark-cli.sh` | JiraCLI 安装与可选 Lark CLI 配置辅助入口。 |

## 环境要求

- Git；私有仓库访问推荐使用已认证的 GitHub CLI。
- Jira 查询需要本机可用的 `jira` 命令和有效 Jira 配置。
- 飞书发布为可选能力，需要 Node.js/npm/npx、官方 `lark-cli` 及相应 OAuth/应用权限。
- OpenClaw 定时场景由 OpenClaw 负责 schedule、时区、session、重试和任务持久化；本仓库不维护生产 cron 状态。

## 安装

推荐先认证 GitHub，再执行私有仓库安装入口：

```bash
gh auth login
gh auth setup-git
sh -c "$(gh api --method GET -H 'Accept: application/vnd.github.raw+json' /repos/active-ailab/active-jira-workflow/contents/install.sh -f ref=main)"
```

从已有 checkout 安装时：

```bash
cd active-jira-workflow
PROJECT_DIR="$(pwd)" sh install.sh
```

安装器会准备源码、检查或安装 jira-cli、执行 `jira init`、把仓库内 Skills 软链接到 Codex/Copilot skills 目录，并生成默认位于 `~/.local/bin/active-jira` 的维护命令。Lark CLI 默认是可选增强能力，可使用 `INSTALL_LARK_CLI=1` 显式启用。

> 注意：安装器脚本中的默认 Skill 列表会扫描仓库内 `*/SKILL.md` 并补充新 Skill；使用旧的显式 `SKILL_REL_PATHS` 配置时，请确认其中包含 `active-jira-automation`。

常用维护命令：

```bash
active-jira skills
active-jira update
active-jira version
sh lark-cli.sh doctor
```

## 快速使用

### 查询 Jira

```bash
jira issue list --raw -q 'project = GENEVA AND status != Closed'
jira issue view GENEVA-123 --raw
```

### 生成长期未处理报告

```bash
python active-jira-report/scripts/generate_stale_jira_report.py \
  --project GENEVA \
  --age 1w \
  --output reports/geneva-stale-jira.md
```

生成器支持完整分页、可选详情补全、评论采样、额外 JQL、离线 JSON 回放以及 `--dry-run`。项目和超期阈值必须显式给出。

### 发布报告到飞书

先预览，再在目标群 ID 明确后执行：

```bash
python active-jira-report/scripts/publish_stale_jira_report_to_lark.py \
  --project GENEVA \
  --age 1w \
  --report-input reports/geneva-stale-jira.md \
  --chat-id oc_xxx \
  --grant-chat-view \
  --dry-run
```

确认无误后移除 `--dry-run`。不要根据群名猜测接收方，应先解析并确认稳定的 `oc_...` ID。

### 创建 OpenClaw Jira 提醒

在 OpenClaw 会话中描述需求，例如“每小时检查 GENEVA 新增的 P0 Bug 并推送到测试告警群”。创建前应确认：

- 只含业务条件的 `base_jql`；
- `created`、`updated` 或 `snapshot` 窗口模式；
- schedule、时区和默认 `isolated` session；
- 稳定的 `target_chat_id`；
- 去重与每次最大通知数量。

运行时入口为：

```bash
python active-jira-automation/scripts/run_automation_task.py <TASK_ID> --jira-bin jira --dry-run
```

OpenClaw 是生产任务状态的唯一来源；仓库中的本地 task store 和 scheduler adapter 主要用于开发与测试。

## 安全边界

- 默认优先只读查询；创建、编辑、流转、评论和删除 Jira 必须有明确用户意图。
- 删除 Jira、覆盖飞书文档、授权和发消息属于高风险操作，应明确目标并先预览。
- 不在仓库中保存 Jira Token、飞书 App Secret、access token 或 refresh token。
- 报告必须保留查询时间、项目、阈值、命令、JQL 和分页状态；查询失败时不得编造结果。
