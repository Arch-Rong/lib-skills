# Lumina-Muse（光之灵感 · 创意工坊）— 项目 Skills

> 最后更新：2026-08-14
> 基于 Monorepo 多应用架构（pnpm workspace），包含 1 个独立应用与 10 个共享包。

---

## 一、技术栈

| 分类 | 技术 | 版本 |
| --- | --- | --- |
| 框架 | React | 19.2 |
| 语言 | TypeScript | ~5.9 |
| 构建 | Vite | 6.4 |
| 包管理 | pnpm workspace (Monorepo) | 9.15 |
| UI 库 | Ant Design | 6.3 |
| Pro 组件 | @ant-design/pro-components | 2.8 |
| AI 对话组件 | @ant-design/x | 2.5 |
| 富文本 | TipTap (StarterKit + Mention + Placeholder + Suggestion) | 3.20 |
| 样式 | TailwindCSS v4 + Less | 4.2 / 4.5 |
| 路由 | React Router DOM | 7.13 |
| 画布引擎 | @xyflow/react (React Flow) | 12.10 |
| 图标库 | lucide-react + @tabler/icons-react | — |
| 图表 | echarts | 6.1 |
| 图像处理 | fabric.js | 7.4 |
| CSS-in-JS | antd-style (createStyles) | 4.1 |
| 状态管理 | zustand (chat-main 项目) | 5.0 |
| 对象存储 | cos-js-sdk-v5 | 1.10 |
| 编译优化 | babel-plugin-react-compiler | 1.0 |
| 自动导入 | unplugin-auto-import (react + react-router-dom) | 21.0 |
| 压缩 | vite-plugin-compression2 (gzip, ≥10KB) | 2.5 |
| 代码压缩 | Terser（生产环境 drop_console + drop_debugger） | 5.46 |

---

## 二、项目架构

### 2.1 Monorepo 总览

```
Lumina-Muse/
├── apps/
│   └── bo/                    # @lumina/bo-app — 独立运营后台应用
├── packages/                  # 10 个共享工作空间包
│   ├── core/                  # @lumina/core — 共享核心
│   ├── ctrl-main/             # @lumina/ctrl-main — 创作控制台
│   ├── chat-main/             # @lumina/chat-main — 对话项目
│   ├── canvas-main/           # @lumina/canvas-main — 画布工作流
│   ├── script-main/           # @lumina/script-main — 剧本项目
│   ├── login/                 # @lumina/login — 登录页
│   ├── console/               # @lumina/console — 控制台
│   ├── guide/                 # @lumina/guide — 使用指南
│   ├── project-summary/       # @lumina/project-summary — 项目汇总
│   └── ecommerce-main/        # @lumina/ecommerce-main — 电商项目
├── src/                       # 根入口
│   ├── router/                # 路由配置 (AuthGuard + lazyWithRetry)
│   ├── components/            # LazyWrapper 组件
│   ├── pages/                 # NotFound404 页面
│   ├── utils/                 # lazyWithRetry 工具
│   ├── App.tsx
│   └── main.tsx
├── docs/                      # 项目文档
├── public/                    # 静态资源
├── vite.config.ts             # 构建配置
├── pnpm-workspace.yaml
└── package.json
```

### 2.2 包依赖关系

```
core ──→ (无内部依赖，被所有包引用)
ctrl-main ──→ core
canvas-main ──→ core
chat-main ──→ core
script-main ──→ core
bo（独立应用）──→ core
login ──→ core + @ant-design/pro-components
project-summary ──→ core
console ──→ core
guide ──→ core
ecommerce-main ──→ core
```

### 2.3 请求架构

| 模式 | 函数 | 超时 | 用途 |
| --- | --- | --- | --- |
| 普通 POST | `postRequest<T>` | 6s（可覆盖） | 登录、列表查询、创建订单等 |
| SSE 流式 | `postChunkedRequest<T>` | 20min | 视频/图片/文本生成（payOrder）|
| FormData | `postFormData<T>` | 6s | 文件上传 |

**请求特性：**
- 雪花 ID（128 位十六进制）作为 `REQUEST-ID` 请求头，支持请求追踪
- 请求头携带 `X-USER`（用户 ID）、`X-ACCESS-TOKEN`（JWT Token）
- Token 自动刷新：流式响应中检测 `replyHeader.session` 字段，通过 `iat` 时间戳比较决定是否替换
- 401 重试机制：SSE 请求遇到 401 时，先检查本地 Token 是否已变更，否则轮询等待最多 30s
- 认证失效处理：`replyHeader.code === 20001` 或 `14001` → 清除凭证并跳转首页
- 错误码：`12000` 成功 | `14001/14002` 鉴权失败 | `14003` 业务警告 | `20001` 强制登出
- 基础 URL：`/v2`（postRequest 默认）| BO 接口前缀：`/bo/v1`（跳过 base URL）| `/bos` rootBoServer 前缀

### 2.4 状态管理

- **React Context**：
  - `CtrlMainContext`：在 CtrlMain 中构建，通过 `<Outlet context={...}>` 向子面板传递 `projectId`、`modelInfos`、`prefillData`、骨架图操作函数等
  - `FavorContext`：收藏状态全局管理（`FavorProvider` + `useFavorContext`），useRef 存储收藏 map，`_version` 状态号触发重渲染
  - `CanvasContext`：画布项目的余额 + 结果历史状态管理（双 slice Reducer：`balanceReducer` + `resultHistoryReducer`）
  - `InputParamsContext`：输入参数共享上下文
  - `SelectionContext`：画布框选状态追踪（`isDragSelecting`）
- **zustand**：chat-main 项目使用 `chatFlowStore` 管理 Flow 状态
- **localStorage 持久化**（按用户 ID 隔离）：
  - 面板输入数据（prompt / images / params）— `panelStorage.ts`
  - 会话 ID — `saveConversationId / loadConversationId`
  - Splitter 尺寸偏好 — `saveSplitterSizes / loadSplitterSizes`
  - 用户信息 — `userStorage.ts`
  - 模型列表缓存 — `modelCache.ts`（8h TTL + token 绑定）
- **sessionStorage**：
  - 跨路由做同款数据传递（`__prefillData__`）
  - 骨架屏倒计时持久化（`__countdown_{id}`），刷新后可恢复
  - Chunk 加载重试标记（`__chunk_reload__`）

### 2.5 构建优化

- **代码分割**（manualChunks）：
  - `vendor-react` — React/ReactDOM/ReactRouter/Scheduler
  - `vendor-tiptap` — TipTap/ProseMirror/Floating-ui/tippy.js
  - `vendor-antd` — antd 核心 + @ant-design/icons + rc-* + @ant-design/pro-* + cssinjs
  - `vendor-dayjs` — dayjs
  - `vendor-cos` — COS SDK
  - `vendor-icons` — @tabler/icons-react + lucide-react
  - `vendor-xyflow` — @xyflow/react
  - `vendor-three` — 打光预览与导演工具共用的 Three.js 引擎
  - Muse 工作空间包按功能拆分：`pkg-core`、`pkg-ctrl-main`、`pkg-canvas-main-*` 等；主应用不再包含 BO 页面代码
