# AI-CONFIG: AI 接入配置与自动回写规则
> AI-META: schema=pai-archive/v1; mode=config; auto_write=confirm; section=自动回写日志; eol=append-only; encoding=utf-8

> 说明：本文件是 AI 读取、写入与回写本档案的**唯一规则入口**。配置变更需用户显式确认。文中 `<...>` 与 `{{...}}` 均为占位符；**任何真实令牌都不得写入本仓库或任何 commit**。

## 0. 仓库坐标（公开元数据）

| 项 | 值 |
| --- | --- |
| GitHub 登录名 | `laoshen1987` |
| 仓库 | `https://github.com/laoshen1987/personal-ai-archive` |
| 默认分支 | `main` |
| 可见性 | 公开 |

以上为公开元数据，已在本文件所有 URL、OpenAPI schema 与配置片段中替换为真实值。文件中剩余的 `{{...}}` **仅用于待填写的个人信息字段**，不得批量替换。各 AI 平台的启用状态、验证证据与剩余一次性动作，以 `README.md` 搭建报告为准。

**本方案不创建、也不需要 PAT**：远端写入由 Git Credential Manager 的浏览器 OAuth（`git push`）完成；连接器因 App 权限不足未参与写入。任何真实令牌都不得写入本仓库或任何 commit。

## 1. 读取侧

### 1.1 统一读取入口

| 用途 | 方式 | 认证 |
| --- | --- | --- |
| 免认证快速读取 | `https://raw.githubusercontent.com/laoshen1987/personal-ai-archive/main/{path}` | 公开仓库无需认证 |
| 权威即时读取 | `GET https://api.github.com/repos/laoshen1987/personal-ai-archive/contents/{path}` | 免认证 60 次/小时；带凭据 5000 次/小时 |

raw 域名有 CDN 缓存（写入后可能滞后数分钟）。凡需要「读取后回写」，一律先用 Contents API 取最新内容与 `sha`：

```bash
curl -sS -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/laoshen1987/personal-ai-archive/contents/memory/memory.md?ref=main"
```

返回 JSON 中 `content` 为 Base64，`sha` 为该文件当前 blob SHA（回写必需）。加 `Accept: application/vnd.github.raw` 可直接取原文。

### 1.2 文件清单（AI 定位用）

| 路径 | 首行标签 | 读 | 写 |
| --- | --- | --- | --- |
| `identity/identity.md` | `# AI-READONLY` | 自由 | 核心区需确认，日志区可追加 |
| `workflow/workflow.md` | `# AI-READONLY` | 自由 | 规则区需确认，日志区可追加 |
| `memory/memory.md` | `# AI-APPEND` | 自由 | 主自动追加区 |
| `skills/skills.md` | `# AI-APPEND` | 自由 | 自动追加 |
| `archive/archive.md` | `# AI-APPEND` | 自由 | 自动追加，只增不删 |
| `ai-integration.md` | `# AI-CONFIG` | 自由 | 需确认 |
| `README.md` | `# AI-REPORT` | 自由 | 需确认 |

每个文件第二行的 `AI-META` 给出 `mode` 与 `auto_write`，AI 应据此判断写入策略，无需任何格式转换。

### 1.3 ChatGPT — Custom GPT Actions

在 GPT 配置页 → Actions → 粘贴以下 OpenAPI 3.1 schema。路径中的登录名已填为 `laoshen1987`，可直接使用；若迁移到其它账号，需同步替换。

```yaml
openapi: 3.1.0
info:
  title: Personal AI Archive
  version: 1.0.0
servers:
  - url: https://api.github.com
paths:
  /repos/laoshen1987/personal-ai-archive/contents/memory/memory.md:
    get:
      operationId: readMemory
      summary: 读取 memory（公开仓库免认证）
      parameters:
        - name: ref
          in: query
          required: false
          schema:
            type: string
            default: main
      responses:
        '200':
          description: Base64 content 与 blob sha
          content:
            application/json:
              schema:
                type: object
                properties:
                  sha:
                    type: string
                  content:
                    type: string
    put:
      operationId: appendMemory
      summary: 追加写入（需平台侧凭据；更新已存在文件必须带当前 sha）
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [message, content]
              properties:
                message:
                  type: string
                content:
                  type: string
                  description: UTF-8 原文的 Base64，单行无换行
                sha:
                  type: string
                  description: 取自 readMemory 的最新 blob sha
                branch:
                  type: string
                  default: main
      responses:
        '200':
          description: 已生成新 commit
components:
  securitySchemes:
    githubPat:
      type: http
      scheme: bearer
```

