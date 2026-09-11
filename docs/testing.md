# 测试

OpenBot 已有能力的功能测试。每项测试包含示例输入和明确的预期结果。

## 前置条件

```sh
bash scripts/start.sh
```

或使用单容器镜像：

```sh
docker build -t openbot .
docker run -p 3001:3001 --env-file .env -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql openbot
```

所有测试假设 API 在 `http://localhost:3001`，应用在 `http://localhost:3010`。
如需指向其他地址，设置 `OPENBOT_API_URL`。

## 1. 健康检查与能力

| # | 操作 | 预期结果 |
|---|------|----------|
| 1.1 | `curl http://localhost:3001/health` | 返回 `200` |
| 1.2 | `curl http://localhost:3001/api/capabilities` | JSON 包含 `licenseStatus: "valid"` 且至少有一个 agent |
| 1.3 | `curl http://localhost:3010/` | HTML 包含 `<title>` 且标题含 OpenBot |

示例命令：

```sh
curl -s http://localhost:3001/health
# 预期输出: 200 (HTTP 状态码)

curl -s http://localhost:3001/api/copilotkit/info | head -c 200
# 预期输出: JSON，包含 "licenseStatus":"valid" 和 agents 列表

curl -s http://localhost:3010/ | grep -o '<title>[^<]*</title>'
# 预期输出: <title>...OpenBot...</title>
```

## 2. Bot 对话

| # | 操作 | 预期结果 |
|---|------|----------|
| 2.1 | 打开 `http://localhost:3010/bot`，发送消息 | Bot 流式返回文字回复 |
| 2.2 | 让 Bot 打开一个网站并返回内容 | 浏览器打开，Bot 返回页面内容 |
| 2.3 | 在 2.2 浏览器打开期间，通过实时屏幕面板接管控制 | 人工控制期间 Bot 的操作被拒绝 |
| 2.4 | 释放控制 | Bot 操作恢复 |

示例输入：

```
2.1 你好，你能做什么？请用三句话介绍自己。

2.2 打开 https://example.com 并告诉我页面上写了什么。

2.3 (点击实时屏幕面板上的"接管控制"按钮)

2.4 (点击"释放控制"按钮)
```

## 3. Coworker 与频道

前置条件：`.env` 中设置 `AGENT_COMPUTER_ALLOW_PRIVATE_HOSTS=true`（本地开发允许私有地址）。

| # | 操作 | 预期结果 |
|---|------|----------|
| 3.1 | 打开 `http://localhost:3010/agents`，创建一个 coworker | 列表中出现新 coworker |
| 3.2 | 与该 coworker 开启频道，发送消息 | 频道打开，coworker 回复 |
| 3.3 | 与同一 coworker 开启第二个频道 | 两个独立对话，线程互不影响 |
| 3.4 | 从 `/agents` 删除该 coworker | coworker 被移除；已有频道仍可读，显示为墓碑状态 |

示例输入：

```
3.1 创建 coworker:
    名称: 测试助手
    标题: 测试与验证
    角色描述: 协助测试系统功能，回答简单问题。
    Endpoint（用 start.sh 时）: http://localhost:4201/ag-ui
    Endpoint（用 Docker 单容器时）: http://host.docker.internal:4201/ag-ui
    （4201 是 agent-langgraph，4200 是 agent-bot；单容器镜像不含 Bot 进程，
     需在宿主机单独运行 agent-langgraph，Docker 内用 host.docker.internal 访问宿主机）

3.2 在频道中发送:
    你好，请告诉我你的角色是什么？

3.3 在第二个频道中发送:
    你好，这是一个新对话，你记得我们之前的对话吗？

3.4 (在 /agents 页面点击该 coworker 的删除按钮)
```

## 4. 浏览器治理与审计

| # | 操作 | 预期结果 |
|---|------|----------|
| 4.1 | 让 Bot 浏览网站，查看审计页面 | 每个浏览器操作都有审计行，`initiator_kind` 为 `person` |
| 4.2 | 添加 deny 规则后再次让 Bot 访问 | 操作被拒绝；审计行记录拒绝 |
| 4.3 | 删除 deny 规则后再次让 Bot 访问 | 同一操作恢复允许 |

示例输入：

```
4.1 Bot 输入:
    打开 https://news.ycombinator.com 并告诉我头条新闻的标题

    然后访问: http://localhost:3010/admin/audit
    预期: 审计页面显示 browser 导航操作，initiator_kind 为 "person"

4.2 在 http://localhost:3010/admin/boundaries 添加 deny 规则:
    规则: deny: page.host == "news.ycombinator.com"

    然后再次 Bot 输入:
    打开 https://news.ycombinator.com 并告诉我头条新闻

    预期: Bot 报告操作被拒绝，审计页面出现拒绝记录

4.3 删除上述 deny 规则后，再次发送相同指令
    预期: 操作恢复正常
```

## 5. Bot 间协作

前置条件：`BOT_HANDOFF_MAX_DEPTH` 至少为 `1`（默认值），且在 agent 页面 **Bots it may ask** 中已授权 Bot A 可向 Bot B 发起协作。

| # | 操作 | 预期结果 |
|---|------|----------|
| 5.1 | 在 Bot A 的频道中，让它咨询 Bot B | Bot B 在自己的频道中回答；Bot A 的频道显示工作去向 |
| 5.2 | 查看审计记录 | Bot B 的操作审计行显示 `initiator_kind: handoff` |

示例输入：

