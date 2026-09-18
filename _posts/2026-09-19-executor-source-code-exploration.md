---
title: "Source code exploration - Executor"
categories:
  - Code
---

最近看到一個開源專案（今年度被YC錄取），叫 [https://executor.sh/s-organization-qet2](Executor)。該專案的核心想法是：整合不同的 Agent (Claude Code, Codex) 到統一的 MCP server，支援不同的 API (openAPI, graphQL, MCP)，藉此就不用每個 Agent 重複設定一份憑證。
整體想法蠻酷的，所以我把整個專案fork下來，當作練手。
整個專案大約有20萬行代碼，並且極有可能大量由agent產生的，因此我不打算逐行去閱讀（那可能我會先瘋掉），所以我會嘗試藉由 Claude code 來理解整個專案。

整個專案核心代碼都放在 `package/`。

## Agent 連上 Executor：`createExecutorMcpServer`

Agent（Claude Code、Codex...）要用 executor 的第一步，是連上一個 MCP server。這個 server 是用 `createExecutorMcpServer` 來組裝一台 server，內部透過 `registerTool` 註冊了execute, skill, resume（核心三個,另外還有依設定開關的 artifact 和 search 相關工具）。建立 session 後，Agent 會透過 MCP 所定義的 `tools/list`、`tools/call` 來呼叫這些工具。整體架構是 code mode，當 agent 透過 skills 定義好的標準格式，例如：`tools.github.repos.delete(...)`，會將這些 code 丟到 `execute` function 裡面。

PS: code mode

The pattern where an LLM writes TypeScript/JavaScript that calls into a pre-registered set of tools, executed in a sandbox

實作參考：

- `createExecutorMcpServer` 函式本體（到 `new McpServer(...)` 為止）：[packages/hosts/mcp/src/tool-server.ts:1111-1230](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/hosts/mcp/src/tool-server.ts#L1111-L1230)
- `ExecutorMcpServerConfig` 型別定義（config 二選一：`ExecutionEngineConfig` 或現成的 `engine`）：[packages/hosts/mcp/src/tool-server.ts:275-279](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/hosts/mcp/src/tool-server.ts#L275-L279)
- `registerTool("execute", ...)`：[packages/hosts/mcp/src/tool-server.ts:1551-1563](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/hosts/mcp/src/tool-server.ts#L1551-L1563)
- `registerTool("skills", ...)`：[packages/hosts/mcp/src/tool-server.ts:1566-1591](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/hosts/mcp/src/tool-server.ts#L1566-L1591)
- `registerTool("resume", ...)`（model 模式 / browser 模式兩種版本）：[packages/hosts/mcp/src/tool-server.ts:1599-1644](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/hosts/mcp/src/tool-server.ts#L1599-L1644)
- `skills` 文件內容本體（`execute` / `create-artifact` / `artifact-style` 三篇 markdown）：[packages/core/execution/src/skills.ts:16-657](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/skills.ts#L16-L657)
- `execute` 標準呼叫格式 `tools.<integration>.<owner>.<connection>.<tool>(args)` 的規則定義：[packages/core/execution/src/skills.ts:57](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/skills.ts#L57)
- session 真正建立的地方（`server.connect(transport)`，stdio 範例）：[apps/local/src/mcp.ts:310-313](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/apps/local/src/mcp.ts#L310-L313)
- "code mode" 一詞的專案自身定義：[packages/kernel/core/README.md:3](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/kernel/core/README.md#L3)

延續前面，把 code 丟進沙盒之前，先製造一個 invoker（透過 `makeFullInvoker`）
```typescript
const invoker = makeFullInvoker(
  executor,
  { onElicitation: elicitationHandler },
  toolDiscoveryProvider,
);
```
然後把 code、invoker 丟進去沙盒
```typescript
fiber = yield* Effect.forkDetach(
  codeExecutor.execute(code, invoker) 
);
```

而在沙盒執行的code會透過 JS Proxy 來 call invoke，這個 inovke 方法實際主要吃兩個參數：path, args，初步會先檢查是不是search、executor.integrations.list、describe.tool，這些都是預先定義內部，如果都不是,就轉給內層 `makeExecutorToolInvoker`，再進入 `executor.execute` 的 6 道安檢流程，通過後才會真的打外部 API。

實作參考：

- `makeFullInvoker` 定義（外層包裝：攔截 `search`/`executor.integrations.list`/`describe.tool`，其餘轉給內層）：[packages/core/execution/src/engine.ts:317-321](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L317-L321)
- `makeFullInvoker` 呼叫 + `forkDetach` 進沙盒（就是上面兩段 code 的原文）：[packages/core/execution/src/engine.ts:697-704](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L697-L704)
- 沙盒內把 `tools.xxx.yyy(...)` 轉成 `invoke({ path, args })` 的 JS Proxy（以 quickjs runtime 為例，其他 runtime 各自有一份）：[packages/kernel/runtime-quickjs/src/index.ts:230](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/kernel/runtime-quickjs/src/index.ts#L230)
- `invoke({ path, args })` 本體、三個分支的起點：[packages/core/execution/src/engine.ts:324](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L324)
  - `path === "search"` 分支：[engine.ts:325-377](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L325-L377)
  - `path === "executor.integrations.list"` 分支：[engine.ts:378](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L378)（起點）
  - `path === "describe.tool"` 分支：[engine.ts:425](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L425)（起點）
- 三個分支都沒中，轉給內層 `base.invoke({ path, args })`（即 `makeExecutorToolInvoker`，真正跨出沙盒、呼叫 `executor.execute` 的地方）：[engine.ts:460](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/engine.ts#L460) / 定義於 [tool-invoker.ts:307](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/execution/src/tool-invoker.ts#L307)
- `executor.execute` 的 6 道安檢流程入口：[packages/core/sdk/src/executor.ts:6231-6291](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L6231-L6291)

這相關流程放在 `packages/core/sdk/src/executor.ts`，在真正打出api之前，會先做去fetch這支工具的相關資訊，例如使用者設定的 policyRules，並確認該操作是否有需要觸發使用者確認 (enforce approval)，確認後才去做 crendential resolution，把對應的 token 撈出來。

其他像是有做 Single-flight refresh gate，來確保同一時間只能有一個程序去 refresh token（大概是透過 Effect 的 `Deferred` 去做）來避免 race condition。

實作參考：

- 三路並行預讀（toolRow / policyRules / connectionRow 用 `Effect.forkChild` 平行發出）：[engine.ts 概念對應，實際在 executor.ts 6298-6391](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L6298-L6391)
- `approvalRequired` / `enforceApproval` 定義：[executor.ts:6152-6186](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L6152-L6186)
- `enforceApproval` 實際被呼叫、擋住後續流程的地方：[executor.ts:6452](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L6452)
- credential resolution（`enforceApproval` 過關後才啟動的 `valuesFiber`）：[executor.ts:6477-6486](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L6477-L6486)
- Single-Flight Refresh Gate 型別 + 建立：[executor.ts:256-269](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L256-L269)
- Gate 的 key 產生方式（`connectionKey`）：[executor.ts:2204-2205](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L2204-L2205)
- 核心邏輯 `refreshConnectionToken`（查 Map → 排隊等 / 發起 + `forkDetach`）：[executor.ts:2897-2943](https://github.com/cjumeow/executor/blob/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src/executor.ts#L2897-L2943)
- Policies 設定頁面（截圖裡那個 UI，讓使用者自己新增 policy）：搜尋 `packages/react/src` 底下 policy 相關 route/component（尚未逐行核對，之後可補精確行號）

最後，呼叫 `invokeWith(values)` 把 credential resolution + integration 設定塞進`credential` 連同 `args` 參數一起交給 `invokeTool(...)`。再把 api 轉交給對應 plugin 的 `backing.ts`，以 openAPI 為例，這裡描述了該工具對應哪個http method、path。

實作參考：

- `invokeWith` / `invokeTool(...)` 交接點：[packages/core/sdk/src/](https://github.com/cjumeow/executor/tree/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/core/sdk/src)
- OpenAPI plugin 的 `invokeTool` 轉接層：[packages/plugins/openapi/src/sdk/](https://github.com/cjumeow/executor/tree/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/plugins/openapi/src/sdk)（`plugin.ts` 薄轉接、`backing.ts` 查 operation binding + 渲染 credential、`invoke.ts` 組 request + 真正 `client.execute`）
- 其他協定的對照組（同一個 `invokeTool` 介面，換掉協定細節）：[packages/plugins/mcp/src/sdk/](https://github.com/cjumeow/executor/tree/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/plugins/mcp/src/sdk)、[packages/plugins/graphql/src/sdk/](https://github.com/cjumeow/executor/tree/eaa1f3a57ffff88aede8e83783ea7ed4471aec1f/packages/plugins/graphql/src/sdk)