要点与限制（如实）：

- 认证支持 None / API key / OAuth；调用可能被要求用户批准。
- 含 `/` 的多段路径模板在 GPT Actions 中支持不稳定，故**每个文件写死一条 path** 最可靠；需要其它文件时复制 path 段并改 `operationId`。
- **`PUT` 需要模型自行生成完整 Base64 文件内容，可靠性有限**。建议 GPT 侧主要承担读取；写入优先交给 Claude MCP、WorkBuddy 或本地脚本执行第 3 节规则。
- 这是**平台侧一次性粘贴动作**，不属于仓库侧自动完成。

### 1.4 Claude — GitHub 连接器

两条官方路径（底层均为 GitHub 开源的 `github-mcp-server`，MCP 为开放标准）：

- **Remote**：在 Claude 的连接器设置中启用 GitHub，服务地址 `https://api.githubcopilot.com/mcp/`，按提示完成授权。
- **Local（Claude Desktop）**：编辑 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<平台侧一次性填入，绝不写入本仓库>"
      }
    }
  }
}
```

Local 路径也可走 OAuth device flow，无需分发令牌。二者均为**平台侧一次性配置**。配置完成后，可直接对 Claude 说「读取我的 `memory/memory.md` 并按 `ai-integration.md` 第 3 节回写」。

### 1.5 WorkBuddy

证据等级说明：**未找到 WorkBuddy 官方公开产品文档页可直连复核，以下路径来自第三方教程**。本机实测（2026-11-07）：官方 GitHub 连接器已连接并启用，可读取账号信息，但写入本仓库返回 403；实际推送改走 GCM 浏览器 OAuth。`~/.workbuddy/mcp.json` 未创建，无需自建 MCP 配置。

1. **首选（零配置、可验证）**：公开仓库 raw URL 直读，直接在对话中给出 URL 即可，无需任何配置文件。
   `https://raw.githubusercontent.com/laoshen1987/personal-ai-archive/main/memory/memory.md`
2. **不推荐（需 PAT）**：自建 `~/.workbuddy/mcp.json` 配 `@modelcontextprotocol/server-github` + `GITHUB_PERSONAL_ACCESS_TOKEN`。与本次「零 PAT」决策冲突，且无官方文档保证生效。
3. **连接器实测边界**：官方 GitHub 连接器已连接，但其 App 令牌对新建仓库无 `Contents: write` 权限（`PUT /contents` 返回 403），故本次推送未使用该通道，全程无需 PAT。

## 2. 写入侧

### 2.1 Contents API PUT 规范

```
PUT https://api.github.com/repos/laoshen1987/personal-ai-archive/contents/{path}
Accept: application/vnd.github+json
Authorization: Bearer <具备写权限的凭据>
X-GitHub-Api-Version: 2022-11-28
Content-Type: application/json; charset=utf-8

{
  "message": "pai: append memory/memory.md | by Claude | task=8f3a1c2d | 2026-11-07T14:02:11Z",
  "content": "<Base64(UTF-8 原文，单行，无换行，无 BOM)>",
  "sha": "<更新已存在文件时必填：该文件当前 blob sha>",
  "branch": "main"
}
```

- `message` 与 `content` 必填；每次成功调用**生成一个 commit**。
- **更新已存在文件必须带当前 blob `sha`**，缺失或不匹配返回 409 / 422。
- 与删除操作并行会冲突，**必须串行写入**。
- 权限：classic PAT 需 `repo` scope；fine-grained PAT 需 `Contents: write`。