- **多应用构建**：根 `pnpm build` 顺序生成 Muse `dist/` 与 BO `dist-bo/`
- **懒加载**：Muse 使用 `lazyWithRetry`，BO 使用路由级 `React.lazy` + `Suspense`
- **Gzip 压缩**：≥10KB 文件自动生成 .gz
- **React Compiler**：通过 `babel-plugin-react-compiler` 启用自动 memoization
- **自动导入**：`unplugin-auto-import` 自动注入 `react` 和 `react-router-dom` 的 import

### 2.6 Chunk 加载容错

| 方案 | 组件/函数 | 作用 |
| --- | --- | --- |
| 方案一 | `lazyWithRetry()` | 包装 `React.lazy`，捕获 import 失败后 3 次延迟重试（800ms 间隔），失败则 `window.location.reload()`，通过 `sessionStorage.__chunk_reload__` 防止无限刷新 |
| 方案二 | `ChunkErrorBoundary` | Error Boundary 兜底，展示「检测到新版本，请刷新页面」提示 |

### 2.7 版本检测

- 构建时自动生成 `version.json`（命名规则：`{版本号}-{分支名}-{YYMMDD}-{git短hash}`）
- `useVersionCheck` 30s 轮询检测新版本
- `VersionUpdateBar` 组件显示更新提示条（全局覆盖所有路由页面）

---

## 三、路由系统

**配置位置：** `src/router/index.tsx`、`packages/core/src/constants/routeConfig.ts`

```typescript
BASE_PATH = '/CtrlMain';  // 项目路由前缀
BASE_NAME = '';           // Router basename（空字符串）
HOME_ROUTE = '/';
LOGIN_ROUTE = '/login';
```

### 3.1 路由表

Muse 主应用部署在 `/`；BO 是同一 Monorepo 中的独立应用，构建后部署在 `/bo/`，页面代码不进入 Muse 构建产物。

| 路径 | 页面 | 权限 |
|------|------|------|
| `/`、`/login` | Login | 无 |
| `/CtrlMain/:projectId` | CtrlMain 工作台（默认 video） | 登录 + 项目权限 |
| `/CtrlMain/:projectId/video` | VideoCtrlPanel | 登录 + 项目权限 |
| `/CtrlMain/:projectId/image` | ImageCtrlPanel | 登录 + 项目权限 |
| `/CtrlMain/:projectId/text` | TextCtrlPanel | 登录 + 项目权限 |
| `/projectSummary` | ProjectSummary | 登录 |
| `/ScriptMain/:projectId` | ScriptMain | 登录 + 项目权限 |
| `/ChatMain/:projectId` | ChatMain | 登录 + 项目权限 |
| `/CanvasMain/:projectId` | CanvasMain | 登录 + 项目权限 |
| `/guide` | Guide | 登录 |
| `/Console` | Console | 登录 |
| `/bo/home` | BillOverview | ADMIN/TEAM_ADMIN |
| `/bo/addMember` | AddMember | ADMIN/TEAM_ADMIN |
| `/bo/listMember` | ListMember | ADMIN/TEAM_ADMIN |
| `/bo/memberDetail/:id` | MemberDetail | ADMIN/TEAM_ADMIN |
| `/bo/project` | BoProject | ADMIN/TEAM_ADMIN |
| `/bo/addCustomer` | AddCustomer | ADMIN |
| `/bo/subAgentData` | SubAgentData | ADMIN |
| `/bo/listCustomer` | ListCustomer | ADMIN |
| `/bo/billOverview` | BillOverview | ADMIN/TEAM_ADMIN |
| `/bo/consumeOverview` | ConsumeOverview | ADMIN/TEAM_ADMIN |
| `/bo/CustomerDetail/:tId` | CustomerDetails | ADMIN |
| `/bo/detailBill` | BillDetail | ADMIN/TEAM_ADMIN |
| `/bo/summyBill` | SummyBill | ADMIN/TEAM_ADMIN |
| `/bo/memberSummyBill` | MemberSummyBill | ADMIN/TEAM_ADMIN |

### 3.2 路由守卫 AuthGuard

- 检查 `sessionStorage.getItem('token')`
- 加载项目列表并校验用户是否有当前 `projectId` 的权限
- 角色校验：`ROOT`（root）、`ADMIN`（admin）、`TEAM_ADMIN`（team_admin）、`USER`（user）、`GUEST`（guest）
- 使用 `useReducer` 管理项目访问状态机（RESET → GRANT / DENY）
- BO 使用 `apps/bo/src/router.tsx` 中的独立守卫；仅允许 admin/team_admin，team_admin 与受限 admin 使用 BO 页面白名单
- 普通用户访问 BO → 跨应用重定向到 Muse `/projectSummary`
- `extractProjectId` 支持 `CtrlMain`、`ScriptMain`、`ChatMain` 提取 projectId，`CanvasMain` 通过 `useParams` 直接获取

---

## 四、API 接口清单

> 所有 API 定义在 `packages/core/src/api/index.ts`，共 **62 个函数**。

### 4.1 请求工具函数

| 函数 | 参数 | 说明 |
|------|------|------|
| `postRequest(url, data, timeout?, retry?, skipBaseUrl?, requestId?, skipAuth?)` | 通用 | 普通 POST 请求 |
| `postChunkedRequest(url, onChunk, data, onComplete?, onError?, timeout?, retry?, customHeaders?)` | SSE | 流式 SSE 请求 |
| `postFormData(url, formData, timeout?, skipBaseUrl?)` | FormData | 文件上传 |

### 4.2 认证

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiLogin(params)` | `/UserApi.login` | 登录（skipAuth=true）|
| `apiChangePassword(old, new)` | `/UserApi.changePassword` | 修改密码 |

### 4.3 项目管理

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiListProjectIds()` | `/ProjectApi.listMyProjectIds` | 获取用户项目 ID 列表 |
| `apiListProjects(param)` | `/ProjectApi.listMyProject` | 分页获取项目列表 |
| `apiAddProject(name, desc, type, cover)` | `/ProjectApi.addProject` | 创建项目 |
| `apiDeleteProject(projectId)` | `/ProjectApi.delProject` | 删除项目 |
| `apiSetProjectPinned(projectId, pinned)` | `/ProjectApi.setProjectPinned` | 当前账号置顶／取消置顶项目 |
| `apiSetProjectInfo(projectId, name?, desc?, coverUrl?, canvas?)` | `/ProjectApi.setProjectInfo` | 修改项目信息（含画布数据）|
| `apiGetProjectInfo(projectId)` | `/ProjectApi.getProjectInfo` | 获取项目详情（含画布数据）|

### 4.4 模型

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiListModels(type)` | `/GenApi.listModel` | V1 模型列表 |
| `apiGetModelInfoList(type)` | `/GenApi.GetModelInfoList` | V2 模型列表（含 options/limit/logo）|

### 4.5 订单系统

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiCreateOrder(req, requestId)` | `/OrderApi.createAndPayOrder` | 仅创建订单 |
| `apiPayOrder(orderId, projectId, onChunk...)` | `/OrderApi.payOrder` | 支付订单（SSE 流式）|
| `apiCreateAndPayOrder(req, requestId)` | `/OrderApi.createAndPayOrder` | V1 创建并支付订单 |
| `apiCreateAndPayOrderV2(req, requestId)` | `/OrderApi.createAndPayOrderV2` | V2 创建并支付订单 |
| `apiGetAtc(type)` | `/GenApi.getAtc` | 模型平均耗时 |
| `apiBatchGetGenResult(requestIds[], projectId)` | `/OrderApi.batchGetGenResult` | 批量查询生成结果 |

