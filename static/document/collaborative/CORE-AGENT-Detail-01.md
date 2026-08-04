# 解决框选下标偏移的细节内容

```mermaid
flowchart LR
  A["初始 XmlText：A | B | C | D"] --> B["Agent 框选 CD<br/>start → C 前<br/>end → D 后"]
  B --> C["真人在 A 后插入 M"]
  C --> D["当前内容：A | M | B | C | D"]
  D --> E["解析相对位置<br/>start = 3，end = 5"]
  E --> F["Agent 仍准确框选 CD"]
```

## 1. Agent 如何控制光标和选区

Agent 没有浏览器里的真实光标，它维护的是一份“协作光标状态”：

```ts
type AgentCursorState = {
  actorId: string
  actorType: 'AGENT'
  name: string
  color: string
  anchor: Y.RelativePosition
  head: Y.RelativePosition
}
```

- 光标：`anchor === head`
- 选区：`anchor !== head`
- 选区方向由 `anchor` 与 `head` 保留
- 前端把该状态作为协作 Presence/Awareness 渲染成 Agent 光标、姓名标签与选区高亮

对于 `ABCD`，Agent 框选 `CD`：

```ts
import * as Y from 'yjs'

const start = Y.createRelativePositionFromTypeIndex(xmlText, 2, 0)
const end = Y.createRelativePositionFromTypeIndex(xmlText, 4, -1)
```

其中：

```text
初始文本：A B C D
绝对位置：0 1 2 3 4

选区：    [   C D   )
           2       4
```

此时不要持久化 `2` 和 `4`，而是持久化 `start` 与 `end` 的编码结果：

```ts
const startBytes = Y.encodeRelativePosition(start)
const endBytes = Y.encodeRelativePosition(end)
```

它们可以存为 PostgreSQL `bytea`，或 Base64 字符串。

当真人在 `A` 后插入 `M`：

```text
原文：ABCD
改后：AMBCD

原选区绝对下标：2..4
新选区绝对下标：3..5
```

Agent 在真正提交前重新解析：

```ts
const startAbsolute = Y.createAbsolutePositionFromRelativePosition(start, ydoc)
const endAbsolute = Y.createAbsolutePositionFromRelativePosition(end, ydoc)

if (!startAbsolute || !endAbsolute) {
  throw new Error('目标节点已经不存在')
}

const from = startAbsolute.index // 3
const to = endAbsolute.index     // 5
const current = xmlText.toString().slice(from, to) // CD
```

因此，插入发生在选区外部时，选区会自然随其所锚定的 CRDT 内容移动。

但相对位置只解决“位置跟随”，不保证“目标语义仍正确”。所以 Agent 提交前必须再验证局部指纹：

```ts
const expectedText = 'CD'
const expectedFingerprint = sha256(expectedText)

const actualText = xmlText.toString().slice(from, to)
if (sha256(actualText) !== expectedFingerprint) {
  return { status: 'CONFLICTED' }
}
```

例如：

```text
初始：ABCD
Agent 目标：CD

真人改为：ABXD
```

相对位置仍可能解析成功，但目标片段已不是 `CD`。此时 Agent 必须停止覆盖，重新读取局部内容。

所以完整原则是：

> RelativePosition 负责跟随内容；局部指纹负责防止错误覆盖。

## 2. 边界插入的语义

选区边界本身可能被并发插入，因此创建相对位置时要规定关联方向。

对“替换 `CD`”而言，通常期望：

```text
start：绑定到 C 一侧
end：绑定到 D 一侧
```

这样，选区中的原始内容仍是 `C` 到 `D`，而不是因为有人恰好在边界输入文字就被 Agent 一并替换。

这部分不要让各业务代码直接散落地传 `assoc` 参数。应封装为唯一的范围服务：

```ts
function createProtectedTextRange(xmlText: Y.XmlText, from: number, to: number) {
  return {
    start: Y.createRelativePositionFromTypeIndex(xmlText, from, 0),
    end: Y.createRelativePositionFromTypeIndex(xmlText, to, -1)
  }
}
```

后续通过协同测试固定边界规则，包括：

- 在选区前插入；
- 在选区后插入；
- 在选区起点插入；
- 在选区终点插入；
- 在选区内部插入；
- 删除整个选区；
- 删除选区所属节点。

## 3. Agent 如何读取 Yjs XML 文档

Tiptap 的协作内容通常存放在：

```ts
const root = ydoc.getXmlFragment('content')
```

其内部不是一个普通字符串，而是类似这样的 Yjs XML 树：

```text
Y.XmlFragment(content)
└── Y.XmlElement(paragraph)
    └── Y.XmlText("ABCD")
```

复杂文档可能是：

```text
Y.XmlFragment(content)
├── Y.XmlElement(heading)
│   └── Y.XmlText("项目背景")
├── Y.XmlElement(paragraph)
│   └── Y.XmlText("ABCD")
└── Y.XmlElement(bullet_list)
    └── ...
```

Agent 不应该直接看到 Yjs 二进制，也不应自己生成 XML/Yjs 操作。

应由 TS Yjs Document Engine 把 Yjs XML 树投影为 Agent 可理解的结构：

```ts
type AgentDocumentProjection = {
  documentId: string
  blocks: Array<{
    ref: string
    type: 'heading' | 'paragraph' | 'listItem' | 'agentSection'
    text: string
    attributes: Record<string, string>
  }>
}
```

例如：