### 2.2 commit message 规范

`pai: <create|append|update|revert> <path> | by <执行者> | task=<8位标识> | <ISO8601 UTC>`

便于在 `GET /repos/laoshen1987/personal-ai-archive/commits` 中按目录、来源 AI、时间定位历史版本。

### 2.3 强制 read-modify-write

任何追加写入前，先 `GET contents/{path}` 取最新 `sha` 与 `content`，解码后在**最新内容**上追加，再 Base64 回写。禁止基于缓存内容直接覆盖。

## 3. 更新规则：AI 完成任务后自动回写

### 3.1 触发条件

AI 完成一个「有可沉淀产出」的任务（新增偏好 / 决策 / 经验 / 技能变化 / 任务完成），且该产出**非敏感信息**。涉及身份核心事实的变更不自动触发，须用户显式确认。

### 3.2 任务产出 → 目标目录映射表

| 任务产出 | 目标文件 | 区域 | 模式 |
| --- | --- | --- | --- |
| 用户偏好、沟通结论、习惯观察 | `memory/memory.md` | 「AI 对话沉淀的个人偏好」+ 日志区 | 追加 |
| 一次重要决策及依据 | `memory/memory.md` | 「重要决策」+ 日志区 | 追加 |
| 经验教训 | `memory/memory.md` | 「历史经验」+ 日志区 | 追加 |
| 已完成项目 / 旧文档 / 已完成任务 | `archive/archive.md` | 归档区 + 日志区 | 追加，只增不删 |
| 新掌握技能 / 学习进展 | `skills/skills.md` | 日志区 | 追加 |
| 工作流 / 工具 / 优先级观察 | `workflow/workflow.md` | 日志区 | 追加；规则区变更需确认 |
| 身份 / 职业 / 人生目标变更 | `identity/identity.md` | 核心事实区 | **必须用户显式确认** |

### 3.3 追加式条目格式

```
- [2026-11-07T14:02:11Z] | src=Claude | task=8f3a1c2d | summary=用户确认偏好「先给结论再给论据」 | target=memory/memory.md#AI对话沉淀的个人偏好 | mode=append | commit=3f9b1a0
```

含时间戳、来源 AI、任务摘要、目标位置与写入模式，便于审计与回退定位。

### 3.4 硬约束

1. 禁止覆盖、禁止删除、禁止修改已有行（占位符 `{{...}}` 也不改）。
2. 禁止为写入而新建文件；只允许写入第 1.2 节清单内的文件。
3. 只在 `<!-- PAI:APPEND:START -->` / `<!-- PAI:APPEND:END -->` 锚点之间，或文末日志区追加。
4. 禁止任何 `force` 操作、禁止改写 commit 历史。
5. 禁止把真实令牌、密码、身份证号等写入任何文件。

### 3.5 冲突与失败兜底

| 情况 | 处理 |
| --- | --- |
| 409 / 422（`sha` 失配） | 重新 `GET` 取新 `sha` 与最新内容，重放追加，重试 1 次 |
| 401 | 立即停止并提示重新授权，不重试 |
| 403 / 429 | 读 `X-RateLimit-Reset`，指数退避最多 3 次 |
| 5xx | 退避重试 2 次 |
| 最终失败 | 条目暂存于 **AI 会话上下文**，在下一次任务开头补写；不落盘、不新建文件 |
| 仍无法写入 | 在 `memory/memory.md` 日志区追加一条 `status=deferred` 记录 |

## 4. Meta Muse / OpenAI Dots 预留段

**核查结论（如实）**：未找到可用于「自动同步个人偏好」的公开可验证 API。

- Meta Muse：公开可验证的只有无状态模型推理 API `https://api.meta.ai/v1`，官方文档明确 "no built-in memory between requests"；产品内的记忆能力不支持外部系统编程同步。
- OpenAI Dots：多个独立来源明确记载无公开 API、不可编程创建或配置。

因此**不编造任何端点**，仅保留禁用状态的抽象契约，待官方开放后启用：

