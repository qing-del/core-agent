# Agent 协作编辑：单节点首发规范

## 1. 首发范围

首发只支持按稳定 `nodeId` 定位的**单节点**编辑。一次 Agent 文本范围只能位于一个目标节点内；不支持跨段、跨标题、跨列表项或跨其他节点的选区和替换。

首发不引入业务 `blockId`、`block revision`、`agentSection`、跨节点 transaction 或 `nodeHash` 校验。大文本“开辟区域”的锚点和分批写入语义也不属于本期定义，后续单独设计。

这里的“节点”是面向 Agent 读取、定位和编辑的已注册文档节点，不是 ProseMirror 内部的文本叶子节点。一个可承载文本范围的 paragraph、heading 或 listItem 等节点，才是 `replaceTextRange` 的目标；范围不得跨越它的边界。

## 2. 上线前必须提供的节点基座

`middleware` 现有编辑器已通过 `Y.Doc + Y.XmlFragment('content')` 同步 Tiptap 文档；`resourceReference` 也已有 `refId`。但它目前没有通用 `nodeId` 或 `nodeVersion`。本节定义的是首发上线前必须补齐的基座能力，而不是已存在的实现。

### 2.1 统一节点模型

每个可被 Agent 定位的已注册文档节点都要能投影为：

```text
Node
├─ nodeId       稳定身份
├─ type         节点类型
├─ attrs        节点属性
├─ content      子内容或文本内容
├─ marks        格式标记
└─ refId        仅 resourceReference 使用，值等于 nodeId

并发保护
├─ nodeVersion  节点自身版本
└─ nodeHash     可选；首发不使用
```

`nodeId` 的生命周期规则如下：

- 节点创建时生成；同一文档内必须唯一，推荐使用 UUID 或等价的不可预测 ID。
- 节点删除后，其 `nodeId` 永久失效，绝不能被后续新节点复用。
- 复制、粘贴或新建节点必须生成新的 `nodeId`，不能继承原节点 ID。
- `resourceReference` 的 `refId` 是该节点的稳定身份，应与通用 `nodeId` 保持相同值；它不是另一套独立 ID。

`nodeVersion` 是节点自身的乐观并发版本：

- `type`、`attrs`、`content` 或 `marks` 的任一持久化变化都必须递增版本。
- 节点仅被移动、重排或改变父节点位置，但节点自身上述字段未变时，不递增版本。
- 每次读取节点投影都必须返回当前 `nodeVersion`；每个 Agent 写请求都必须携带其读取或 prepare 时获得的 `expectedNodeVersion`。

首发不计算或校验 `nodeHash`。它只预留给后续诊断、审计或跨版本兼容；不得把它作为首发提交的额外前置条件。

## 3. 节点能力注册与 Agent 投影

`nodeId` 只解决"找到哪个节点"，并不授权 Agent 任意改写节点。每个已注册节点类型必须通过 `AgentNodeCapability` 声明 Agent 可执行的操作、允许的输入字段和 Schema 校验器。

```ts
type AgentNodeCapability = {
  type: string
  operations: readonly string[]
  validate(operation: AgentNodeOperation, node: AgentNodeProjection): ValidationResult
}
```

首发只固定统一外层契约，不预先假设每一种节点的字段写法：

```ts
type ApplyNodeOperationRequest = {
  nodeId: string
  expectedNodeVersion: number
  operation: AgentNodeOperation
}

applyNodeOperation(request: ApplyNodeOperationRequest)
```

`AgentNodeOperation` 是按节点能力扩展的判别联合。首发已定义的操作只有文本节点能力提供的 `replaceTextRange`；其他节点类型的具体 operation、属性白名单和校验规则留待对应適配器定义。节点没有能力注冊、能力未声明该 operation，或输入未通过能力校验时，统一返回 `UNSUPPORTED`。

`resourceReference` 虽已在前端注冊，也具备 `refId`，但首发尚未定义它的 Agent 编辑能力。因此 Agent 对它的任何写入仍返回 `UNSUPPORTED`，直到其专用能力、属性白名单和审计规则被补充。

Agent 读取的是不暴露 Yjs 内部坐标的投影，例如：

```ts
type AgentNodeProjection = {
  nodeId: string
  nodeVersion: number
  type: string
  attrs: Record<string, unknown>
  content: unknown
  marks: unknown
}
```

投影中未定义的节点字段不得由模型猜测或写入。 `nodeId` 只在当前文档内有效；跨文档使用同一 ID 必须被拒绝。

## 4. 单节点文本范围操作

### 4.1 准备范围

文本节点能力提供以下 prepare 操作：

```ts
prepareTextRange({
  nodeId,
  expectedNodeVersion,
  targetText,
  before,
  after
})
```

