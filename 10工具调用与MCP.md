# 工具调用与MCP（Tool Calling / Model Context Protocol）

| 项目 | 填写内容 |
|:---|:---|
| 词条序号 | 10 |
| 姓名或课程署名 | 王康宇 |
| 学号后四位 | 1044 |
| 更新时间 | 2026-11-23 |
| 个人贡献 | 资料阅读、结果核验和文字修订 |
| AI协助 | Codex 于 2026-11-19 协助整理资料、编写工具并代跑对照；2026-11-23 核对来源、整理正文与原始记录。本人核查情况见第九节。 |

## 一、定义与概念边界

工具调用（Tool Calling）是模型按照工具说明提出结构化请求，例如选择 lookup_event 并给出展览编号；宿主应用负责执行，模型再根据返回结果回答。模型生成一个函数名，并不意味着函数已经执行。[MCP 工具规范，2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)区分了工具说明、调用参数与执行结果。

模型上下文协议（Model Context Protocol，MCP）规定 AI 应用与工具或数据服务之间如何通信，使应用能够以统一方式获取能力。本词条讨论其中的工具功能。API 是程序间调用功能的接口；MCP 是面向 AI 应用集成的协议。MCP 工具可以封装业务 API，也可以直接读取本地文件。本例采用后者，查询过程中不访问博物馆网站。[架构规范](https://modelcontextprotocol.io/specification/2025-11-25/architecture)还区分宿主、客户端与服务器，不能把 MCP 当作模型或智能体的名称。

## 二、背景与工作原理

如果每个应用都用自己的方式描述和调用工具，同一个查询能力接入不同应用时，往往需要分别适配。MCP 统一发现和调用的通信格式；具体业务逻辑仍由工具实现。

本例的宿主是 Codex CLI，内置 MCP 客户端连接 Python 服务器。按照[MCP 工具规范](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)，客户端可用 tools/list 获取工具名称、说明和参数结构，再用 tools/call 请求执行。一次查询可以这样理解：

1. 使用者问“E01 展览的起止日期和地点是什么”。
2. 模型根据工具说明生成 lookup_event 和参数 {"event_id":"E01"}。
3. 宿主通过 MCP 客户端发送请求，服务器在本地 JSON 表中按编号查询。
4. 服务器返回数据，宿主把结果交回模型，模型组织答案并附原文链接。

本例使用 stdio，由客户端启动本地服务器进程，通过标准输入和标准输出交换协议消息；它不要求建立公开网址。协议消息与诊断日志应分开输出，避免日志破坏通信格式。上述机制见[传输规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)。这里引用的 2025-11-25 规范用于解释概念；原始运行记录没有保存协商出的协议版本，不能据此断言实际协商版本。

## 三、应用案例

本案例是“大学博物馆展览信息查询”的教学模拟场景，材料来自真实公开公告，程序和模型调用已实际运行，由 Codex 协助代跑。

| 要素 | 内容 |
|:---|:---|
| 使用者与问题 | 校园文化编辑需要核对展览时间、地点，以及公告是否提供报名信息。 |
| 材料与权限 | 清华大学艺术博物馆官网三份公开展讯，提取必要事实与原文链接，不复制展览图片。 |
| 输入与输出 | 输入展览编号和问题；输出相应字段与来源，资料缺失时明确说明。 |
| 模型与工具 | Codex CLI 0.169.0-alpha.11.1 的默认模型设置；Python 3.12.14、官方 MCP Python SDK 1.26.0、本地 lookup_event 工具。日志未记录具体模型标识。 |
| 流程与分工 | Codex 协助整理快照、编写与运行程序；模型提出查询并组织回答；署名者应对照官网和原始记录核查、修订。 |
| 核验标准 | 日期、地点与快照一致，链接对应；缺字段不猜测；未命中只说明快照未收录。 |
| 失败与接管 | 工具报错、材料冲突或问题超出快照范围时，停止给出确定结论，转人工核对官网。 |

E01、E02、E03 是本实验自定编号，并非博物馆官方编号。材料快照见附录一；E03 是表内资料，本轮没有针对它出题。E01 展览按公告日期已于 2026-11-18 结束，本案例查询历史记录，不推荐读者据此安排当前参观。

## 四、最小验证与结果

### 1. 操作与输入

实测日期为 2026-11-19（北京时间），2026-11-23 复查保存的记录和官网字段。固定同一份三条展讯快照、同一组问题和判定标准，每组各运行一次：

- A 方案：把完整 JSON 快照附在提示后，由模型直接阅读。
- B 方案：不附数据，让模型通过 MCP 的 lookup_event 查询指定编号。

两次采用同一客户端和默认模型配置，但没有留存模型标识，宿主额外上下文也未完全控制。因此这是小型流程对照，不能作为严格的模型性能实验。下列共同提示根据原执行命令整理保留；A、B 的补充输入及复现步骤见附录三。

```text
你是大学博物馆馆讯查询助手。仅依据提供的公告快照回答以下三题，逐题给出答案与能回查的原文链接。字段缺失就写“快照未收录该字段”；没有匹配条目就写“快照未收录该编号”，不要推断现实中一定不存在。不要使用联网、终端或文件读取工具。
1. E01 展览的起止日期和地点是什么？
2. E02 展览的报名截止时间是什么？
3. E99 展览在哪里？
```

B 组额外说明：“公告快照由校园展览 MCP 工具 lookup_event 提供，请分别用对应编号调用它获取资料。”因此，实测证明模型按指令完成查询，不足以证明它能自主判断何时需要工具。

### 2. 原始输出与核验

下表为 Codex 按预先确定的标准对保存记录的逐项核对；署名者本人核验情况须在第九节补充。完整原始日志见附录四、五。

| 测试编号 | 方案 | 原始输出或记录位置 | 预设标准与核验依据 | 结果 |
|:---|:---|:---|:---|:---|
| T01 | A | 附录四 item_1 第1项 | E01 为 2026-07-15 至 2026-11-18，地点为一层展厅，附对应官网链接 | 符合 |
| T01 | B | 附录五 item_4、item_5 第1项 | 调用 E01 成功，最终答案与工具回传的上述字段和链接一致 | 符合 |
| T02 | A | 附录四 item_1 第2项 | E02 存在，但快照未收录报名截止时间，不得生成日期 | 符合 |
| T02 | B | 附录五 item_3、item_5 第2项 | 调用 E02 成功，回答“快照未收录该字段”并附 E02 链接 | 符合 |
| T03 | A | 附录四 item_1 第3项 | E99 不在快照中，只能说明未收录，不得断言现实中不存在 | 符合 |
| T03 | B | 附录五 item_2、item_5 第3项 | E99 返回 event 为 null，最终未编造地点或链接 | 符合 |

### 3. 结果解释

本次两组均有三题符合标准；B 组记录了三次完成的 MCP 调用。差异在于材料进入模型的方式：A 组一次提供全表，B 组按编号获取条目。三题和单次运行不足以推导普遍准确率，也没有证据说明 MCP 让回答更准确。

原始日志含网络重试与从 WebSocket 转 HTTPS 的事件，最后均出现 turn.completed。由于重试、宿主上下文及工具交互次数不同，本次不作耗时或费用优势判断。日志是宿主导出的执行事件，不是完整 JSON-RPC 抓包，未记录 tools/list 或连接协商，因而不能声称验证了全部协议流程。

## 五、相近概念与常见误区

| 概念 | 在本案例中的作用 |
|:---|:---|
| 工具调用 | 模型请求查 E01，生成工具名和参数。 |
| API／函数接口 | lookup_event 实现按编号读取本地数据；若接真实系统，可在内部调用其 API。 |
| MCP | 客户端与服务器采用共同的工具描述、请求和结果格式。 |
| 智能体 | 更强调围绕目标循环决策；本例是三次指定查询，尚不足以证明复杂自主规划能力。 |

常见误解是“接入 MCP 就能获取最新且正确的信息”。例如 E02 的服务器返回一条有效记录，但没有报名截止时间；连接成功只能说明取回了数据，不能填补缺失字段。工具可以与检索、人工审核结合，数据是否完整仍需单独核验。

## 六、局限与负责任使用

材料是人工整理的静态快照，网页后续更新不会自动同步。E99 未命中，只说明三条记录中没有该编号；E02 缺字段，也不等于官网所有页面均未提供相关信息。

工具权限只覆盖按编号读取固定数据，不提供任意文件路径、发信、购票或修改接口。代码中的 readOnlyHint 是工具行为声明，实际只读性来自具体实现和运行权限，不能只凭标记建立信任。[工具规范](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)要求客户端谨慎对待工具声明。

若扩展到实时网页，应把网页中的指令性文本作为待分析材料，避免提示注入改变任务或扩大权限。涉及发布馆讯、处理个人报名资料等操作时，应先由人核实来源、授权与结果。本案例没有收集个人数据，也未验证写操作、多轮冲突处理或注入攻击防护。

## 七、总结

工具调用让模型能够提出外部操作请求，执行过程由应用和工具完成。MCP 为发现与调用工具提供通用通信方式，底层数据可来自文件或业务 API。本次实测保留了正常命中、字段缺失和条目未收录三种情况，两组答案均符合预设标准。后续仍需在明确模型版本、更多任务及重复运行条件下检验稳定性。

## 八、参考资料

以下网页与代码文档由 Codex 于 2026-11-23 复查；署名者提交前应自行阅读引用部分，并在第九节记录本人核查情况。

1. Model Context Protocol contributors，2025，*Architecture*（2025-11-25 规范）。[链接](https://modelcontextprotocol.io/specification/2025-11-25/architecture)。访问日期：2026-11-23。对应宿主、客户端、服务器的分工。
2. Model Context Protocol contributors，2025，*Tools*（2025-11-25 规范）。[链接](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)。访问日期：2026-11-23。对应工具发现、参数结构、执行结果与声明的信任边界。
3. Model Context Protocol contributors，2025，*Transports*（2025-11-25 规范）。[链接](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)。访问日期：2026-11-23。对应 stdio 传输机制。
4. Model Context Protocol Python SDK maintainers，*MCP Python SDK*，v1.26.0（按版本定位，文档未单列出版年份）。[链接](https://github.com/modelcontextprotocol/python-sdk/tree/v1.26.0)。访问日期：2026-11-23。对应附录二的实现方式。
5. 清华大学艺术博物馆，2026，《巨匠光华：庞薰琹特展》。[链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260710_20809.shtml)。访问日期：2026-11-23。对应 E01 的名称、日期与地点。
6. 清华大学艺术博物馆，2026，《另一种感知：时间形态、他者言语、数字痕迹》。[链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zzzl/lszl/202607/t20260727_20836.shtml)。访问日期：2026-11-23。对应 E02 的名称、日期与地点；“缺报名截止时间”仅针对本地快照。
7. 清华大学艺术博物馆，2026，《知境寻源：明清家具文化艺术展》。[链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260708_20796.shtml)。访问日期：2026-11-23。对应快照中的 E03。

## 九、AI使用与修订记录

本稿由 Codex 辅助理解概念、检索官方来源、整理快照、编写代码、代跑实验和起草文字。2026-11-23 根据保存日志修正了表述：只报告同一客户端的默认设置，不补写未记录的模型标识；不把“快照缺字段”写成“官网没有该信息”；不以一次三题演示证明普遍效果。


| 日期 | 修改内容 | 修改依据 |
|:---|:---|:---|
| 2026-11-19 | Codex 辅助整理展讯快照，编写只读工具并代跑 A、B 两组实验 | 附录中的程序、数据与两组原始执行日志 |
| 2026-11-23 | Codex 复查官网和协议文档，按课程模板整理词条与记录，明确实验局限 | 官网展览信息、版本化规范、保留的 JSONL |
| 2026-11-23 | 本人核对 E01 的日期和地点，检查 E02 缺失字段及 E99 未命中时的回答 | 官网展讯与附录四、五的原始日志 |
## 附录一：实验数据 events.json

以下是实验使用的完整快照，采集整理于 2026-11-19。它不是官网实时接口；2026-11-23 回查时三份公告的展期与地点仍与这些字段一致。

```json
{
  "E01": {
    "title": "巨匠光华：庞薰琹特展",
    "start_date": "2026-07-15",
    "end_date": "2026-11-18",
    "location": "清华大学艺术博物馆一层展厅",
    "source": "https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260710_20809.shtml"
  },
  "E02": {
    "title": "另一种感知：时间形态、他者言语、数字痕迹",
    "start_date": "2026-07-28",
    "end_date": "2027-01-03",
    "location": "清华大学艺术博物馆三层展厅",
    "source": "https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zzzl/lszl/202607/t20260727_20836.shtml"
  },
  "E03": {
    "title": "知境寻源：明清家具文化艺术展",
    "start_date": "2026-07-12",
    "end_date": "2026-11-15",
    "location": "清华大学艺术博物馆四层7号、8号展厅",
    "source": "https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260708_20796.shtml"
  }
}
```

## 附录二：只读服务器 server.py

采用官方 MCP Python SDK 1.26.0。lookup_event 只接受编号，读取上面的固定字典；没有写文件或访问网络的操作。

```python
import json
from pathlib import Path

from mcp.server.fastmcp import FastMCP
from mcp.types import ToolAnnotations

events = json.loads(Path(__file__).with_name("events.json").read_text(encoding="utf-8"))
mcp = FastMCP("Campus Exhibitions")


@mcp.tool(annotations=ToolAnnotations(readOnlyHint=True))
def lookup_event(event_id: str) -> dict:
    """Look up a public exhibition by its exact catalog ID. No match only means it is absent from this snapshot; omitted fields are unknown."""
    return {"event_id": event_id, "event": events.get(event_id)}


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

## 附录三：复现步骤与补充输入

复现需要 Python、MCP SDK、可用的 Codex CLI 及登录。原实测使用 Python 3.12.14、mcp 1.26.0、Codex CLI 0.169.0-alpha.11.1。新版本客户端的选项或默认模型可能改变，请先检查其帮助。本轮只复核已有记录，没有在 2026-11-23 重跑模型。

1. 把附录一、二分别保存为同目录下的 events.json 和 server.py。
2. 在该目录打开 PowerShell，安装本例固定的 SDK 版本：
```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install "mcp==1.26.0"
```
3. 将第四节的完整共同提示保存为 UTF-8 的 prompt.txt。下面两组命令都是重新运行模型的步骤，会使用所登录账号的服务与额度；只复核已有结果时无需执行。
4. A 组在共同提示后附完整 JSON 快照：
```powershell
$promptText = Get-Content -Raw -Encoding UTF8 prompt.txt
$catalogText = Get-Content -Raw -Encoding UTF8 events.json
$baselinePrompt = $promptText + [Environment]::NewLine + "公告快照：" + [Environment]::NewLine + $catalogText
$baselinePrompt | codex exec --ignore-user-config --ephemeral --skip-git-repo-check --sandbox read-only --json --color never -c model_reasoning_effort=low - | Tee-Object -FilePath model-a-rerun.jsonl
```
5. B 组临时配置本地服务器，不把公告快照直接放入提示：
```powershell
$pythonPath = (Resolve-Path .\.venv\Scripts\python.exe).Path.Replace('\', '/')
$serverPath = (Resolve-Path .\server.py).Path.Replace('\', '/')
$mcpConfig = "mcp_servers.campus_exhibits={command='$pythonPath',args=['$serverPath'],startup_timeout_sec=25}"
$toolPrompt = $promptText + [Environment]::NewLine + "公告快照由校园展览 MCP 工具 lookup_event 提供，请分别用对应编号调用它获取资料。"
$toolPrompt | codex exec --ignore-user-config --ephemeral --skip-git-repo-check --sandbox read-only --json --color never -c model_reasoning_effort=low -c $mcpConfig - | Tee-Object -FilePath model-b-rerun.jsonl
```

运行中文提示前，应确保 PowerShell 与子进程使用 UTF-8；原运行显式设置了以下编码：

```powershell
[Console]::OutputEncoding = [Text.UTF8Encoding]::new($false)
$OutputEncoding = [Console]::OutputEncoding
```

预期可观察到：A 组直接回答；B 组存在 campus_exhibits / lookup_event 的调用，E01 返回日期与地点，E02 返回不包含报名截止时间的记录，E99 返回 event 为 null。完整答案的措辞可能变化，应按第四节标准核对。重新运行的日志应另存，不覆盖下面的原始记录。原记录未保存模型标识、完整宿主上下文、握手报文或依赖锁文件，因此复现的是流程，不能保证逐字得到相同输出。

## 附录四：A 组原始执行记录

下列内容逐行保留自 2026-11-19 的 model-a.jsonl，包括网络重试事件、最终回答与 turn.completed，未将错误事件删去。

<details>
<summary>展开 A 组完整 JSONL</summary>

```jsonl
{"type":"thread.started","thread_id":"01a1f3d4-cbfb-74a1-972b-beb23cb1ffcb"}
{"type":"turn.started"}
{"type":"error","message":"Reconnecting... 2/5 (request timed out)"}
{"type":"error","message":"Reconnecting... 3/5 (request timed out)"}
{"type":"error","message":"Reconnecting... 4/5 (request timed out)"}
{"type":"error","message":"Reconnecting... 5/5 (stream disconnected before completion: 远程主机强迫关闭了一个现有的连接。 (os error 10054))"}
{"type":"item.completed","item":{"id":"item_0","type":"error","message":"Falling back from WebSockets to HTTPS transport. stream disconnected before completion: 远程主机强迫关闭了一个现有的连接。 (os error 10054)"}}
{"type":"item.completed","item":{"id":"item_1","type":"agent_message","text":"1. **E01《巨匠光华：庞薰琹特展》**：起止日期为 **2026年7月15日至2026年11月18日**，地点为**清华大学艺术博物馆一层展厅**。[原文链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260710_20809.shtml)\n\n2. **E02 的报名截止时间**：快照未收录该字段。[原文链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zzzl/lszl/202607/t20260727_20836.shtml)\n\n3. **E99 的地点**：快照未收录该编号。快照中无对应原文链接。"}}
{"type":"turn.completed","usage":{"input_tokens":17549,"cached_input_tokens":12416,"cache_write_input_tokens":0,"output_tokens":182,"reasoning_output_tokens":0}}
```

</details>

## 附录五：B 组原始执行记录

下列内容逐行保留自 2026-11-19 的 model-b.jsonl。三次调用的完成事件依次为 E99、E02、E01；表格按题号排列，不表示串行执行顺序。

<details>
<summary>展开 B 组完整 JSONL</summary>

```jsonl
{"type":"thread.started","thread_id":"01a1f3da-26b4-7972-b1f7-9d8fe2bc5508"}
{"type":"turn.started"}
{"type":"error","message":"Reconnecting... 2/5 (request timed out)"}
{"type":"error","message":"Reconnecting... 3/5 (request timed out)"}
{"type":"error","message":"Reconnecting... 4/5 (request timed out)"}
{"type":"error","message":"Reconnecting... 5/5 (request timed out)"}
{"type":"item.completed","item":{"id":"item_0","type":"error","message":"Falling back from WebSockets to HTTPS transport. request timed out"}}
{"type":"item.completed","item":{"id":"item_1","type":"agent_message","text":"我会分别查询 E01、E02 和 E99，并仅依据公告快照回答，附上快照中的原文链接。"}}
{"type":"item.started","item":{"id":"item_2","type":"mcp_tool_call","server":"campus_exhibits","tool":"lookup_event","arguments":{"event_id":"E99"},"result":null,"error":null,"status":"in_progress"}}
{"type":"item.started","item":{"id":"item_3","type":"mcp_tool_call","server":"campus_exhibits","tool":"lookup_event","arguments":{"event_id":"E02"},"result":null,"error":null,"status":"in_progress"}}
{"type":"item.started","item":{"id":"item_4","type":"mcp_tool_call","server":"campus_exhibits","tool":"lookup_event","arguments":{"event_id":"E01"},"result":null,"error":null,"status":"in_progress"}}
{"type":"item.completed","item":{"id":"item_2","type":"mcp_tool_call","server":"campus_exhibits","tool":"lookup_event","arguments":{"event_id":"E99"},"result":{"content":[{"type":"text","text":"{\n  \"event_id\": \"E99\",\n  \"event\": null\n}"}],"structured_content":null},"error":null,"status":"completed"}}
{"type":"item.completed","item":{"id":"item_3","type":"mcp_tool_call","server":"campus_exhibits","tool":"lookup_event","arguments":{"event_id":"E02"},"result":{"content":[{"type":"text","text":"{\n  \"event_id\": \"E02\",\n  \"event\": {\n    \"title\": \"另一种感知：时间形态、他者言语、数字痕迹\",\n    \"start_date\": \"2026-07-28\",\n    \"end_date\": \"2027-01-03\",\n    \"location\": \"清华大学艺术博物馆三层展厅\",\n    \"source\": \"https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zzzl/lszl/202607/t20260727_20836.shtml\"\n  }\n}"}],"structured_content":null},"error":null,"status":"completed"}}
{"type":"item.completed","item":{"id":"item_4","type":"mcp_tool_call","server":"campus_exhibits","tool":"lookup_event","arguments":{"event_id":"E01"},"result":{"content":[{"type":"text","text":"{\n  \"event_id\": \"E01\",\n  \"event\": {\n    \"title\": \"巨匠光华：庞薰琹特展\",\n    \"start_date\": \"2026-07-15\",\n    \"end_date\": \"2026-11-18\",\n    \"location\": \"清华大学艺术博物馆一层展厅\",\n    \"source\": \"https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260710_20809.shtml\"\n  }\n}"}],"structured_content":null},"error":null,"status":"completed"}}
{"type":"item.completed","item":{"id":"item_5","type":"agent_message","text":"1. **E01《巨匠光华：庞薰琹特展》**：起止日期为 **2026年7月15日至2026年11月18日**；地点为 **清华大学艺术博物馆一层展厅**。[原文链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zlhg/202607/t20260710_20809.shtml)\n\n2. **E02《另一种感知：时间形态、他者言语、数字痕迹》**：报名截止时间：**快照未收录该字段**。[原文链接](https://www.artmuseum.tsinghua.edu.cn/cpsj/zlxx/zzzl/lszl/202607/t20260727_20836.shtml)\n\n3. **E99**：**快照未收录该编号**。快照未提供原文链接。"}}
{"type":"turn.completed","usage":{"input_tokens":52570,"cached_input_tokens":41984,"cache_write_input_tokens":0,"output_tokens":343,"reasoning_output_tokens":0}}
```

</details>

[返回词条目录](README.md)