### 4.6 素材与文件夹

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiListResultHistory(item, projectId, type)` | `/MaterialApi.listRecord` | 获取生成结果历史（游标分页）|
| `apiSearchRecord({projectId, keyword, type?, folderId?, page})` | `/MaterialApi.SearcRecord` | 按名称或提示词搜索生成资产；复用历史查询权限与返回结构，页码从 1 开始，每页 10 条 |
| `apiGetRecord(request)` | `/MaterialApi.getRecords` | 批量查询指定记录 |
| `apiFavorResult(projectId, resultId)` | `/MaterialApi.favorResult` | 收藏/取消收藏 |
| `apiUpdateRecord(record)` | `/MaterialApi.UpdateRecord` | 更新记录名称 |
| `apiCreateFolder(name)` | `/MaterialApi.CreateFolder` | 创建文件夹 |
| `apiUpdateFolder(id, name)` | `/MaterialApi.UpdateFolder` | 重命名文件夹 |
| `apiDeleteFolder(id)` | `/MaterialApi.DeleteFolder` | 删除文件夹 |
| `apiListFolders()` | `/MaterialApi.ListFolders` | 文件夹列表 |
| `apiAddRecordToFolder(recordIds, folderId)` | `/MaterialApi.AddRecordToFolder` | 记录移入文件夹 |
| `apiRemoveRecordFromFolder(resultIdByRecord, folderId)` | `/MaterialApi.RemoveRecordFromFolder` | 记录移出文件夹 |
| `apiListAssetIdsFromFolder(folderId, projectId)` | `/MaterialApi.GetAssetIdsFromFolder` | 文件夹内资产列表 |

### 4.7 素材资产 (VolcEngine)

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiGetAssetGroupId()` | `/MaterialApi.GetMaterialGroupId` | 获取素材组 ID |
| `apiCreateAsset(req)` | `/MaterialApi.CreateAsset` | 创建素材资产 |
| `apiListAssets(req)` | `/MaterialApi.ListAssets` | 查询资产列表 |
| `apiGetAsset(req)` | `/MaterialApi.GetAsset` | 获取资产详情 |
| `apiUpdateAsset(req)` | `/MaterialApi.UpdateAsset` | 更新资产 |
| `apiListMaterial(...)` | `/MaterialApi.ListMaterial` | 获取素材列表 |
| `apiGetMaterialStorage(type)` | `/MaterialApi.GetMaterialStorage` | 存储信息 |
| `apiUpdateMaterial(info)` | `/MaterialApi.UpdateMaterial` | 更新素材 |

### 4.8 聊天

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiCreateChat(classType?)` | `/Chat.CreateConversion` | 创建会话 |
| `apiGetChatList(pageNum?)` | `/Chat.ListConversations` | 历史会话列表 |
| `apiGetChatHistory(id, msgId, dir)` | `/Chat.ListMessages` | 消息历史 |
| `apiRenameChatConversation(id, title)` | `/Chat.RenameConversation` | 重命名会话 |
| `apiChangeConversationName(id, name)` | `/Chat.ChangeConversationName` | 修改会话名称 |
| `apiGenText(param, onChunk, onComplete, onError)` | `/GenApi.genText` | 流式文本生成（SSE，旧接口）|
| `apiCancelGenText(requestId)` | `/GenApi.cancelGenText` | 取消文本生成 |
| `apiChat(request, onChunk,...)` | `/chat` | 多模态聊天（SSE）|
| `apiIntentAnalyze(req)` | `/Chat.IntentAnalyze` | 意图分析 |
| `apiListSkills()` | `/Skill.List` | Flow 技能列表 |
| `apiFlowAnalyze(req)` | `/Chat.FlowAnalyze` | Flow 分析 |
| `apiFlowStep(req)` | `/Chat.FlowStep` | Flow 步骤 |
| `apiFlowState(req)` | `/Chat.FlowState` | Flow 状态 |

### 4.9 系统提示词

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiListSystemPrompt(params)` | — | 预设列表 |
| `apiAddSystemPrompt(params)` | — | 新增预设 |
| `apiUpdateSystemPrompt(params)` | — | 更新预设 |
| `apiDeleteSystemPrompt(params)` | — | 删除预设 |
| `apiListSystemPromptClass()` | — | 分类列表 |
| `apiAddSystemPromptClass(params)` | — | 新增分类 |
| `apiUpdateSystemPromptClass(params)` | — | 更新分类 |
| `apiDeleteSystemPromptClass(params)` | — | 删除分类 |

### 4.10 视频增强

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiEnhanceVideo(request)` | `/MediaApi.enhanceVideo` | 视频超分 |

### 4.11 BO 后台团队管理

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiAddMember(req)` | `/v1/TeamApi.addMember` | 添加成员 |
| `apiListMembers(req)` | `/v1/TeamApi.listMembersInfo` | 成员列表 |
| `apiGetMemberDetail(uId)` | — | 成员详情 |
| `apiGetMemberProjects(req)` | — | 成员项目列表 |
| `apiRemoveProjectMember(req)` | — | 移除项目成员 |
| `apiAddProjectMember(req)` | — | 添加项目成员 |
| `apiGetProjectList()` | `/v1/ProjectApi.List` | 项目列表 |

### 4.12 BO 后台账单与消费

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiGetTeamConsumeAnalysis(req)` | `/v1/BillApi.GetTeamConsumeAnalysis` | 团队消耗分析 |
| `apiGetProjectBillStat(projectId)` | `/v1/BillApi.GetProjectBillStat` | 项目账单统计 |
| `apiGetProjectBillList(req)` | `/v1/BillApi.GetProjectBillList` | 项目账单流水 |
| `apiGetBalance()` | `/WalletApi.Check` | 查询余额 |
| `apiListBillDetail(params)` | `/bos/rootApi.listBillDetail` | 账单明细列表 |
| `apiListBillDetailSummary(params)` | — | 账单明细汇总 |
| `apiListBillDetailDownload(params)` | — | 账单明细下载（返回 COS CSV 地址）|
| `apiListBillsWithinRecentDays(days)` | — | 近 N 天的账单趋势 |
| `apiListUserBillDetail(params)` | — | 用户账单明细 |
| `apiGetBalanceLog(req)` | — | 余额变动记录 |

### 4.13 BO 后台客户管理

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiCreateSubAgent(req)` | `/v1/SubAgentApi.createSubAgent` | 创建客户 |
| `apiListSubAgents(page, pageSize)` | `/v1/SubAgentApi.listSubAgents` | 客户列表 |
| `apiSetDiscount({subAgentId, modelDiscounts})` | `/v1/SubAgentApi.setDiscount` | 设置折扣 |
| `apiGetSubAgentStats(timePeriod)` | `/v1/SubAgentApi.getStats` | 客户统计 |
| `apiSubAgentRecharge(req)` | `/v1/SubAgentApi.recharge` | 客户充值 |
| `apiListRechargeRecords({page,pageSize,subAgentId})` | `/v1/SubAgentApi.listRechargeRecords` | 充值记录 |

### 4.14 其他

| 函数 | 路由 | 说明 |
|------|------|------|
| `apiGetGenSign(bizParams)` | `/api/getSign` | 获取直连签名 |
| `apiUploadToCos(file, onProgress)` | COS SDK | 上传文件到 COS |
| `apiSplitNovel(req)` | — | 小说角色拆分 |

---

## 五、核心类型定义