```
5.1 在 Bot A（如 "Risk Analyst"）的频道中发送:
    请咨询 Knowledge Bot，问它：我们的合规政策文档存放在哪里？

    预期: Bot A 的频道显示"已将工作交给 Knowledge"
          Knowledge Bot 的频道出现新消息，回答该问题

5.2 访问 http://localhost:3010/admin/audit
    预期: Bot B 的审计行 initiator_kind 为 "handoff"
```

## 6. 定时任务（Routines）

前置条件：`WORKER_SHARED_SECRET` 已设置，worker 正在运行，且 Bot 已被授予 `create_routine` 权限。

| # | 操作 | 预期结果 |
|---|------|----------|
| 6.1 | 在频道中让 Bot 创建定时任务 | Bot 创建任务；Routines 页面显示下次运行时间 |
| 6.2 | 等待到下次计划运行时间 | Bot 的回复自动出现在频道中 |
| 6.3 | 让 Bot 删除定时任务 | 任务被移除，不再触发 |
| 6.4 | 让 Bot 列出定时任务 | Bot 列出剩余任务（如有） |

示例输入：

```
6.1 在频道中发送:
    请每 15 分钟在这个频道发布一次当前时间，用中文回复。

    预期: Bot 确认创建，Routines 页面显示下次运行时间

6.2 等待 15 分钟
    预期: 频道自动出现 Bot 发布的时间消息

6.3 发送:
    请删除我刚才创建的"每 15 分钟发布时间"的定时任务

    预期: Bot 确认删除

6.4 发送:
    请列出我当前所有的定时任务

    预期: Bot 回复空列表或剩余任务
```

## 7. 技能（Skills）

| # | 操作 | 预期结果 |
|---|------|----------|
| 7.1 | 在拥有 `skill-creator` 技能的 Bot 频道中，让它帮你编写技能 | Bot 进行访谈，然后展示包含命令、标题和说明的卡片 |
| 7.2 | 点击卡片上的保存按钮 | 技能出现在 Skills 页面 |
| 7.3 | 在 Skills 页面将该技能授予某个 Bot | 该 Bot 的授权列表中显示此技能 |
| 7.4 | 在频道中发送匹配该技能用途的消息 | Bot 仅被提供该技能相关的工具（当工具总数超过 12 个时） |

示例输入：

```
7.1 在频道中发送:
    帮我创建一个技能，叫做"翻译助手"。它的作用是：用户发来一段文字，
    把它翻译成英文。步骤是：先识别文字语言，再翻译，最后返回结果。

    预期: Bot 开始访谈，询问细节，最终展示保存卡片

7.2 (点击卡片上的"保存"按钮)

7.3 访问 http://localhost:3010/skills
    找到"翻译助手"技能，点击"授予"，选择一个 Bot

7.4 在该 Bot 的频道中发送:
    请把这句话翻译成英文：今天天气真好

    预期: Bot 执行翻译并返回英文结果
```

## 8. 生成式界面（Generative UI）

前置条件：`.env` 中设置 `OPENBOT_GENERATIVE_UI=true`。

| # | 操作 | 预期结果 |
|---|------|----------|
| 8.1 | 让 Bot 构建一个简单界面 | Bot 生成并流式渲染一个界面，显示在对话中 |
| 8.2 | 与生成的界面交互 | 界面响应交互（如计数器递增） |

示例输入：

```
8.1 在频道中发送:
    请生成一个交互界面：一个计数器按钮，点击后数字加 1，
    再加一个重置按钮，点击后归零。

    预期: 对话中出现一个可交互的界面，包含计数器按钮和重置按钮

8.2 操作:
    点击计数器按钮 3 次
    预期: 数字显示 3

    点击重置按钮
    预期: 数字显示 0
```

## 9. MCP 插件

前置条件：管理员已在 `http://localhost:3010/admin/plugins` 启用 Google Drive 或 Notion，且用户已连接自己的账号。

| # | 操作 | 预期结果 |
|---|------|----------|
| 9.1 | 让 Bot 搜索文件 | Bot 调用搜索工具，返回用户 Drive 中的结果 |
| 9.2 | 让 Bot 读取文件内容 | Bot 读取并返回文件内容，附带来源引用 |
| 9.3 | 从 Plugins 页面断开 Google 账号 | 再次让 Bot 搜索时返回"未连接" |

示例输入：

```
9.1 在频道中发送:
    在我的 Google Drive 中搜索包含"季度报告"的文件

    预期: Bot 调用 google-drive/search_files 工具，返回匹配的文件列表

9.2 发送:
    请读取第一个搜索结果的文件内容

    预期: Bot 调用 google-drive/read_file_content 工具，返回文件内容并注明来源

9.3 访问 http://localhost:3010/admin/plugins，断开 Google 账号连接

    然后再次发送:
    在我的 Google Drive 中搜索包含"季度报告"的文件

    预期: Bot 返回"Google Drive 未连接"或类似提示
```

## 10. 停止与清理

```sh
bash scripts/stop.sh
```

或对于容器：

```sh
docker stop $(docker ps -q --filter ancestor=openbot)
```

## 自动化测试

```sh
bun run test           # 单元和集成测试
bun run test:smoke     # 针对运行中的实例进行 HTTP 全流程测试
bun run test:ci        # CI 套件，验证预期测试数量
```

`test:smoke` 需要运行中的实例。`test` 不需要，但集成测试需要带 pgvector 的 PostgreSQL。
