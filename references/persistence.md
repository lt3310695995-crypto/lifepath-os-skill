# LifePath V6.1 持久化接口

在需要读取记录、保存草稿、使用固定访谈或恢复进度时读取。以下对应本项目 V6.1；先检查当前工具列表，若运行版本不同，以实际接口为准。

## 接入与能力范围

优先使用已连接的 LifePath MCP。没有连接但可访问本项目时，可从项目根目录运行 CLI；本地网页服务已启动且确认属于本项目时，也可用 HTTP 接口。不要因为开始访谈而自动修改全局 MCP 配置或安装技能。

- Python 3.10+，项目根目录包含 `lifepath/` 和 `web/`。
- 默认数据库：`~/.lifepath-os/life.db`。启动 HTTP：`python -m lifepath.cli serve`，默认 `http://127.0.0.1:8765`。
- MCP 启动命令：`python -m lifepath.mcp_server`。这是项目的最小 stdio 实现，不能据此宣称任何客户端均已验证兼容。
- MCP 的七个工具：`list_events`、`create_event`、`confirm_event`、`start_session`、`answer_session`、`list_branches`、`add_branch`。
- CLI 对应命令：`events`、`add`、`confirm`、`start`、`answer`、`branches`、`branch`，另有 `modules`、`serve`。
- 当前 MCP 和 CLI 都没有恢复会话、修改事件或删除事件工具；这些能力可通过 HTTP 或网页使用。不得虚构 MCP 工具名。

## 字段与 HTTP 路径

POST 使用 JSON 对象；GET 不带写入请求体。下面的路径相对于服务根地址。不要通过请求传入 `db` 或另选数据库来规避既有记录。

- `GET /api/modules`：六个合法模块名。默认模块为“成长与教育”，调用时根据内容明确传入。
- `GET /api/events` / MCP `list_events`：返回包括候选在内的事件；读回时检查 `id`、`status` 及内容。返回顺序是创建顺序，不是人生时间顺序。
- `POST /api/events` / `create_event`：`title` 必填；可传 `description`、`time_label`、`module`。创建结果为 `{id, status: "candidate"}`。
- `POST /api/confirm` / `confirm_event`：`{id}`，返回 `{updated: true/false}`。
- `POST /api/update-event`：`id`、`title`，并发送完整的 `description`、`time_label`、`module`；遗漏的可选字段会重置为默认值。编辑不会自动将已确认事件改回候选。
- `POST /api/delete-event`：`{id}`，返回 `{deleted: true/false}`；按用户明确指定的对象删除。
- `POST /api/sessions` / `start_session`：`{module}`，返回会话 `id`、`module`、首个 `question`。
- `POST /api/answers` / `answer_session`：`{id, answer}`；返回 `done`、下一条 `question`，结束时可能返回 `candidate`。
- `GET /api/session?id=会话ID`：读取 `step`、`status`、`answers`、`question`、`candidates`，用于恢复和核实写入。查询参数需 URL 编码。
- `POST /api/branches` / `add_branch`：`title`、`premise`、`risk`、`uncertainty` 全部必填；返回 `{id}`。
- `GET /api/branches` / `list_branches`：读取已保存的未来假设。当前没有编辑或删除分支接口。

`title` 使用非空、最多 120 字符的标题，`description` 最多 5000 字符。虽然创建接口校验不足，仍按编辑接口的限制组织内容；不要静默截断用户的关键原话。事件表没有单独的证据、感受、推测或行动字段：原话依据和用户感受可在 `description` 中用文字标注，助手推测留在对话中。

未来分支没有 `opportunity`、`action`、`review_at` 等字段。用户要求保存完整假设时，可把相关经历 ID、机会写入 `premise`，把验证动作和观察标准写入 `uncertainty`，使用清楚的文字标签；不要添加接口不支持的参数。这只保存文本，不会安排提醒。

## 两种访谈方式不要混用

**代理自适应访谈（默认）**

在对话中根据用户内容追问，使用 `create_event` 保存用户要求记录的候选事件。不调用 `start_session` 或 `answer_session` 来模拟动态追问，也不声称该过程可通过后端会话 ID 恢复。

**程序固定三问（用户选择此方式或已有会话时）**

1. `start_session` 一次，保留返回的真实会话 ID。按接口返回的问题推进，不替换问题后把回答塞进不对应的步骤。
2. 每次回答最多 5000 字符；`answer_session` 会立即存储回答并推进一步。跳过时发送 `跳过`。仅暂停或要求总结时，不提交虚构答案，不调用不存在的暂停接口。
3. `done=true` 时程序最多生成一条候选事件，使用返回的 `candidate.id`，不能再调用 `create_event` 重复保存相同内容。
4. 自动整理只是拼接回答，并非 AI 提取；前 70 字符用作标题、拼接内容最多保留 3000 字符。应核对原始回答；首题跳过但后续有内容时，标题可能仍为“跳过”，需修正后再确认。
5. 只有 HTTP 或其他实际支持读取会话的工具可用于从后端恢复。仅有现行 MCP/CLI、又没有足够对话进度时，说明无法可靠恢复，建议在网页恢复；不要猜测步骤并写入答案。

## 写入结果与重试

- 保存前在授权范围内核对相关已有事件；同标题不等于同事件，时间、描述和来源也要比对。
- 确认前展示具体候选版本，收到明确认可后使用对应 ID。若先修改，则修改成功后才确认；不因“保存一下”就把尚未展示的助手改写自动确认。
- 成功调用后按 ID 读回状态和关键内容。`updated=false` 不等于成功，可能是已确认或 ID 不存在，先读回区分。
- 超时或响应丢失时，不盲目重试创建事件、创建分支、启动会话或提交回答。先用列表/会话读取核实；无法核实会话进度时报告状态未知，保留草稿，停止相关写入。
- 固定访谈自动草稿若需要拆分，先展示拆分结果；在具备编辑/删除能力并获准替换原草稿后处理。只具备 MCP 时不生成一组重复记录来掩盖无法处理原草稿的问题。

## 使用限制

网页时间线仅是示意图，曲线起伏由绘图算法产生，不代表情绪、成就或幸福程度。教育阶段和具体年份的混合排序不可靠，回顾时保留原始时间标签，不从节点位置推导先后或因果。

删除事件不会删除固定访谈中的原始回答；当前没有完整删除访谈、清除所有个人数据或备份恢复的产品接口。收到“彻底删除我的记录”时先明确范围和现有能力，不能只删事件后宣称全部清除。

数据库无应用层加密，HTTP 服务仅用于本机。不要为跨设备访问擅自改为公网监听。