### 5.1 响应基础 (`SimpleResponse`)

```typescript
interface ReplyHeader {
    code: number;       // 12000=成功, 14001/14002=鉴权, 14003=警告, 20001=强制登出
    message: string;
    session?: string;   // Token 刷新凭据
}
interface SimpleResponse { replyHeader: ReplyHeader }
interface ApiResponse<T> extends SimpleResponse { data: T }
```

### 5.2 GenResultVo（生成结果）

```typescript
interface GenResultVo {
    userInfo: UserInfoSummary;
    projectId: string;
    requestId: string;
    orderId: string;
    isFavor: boolean;
    resolution: string;
    resultId: string;
    type: number;         // 1-文字 2-图片 3-视频 4-超分 5-音频
    result: string;       // 结果 URL
    status: number;       // 订单状态
    modelId: number;
    prompt: ContentItem[];
    referenceImg: ResItemVo[];    // 注意字段名为 referenceImg
    referenceVideo: ResItemVo[];
    referenceAudio: ResItemVo[];
    param: string;        // 生成参数 JSON
    func: string;         // 功能参数 JSON
    seed: number;
    tc: number;           // 生成耗时(秒)
    cost: number;         // 消费金额(万分之二元)
    clientParams: string; // 透传前端参数 JSON
    createTime: number;   // 创建时间(毫秒)
}
```

### 5.3 ResItemVo（资源引用项）

```typescript
interface ResItemVo {
    id: string;
    type: number;          // 1-文字 2-图片 3-视频 4-超分 5-音频 6-主体
    title: string;
    description: string;
    referImage: string;
    referVideo: string;
    referAudio: string;
}
```

### 5.4 CreateOrderV2Req（V2 订单参数）

```typescript
interface CreateOrderV2Req {
    projectId: string;
    id: number;            // 模型 ID
    params: string;        // JSON 字符串
    clientParams?: string; // JSON 字符串（透传前端参数）
}

interface CreateOrderV2Params {
    prompt?: ContentItem[];
    referImage?: string[];
    referImageRole?: string[];
    referVideo?: string[];
    referVideoRole?: string[];
    referAudio?: string[];
    referAudioRole?: string[];
    [key: string]: string | number | boolean | ContentItem[] | string[] | undefined;
}
```

### 5.5 ModelInfoV2（V2 模型信息）

```typescript
interface ModelInfoV2 {
    id: number;
    title: string;         // 显示名称（用户友好）
    name: string;          // 模型名称
    version: string;
    description: string;
    logo: string;
    type: number;          // 生成类型编码
    limit: string;         // JSON: {MaxText, MaxAudio, MaxImage, MaxVideo}
    options: string;       // JSON 参数选项数组
}
```

### 5.6 ContentItem（提示词内容）

```typescript
interface ContentItem {
    type: 'text';
    content: {
        id: string;
        label: string;
    };
}
```

### 5.7 UserInfo（用户信息）

```typescript
interface UserInfo {
    uId: string;
    userName: string;
    nickName: string;
    userRole: UserRoleType;  // 'root' | 'admin' | 'team_admin' | 'user' | 'guest'
    tId?: string;
    auth?: string;
    avatarUrl?: string;
}
```

### 5.8 枚举与常量

```typescript
// 生成结果类型
const GEN_RESULT_TYPE = { TEXT: 1, IMAGE: 2, VIDEO: 3, VIDEO_ENCH: 4, AUDIO: 5 } as const;

// 资源项类型
const RES_ITEM_TYPE = { TEXT: 1, IMAGE: 2, VIDEO: 3, VIDEO_ENCH: 4, AUDIO: 5, SUBJECT: 6 } as const;

// 视频生成模式
const GEN_CLASS_TYPE = {
    TEXT_TO: 1,       // 文生视频
    IMG_TO: 2,        // 参考图生视频
    KEYFRAME: 3,      // 关键帧
    REFER: 4,         // 智能参考（默认）
    ACTION_CTRL: 5,   // 动作控制
    MOTION_BRUSH: 6,  // 运动画笔
    CAMERA_CTRL: 7,   // 镜头控制
} as const;

// 订单类型
const GEN_ORDER_TYPE = {
    TEXT: 1, IMAGE: 2, VIDEO: 3, ENHANCE_VIDEO: 4
} as const;

// t_app_info type 编码规则
// 千位=输出类型(1文字/2图片/3视频/4超分)
// 百位=支持文字输入(1是)
// 十位=支持图片输入(1是)
// 个位=支持多媒体输入(1是)
// 例：1100=纯文字生成，3111=视频+文字+图片+多媒体输入
```

### 5.9 项目类型

```typescript
const ProjectType = {
    STORY: 'story',
    SCRIPT: 'script',
    CANVAS: 'canvas',
    NORMAL: 'normal',
    BATCH: 'batch',
    CHAT: 'chat',
} as const;

// 项目类型配置
const PROJECT_TYPE_LIST = [
    { type: 'normal', label: '普通项目', emoji: '📁', route: '/CtrlMain', color: '#555E66', available: true },
    { type: 'chat', label: '对话项目', emoji: '💬', route: '/ChatMain', color: '#635598', available: true },
    { type: 'script', label: '剧本项目', emoji: '🎬', route: '/ScriptMain', color: '#559877', available: true },
    { type: 'canvas', label: '画布项目', emoji: '🎨', route: '/CanvasMain', color: '#467A9F', available: true },
    { type: 'story', label: '故事项目', emoji: '📖', route: '/StoryMain', color: '#351b92', available: false },
    { type: 'batch', label: '批处理项目', emoji: '⚡', route: '/BatchMain', color: '#9F8746', available: false },
];
```

### 5.10 订单/账单类型

```typescript
interface CreateOrderReq {
    projectId: string;
    id: number;           // 模型 ID
    params: string;       // JSON 字符串
    clientParams?: string;
    requestId?: string;
    skeletonId?: string;
}

interface BillDetailRecordVo {
    createTime: number;
    productName: string;
    modelName: string;
    unit: number;
    cost: string;         // 万分之二元
    unitPrice: string;    // 单价(万分之二元/单位)
    discount: string;     // 折扣率(1=100%)
    originPrice: string;  // 折前价(万分之二元)
    finalPrice: string;   // 折后价(万分之二元)
    userInfo: UserInfoSummary;
    type: number;         // 1-文字 2-图片 3-视频 4-超分 5-音频
    result?: string;      // 结果 URL
}
```

---

## 六、包详解：`@lumina/core`（共享核心）

**位置**：`packages/core/src/`

### 6.1 导出总览

| 类别 | 数量 | 内容 |
|------|------|------|
| API 函数 | 62 个 | `apiLogin` ~ `apiSplitNovel` |
| 请求工具 | 4 个 | `postRequest`, `postChunkedRequest`, `postFormData`, `generateRequestId` |
| 类型接口 | 30+ | `GenResultVo`, `CreateOrderV2Req`, `ModelInfoV2`, `UserInfo` 等 |
| React 组件 | 7 个 | `MainLayout`, `LeftMenu`, `VersionUpdateBar`, `CustomModal`, `CustomPopover`, `CustomDropdown`, `EditableName` |
| React Context | 4 个 | `FavorContext`, `CanvasContext`, `CtrlMainContext`, `InputParamsContext` |
| 工具函数 | 20+ | `panelStorage`, `modelCache`, `userStorage`, `mediaType`, `timestampFormat` 等 |
| Mock API | 5 个 | `fetchModelPriceList`, `mockWorkflowCanvas` 等 |
| 常量 | 10+ | `ProjectType`, `GEN_CLASS_TYPE`, `BASE_PATH` 等 |

