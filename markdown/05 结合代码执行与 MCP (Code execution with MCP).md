# 结合代码执行与 MCP：构建更高效的智能体

Code execution with MCP: Building more efficient agents

>   **发布日期：**2025 年 11 月 4 日 **团队：**Anthropic Engineering **作者：**Adam Jones & Conor Kelly **预计阅读：**约 12 分钟  [查看官方英文原文 ↗](https://www.anthropic.com/research/code-execution-with-mcp)

 模型上下文协议（MCP）是连接 AI 智能体与外部系统的开放标准。过去，每把一个智能体接入一个新的工具或数据源，都得为这对组合单独写一套定制适配，造成生态割裂与大量重复劳动，难以规模化构建真正互联的系统。MCP 提供了统一协议——开发者只需在智能体中实现一次 MCP，即可一键解锁整个集成生态。

 自 2024 年 11 月发布以来，MCP 普及速度极快：社区已构建数千个 MCP 服务器，所有主流编程语言均有对应 SDK，行业已将 MCP 奉为连接智能体与工具数据的事实标准。

 如今，开发者动辄让智能体横跨数十个 MCP 服务器、接入成百上千个工具。但随着所连接工具的数量激增，**启动时就把全部工具定义一股脑塞进上下文，加上所有中间结果都要反复穿过上下文窗口**，正显著拖慢智能体响应速度并推高费用。

 本文将探讨如何借助**代码执行（code execution）**让智能体更高效地与 MCP 服务器交互——用更少的 Token 驾驭更多工具。

## 一、工具带来的过量 Token 消耗正拖垮智能体效率

随着 MCP 使用规模扩大，有两种常见模式会显著拉高成本与延迟：

1. 工具定义过载，挤爆上下文窗口；
1. 中间工具返回结果消耗大量额外 Token。


### 1. 工具定义挤爆上下文窗口

 绝大多数 MCP 客户端在启动时会把**所有工具定义**直接预先塞入上下文，通过直接工具调用（direct tool-calling）语法暴露给模型。这些工具定义通常长这样：

```text
gdrive.getDocument
     Description: Retrieves a document from Google Drive
     Parameters:
                documentId (required, string): The ID of the document to retrieve
                fields (optional, string): Specific fields to return
     Returns: Document object with title, body content, metadata, permissions, etc.
```



```text
salesforce.updateRecord
    Description: Updates a record in Salesforce
    Parameters:
               objectType (required, string): Type of Salesforce object (Lead, Contact, Account, etc.)
               recordId (required, string): The ID of the record to update
               data (required, object): Fields to update with their new values
     Returns: Updated record object with confirmation
```



 大量工具描述会挤占上下文空间，拉长响应时间并推高账单。如果智能体连接了上千个工具，**甚至在还没读到用户请求之前，模型就得先处理数十万 Token**。

### 2. 中间工具结果消耗海量额外 Token

 绝大多数 MCP 客户端允许模型直接调用 MCP 工具。假设你吩咐智能体：“把我在 Google Drive 里的会议纪要下载下来，附加到 Salesforce 的销售线索记录里。”模型会发起如下调用：

```text
TOOL CALL: gdrive.getDocument(documentId: "abc123")
        → returns "Discussed Q4 goals...\n[full transcript text]"
           (loaded into model context / 全量载入模型上下文)

TOOL CALL: salesforce.updateRecord(
            objectType: "SalesMeeting",
            recordId: "00Q5f000001abcXYZ",
            data: { "Notes": "Discussed Q4 goals...\n[full transcript text written out]" }
        )
        (model needs to write entire transcript into context again / 模型需将整篇逐字稿重新写进上下文)
```



 每一个中间结果都必须先流经模型本身。在这个例子里，整篇通话逐字稿流过了两次——对于一场 2 小时的销售会议，仅这一步就意味着要额外处理 **50,000 Tokens**；更大的文档甚至会直接超出上下文窗口上限，导致整个工作流崩溃。

 而且面对大型文档或复杂数据结构时，模型在工具调用之间抄写搬运数据，也更容易出错。

> **【图表说明】** 官方原图（中文化）：传统 MCP 客户端将全部工具定义载入上下文窗口，并在消息循环中让每次工具调用与返回结果都完整流经模型。



## 二、结合代码执行与 MCP：大幅提升上下文效率

 如今为智能体配备代码执行环境已越来越普遍，这带来了一个绝佳解法：**把 MCP 服务器呈现为“代码 API”，而不是直接的工具调用列表。**智能体通过编写代码来与 MCP 服务器交互，一举解决上述两大难题——它只需按需加载当前任务用得上的工具，并在代码执行环境内部直接完成数据处理，只把精简结果传回模型。

 实现方式有多种，其中一种是从已连接的 MCP 服务器中生成所有可用工具的代码文件树。以下是 TypeScript 实现示意：

```directory
servers
├── google-drive
│   ├── getDocument.ts
│   ├── ... (other tools)
│   └── index.ts
├── salesforce
│   ├── updateRecord.ts
│   ├── ... (other tools)
│   └── index.ts
└── ... (other servers)
```



每个工具对应一个清晰的类型定义文件，例如：

```typescript
// ./servers/google-drive/getDocument.ts
import { callMCPTool } from "../../../client.js";

interface GetDocumentInput {
  documentId: string;
}

interface GetDocumentResponse {
  content: string;
}

/* Read a document from Google Drive */
export async function getDocument(input: GetDocumentInput): Promise<GetDocumentResponse> {
  return callMCPTool<GetDocumentResponse>('google_drive__get_document', input);
}
```



于是，前面那段“从 Google Drive 读纪要并写入 Salesforce”的搬运任务，直接变成了一段极简代码：

```typescript
// Read transcript from Google Docs and add to Salesforce prospect
import * as gdrive from './servers/google-drive';
import * as salesforce from './servers/salesforce';

const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  recordId: '00Q5f000001abcXYZ',
  data: { Notes: transcript }
});
```



 智能体通过探索文件系统按需发现工具：先列出 `./servers/` 目录找到可用服务（如 `google-drive` 与 `salesforce`），再仅读取当前任务所需的具体工具文件（如 `getDocument.ts` 与 `updateRecord.ts`）以了解接口签名。这使得智能体只加载当前任务必需的工具定义，**直接将 Token 消耗从 150,000 Tokens 骤降至 2,000 Tokens——时间和费用节省高达 98.7%。**

 Cloudflare 也发布过类似发现，将结合代码执行调用 MCP 称为 “Code Mode”。核心洞察殊途同归：**大语言模型极度擅长写代码，开发者应当善用这一长处，构建能更高效驾驭 MCP 服务器的智能体。**

## 三、代码执行 + MCP 带来的五大工程红利

 结合代码执行与 MCP，能够通过按需加载工具、在数据到达模型前先行过滤、以及在单一步骤内执行复杂逻辑，让智能体更高效地利用上下文；同时，这一模式还在安全隐私与状态管理方面带来显著收益。

### 1. 渐进式披露（Progressive disclosure）

 模型天生擅长浏览文件系统。把工具以代码文件形式组织在目录树里，让模型得以**按需读取工具定义**，而非启动时一口气全吞。

 另一种方式是在服务端添加一个 `search_tools` 工具，用于检索相关定义。例如，在使用上文假设的 Salesforce 服务器时，智能体可以搜索 `"salesforce"` 并只加载当前任务所需的工具。在 `search_tools` 中加入一个“详细程度”参数（允许智能体按需选择仅返回名称、名称与描述，或带完整 Schema 的全部定义），同样有助于节省上下文并高效定位工具。

### 2. 高上下文效率的工具结果过滤（Context efficient tool results）

 处理大型数据集时，智能体可以先在代码里完成筛选与转换，再输出精简结果。想象拉取一张 10,000 行的电子表格：

```typescript
// Without code execution - all rows flow through context
TOOL CALL: gdrive.getSheet(sheetId: 'abc123')
        → returns 10,000 rows in context to filter manually

// With code execution - filter in the execution environment
const allRows = await gdrive.getSheet({ sheetId: 'abc123' });
const pendingOrders = allRows.filter(row =>
  row["Status"] === 'pending'
);
console.log(`Found ${pendingOrders.length} pending orders`);
console.log(pendingOrders.slice(0, 5)); // Only log first 5 for review
```



 模型最终看到的只有 5 行，而不是 10,000 行。聚合计算、多数据源关联（join）、特定字段提取全部适用这一模式，彻底告别上下文膨胀。

### 3. 更强大且省上下文的控制流（More powerful and context-efficient control flow）

 循环、条件分支与错误处理都可以直接用常规代码实现，不必笨拙地在智能体主循环里一次次串联单个工具调用。例如，如果你需要在 Slack 中等待一条部署完成通知，智能体可以写出如下代码：

```typescript
let found = false;
while (!found) {
  const messages = await slack.getChannelHistory({ channel: 'C123456' });
  found = messages.some(m => m.text.includes('deployment complete'));
  if (!found) await new Promise(r => setTimeout(r, 5000));
}
console.log('Deployment notification received');
```



 这比在智能体主循环里反复交替执行 MCP 工具调用和 `sleep` 命令高效得多。

 此外，直接写出可供执行的条件判断树还能节省“首字延迟（time to first token，TTFT）”：智能体无需苦等模型一步步推理 `if` 分支，而是把条件求值直接交给代码执行环境完成。

### 4. 隐私保护操作（Privacy-preserving operations）

 采用代码执行调用 MCP 时，中间数据默认停留在执行环境中，智能体只能看到你显式打印或返回的内容——意味着你不想暴露给模型的数据可以直接在工作流内流转，**自始至终不进入模型上下文**。

 对于更高敏感度的业务，执行框架（agent harness）甚至可以自动对敏感数据做**脱敏令牌化（tokenization）**。例如把客户联系方式从电子表格批量导入 Salesforce，智能体写出如下代码：

```typescript
const sheet = await gdrive.getSheet({ sheetId: 'abc123' });
for (const row of sheet.rows) {
  await salesforce.updateRecord({
    objectType: 'Lead',
    recordId: row.salesforceId,
    data: {
      Email: row.email,
      Phone: row.phone,
      Name: row.name
    }
  });
}
console.log(`Updated ${sheet.rows.length} leads`);
```



MCP 客户端在数据到达模型前拦截并将个人身份敏感信息（PII）替换为占位令牌：

```typescript
// What the agent would see, if it logged the sheet.rows:
[
  { salesforceId: '00Q...', email: '[EMAIL_1]', phone: '[PHONE_1]', name: '[NAME_1]' },
  { salesforceId: '00Q...', email: '[EMAIL_2]', phone: '[PHONE_2]', name: '[NAME_2]' },
  ...
]
```



 随后，当数据传入下一个 MCP 工具调用时，MCP 客户端会通过内部映射表将其还原为真实值。真实的邮箱、手机号和姓名直接从 Google Sheets 流向 Salesforce，**从未穿过模型本身**，防止智能体意外打印日志或误处理敏感信息；你还可以借此定义确定性的安全流转规则，严格限定数据允许从哪里流向哪里。

### 5. 状态持久化与 Skills 沉淀（State persistence and skills）

 具备文件系统访问权限的代码执行环境，让智能体能够跨操作持久化维护状态。智能体可以将中间结果写入文件，从而实现断点续跑并追踪进度：

```typescript
const leads = await salesforce.query({
  query: 'SELECT Id, Email FROM Lead LIMIT 1000'
});
const csvData = leads.map(l => `${l.Id},${l.Email}`).join('\n');
await fs.writeFile('./workspace/leads.csv', csvData);

// Later execution picks up where it left off
const saved = await fs.readFile('./workspace/leads.csv', 'utf-8');
```



 智能体还可以把自己的代码持久化封装为可复用函数。一旦针对某项任务跑通了有效代码，就可以将这份实现保存下来供未来复用：

```typescript
// In ./skills/save-sheet-as-csv.ts
import * as gdrive from './servers/google-drive';
export async function saveSheetAsCsv(sheetId: string) {
  const data = await gdrive.getSheet({ sheetId });
  const csv = data.map(row => row.join(',')).join('\n');
  await fs.writeFile(`./workspace/sheet-${sheetId}.csv`, csv);
  return `./workspace/sheet-${sheetId}.csv`;
}

// Later, in any agent execution:
import { saveSheetAsCsv } from './skills/save-sheet-as-csv';
const csvPath = await saveSheetAsCsv('abc123');
```



 这与 **Skills（技能）**的理念紧密呼应——即由可复用的指令、脚本和资源组成的文件夹，帮助模型在特定专业任务上提升表现。为这些保存下来的函数配上一份 `SKILL.md` 文件，就能构建出可供模型引用与使用的结构化技能。随着时间推移，你的智能体得以亲手积累起一座高阶能力工具箱，持续演化出最高效工作所需的工程脚手架。

> **工程权衡提醒**


引入代码执行也伴随着自身的复杂度：运行智能体生成的代码需要配备具备完善**沙箱隔离、资源配额限制与行为监控**的安全执行环境。相比直接工具调用，这会增加基础设施运维与安全防护开销，你在架构选型时应把 Token 成本降低、延迟下降与工具组合优势，同这些实现与运维成本放在同一张天平上权衡。

## 四、总结（Summary）

 MCP 为智能体接入海量工具与系统奠定了基础协议；然而，一旦挂载的服务器过多，工具定义与中间结果极易消耗过量 Token，拖累智能体的运行效率。

 尽管上下文管理、工具组合与状态持久化看似都是 AI 时代的新难题，但它们在传统软件工程中早有成熟解法。**结合代码执行，正是把久经考验的软件工程范式引入智能体领域，让模型用熟悉的编程结构更高效地驾驭 MCP 服务器。**如果你在实践中落地了这一方案，我们非常鼓励你将实践心得分享给 [MCP 社区](https://modelcontextprotocol.io/community/communication)。

> **致谢（Authors & Acknowledgements）**


本文由 Adam Jones 和 Conor Kelly 撰写；感谢 Jeremy Fox、Jerome Swannack、Stuart Ritchie、Molly Vorwerck、Matt Samuels 和 Maggie Vo 对初稿提出的宝贵意见。

---

## 💡 译者附注（非原文）：如何把本篇用到 AI 客服与全自动运营 Agent



- **全自动运营批量对账必须走“代码执行 + 业务 MCP”**：比如每天核对 5,000 笔支付流水与 ERP 订单状态、筛选超时未发货包裹，让 GPT-6.2 写一段 TypeScript 在沙箱内跑过滤和分组统计，只把异常 Top 10 和汇总指标打印回上下文，避免账单暴涨和模型漏算。
- **用 MCP 客户端 PII 脱敏守住客服隐私红线**：当客服 Agent 需要把客户手机号、收货地址从订单 MCP 同步到物流补发 MCP 时，网关层自动把敏感字段替换成 `[PHONE_1]`、`[ADDR_1]` 占位符再传给模型，真正调用物流接口时再由客户端还原，彻底杜绝客户 PII 进入模型日志。
- **把高频运营查询脚本沉淀成团队可复用 Skills**：借助 Codex 把跑通的大促复盘取数脚本、异常退款排查脚本固化到 `./skills/`，以后全自动运营 Agent 直接 `import` 调用，兼具确定性与极低延迟。
- **明确沙箱安全边界**：仅在隔离容器/微虚拟机里执行模型生成代码，限制出网白名单仅允许访问内部业务 MCP 网关，严禁直连公网或生产数据库主库。
