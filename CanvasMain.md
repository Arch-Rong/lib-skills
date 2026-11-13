# CanvasMain 画布模块说明文档

## 1. 概述

CanvasMain 是 Lumina-Muse 项目中的 **AI 工作流画布模块**，基于 [React Flow (@xyflow/react)](https://reactflow.dev/) 构建。用户可以在可视化画布上通过拖拽、连线的方式编排 AI 生成节点（文本、图片、视频等），组成完整的内容创作工作流。

**核心能力：**
- 可视化节点编排（拖拽、连线、缩放）
- 多种 AI 生成节点类型（文生文、文生图、文生视频等）
- 多工作流管理（创建、切换、重命名）
- 自动保存与持久化（localStorage / API 可插拔）
- 节点间数据流传递（上游输出自动注入下游输入）

画布项目创建时可勾选“团队项目”，默认创建个人项目。`ProjectInfo.isTeamProject` 由后端持久化，并通过现有项目详情加载流程传入 `WorkflowBuilder`；该属性不写入画布 JSON。个人项目沿用自动保存及冲突恢复，团队项目启用已有节点锁、SSE 和版本同步。顶部模式标签只读，普通项目编辑不改变项目属性。BO 项目总览通过现有项目成员列表接口展示相同类型，个人项目的成员添加同时由 UI 和后端限制。

---

## 2. 目录结构

```
src/pages/CanvasMain/
├── index.tsx                        # 页面入口，路由组件
├── types.ts                         # 所有 TypeScript 类型定义
├── exports.ts                       # 模块公开导出入口
├── constants/
│   └── nodeTypes.tsx                # 节点类型配置、连接规则、操作卡片配置
├── components/
│   ├── WorkflowBuilder.tsx          # 核心画布组件（ReactFlow 容器）
│   ├── nodes/                       # 节点组件
│   │   ├── BaseNode.tsx             # 通用节点基础组件（逻辑层）
│   │   ├── TextNodeWithBaseNode.tsx  # 文生文节点
│   │   ├── ImageNodeWithBaseNode.tsx # 文生图节点
│   │   ├── VideoNodeWithBaseNode.tsx # 文生视频节点
│   │   ├── OutputTextNodeWithBaseNode.tsx # 输出文本节点
│   │   ├── UploadNodeWithBaseNode.tsx # 上传节点（图片/视频/音频）
│   │   ├── ModelSelect.tsx          # 模型选择器
│   │   ├── ModelParamsSelector.tsx   # 模型参数选择器
│   │   ├── NodeKeyboard.tsx         # 节点键盘（输入区域）
│   │   ├── view/                    # 节点视图层
│   │   │   ├── BaseNodeView.tsx
│   │   │   ├── TextNodeView.tsx
│   │   │   ├── ImageNodeView.tsx
│   │   │   └── VideoNodeView.tsx
│   │   └── css/                     # 节点样式
│   └── ui/                          # UI 组件
│       ├── ActionCard.tsx           # 添加节点操作卡片（逻辑层）
│       ├── NodeContextMenu.tsx      # 节点右键菜单（逻辑层）
│       ├── EdgeScissorsButton.tsx   # 连线剪刀删除按钮（逻辑层）
│       ├── view/                    # UI 视图层
│       │   ├── ActionCardView.tsx
│       │   ├── NodeContextMenuView.tsx
│       │   ├── EdgeScissorsButtonView.tsx
│       │   └── FloatButtonView.tsx  # 悬浮工具栏视图
│       └── css/                     # UI 样式
├── hooks/
│   └── useUpstreamData.ts          # 上游节点数据监听 Hook
└── utils/
    ├── connectionUtils.ts           # 连接验证工具
    ├── nodeUtils.ts                 # 节点创建工具
    ├── dataUtils.ts                 # 数据类型检测（图片/视频判断）
    └── imageUtils.ts                # 图片处理工具（URL 转 Base64）
```

---

## 3. 架构设计

### 3.1 分层架构

模块采用 **逻辑层 / 视图层分离** 的设计模式：

```
┌─────────────────────────────────────────────┐
│              CanvasMain (页面入口)             │
│         路由参数解析 + API 注入                 │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           WorkflowBuilder (核心画布)           │
│    ReactFlow 容器 + 工作流管理 + 自动保存       │
└──────────────────┬──────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼───┐    ┌────▼────┐   ┌────▼────┐
│ Nodes │    │   UI    │   │  Hooks  │
│ 节点层 │    │ 交互层   │   │ 数据层  │
└───┬───┘    └────┬────┘   └─────────┘
    │             │
┌───▼───┐    ┌────▼────┐
│ View  │    │  View   │
│ 视图层 │    │  视图层  │
└───────┘    └─────────┘
```

### 3.2 数据流

```
localStorage / API ──► WorkflowBuilder ──► nodes + edges (React State)
                                │                    │
                                │               ReactFlow
                                │                    │
                         自动保存 (防抖 1s)     ◄──── 用户操作
                         节流 (最小间隔 3s)
```

---

## 4. 核心组件

### 4.1 CanvasMain（页面入口）

**文件：** `index.tsx`

作为路由页面组件，职责：
1. 从路由获取 `projectId` 参数
2. 基于 `localStorage` 实现数据持久化（可替换为 API 调用）
3. 通过 Props 将 API 回调注入 `WorkflowBuilder`
4. 通过 `renderFloatingToolbar` 渲染悬浮按钮组

**提供的 API 回调：**

| 回调 | 说明 |
|------|------|
| `handleFetchWorkflowList` | 分页获取工作流列表 |
| `handleGetWorkflowCanvas` | 获取指定工作流的画布数据 |
| `handleSaveWorkflowCanvas` | 保存工作流画布（含节点、连线、视口） |
| `handleCreateWorkflow` | 创建新工作流 |
| `handleUpdateWorkflowName` | 更新工作流名称 |

### 4.2 WorkflowBuilder（核心画布）

**文件：** `components/WorkflowBuilder.tsx`

核心画布组件，基于 ReactFlow 构建，负责：

**Props（WorkflowBuilderProps）：**

| Prop | 类型 | 说明 |
|------|------|------|
| `onFetchWorkflowList` | `(params) => Promise<WorkflowListResponse>` | 获取工作流列表 |
| `onGetWorkflowCanvas` | `(params) => Promise<WorkflowData>` | 获取画布数据 |
| `onSaveWorkflowCanvas` | `(params) => Promise<{...}>` | 保存画布 |
| `onCreateWorkflow` | `(params) => Promise<WorkflowData>` | 创建工作流 |
| `onUpdateWorkflowName` | `(params) => Promise<{...}>` | 更新名称 |
| `onStateChange` | `(state: WorkflowBuilderState) => void` | 状态变化回调 |
| `renderFloatingToolbar` | `(ctx) => ReactNode` | 自定义悬浮工具栏 |

**核心功能：**

- **工作流管理：** 列表加载、分页、创建、切换（含未保存提醒）、重命名
- **画布操作：** 节点拖拽、连线（贝塞尔曲线）、框选、缩放/平移
- **节点操作：** 添加、复制/粘贴（支持批量）、删除（支持批量）、右键菜单
- **连线操作：** 连接创建（防重复）、剪刀删除、选中高亮
- **自动保存：** 防抖 1s + 节流 3s + 快照比对避免无效保存
- **离开保护：** 页面关闭前提醒、标签页切换自动保存、路由切换保存确认
- **快捷键：** `Ctrl+C` 复制 / `Ctrl+V` 粘贴 / `Ctrl+A` 全选 / `Delete` 删除
- **防拖拽卡死：** 全局监听鼠标事件，5s 超时强制停止拖拽
- **节点内容刷新：** BaseNode 展示层在移动期间复用上次内容，坐标及选框仍实时跟随；连续 120ms 没有累计达到 1 个屏幕像素的有效位移时显示最新内容，松手、失焦或拖动中断时立即刷新。任务、数据更新及原有保存流程继续执行。

**注册的节点类型：**

```typescript
const nodeTypes = {
  textNode: TextNodeWithBaseNode,       // 文生文
  outputTextNode: OutputTextNodeWithBaseNode, // 输出文本
  imageNode: ImageNodeWithBaseNode,     // 文生图
  videoNode: VideoNodeWithBaseNode,     // 文生视频
  uploadNode: UploadNodeWithBaseNode // 上传节点（图片/视频/音频）
};
```

### 4.3 BaseNode（通用节点基础组件）

**文件：** `components/nodes/BaseNode.tsx`

所有业务节点的基础组件，提供通用的节点交互能力：

- **屏幕区域（Screen）：** 显示节点内容，hover 显示连接点，点击展开键盘
- **键盘区域（Keyboard）：** 输入区域，与连接点互斥显示
- **左右连接点（Handle）：** 左侧为输入（target），右侧为输出（source）
- **节点缩放：** 支持右侧/底部/角落拖拽缩放，最小 250px
- **连接点跟随鼠标：** 连接点图标在 20px 范围内跟随鼠标移动
- **节点类型选择器：** 拖拽连接点到空白处时弹出，可快速创建新节点并建立连接
- **连接规则校验：** 基于 `DEFAULT_ALLOWED_CONNECTIONS` 验证连接合法性

### 4.4 业务节点

| 节点 | 类型标识 | 说明 |
|------|---------|------|
| TextNodeWithBaseNode | `textNode` | 文生文节点，文本输入生成文本输出 |
| OutputTextNodeWithBaseNode | `outputTextNode` | 纯输出文本节点，支持直接编辑 |
| ImageNodeWithBaseNode | `imageNode` | 文生图节点，文本输入生成图片 |
| VideoNodeWithBaseNode | `videoNode` | 文生视频节点，文本输入生成视频 |
| UploadNodeWithBaseNode | `uploadNode` | 上传节点，支持图片/视频/音频上传 |

### 4.5 UI 组件

| 组件 | 说明 |
|------|------|
| `FloatButtonView` | 悬浮工具栏：添加节点、返回项目页、素材库、文件管理 |
| `ActionCard` | 添加节点操作卡片，展示可用节点类型列表 |
| `NodeContextMenu` | 节点右键菜单：复制、粘贴、删除 |
| `EdgeScissorsButton` | 连线上的剪刀图标，点击删除连线 |

---

## 5. 节点连接规则

定义在 `constants/nodeTypes.tsx` 的 `DEFAULT_ALLOWED_CONNECTIONS` 中：

| 节点类型 | 允许的输入来源 | 允许的输出目标 |
|---------|--------------|--------------|
| text（文本） | textNode, imageNode, uploadNode | textNode, imageNode, videoNode |
| image（图片） | textNode, imageNode | textNode, imageNode, videoNode |
| video（视频） | textNode, imageNode | textNode, outputTextNode |
| upload（上传） | 无（通常不需要输入） | textNode, outputTextNode, imageNode, videoNode |

---

## 6. Hooks

### useUpstreamData

**文件：** `hooks/useUpstreamData.ts`

监听当前节点的上游连接变化，自动收集上游节点的输出数据。

```typescript
const { upstreamData, upstreamDataType, upstreamTextNodes, upstreamImageNodes } = useUpstreamData(
  nodeId,           // 当前节点 ID
  onChangeCallback, // 上游数据变化回调
  checkImage,       // 是否检测图片类型（默认 false）
  checkInterval     // 检查间隔（默认 500ms）
);
```

**返回值（UseUpstreamDataReturn）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `upstreamData` | `string \| null` | 第一个上游节点的输出（向后兼容） |
| `upstreamDataType` | `'text' \| 'image' \| null` | 上游数据类型 |
| `upstreamTextNodes` | `UpstreamNodeInfo[]` | 所有上游文本节点信息 |
| `upstreamImageNodes` | `UpstreamNodeInfo[]` | 所有上游图片节点信息 |

---

## 7. 工具函数

### connectionUtils.ts

| 函数 | 说明 |
|------|------|
| `checkDuplicateConnection` | 检查两个节点间是否已存在连接 |
| `getValidTargetHandle` | 检测鼠标位置下的合法目标连接点 |
| `checkNodeConnectionToHandle` | 检查节点左右连接点的连接状态 |

### nodeUtils.ts

| 函数 | 说明 |
|------|------|
| `createNodeConfig` | 创建新节点的完整配置 |
| `createEdgeConfig` | 创建新连线的完整配置 |
| `calculateNewNodePosition` | 根据拖拽位置和连接方向计算新节点位置 |

### dataUtils.ts

| 函数 | 说明 |
|------|------|
| `isImageData` | 检测字符串是否为图片数据（Base64 或 URL） |
| `isVideoData` | 检测字符串是否为视频数据（Base64 或 URL） |

### imageUtils.ts

| 函数 | 说明 |
|------|------|
| `convertUrlToBase64` | 将图片 URL 转换为 Base64 |
| `processImages` | 批量处理图片数组，统一转为 Base64 |

---

## 8. 类型定义

所有类型集中在 `types.ts` 中，主要分类：

| 分类 | 主要类型 |
|------|---------|
| 模型相关 | `ModelItem`, `ModelFuncMap`, `ModelParamMap`, `ParsedModelParams` |
| 节点相关 | `BaseNodeData`, `BaseNodeProps`, `BusinessNodeProps`, `NodeTypeConfig` |
| 连接相关 | `ConnectionData`, `AllowedConnections`, `AllowedConnectionsMap` |
| 工作流 | `WorkflowData`, `WorkflowListResponse`, `WorkflowBuilderProps`, `WorkflowBuilderState` |
| UI 组件 | `ActionCardProps`, `NodeContextMenuProps`, `EdgeScissorsButtonProps` |
| API 参数 | `GenerateTextParams`, `GenerateImageParams`, `GenerateVideoParams` |

---

## 9. 自动保存机制

```
用户操作 → nodes/edges 状态变化
            │
            ▼
      防抖 1s (DEBOUNCE_DELAY)
            │
            ▼
     节流 3s (MIN_SAVE_INTERVAL)
            │
            ▼
     快照比对 (buildSnapshot)
   排除 selected 等无关字段
            │
     有实质变化？──── 否 ──► 跳过
            │
            是
            ▼
     调用 saveWorkflowCanvas
     保存节点 + 连线 + viewport
```

**额外保存触发：**
- 页面 `visibilitychange`（切换标签页 / 最小化时）
- 工作流切换前（弹窗确认）
- 页面关闭前（`beforeunload`）

---

## 10. 快捷键

左下角键盘按钮可展开快捷键说明。Windows/Linux 使用 Ctrl，macOS 使用 Cmd；成组/解组也支持 Alt/Option。

| 快捷键 | 功能 |
|--------|------|
| `Ctrl/Cmd + A` | 全选所有节点 |
| `Ctrl/Cmd + C` | 复制选中节点（支持批量） |
| `Ctrl/Cmd + V` | 粘贴节点（批量粘贴保持相对位置） |
| `Ctrl/Cmd/Alt + G` | 将选中的多个节点成组 |
| `Ctrl/Cmd/Alt + Shift + G` | 解除选中的组 |
| `Ctrl/Cmd + L` | 将多选节点按从左到右的顺序连线；单选图片/视频节点时进入参考选择 |
| `Alt/Option + H` | 隐藏当前画布的连线及连线删除按钮 |
| `Alt/Option + S` | 显示当前画布的连线及连线删除按钮 |
| `Ctrl/Cmd + Enter` | 点击当前节点的生成按钮，沿用按钮禁用态和生成校验 |
| `Tab` | 在鼠标位置打开新建节点菜单；输入和按钮内保留原有 Tab 行为 |
| `Alt/Option + 拖动节点` | 复制并移动副本，保留原节点和已有复制缓冲区 |
| `Ctrl + Alt/Option + 拖动节点` | 复制无结果的节点，保留提示词、模型参数、名称、尺寸及关联连线；不继承历史结果或生成中的任务 |
| `Ctrl/Cmd + Z` | 撤销 |
| `Ctrl/Cmd + Shift + Z` / `Ctrl/Cmd + Y` | 重做 |
| `Ctrl/Cmd + +` / `Ctrl/Cmd + -` / `Ctrl/Cmd + 0` | 放大 / 缩小 / 适应画布 |
| `Alt + Shift + F` | 整理画布 |
| `Delete / Backspace` | 删除选中节点（支持批量） |
| 鼠标左键拖拽 | 框选节点 / 移动节点 |
| 鼠标右键拖拽 / 中键拖拽 | 平移画布 |
| `Space + 左键拖动` | 平移画布 |
| 鼠标滚轮 | 平移画布 |
| 触控板双指捏合 | 缩放画布 |
| `Shift / Ctrl + 点击` | 多选节点 |

文字输入、弹窗和剪辑板保留各自快捷键；节点提示词区支持 `Ctrl/Cmd + Enter` 生成。连线沿用实际可用端口、类型/引用数量校验和协作锁。连线显隐仅影响当前视图，不修改节点引用或保存数据，重新进入画布时默认显示。

---

## 11. 可扩展性

### API 层可替换

`CanvasMain` 通过 Props 注入 API 回调给 `WorkflowBuilder`，当前使用 `localStorage` 实现。替换为远程 API 只需修改 `index.tsx` 中的回调实现，无需改动画布组件。

### 节点类型可扩展

1. 在 `components/nodes/` 下创建新的节点组件（继承 BaseNode）
2. 在 `constants/nodeTypes.tsx` 中注册节点配置和连接规则
3. 在 `WorkflowBuilder.tsx` 的 `nodeTypes` 中注册

### 模块化导出

通过 `exports.ts` 统一导出所有组件、类型、Hooks 和工具函数，可作为独立模块被其他页面引用。

## 12. 脚本节点四步流程

1. **确认镜头**：在节点参数中选择文本模型和视频模型。视频模型有默认值，未展开参数面板时也会使用同一默认规则。后端按模型名称、版本及当前公司配置读取 `duration` 的有效正数，得到最小时长、最大时长和支持的档位；据此组织镜头，直接将超长动作或对白拆成多条分镜。时长、景别、光影、音效、对白、运镜等镜头信息在这一步补齐，不生成任何生图或视频提示词。
2. **准备资产**：打开现有资产生成弹窗，勾选缺图元素，点击“合成缺失提示词”。仅为缺图、未生成中且提示词为空的元素调用文本模型。已有提示词可继续编辑；合成结果保存后，另点“生成”提交图片任务。打开弹窗不会自动合成或生图。
3. **合成提示词**：确认镜头与元素后，明确选择智能合成或自动拼接，生成 Start 起始帧、Apex 动作峰值帧、End 结束帧三条独立图片提示词和视频提示词。智能合成使用相邻镜头作上下文，只更新选中镜头的提示词与连续性说明，不覆盖已确认的时长、景别、光影、运镜、台词和资产绑定。
4. **合成分镜参考**：复用第3步的三条图片提示词及该镜头关联的角色、场景、道具图片，通过已有生图流程分别生成 Start、Apex、End 三帧。每帧使用独立的图片节点和请求 ID，同一镜头的三帧放在同一组，并按 Start → Apex → End 横向排列，不同镜头按行向下排列；分批创建时保留对应帧的位置。只提交勾选的待生成帧，成功帧与在途帧会跳过，失败帧可单独重试。旧版单张参考保留并标记为旧版，不猜测其动作相位。其他步骤的提示词合成入口统一跳转第3步。

模型缓存为空或失效时，脚本节点复用现有模型列表接口补载，两个选择器随加载结果更新；获取失败可点击“重新加载”。打开参数不会自动拿锁保存默认模型，明确切换时沿用既有协作锁和保存流程。

第4步的“批量生成视频”继续使用脚本所选的视频模型，按勾选镜头各自的时长生成；只允许当前提示词及三帧参考均就绪的镜头。默认按 Start → Apex → End 作为三帧参考，也可选择 Start/End 首尾帧模式。画幅、分辨率等选项读取模型声明，时长逐镜校验。每个镜头复用普通视频节点的下单、素材注册、轮询和结果保存流程；完成后在该镜头显示视频链接及“定位视频节点”入口。已有视频与在途任务跳过，未完成的空视频节点原位重试，重新打开画布后继续恢复结果。

脚本连接的小说原文或文本节点正文显示在键盘顶部的文本引用栏，复用图片节点的引用卡片和“参考”选择入口。内容编辑区仅显示脚本节点自己保存的正文；未编辑正文时，分析仍读取连接的文本。删除引用仅移除对应连线，保留源节点及手写正文。

视频模型长期按名称和版本保存，供应商记录 ID 变化不会改变所选模型语义。模型没有有效 `duration` 配置时明确报错。每个分镜最终时长必须不小于 `min_duration_seconds`、不超过 `max_duration_seconds`，且属于模型支持的 `supported_duration_seconds`。模型返回的数值时长未命中选项时，后端本地吸附到最近有效档位，距离相同时取较长档位，越界时取最近端点。例如 `[5,10]` 中 7 秒吸附到 5 秒，7.5 秒吸附到 10 秒。审核说明记录调整，供用户核对动作与对白节奏；不增加模型调用。

元素提示词保存在现有 `scriptAssetBatch.prompts` 中，已有图片、已有非空提示词、生成中及待确认任务不会被批量覆盖。合成失败保留草稿；合成成功但保存失败时重试保存，不重复调用模型。

资产、分镜参考及视频的批量生成失败时，弹窗按资产名称或镜号显示参数校验、下单拒绝的具体原因，并保留提示词和生成参数供调整后重试。参考图片超出所选模型上限时沿用现有校验拦截，不自动丢弃资产图片或提交订单。

脚本编辑弹窗复用本页已持有的有效编辑租约，并等待尚未完成的申请或释放；输入框和弹窗全部结束使用后才释放原锁。生成租约继续独占，其他成员的租约不会被复用。

批量提交持有本页生成租约时，节点轮询的静默申请先在本地等待原租约释放，不每秒重复申请远端锁；释放后沿用原轮询接管。保存失败会阻止当前项下单，并提示此前已提交的任务数量。资产、三帧参考和镜头视频任务在连续空查询时均保留原请求 ID，等待确认后恢复结果，避免再次下单。

脚本分析以稳定产出可编辑结果为优先。原文、台词、人物命名、镜号顺序及画面绑定一致性等内容问题写入已有审核备注或待确认事项，不中断分析；空白创作文本可留待补充。原文核对兼容排版和 Markdown 标点转义差异，备注保留元素或镜号、字段路径及有长度限制的原文对照片段，供用户自行核对。

JSON 语法、必需业务字段、类型及枚举、重复或空编号、待处理镜头或元素的结果缺失仍需修复，保证结果可显示并按 ID 合并。每个阶段在首次生成后最多修复 5 次，成功即继续，达到上限明确报错；括号或引号不闭合也进入修复。后续修复携带原始输出、上次结果和最新错误，沿用现有进度、取消、超时及计费流程。缺省审核数组和可空标签本地补齐，额外字段只按当前阶段约定投影，避免覆盖已确认内容。供应商明确返回长度截断、容量不足或调用取消时仍停止并提示原因。此行为需配套新版后端；本次未部署或调用真实模型。

旧画布和旧提示词继续保留。应用当前视频模型时长范围和档位需要重新分析；模型切换不会自动调用模型。首次新分析的图片和视频提示词为空，第3步完成度为零。

本次接入需配套后端接口和 V40 提示词配置，配置发布说明见后端 `docs/canvas-script-four-step-release.md`。三帧版本另需配套 V41 提示词更新；不包含视频批量下单或自动视频接力。

2026-10-16 已在当前后端使用的 `online` 库执行并核验 V40，第二步所需的元素提示词配置已启用；旧配置已备份。配置按请求读取，补齐配置后可直接重试合成，无需为此重启后端。其他数据库仍需单独核对和更新。

三帧提示词字段沿用 `picture_prompt` 表示 Start，新增 `apex_picture_prompt` 与 `end_picture_prompt`。旧画布缺少新字段时保留原内容，可在第3步手动合成补齐；进入步骤不自动调用模型。三帧按动作相位组织，身份、造型、物品和空间保持一致。

2026-10-16 19:45（Asia/Shanghai）已按用户授权在 `online` 执行 V41，仅升级并启用镜头三帧提示词模板，原记录完整备份于 `t_internal_prompt_config_backup_v41`，提交前后核验通过。其他四条脚本配置、模型选项和价格未变；本次未重启后端或提交生成任务。使用时需运行三帧版本的后端，再到第3步重新合成所需镜头。