其中 `targetText` 是 Agent 要修改的原文，`before` 和 `after` 是同一节点文本内的相邻上下文。Engine 只在以下条件同时成立时创建范围：

1. `nodeId` 能定位到当前文档内的一个已注冊、支持文本范围操作的节点；
2. 当前版本与 `expectedNodeVersion` 一致；
3. `targetText + before + after` 能在该节点内唯一定位起止边界；
4. 起止边界均属于同一节点，且起点不大于终点。

多个命中、上下文不足或范围越过节点边界时返回 `AMBIGUOUS_SELECTION`；节点不存在返回 `NODE_NOT_FOUND`；节点不支持文本范围操作返回 `UNSUPPORTED`。Engine 不得退化为"命中第一个相同文本"。

prepare 成功后，Worker 只在内部保存下列材料，并向 Agent 返回不透明的 `rangeRef` 和实际选中的文本：

```ts
type PreparedTextRange = {
  agentRunId: string
  nodeId: string
  preparedNodeVersion: number
  start: Uint8Array // 同一文本节点内的 encodeRelativePosition(start)
  end: Uint8Array   // 同一文本节点内的 encodeRelativePosition(end)
  expectedTextFingerprint: string
  createdAt: number
}
```

文本指纹是目标范围当前纯文本的原始 UTF-8 字节的 SHA-256；不进行 Unicode、空白、格式或节点结构归一化。 `rangeRef` 仅属于创建它的 Agent run，且一次性有效：成功、冲突、取消或创建 10 分钟后失效；失效、跨 run 使用或重复提交都返回 `RANGE_EXPIRED`。

### 4.2 提交范围

提交仍走统一入口：

```ts
applyNodeOperation({
  nodeId,
  expectedNodeVersion,
  operation: {
    kind: 'replaceTextRange',
    rangeRef,
    replacement
  }
})
```

`nodeId` 和 `expectedNodeVersion` 必须与 `rangeRef` 的准备记录一致，否则返回 `CONFLICTED`。 `replacement` 在首发只能是无换行、无 mark、无 HTML、无 Markdown、无 Tiptap JSON 的纯文本。目标节点当前 Schema 不允许该替换时返回 `SCHEMA_REJECTED`。

Worker 先定位当前 `nodeId`，再验证 `rangeRef` 的两个相对端点仍归属于该节点内部。端点无法解析、已越过节点边界或顺序无效时返回 `INVALID_RANGE`；节点已删除时返回 `NODE_NOT_FOUND`。不允许将端点解析到另一个节点后继续提交。

### 4.3 `nodeVersion` 与文本范围的两层判定

普通节点操作以版本作为绝对前置条件：当前 `nodeVersion !== expectedNodeVersion` 时直接返回 `CONFLICTED`，不会尝试属性、marks 或结构自动合并。

`replaceTextRange` 是唯一例外，按以下两层规则处理：

```text
当前 nodeVersion 等于 preparedNodeVersion
  → 校验节点能力、rangeRef 与 Schema 后直接提交

当前 nodeVersion 不等于 preparedNodeVersion
  → 重新解析同一节点内的两个 RelativePosition
  → 校验范围有效，并对当前选区文本计算 SHA-256
  → 指纹一致：允许提交
  → 指纹不一致：CONFLICTED
```

这使 `nodeVersion` 既能明确指出"节点自 prepare 后发生过变化"，又不会誤伤目标文本未变化的边界编辑。二次校验只比较文本；若版本变化仅来自 `attrs` 或 `marks`，文本相同仍可提交，最终格式保留或变化由该节点的 Tiptap/ProseMirror transaction 和能力適配器决定。

## 5. 同节点边界插入语义

在同一个文本节点中：

```text
初始：abcd
Agent 选择：bc
```

范围起点固定使用 `assoc = 0`，终点固定使用 `assoc = -1`：

```ts
start = Y.createRelativePositionFromTypeIndex(xmlText, from, 0)
end = Y.createRelativePositionFromTypeIndex(xmlText, to, -1)
```

规则如下：

```text
真人在 a 后插入 m：ambcd
当前保护范围仍为 bc，文本指纹仍为 bc
→ nodeVersion 已变化，但二次校验通过；Agent 可替换 bc

真人在 d 前插入 m：abcdm
当前保护范围仍为 bc，文本指纹仍为 bc
→ nodeVersion 已变化，但二次校验通过；Agent 可替换 bc

真人在 b 与 c 之间插入 m：abmcd
当前保护范围变为 bmc，文本指纹变化
→ CONFLICTED
```