### 6.2 核心 API 文件结构

```
packages/core/src/api/
├── index.ts            # 62 个 API 函数导出
├── request.ts          # 请求引擎（postRequest / postChunkedRequest / postFormData）
├── base/
│   └── simpleReply.ts  # SimpleResponse / ApiResponse 基础类型
├── modules/            # 旧版模块化类型定义（被 index.ts 替代）
│   ├── favor.ts        # 收藏 + 文件夹类型
│   ├── genResult.ts    # GenResultVo + ResItemVo
│   ├── order.ts        # 订单类型 + GEN_CLASS_TYPE
│   └── ...
└── mock/
    ├── billOverview.ts  # 账单概览模拟数据
    ├── consumeOverview.ts
    ├── modelPrice.ts
    └── subAgent.ts
```

### 6.3 组件详解

| 组件 | 路径 | 说明 |
|------|------|------|
| `MainLayout` | `components/Layout/` | 主布局（LeftMenu + Outlet + VersionUpdateBar）|
| `LeftMenu` | `components/LeftMenu/` | 左侧图标导航菜单 |
| `VersionUpdateBar` | `components/VersionUpdateBar/` | 底部版本检测提示条 |
| `CustomModal` | `components/CustomModal/` | 通用 Modal 封装（深色主题）|
| `CustomPopover` | `components/CustomPopover/` | Popover 封装 |
| `CustomDropdown` | `components/CustomDropdown/` | Dropdown 封装 |
| `EditableName` | `components/EditableName/` | 可编辑名称组件 |
| `ImageItem` / `VideoItem` | `components/material/` | 素材缩略图组件 |

### 6.4 Context

| Context | 文件 | 说明 |
|---------|------|------|
| `FavorContext` | `contexts/FavorContext.tsx` | 收藏状态管理（FavorProvider + useFavorContext）|
| `CanvasContext` | `contexts/CanvasContext.tsx` | 画布项目余额 + 结果历史（双 Reducer）|
| `InputParamsContext` | `contexts/InputParamsContext.tsx` | 输入参数共享 |
| `CtrlMainContext` | `components/Data/` | CtrlMain 出口上下文类型定义 |

### 6.5 工具函数

| 模块 | 函数 | 说明 |
|------|------|------|
| `panelStorage.ts` | `savePanelPrompt / loadPanelPrompt / clearPanelPrompt` | Prompt 持久化 |
| | `savePanelItems / loadPanelItems / clearPanelItems` | 资源列表持久化 |
| | `savePanelParams / loadPanelParams / clearPanelParams` | 参数持久化 |
| | `saveConversationId / loadConversationId / clearConversationId` | 会话 ID |
| | `saveSplitterSizes / loadSplitterSizes` | Splitter 尺寸 |
| | `saveCountdownData / loadCountdownData / clearCountdownData` | 倒计时（sessionStorage）|
| | `savePanelSystemPrompt / loadPanelSystemPrompt / clearPanelSystemPrompt` | 系统提示词 |
| `modelCache.ts` | `getModelListCacheV2 / setModelListCacheV2 / removeModelListCacheV2` | 模型缓存（8h TTL）|
| | `startModelRefreshTimer` | 30分钟定时刷新 |
| `userStorage.ts` | `getLoginUserInfo / saveLoginUserInfo / clearLoginUserInfo / getCurrentUserId` | 用户信息 |
| `mediaType.ts` | `isImage / isVideo / isAudio / getMediaType` | 媒体类型判断 |
| `SnowflakeIdGenerator.ts` | `generateId()` | 128 位雪花 ID（时间戳64+机器ID16+序列号16+随机数32）|
| `timestampFormat.ts` | `formatTimestamp(ts, format)` | 时间戳格式化 |
| `url.ts` | `ensureHttps` | URL 协议规范化 |
| `translateErrorText.ts` | `translateErrorText(msg)` | 错误信息中文翻译 |

### 6.6 本地存储 Key 规范

```
面板持久化: __panel_{uId}_{category}_{panelId}
用户信息:   {uId}-userInfo
模型缓存:   __model_list_v2_{token}
预填数据:   __prefillData__ (sessionStorage)
倒计时:     __countdown_{id} (sessionStorage)
Chunk 重试: __chunk_reload__ (sessionStorage)
```

---

## 七、包详解：`@lumina/ctrl-main`（创作控制台）

**位置**：`packages/ctrl-main/src/`

### 7.1 页面结构

```
CtrlMain 三栏布局
├── LeftMenu（左侧图标导航）
├── Splitter（可拖拽分栏）
│   ├── 左侧面板 (Outlet 子路由)
│   │   ├── video → VideoCtrlPanel（生视频）
│   │   ├── image → ImageCtrlPanel（生图）
│   │   └── text → TextCtrlPanel（文字优化/对话）
│   └── 中间面板 (SimpleChatPanel，可折叠)
└── ResultPanel（右侧结果面板）
```

### 7.2 文件结构

```
packages/ctrl-main/src/pages/
├── index.tsx              # CtrlMain 三栏布局 + 状态管理
├── assets/                # 图片资源
├── components/            # 共享组件
│   ├── AssetPanel/        # 资产库（AssetToolbar + AssetGrid + AssetUploader）
│   ├── AssetLibraryPanel/ # 素材库（MaterialLibraryPanel + MaterialCard）
│   ├── InputContain/      # TagTextArea 富文本编辑器（TipTap + @mention）
│   ├── NavMenu/           # 顶部导航 Menu
│   ├── ParamsContain/     # 动态参数面板（RatioIconGroup）
│   ├── PreViewComponent/  # 预览组件（ItemPreviewComponent）
│   ├── ResultItem/        # 结果卡片
│   ├── SimpleChatPanel/   # 聊天侧边栏
│   └── ThumbComponent/    # 缩略图导航
├── hooks/                 # 自定义 Hooks
│   ├── usePanelCommonLogic.ts  # 图片/视频面板共享逻辑
│   ├── useSkeletonPolling.ts   # 骨架屏轮询（5s 间隔）
│   ├── useResultHistory.ts     # 结果历史加载 + 滚动分页
│   ├── useClone.ts             # "做同款"预填逻辑
│   ├── useEnhance.ts           # 视频超分逻辑
│   └── useGenAgain.ts          # 重新生成逻辑
├── utils/                 # 工具函数
├── VideoCtrlPanel/        # 视频生成面板
│   └── index.tsx
├── ImageCtrlPanel/        # 图片生成面板
│   └── index.tsx
├── TextCtrlPanel/         # 文字优化/对话面板
│   ├── index.tsx
│   ├── ChatInputBar.tsx
│   ├── ChatMessageArea.tsx
│   └── ChatHistorySidebar.tsx
└── ResultPanel/           # 结果展示面板
    └── index.tsx
```

### 7.3 VideoCtrlPanel（生视频）

- 路由：`/CtrlMain/:projectId/video`
- 4 种生成模式（SwitchComponent）：
  - 智能参考（REFER, classType=4）— 默认
  - 首尾帧（KEYFRAME, classType=3）— 双 ItemPreviewComponent
  - 文生视频（TEXT_TO_VIDEO, classType=1）
  - 参考图生视频（IMG_TO, classType=2）