```yaml
# PAI-RESERVED · 预留位，未启用，非真实端点
x-pai-reserved-sync:
  disabled: true
  reason: "未找到公开可验证 API，禁止臆造端点"
  server: "https://api.invalid.example/v1"
  paths: {}
  abstract_contract:
    read: "GET  <vendor>/v1/preferences → 映射 memory/memory.md#AI对话沉淀的个人偏好"
    write: "POST <vendor>/v1/preferences ← 来源 memory/memory.md 的偏好条目"
    auth: "OAuth 2.0 / 用户显式授权"
  enable_when:
    - "厂商公布官方偏好同步 REST 端点与鉴权方式"
    - "用户在对应平台侧完成一次性授权"
    - "填入真实 server 与 paths，并把 disabled 置为 false"
```

`server` 使用保留占位域名 `api.invalid.example`，`paths` 刻意为空，禁止解析或调用。

## 5. 版本保护与回退

### 5.1 分支保护配置

`PUT /repos/laoshen1987/personal-ai-archive/branches/main/protection` 请求体：

```json
{
  "required_status_checks": null,
  "enforce_admins": true,
  "required_pull_request_reviews": null,
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
```

- 四个 Required 字段均可为 `null`；`null` 表示禁用对应保护。
- `enforce_admins: true` 必须设置：否则管理员（含 owner 级凭据）默认可绕过强推限制。官方释义 "Enforce all configured restrictions for administrators"。
- `allow_force_pushes: false` / `allow_deletions: false` 阻止强推改写历史与删除分支。
- 三个 `null` + `lock_branch` 默认 false ⇒ **不要求 PR 审核，AI 仍可直接推送**。
- GitHub Free 的公开仓库可用；设置者需 admin / owner 权限（fine-grained 凭据需 `Administration: write`）。
- 不发送 `required_signatures`（不属于该端点请求体）。

**残留风险（如实）**：持有 `Administration: write` 的凭据仍可先删除保护规则再强推。可选加固：运行时 AI 回写改用仅 `Contents: write` 的最小权限凭据。副作用：`enforce_admins: true` 同样约束 owner 本人，日后只能用 `git revert` 清理，不能强推。

**无需 PAT 的网页设置路径**：仓库 → Settings → Branches → Add branch protection rule → Branch name pattern 填 `main` → 勾选「Do not allow bypassing the above settings」→ 不勾「Require a pull request before merging」，不勾「Allow force pushes」，不勾「Allow deletions」→ Create。

**当前生效状态（2026-11-08 实测）**：用户已按上述路径启用。公开端点 `GET /branches/main` 返回 `protected:true`；认证 `GET /branches/main/protection` 返回 HTTP 200，实测 `enforce_admins:true`、禁强推、禁删分支，`required_pull_request_reviews` / `required_status_checks` / `restrictions` 均为 `null`，与上方 JSON 完全一致。保护开启后普通 `git push origin main` 成功，AI 直接写入通道正常。

### 5.2 提交记录

Git 提交历史是版本保护的主体：每次 `git commit`、网页编辑文件、Contents API 的 `PUT` 都会生成一个 commit。这不是需要开启的开关，而是 Git 数据模型本身。配合上节的禁强推，历史不可被改写。

### 5.3 一键回退到任意历史版本

先定位历史版本：

```bash
curl -sS "https://api.github.com/repos/laoshen1987/personal-ai-archive/commits?per_page=100"
git log --date=iso --pretty='%h | %ad | %s'
```

再执行回退（把 `<SHA>` 换成目标 commit）：

```bash
git -c user.name="Personal AI Archive" -c user.email="archive@users.noreply.github.com" \
    revert --no-commit <SHA>..HEAD \
 && git -c user.name="Personal AI Archive" -c user.email="archive@users.noreply.github.com" \
         commit -m "pai: revert to <SHA> | by user | $(date -u +%Y-%m-%dT%H:%M:%SZ)" \
 && git push origin main
```