位于 `from` 的插入属于范围左侧，位于 `to` 的插入属于范围右侧，均不进入保护范围；只有严格落在两端之间的修改会改变范围文本并造成冲突。业务代码不得自行选择其他 `assoc` 组合，也不得保存易失效的绝对文本下标。

## 6. 非首发范围

以下能力不在本期实现或验收范围，文档不得再将它们描述为首发可用：

- 跨 paragraph、heading、listItem 或任意节点的选区、替换、合并和拆分；
- 以 ProseMirror 全局 `from/to` 或跨节点 `RelativePosition` 提交的 Agent 操作；
- `agentSection`、业务 `blockId`、block revision、结构指纹与 `nodeHash` 校验；
- Agent 创建标题、列表、表格、图片、引用或其他富文本节点；
- "开辛区域"、`insertLargeContent` 的锁点规则、分块事务和部分失败恢复。

大文本的 WebSocket 帧限制、ACK 与重连重放仍是现有协作传输层事实，但在新的节点模型中，只有在“新节点创建、插入锚点和批次原子性”另行定义后，才能形成 Agent Tool 契约。

## 7. Worker 与协作链路

首发架构保持既有分工：Java `core-agent` 负责 Agent run 编排和审计；受管 Node/TypeScript Worker 负责 `Y.Doc`、同节点 RelativePosition、节点投影与每文档 Agent 串行队列。Worker 以独立 `core_agent` 身份接入 Document WebSocket，因此页面关闭h后仍可继续。

Worker 使用已有 `DocumentCollaborationClient` 的协议语义，而不是复用浏览器类本身：完成 `SYNC_COMPLETE` 后发送本地 update；每条 update 生成 UUID 并保留在 `pendingUpdates`；收到 `UPDATE_ACCEPTED` 后移除；重连 bootstrap 完成后重放未确认 update。

现有 middleware WebSocket 只允许 `clientId = user` 写入。上线前必须让具备 `document:write` 的受控 `core_agent` 身份通过该通道，并保留 Agent run、发起用户和实际写入主体的审计关联。Java WebSocket 层继续只路由不透明的 Yjs update，不参与节点定位、版本比较或范围校验。

同一文档中的 Agent prepare 和 commit 必须在 Worker 内串行。每次执行前先合并已经收到的远端 update，再读取节点版本或解析范围。真人编辑不受 Worker 队列或文档锁限制；若真人 update 在 Agent 校验前到达，按本规范返回冲突或通过二次校验；真正同时到达的 update 仍由 Yjs 合并，不承诺服务端原子 compare-and-commit。

## 8. 结果码与首发验收

统一使用以下结果码：

- `APPLIED`：节点操作已通过校验并生成本地 Yjs update，进入 ACK 确认链路；
- `CONFLICTED`：`nodeVersion` 不符合规则，或文本范围二次校验的指纹不一致；
- `NODE_NOT_FOUND`：`nodeId` 在当前文档中不存在或目标节点已删除；
- `INVALID_RANGE`：文本范围端点无法映射到同一目标节点，或端点顺序无效；
- `AMBIGUOUS_SELECTION`：目标文本与上下文不能在目标节点内唯一定位；
- `UNSUPPORTED`：节点、操作、输入字段或输出超出已注册能力；
- `SCHEMA_REJECTED`：当前 Tiptap/ProseMirror Schema 不接受该操作；
- `RANGE_EXPIRED`：`rangeRef` 超時、已使用、已取消或不属于当前 Agent run。

首发至少验收以下场景：

- `nodeId` 创建即生成且不复用；删除节点后旧 ID 不可定位，复制/新建节点生成新 ID；
- `type`、`attrs`、`content`、`marks` 任一变化都递增 `nodeVersion`；纯位置移动不递增；
- 同一文本节点内的 `replaceTextRange` 成功；跨节点或跨段范围返回 `UNSUPPORTED`；
- 普通节点操作版本不一致返回 `CONFLICTED`；
- 文本范围版本不一致时，左右边界插入可提交，内部插入、删除或替换返回 `CONFLICTED`；
- 节点删除返回 `NODE_NOT_FOUND`，端点失效或跨出目标节点返回 `INVALID_RANGE`；
- 未注册节点或未定义 Agent 能力的已注册节点返回 `UNSUPPORTED`；
- `rangeRef` 超時、跨 run 使用和重复提交都返回 `RANGE_EXPIRED`；
- Worker 完成 bootstrap 后写入，收到 `UPDATE_ACCEPTED` 清除待确认 update，重连后重放未确认 update。

最终模型可收敛为一句话：

> Agent 用 `nodeId` 定位一个节点，以 `nodeVersion` 进行乐观并发控制；只有同节点纯文本范围可在版本变化后通过 RelativePosition 和文本指纹进行有限的二次校验。