```json
{
  "documentId": "doc_001",
  "blocks": [
    {
      "ref": "block_7",
      "type": "paragraph",
      "text": "ABCD",
      "attributes": {}
    }
  ]
}
```

Agent 可以读到：

```text
[block_7]
ABCD
```

但模型不应直接提交：

```json
{ "from": 2, "to": 4 }
```

而应通过受控工具选择内容：

```text
selectText(block_7, "CD")
```

TS 引擎定位后生成并保存真实的 `RelativePosition`、局部指纹和上下文信息，返回一个不透明引用：

```json
{
  "rangeRef": "range_01J...",
  "selectedText": "CD"
}
```

Agent 后续只使用：

```text
replaceRange(range_01J..., "新的内容")
```

而不是接触 Yjs 内部坐标。

## 4. Agent 如何更新 Y.XmlText

小范围文本替换应在同一个 Yjs transaction 中完成：

```ts
function replaceIfUnchanged(
  ydoc: Y.Doc,
  xmlText: Y.XmlText,
  range: PreparedRange,
  replacement: string
) {
  const start = Y.createAbsolutePositionFromRelativePosition(range.start, ydoc)
  const end = Y.createAbsolutePositionFromRelativePosition(range.end, ydoc)

  if (!start || !end || start.type !== xmlText || end.type !== xmlText) {
    return { status: 'CONFLICTED', reason: '目标文本节点已经变化' }
  }

  const from = start.index
  const to = end.index
  const actual = xmlText.toString().slice(from, to)

  if (sha256(actual) !== range.expectedFingerprint) {
    return { status: 'CONFLICTED', reason: '目标内容已被修改' }
  }

  ydoc.transact(() => {
    xmlText.delete(from, to - from)
    xmlText.insert(from, replacement)
  }, {
    source: 'agent',
    agentRunId: range.agentRunId
  })

  return { status: 'APPLIED' }
}
```

这次 transaction 产生的就是普通 Yjs update：

```ts
const update = Y.encodeStateAsUpdate(ydoc, beforeStateVector)
```

它应与真人编辑产生的 update 走同一条链路：

```text
Agent Yjs transaction
  → Yjs binary update
  → PostgreSQL update log
  → WebSocket 广播
  → 真人客户端 applyUpdate
  → Tiptap 自动刷新
```

Agent 不是旁路写数据库，也不是把全文覆盖回去；它是一个受控的虚拟协作者。

## 5. Agent 如何更新 XML 节点

这里建议区分两类操作。

### 文本级操作

操作目标是既有 `Y.XmlText`：

```ts
xmlText.insert(index, text)
xmlText.delete(index, length)
xmlText.applyDelta(delta)
```

适用于：

- 替换句子；
- 改写段落；
- 添加行内文本；
- 保留原有段落结构。

首发版本优先只开放这一类操作。

### 结构级操作

操作目标是 `Y.XmlFragment` 或 `Y.XmlElement`：

```ts
const section = new Y.XmlElement('agentSection')
section.setAttribute('sectionId', crypto.randomUUID())
section.setAttribute('agentRunId', agentRunId)
section.setAttribute('status', 'draft')

const paragraph = new Y.XmlElement('paragraph')
const text = new Y.XmlText()
text.insert(0, 'Agent 生成的草稿内容')

paragraph.insert(0, [text])
section.insert(0, [paragraph])

root.insert(targetBlockIndex + 1, [section])
```

这对应“区域开辟”。

但 `agentSection` 必须是 Tiptap Schema 中正式声明的节点，不能只是服务端临时拼装的未知 XML 标签。否则前端在同步、复制、编辑或重新序列化时可能丢失它的属性。

建议它至少包含：

```text
agentSection
├── sectionId
├── agentRunId
├── status: drafting | ready | accepted | conflicted
├── title
└── content
```

它可以在 UI 中呈现为：

```text
┌─ Agent 草稿：方案补充 ──────────┐
│  当前状态：待确认               │
│                                 │
│  ... Agent 生成的大段内容 ...   │
└─────────────────────────────────┘
```

## 6. XML 直接操作的边界

Yjs XML 很适合做底层精确更新，但 Agent 不应自由拼装任意 XML。

首发建议限制为：

- 既有 `Y.XmlText` 的局部替换；
- 固定结构的 `agentSection` 插入；
- 固定结构的段落、标题、列表生成；
- 所有富文本、表格、图片、引用等复杂节点，后续通过专门的 Tiptap / ProseMirror 适配层生成。

特别是富文本格式，`Y.XmlText` 需要通过 Delta 表达：

```ts
xmlText.applyDelta([
  { insert: '重点内容', attributes: { bold: true } }
])
```

不要把 HTML 字符串直接塞进 `Y.XmlText`，也不要让模型自行构造 Yjs update 二进制。

## 7. 单机版最重要的约束

单机版本中，每个文档应有一个串行执行器：

```text
真人 Yjs update
Agent 光标更新
Agent prepare replace
Agent commit replace
Agent open section
```

这些动作按文档顺序进入同一队列。这样 Agent 在“重新解析锚点 → 校验指纹 → XML transaction”之间不会被另一条 Agent 写入打断。

真人编辑仍可持续进入队列；真人修改同一目标时，Agent 的局部校验失败并进入冲突，而不是覆盖真人内容。

最终可以把 Agent 的操作模型收敛成一句话：

> Agent 读取的是 Yjs XML 的语义投影，保存的是相对位置，提交的是经过局部指纹校验的 Yjs transaction。