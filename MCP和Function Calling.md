
## 一、Function Calling 详解
① 你把工具定义（JSON Schema）随请求一起发给模型
   {
     "name": "get_transaction_history",
     "description": "查询客户最近的交易记录",
     "parameters": {
       "account_no": {"type": "string", "description": "账号"},
       "limit": {"type": "integer", "default": 5}
     }
   }
② 模型返回的不是自然语言，而是结构化输出
   {"tool_calls": [{"name": "get_transaction_history",
                    "arguments": {"account_no": "...", "limit": 5}}]}
③ ★ 你的代码去执行这个调用 ★ ← 这一步和模型无关
   你的后端去查数据库 / 调内部 API
④ 你把结果原样塞回消息列表，再发给模型
   {"role": "tool", "content": "最近5笔交易：..."}
⑤ 模型基于真实结果生成最终回答

关键在 ③：模型只是"说出了意图"，执行权始终在你的代码手里。这正是它安全的原因——模型碰不到你的数据库，它只能请求，你的代码决定给不给、给多少。

### 完整循环（多轮工具调用）

真实场景往往要调好几次

## 二、MCP 详解

它要解决什么问题

N × M 问题：假设你有 M 个 AI 应用（Claude Code、Cursor、自研 Agent），N 个外部系统（数据库、工单、IM、知识库）。

没有标准的做法：每个应用都要为每个系统写一遍适配 → N × M 个集成。

MCP 的做法：每个系统做成一个 MCP 服务器，每个应用实现一个 MCP 客户端 → N + M 个集成。

AWS 的表述很准确：MCP 的设计理念是充当"通用翻译器"，并使"一次集成、多处复用"成为可能

### 架构：三个角色

MCP 遵循客户端-服务器架构：

|角色|是什么|例子|
|---|---|---|
|MCP 主机|协调和管理一个或多个 MCP 客户端的 AI 应用|Claude Desktop、VS Code|
|MCP 客户端|维护与某个 MCP 服务器连接的组件|主机为每个服务器创建一个客户端|
|MCP 服务器|提供上下文/工具的程序|文件系统服务器、Sentry 服务器|

注意一个细节：一个主机连多个服务器时，会为每个服务器实例化一个独立的客户端

### MCP 服务器能提供什么

五类能力：

|能力|作用|
|---|---|
|Resources（资源）|提供数据（文档、记录）|
|Prompts（提示模板）|预置的提示词模板|
|Tools（工具函数）|可被调用的操作 ★ 这是和 Function Calling 的接口|
|Elicitation（用户输入请求）|需要时向用户要信息|
|Sampling（LLM 内容生成）|反向请求主机生成内容|

注意 Tools 这一项：MCP 服务器暴露的 Tool，最终就是要变成模型能调用的函数定义——所以 MCP 和 Function Calling 在这里接头。

![[Pasted image 20261012105758.png]]

![[Pasted image 20261012111232.png]]

![[Pasted image 20261012111911.png]]