- `-c user.name/-c user.email` 就地补齐提交者身份（本机实测全局 git config 为空）。
- `--no-commit` 与 `commit` 成对，避免打开编辑器。
- 用 `git revert` 而非 `git reset --hard`：新建反向提交、不改写历史、普通 push 可完成、与分支保护兼容。GitHub REST 无针对任意 commit 的通用 revert 端点。
- 语义是「把内容恢复到该历史版本」，保留完整审计链；不是把分支指针移回去。

## 6. 诚实对照表

| 动作 | 执行方 | 是否一次性 | 是否仓库侧自动完成 |
| --- | --- | --- | --- |
| 生成 5 目录、7 个 Markdown 文件 | 本地执行器 + GitHub 连接器 | — | **是** |
| 网页新建空公开仓库 | 用户 | 是 | 否（App 无建仓权限） |
| 推送 7 个文件到 GitHub | AI（GCM 浏览器 OAuth / `git push`） | 否 | **是** |
| 启用分支保护 | 用户（网页） | 是 | 否（**已完成**，2026-11-08 经远端验证） |
| 在 ChatGPT Actions 粘贴 OpenAPI | 用户 | 是 | 否 |
| 在 Claude 配 MCP / 连接器 | 用户 | 是 | 否 |
| 在 WorkBuddy 给出 raw URL（零配置）或建 mcp.json | 用户 | 是 | 否 |
| 让 AI 自动回写生效 | 用户 + 各 AI 平台 | 是 | 否 |
| Muse / Dots 偏好同步 | 厂商未开放 | — | 否（预留） |

**工程结论**：「用户零配置零手动」不成立——首次授权与平台侧配置不可省略。准确表述为「一次性授权与配置 ⇒ 之后可全自动回写」。

## 7. 部署与校验（全程无需 PAT）

### 7.1 实际采用的路径：GCM 浏览器 OAuth（`git push`）

本档案最终通过 Git Credential Manager 的浏览器 OAuth 完成 `git push`，**无需创建 PAT**，并将本地 6 条提交与 7 个文件完整同步到远端 `main`。

**连接器实测边界（如实记录）**：官方 GitHub 连接器已连接，`get_me` 可读取账号，但 `POST /user/repos` 与 `PUT /repos/.../contents/{path}` 均返回 `403 Resource not accessible by integration`，分支保护接口亦未暴露。因此连接器未用于任何写操作。

**误判修正**：空仓库的 `409 Git Repository is empty` 匿名调用同样会返回，不能据此推断 App 授权已覆盖该仓库。实测写入 403 才确认权限不足，遂改走 GCM 路径。

**执行要点**：本机 `credential.helper=manager`（system 级），允许 GCM 交互后 `git -c http.version=HTTP/1.1 push -u origin main` 一次成功；若强制非交互（`GIT_TERMINAL_PROMPT=0`），会返回 `Empty reply from server`。每次 `git commit` 与 `git push` 均保留完整提交链。


### 7.2 从零复现命令

使用 Git Credential Manager 的 **Sign in with your browser**（本机 `credential.helper=manager` 已配置）或 GitHub Desktop 推送，同样无需创建 PAT：

```bash
cd personal-ai-archive
git init -b main
git add .
git -c user.name="Personal AI Archive" -c user.email="archive@users.noreply.github.com" \
    commit -m "pai: initial archive | by user | <ISO8601>"
git remote add origin "https://github.com/laoshen1987/personal-ai-archive.git"
git push -u origin main
```

### 7.3 推送后只读校验

```bash
curl -sS "https://api.github.com/repos/laoshen1987/personal-ai-archive/contents/" | grep '"name"'
# 保护详情端点需认证（匿名返回 401）；匿名请用公开分支端点查 protected 标志：
curl -sS "https://api.github.com/repos/laoshen1987/personal-ai-archive/branches/main" | grep '"protected"'
curl -sS -o /dev/null -w "%{http_code}\n" "https://raw.githubusercontent.com/laoshen1987/personal-ai-archive/main/memory/memory.md"
```

## 8. 自动回写日志（AI-APPEND / 仅追加）
<!-- PAI:APPEND:START -->
<!-- PAI:APPEND:END -->