- 下拉选择模型 + 动态参数（ParamsContain）
- 生成音频 Checkbox
- clientParams 结构：
```json
{
    "classType": 4,
    "prompt": [{"type":"text","content":{"id":"0","label":"提示词"}}],
    "referImage": [{"id":"图1","type":2,"title":"图1","referImage":"url..."}],
    "referVideo": [],
    "referAudio": [],
    "resolution": "480p",
    "duration": 4,
    "ratio": "16:9"
}
```

### 7.4 ImageCtrlPanel（生图）

- 路由：`/CtrlMain/:projectId/image`
- 2 种子模式：参考生图（refer2Image）、文生图（text2Image）
- 参考图片上传、@mention 标签引用
- 模型选择 + 动态参数
- 一键清空按钮

### 7.5 TextCtrlPanel（文字对话）

- 路由：`/CtrlMain/:projectId/text`
- 聊天式对话界面：
  - `ChatHistorySidebar` — 历史会话列表
  - `ChatInputBar` — 输入栏（模型选择、侧边栏切换、新建会话、中断发送）
  - `ChatMessageArea` — 消息展示区（流式显示）
- 支持 SSE 流式文本生成 + 多模态图文混合消息
- 会话 ID 持久化（按用户+项目隔离）
- 系统提示词编辑 Modal

### 7.6 ResultPanel（结果面板）

- 瀑布流展示生成结果（视频/图片/文字）
- 滚动分页加载（`useResultHistory`，游标分页）
- 筛选类型切换（全部/图片/视频/文字）
- 收藏管理（`FavorContext`）
- 骨架屏占位 + 实时倒计时
- 结果操作：下载、收藏、做同款、超分、重新生成
- 预览弹窗（`PreViewResultItem`）

### 7.7 ParamsContain（动态参数面板）

- 根据模型 `options` JSON 动态渲染 UI 控件：
  - Segmented（分段选择）
  - Slider（滑块）
  - Select（下拉选择）
  - RatioIconGroup（宽高比图标选择）
- 支持做同款回填
- 参数值持久化到 localStorage

### 7.8 ItemPreviewComponent（资源预览）

- 拖拽/上传图片、视频、音频到预览区域
- 自动生成 @mention 标签编号
- 各类型数量限制（由模型 limit 控制）
- 支持素材库拖拽

### 7.9 生成流程

1. 用户输入 prompt + 选择模型 + 参数
2. 构建 `clientParams` → 调用 `apiCreateAndPayOrderV2`
3. 前端添加骨架屏项，倒计时持久化到 sessionStorage
4. 每 5s 调用 `apiBatchGetGenResult` 轮询
5. 收到结果后替换骨架屏，刷新余额
6. 失败时 catch 块统一处理错误

---

## 八、包详解：`@lumina/canvas-main`（画布工作流）

**位置**：`packages/canvas-main/src/`

### 8.1 架构

```
CanvasMain 页面
├── CanvasProvider（余额 + 结果历史 Context）
└── CanvasMainContent
    ├── Edge 浏览器提示横幅
    ├── 项目名称（左上）
    ├── 余额显示（右上）
    ├── FloatButtonView（左侧悬浮按钮组）
    └── WorkflowBuilder（核心画布组件）
```

### 8.2 WorkflowBuilder

基于 `@xyflow/react` (React Flow) 构建的可视化工作流编辑器，~65.5 KB。

**核心功能：**
- ReactFlow 渲染节点和边，支持贝塞尔曲线连线
- 节点操作：单选 / 多选 / 框选 / 全选（Ctrl+A）
- 快捷键：Ctrl+C 复制 / Ctrl+V 粘贴 / Delete 批量删除
- 右键菜单（NodeContextMenu）
- 连线操作：点击连线显示剪刀图标删除
- 悬浮按钮组：添加节点、返回项目页、素材库
- 自动保存：output 变化 → 防抖 2s + 节流 10s

### 8.3 节点类型系统

| 节点类型 | 组件 | 功能 | 状态 |
|---------|------|------|------|
| `imageNode` | `ImageNodeWithBaseNode` | 文生图 / 参考生图 | ✅ 真实 API |
| `videoNode` | `VideoNodeWithBaseNode` | 文生视频 | ✅ 真实 API |
| `textNode` | `TextNodeWithBaseNode` | AI 文本生成 | ✅ |
| `outputTextNode` | `OutputTextNodeWithBaseNode` | 纯输出文本 | ✅ |
| `uploadNode` | `UploadNodeWithBaseNode` | 上传节点（图片/视频/音频） | ✅ |
| `novelNode` | `NovelNodeWithBaseNode` | 小说输入 | ✅ 新增 |
| `characterListNode` | `CharacterListNodeWithBaseNode` | 角色列表 | ✅ 新增 |
| `characterDescNode` | `CharacterDescNodeWithBaseNode` | 角色描述 | ✅ 新增 |

### 8.4 BaseNode 基础组件

- 三段式布局：Screen（屏幕）+ Keyboard（键盘）+ Handle（连接点）
- 屏幕与键盘互斥显示：hover 显示连接点、点击屏幕展开键盘
- 节点大小可拖拽调整（最小 250px），三方向缩放
- 连接点跟随鼠标偏移（磁性吸附效果，最大 20px）
- 拖拽连接点到空白处弹出节点类型选择器

### 8.5 文件结构

```
packages/canvas-main/src/components/
├── WorkflowBuilder/
│   ├── WorkflowBuilder.tsx     # 核心画布组件 (65.5 KB)
│   ├── types.ts                # 类型定义
│   ├── constants/
│   │   └── nodeTypes.tsx       # 节点类型配置
│   ├── components/
│   │   └── ParamsContain/      # 动态参数面板（含 RatioIconGroup）
│   ├── contexts/
│   │   └── SelectionContext.tsx # 框选状态
│   ├── hooks/
│   │   ├── useUpstreamData.ts  # 获取上游节点数据
│   │   └── useUpstreamCharacters.ts # 上游角色数据
│   ├── nodes/
│   │   ├── BaseNode.tsx        # 通用基础节点 (38.6 KB)
│   │   ├── ImageNodeWithBaseNode.tsx
│   │   ├── VideoNodeWithBaseNode.tsx
│   │   ├── TextNodeWithBaseNode.tsx
│   │   ├── OutputTextNodeWithBaseNode.tsx
│   │   ├── UploadNodeWithBaseNode.tsx
│   │   ├── NodeKeyboard.tsx    # 键盘输入面板
│   │   ├── ModelSelect.tsx     # 模型选择
│   │   ├── ModelParamsSelector.tsx
│   │   ├── css/                # 6 个 CSS 样式文件
│   │   └── view/               # 视图层组件
│   │       ├── BaseNodeView.tsx
│   │       ├── ImageNodeView.tsx (含 Redraw 重绘)
│   │       ├── VideoNodeView.tsx
│   │       ├── TextNodeView.tsx
│   │       └── novel/          # 小说角色节点
│   │           ├── NovelNodeView.tsx
│   │           ├── CharacterListNodeView.tsx
│   │           └── CharacterDescNodeView.tsx
│   └── ui/
│       ├── ActionCard.tsx      # 节点类型选择卡片
│       ├── EdgeScissorsButton.tsx # 连线剪刀按钮
│       ├── FloatButtonView.tsx # 悬浮按钮组
│       ├── ImageMosaic.tsx     # 马赛克编辑器 (fabric.js)
│       ├── MultiNodeSelector.tsx # 多节点操作面板
│       └── NodeContextMenu.tsx # 右键菜单
└── InputContain/               # 富文本编辑器（TagTextArea）
    └── TagTextArea/
        ├── index.ts
        ├── TagTextArea.tsx
        ├── buildDocJson.ts
        ├── helpers.ts
        ├── parseEditorContent.ts
        ├── styles.ts
        ├── types.ts
        └── useTagEditor.ts
```

### 8.6 关键特性

- **多节点框选**：框选后显示 SelectionOverlay + 浮动面板（生图/生视频）
- **重绘模式**：ImageMosaic 马赛克编辑器（fabric.js），支持涂抹/擦除/撤销/重做
- **自动保存**：output 变化、页面隐藏时自动保存
- **模型选择**：集成 `ParamsContain` 动态参数面板
- **@mention 标签**：引用上游节点数据
- **连接类型约束**：不同类型节点限制可连边类型

---

## 九、包详解：`@lumina/chat-main`（对话项目）

**位置**：`packages/chat-main/src/`

### 9.1 架构

```
ChatMain
├── 三栏布局（antd Splitter）
│   ├── 左栏：StoryPanel（故事板）
│   ├── 中栏：PreviewPanel（预览区）+ FilePanel（文件资源）
│   └── 右栏：DialogView（对话面板）
└── 导航标签：对话 / 文件区 / 剪辑框 / 故事板
```

### 9.2 文件结构

```
packages/chat-main/src/
├── index.tsx                     # 页面入口 + 三栏布局
├── modules/
│   ├── DialogView/              # 对话面板（主组件）
│   │   ├── index.tsx            # 聊天界面
│   │   ├── ChatMessages.tsx     # Store 驱动的消息列表
│   │   ├── ChatInput.tsx        # 输入栏 + 技能选择器
│   │   ├── ContentBlockRenderer.tsx # 5 种内容块渲染
│   │   ├── componentMap.ts      # 13 个组件映射
│   │   └── InteractiveFlowUI.tsx # Flow 交互 UI
│   ├── StoryPanel/              # 故事板面板
│   ├── PreviewPanel/            # 预览面板
│   ├── FilePanel/               # 文件资源面板（DirectoryTree）
│   ├── RecommendationCards/     # 推荐卡片
│   └── flowConfig.ts            # Flow 步骤配置
├── stores/
│   └── chatFlowStore.ts         # zustand store（会话/消息/Flow 状态）
└── types.ts                     # FlowResponse / AgentMessage 类型
```

### 9.3 核心流程

1. 用户输入 → `apiIntentAnalyze` 意图分析
2. 图片意图 → `confirm_intent` → 选择模型 → `editPrompt` → 选比例 → 选分辨率 → 生成
3. 聊天意图 → 普通对话（SSE 流式 `apiChat`）
4. Flow 技能：支持 8 个剧情短片 Skill 组件

---

## 十、应用详解：`@lumina/bo-app`（运营后台）

**位置**：`apps/bo/src/`；Vite base 与 Router basename 均为 `/bo/`，独立输出到 `dist-bo/`。

### 10.1 页面路由

| 路由 | 组件 | 说明 |
|------|------|------|
| `/bo/home` | BillOverview | 账单总览 Dashboard |
| `/bo/billOverview` | BillOverview | 同上 |
| `/bo/consumeOverview` | ConsumeOverview | 团队消耗分析（team_admin 首页）|
| `/bo/detailBill` | BillDetail | 账单明细 |
| `/bo/summyBill` | SummyBill | 团队消耗汇总 |
| `/bo/memberSummyBill` | MemberConsumption | 成员消耗汇总 |
| `/bo/addMember` | AddMember | 添加成员 |
| `/bo/listMember` | ListMember | 成员列表 |
| `/bo/memberDetail/:id` | MemberDetail | 成员详情 |
| `/bo/project` | BoProject | 项目管理 |
| `/bo/addCustomer` | AddCustomer | 新增客户 |
| `/bo/listCustomer` | ListCustomer | 客户列表 |
| `/bo/CustomerDetail/:tId` | CustomerDetails | 客户详情 |
| `/bo/subAgentData` | SubAgentData | 客户数据中心 |

### 10.2 文件结构

```
apps/bo/src/pages/
├── BillOverview/       # 账单总览（全功能 Dashboard，手写 SVG 图表）
│   ├── index.tsx       # 统计卡片 + 趋势图 + 饼图
│   └── components/     # AreaLineChart, SimpleBarChart, DonutChart 等
├── ConsumeOverview/    # 团队消耗分析（真实 API）
│   ├── index.tsx       # 卡片 + 多线折线图 + 堆积图
│   └── components/     # MultiLineChart, StackedBarChart
├── Bill/               # 账单明细
│   ├── index.tsx       # BillDetail（表格 + 筛选 + 下载）
│   ├── SummyBill.tsx   # 团队汇总
│   └── MemberConsumption.tsx # 成员消耗汇总
├── Team/               # 团队管理
│   ├── addMember.tsx
│   ├── listMember.tsx
│   └── memberDetail.tsx
├── Custom/             # 客户管理
│   ├── AddCustomer.tsx # 新增客户（CreateCustomerCard）
│   ├── ListCustomer.tsx# 客户列表
│   ├── CustomerDetails.tsx
│   ├── SubAgentData.tsx# 客户数据中心
│   └── components/    # DiscountSetting, UsageStatsCard
└── Project/            # 项目管理
    └── index.tsx       # 项目列表
```

---

## 十一、其他包

### 11.1 `@lumina/script-main`（剧本项目）

- 路由：`/ScriptMain/:projectId`
- 组件：`WelcomeScreen`, `EpisodePanel`, `StoryTab`, `StoryToScriptPanel`, `AssetLibraryModal`, `ProjectConfigModal`
- AI 四阶段分析：角色→场景→片段切分→剧本转换
- 小说角色拆分（API: `apiSplitNovel`）

### 11.2 `@lumina/login`（登录）

- 路由：`/`、`/login`
- 账号密码 → MFA 验证
- 开发后门账号 `1234`（跳过 checkAccount，直接 MFA）
- 首次登录自动初始化所有面板默认值
- COS 直连上传

### 11.3 `@lumina/project-summary`（项目汇总）

- 路由：`/projectSummary`
- 项目卡片网格（ProjectItem）
- 项目 CRUD（创建/删除/重命名/换封面）
- 项目类型筛选
- 右上角用户菜单（修改密码/退出登录）

### 11.4 `@lumina/console`（控制台）

- 路由：`/Console`
- 占位页面，"正在建设中"

### 11.5 `@lumina/guide`（使用指南）

- 路由：`/guide`
- iframe 嵌入飞书文档
- 左侧目录切换

---

## 十二、数据持久化

### 12.1 panelStorage API

| 存储类型 | Key 前缀 | 函数 | 作用域 |
|---------|---------|------|--------|
| Prompt 文本 | `__prompt_` | `savePanelPrompt / loadPanelPrompt / clearPanelPrompt` | localStorage |
| 资源列表 | `__images_ / __videos_ / __audios_` | `savePanelItems / loadPanelItems / clearPanelItems` | localStorage |
| 模型参数 | `__params_` | `savePanelParams / loadPanelParams / clearPanelParams` | localStorage |
| 系统提示词 | `__systemPrompt_` | `savePanelSystemPrompt / loadPanelSystemPrompt / clearPanelSystemPrompt` | localStorage |
| 聊天会话 ID | `__conversation_` | `saveConversationId / loadConversationId / clearConversationId` | localStorage |
| Splitter 尺寸 | `__splitter_` | `saveSplitterSizes / loadSplitterSizes` | localStorage |
| 倒计时 | `__countdown_` | `saveCountdownData / loadCountdownData / clearCountdownData` | sessionStorage |
| 预填数据 | `__prefillData__` | 直接读写 sessionStorage | sessionStorage |

面板 ID 列表：`text`, `image2Image`, `text2Image`, `image2Video`, `keyFrame`, `txt2Video`

### 12.2 模型缓存

- Key：`__model_list_v2_{tokenHash}`（localStorage）
- TTL：8 小时
- 登录时加载 + 每 30 分钟静默刷新
- `getModelListCacheV2` 返回空数组替代 null，简化调用方

### 12.3 用户信息

- Key：`{uId}-userInfo`（localStorage）
- 登录时 `localStorage.clear()` 后存储
- 退出登录清除，保留 conversationId

---

## 十三、开发规范

### 13.1 编码原则（必须遵守）

1. **不要修改架构**：沿用现有模块划分、状态管理、数据流及调用关系，不借功能开发、修复或清理之机擅自调整架构。
2. **不要新造逻辑**：先查找现有业务流程和处理入口，在原有流程中完成需求；不重复实现已有能力，不另建一套连接、校验、协作锁、保存等处理流程。
3. **尽量复用代码**：优先复用现有组件、函数、样式和公共工具，只保留需求必需的差异；避免复制实现，也避免为复用而引入不必要的抽象或通用框架。

### 13.2 原则冲突必须先询问用户

需求、实现方案或其他规范与上述任一原则发生冲突，或无法确定是否冲突时，**必须在实施相关代码修改前询问用户，并等待用户明确同意**。

询问时应说明冲突的原则、现有实现为何无法满足需求、拟采用的最小改动方案及影响范围。未经用户对该冲突的明确同意，不得自行例外处理，也不得先改完再补问；用户未回复不代表同意。功能可用、测试通过、性能优化或代码精简均不能替代冲突确认。

### 13.3 具体开发约定

1. **路由 basename**: 当前为 `''`（空字符串），`BASE_PATH` = `/CtrlMain`
2. **菜单 key 解析**: `pathname.split('/')[2]`（index 2），因第一段为空字符串
3. **面板持久化**: 使用 `panelStorage.ts`，按用户隔离
4. **模型缓存**: localStorage + 8h TTL + token 绑定
5. **ResItemVo 字段名称**: 使用 `referImage`/`referVideo`/`referAudio`（不再使用 `referenceImage`）
6. **API 错误码**: 12000 成功，14001/14002 鉴权，14003 业务警告，20001 强制登出
7. **用户信息存储**: `{uId}-userInfo`，登录时 `localStorage.clear()`
8. **收藏管理**: 使用 `FavorContext`（ref + version 模式，避免闭包过期）
9. **生成订单**: 使用 V2 接口 (`apiCreateAndPayOrderV2`)，`params` 和 `clientParams` 均为 JSON 字符串
10. **版本号规则**: 只升小版本号（patch），不升大/中版本号
11. **React Compiler**: 启用 `babel-plugin-react-compiler`，需遵循其规则（useCallback/useMemo 仍可用但不必须）
12. **自动导入**: `react` 和 `react-router-dom` 由 `unplugin-auto-import` 自动注入，无需手动 import

---

## 十四、构建配置

### 14.1 Vite 代理

| 应用 | 前缀 | 目标 | 说明 |
|------|------|------|------|
| Muse | `/v2`、`/api` | `106.52.172.123:8090` | 主 API |
| Muse | `/v3` | `localhost:8091` | Agent SSE API |
| BO | `/bo/v1` | `localhost:8080` | BO 后台 API |
| BO | `/bos` | `localhost:8080` | rootBoServer API |
| BO | `/v2`、`/api` | `localhost:8082` | 共享主 API |

### 14.2 构建插件

| 插件 | 说明 |
|------|------|
| `@vitejs/plugin-react` | React 编译 + babel-plugin-react-compiler |
| `unplugin-auto-import` | 自动导入 react + react-router-dom |
| `vite-plugin-compression2` | gzip 压缩（≥10KB）|
| `cspPlugin` | 环境变量 `VITE_ENABLE_HTTPS=true` 时注入 CSC 头 |
| `versionJsonPlugin` | 构建时生成 version.json（版本-分支-日期-hash）|

### 14.3 输出结构

```
dist/
├── assets/
│   ├── vendor-react.[hash].js
│   ├── vendor-tiptap.[hash].js
│   ├── vendor-antd.[hash].js         # ~514KB gzip
│   ├── vendor-dayjs.[hash].js
│   ├── vendor-cos.[hash].js
│   ├── vendor-icons.[hash].js        # lucide-react + @tabler/icons-react
│   ├── vendor-xyflow.[hash].js
│   ├── pkg-core.[hash].js            # core + login
│   ├── pkg-ctrl-main-utils.[hash].js
│   ├── pkg-ctrl-main-components.[hash].js
│   ├── pkg-ctrl-main-hooks.[hash].js
│   ├── pkg-ctrl-main-text.[hash].js
│   ├── pkg-ctrl-main-video.[hash].js
│   ├── pkg-ctrl-main-image.[hash].js
│   ├── pkg-canvas-main-wf.[hash].js  # WorkflowBuilder (大块)
│   ├── pkg-canvas-main-input.[hash].js
│   └── ...                           # 其他自动拆分 chunk
└── version.json                      # 构建版本信息

dist-bo/
├── index.html                        # `/bo/` 独立入口
└── assets/                           # BO 页面与按路由拆分的 chunk
```

---

## 十五、已知问题与注意事项

### 15.1 安全风险

- ⚠️ `packages/core/src/api/modules/cos.ts` 中硬编码了腾讯云 COS SecretId/SecretKey
- **建议**：通过后端接口获取 STS 临时密钥，前端不存储永久密钥

### 15.2 路由配置

- `BASE_PATH = '/CtrlMain'`，`BASE_NAME` 为空字符串
- `extractProjectId` 支持 `CtrlMain`、`ScriptMain`、`ChatMain`，`CanvasMain` 用 `useParams`

### 15.3 数据持久化

- localStorage 容量上限（5-10MB），写入时已做 `try-catch`
- 退出登录不清除 `conversationId`
- sessionStorage 用于倒计时（刷新不丢失，关标签页清理）和 Chunk 重试标记

### 15.4 wallet 余额

- 已对接真实余额接口 `apiGetBalance()` → `/WalletApi.Check`
- 画布项目通过 `CanvasContext` 获取余额
- CtrlMain 通过 `refreshBalance` 回调获取

### 15.5 vendor-antd 体积

- `vendor-antd` chunk 约 514KB gzip
- 其中约 250KB 来自 `@ant-design/icons`（2000+ 图标）
- CDN 方案不可行（无 tree-shaking，更大）
- 独立拆分 icons 会引发循环依赖（antd 内部引用 icons）
- 如需优化，可用 `lucide-react` 替换 `@ant-design/icons`（可省 ~240KB gzip）
