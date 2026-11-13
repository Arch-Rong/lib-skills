# Bo 后台管理接口设计方案

> 核心思路：最大化复用现有接口 + 前端加工，减少新增接口数量

## 一、总览

### 现有可复用接口

| 接口 | 用途 | 所在文件 |
|---|---|---|
| `apiListMembers` | 用户管理列表、仪表盘(total) | `src/api/modules/team.ts` |
| `apiListProjects` | 项目管理列表、仪表盘(total) | `src/api/modules/project.ts` |
| `apiAddProject` | 项目管理-创建项目 | `src/api/modules/project.ts` |
| `apiAddMember` | 新增用户 | `src/api/modules/team.ts` |

### 新增接口（仅 4 个）

| # | 接口 | 页面 |
|---|---|---|
| 1 | `BoApi.getDashboardStats` | 首页仪表盘 |
| 2 | `BoApi.getBillOverview` | 用量总量 |
| 3 | `BoApi.getBillDetail` | 账单明细 |
| 4 | `BoApi.getMemberDetail` | 用户详情 |

### 现有接口扩展字段（2 处）

| 接口 | 扩展内容 |
|---|---|
| `ProjectInfo` | 新增 `memberCount`、`memberAvatars`、`visibility` 字段 |
| `ListMembersReq` | 新增 `keyword`、`status` 筛选参数（可选） |

---

## 二、新增接口定义

### 2.1 仪表盘统计 — `BoApi.getDashboardStats`

**页面**：`Bo/home.tsx`

**请求**：无参数（GET）

**响应**：

```typescript
export interface DashboardStatsRsp {
    totalUsers: number;
    todayNewUsers: number;
    totalRevenue: number;       // 总消费额
    totalProjects: number;
    /** 用户增长趋势（近30天） */
    userTrend: { date: string; value: number }[];
}
```

**说明**：
- 一个接口解决首页所有数据，避免前端多次调用再拼接
- 也可以不新增此接口，前端并行调 `apiListMembers` + `apiListProjects` 取 `total`，但趋势和消费数据还是需要后端提供

---

### 2.2 用量总量 — `BoApi.getBillOverview`

**页面**：`Bo/SummyBill/index.tsx`

**请求**：

```typescript
export interface BillOverviewReq {
    period: '7d' | '30d' | '90d';
    page?: number;
    pageSize?: number;
}
```

**响应**：

```typescript
export interface BillOverviewRsp {
    /** 统计卡片 */
    summary: {
        teamCost: number;
        teamCostChange: number;       // 较上期变化百分比，正数=增长，负数=下降
        resaleCost: number;
        resaleCostChange: number;
        resaleProfit: number;
        resaleProfitChange: number;
        discount: number;             // 如 0.85
    };
    /** 趋势图数据 */
    trend: { date: string; teamCost: number; resaleCost: number }[];
    /** 历史明细（分页） */
    history: {
        date: string;
        totalAmount: number;
        teamCost: number;
        resaleCost: number;
        profit: number;
        /** 模型明细 */
        models: { model: string; selfCost: number; profit: number }[];
    }[];
    historyTotal: number;
}
```

**说明**：
- 一个接口返回 统计卡片 + 趋势图 + 历史明细
- 前端根据 `period` 切换时重新调用
- 前端加工：`teamCostChange > 0` → 显示 `↑ 较上月增长 X%`；`< 0` → 显示 `↓ 较上月下降 X%`

---

### 2.3 账单明细 — `BoApi.getBillDetail`

**页面**：`Bo/BillDetail/index.tsx`

**请求**：

```typescript
export interface BillDetailReq {
    page: number;
    pageSize?: number;
    keyword?: string;
    status?: 'active' | 'overBudget' | 'archived';
    dateRange?: 'week' | 'month' | 'quarter' | 'year';
}
```

**响应**：

```typescript
export interface BillDetailRsp {
    data: {
        projectId: string;
        name: string;
        totalAmount: number;
        status: 'active' | 'overBudget' | 'archived';
        lastExpenseDate: string;
        /** 展开后的消费明细 */
        expenses: {
            id: string;
            date: string;
            member: string;
            expenseItem: string;
            amount: number;
        }[];
    }[];
    total: number;
}
```

---

### 2.4 用户详情 — `BoApi.getMemberDetail`

**页面**：`Bo/memberDetail.tsx`

**请求**：

```typescript
export interface MemberDetailReq {
    memberId: string;
}
```

**响应**：

```typescript
export interface MemberDetailRsp {
    /** 基本信息（复用 Member 字段 + 扩展） */
    member: {
        id: string;
        name: string;
        avatar: string;
        role: string;
        lastLogin: string;
        email: string;
        registerTime: string;
        lastLoginFull: string;
        loginIp: string;
    };
    /** 近期消费记录 */
    recentConsumptions: {
        id: string;
        title: string;
        date: string;
        amount: number;
        type: number;    // 1-server 2-database 3-storage
    }[];
    /** 所属项目 */
    projects: {
        id: string;
        name: string;
        role: string;
    }[];
}
```

**说明**：
- `type` 字段前端维护图标映射：`1 → server`、`2 → database`、`3 → storage`

---

## 三、现有接口扩展

### 3.1 `ProjectInfo` 扩展字段

```typescript
export interface ProjectInfo {
    projectId: string;
    title: string;
    description?: string;
    type?: string;
    createTime?: number;
    updateTime?: number;
    // ===== Bo 需要的扩展字段 =====
    memberCount?: number;            // 成员总数
    memberAvatars?: string[];        // 前几个成员头像 URL
    visibility?: 'partial' | 'all'; // 可见性
}
```

### 3.2 `ListMembersReq` 扩展参数（可选）

```typescript
export interface ListMembersReq {
    page: number;
    pageSize?: number;
    // ===== 可选扩展 =====
    keyword?: string;     // 搜索关键字
    status?: string;      // 状态筛选
}
```

---

## 四、前端加工逻辑汇总

| 加工场景 | 说明 |
|---|---|
| 仪表盘复用 total | 前端并行调 `apiListMembers` + `apiListProjects` 取 `rsp.total` |
| `BillOverview.summary.xxxChange` → UI 趋势 | 正数 = `↑ 增长 X%`，负数 = `↓ 下降 X%`，前端判断渲染 |
| `MemberDetail.recentConsumptions.type` → icon | `1 → server`，`2 → database`，`3 → storage`，前端维护映射 |
| `ProjectInfo.createTime` 时间戳 → 格式化 | 前端 `dayjs(createTime).format('YYYY-MM-DD HH:mm')` |
| `Member.role` → 角色中文 | 已有逻辑：`'admin' → '管理员'`，`其他 → '普通用户'` |
| `Member.avatar` 兜底 | 已有逻辑：无头像时用 `avatarList[idx % avatarList.length]` |

---

## 五、接口文件规划

新增接口统一放在 `src/api/modules/bo.ts`，导出后在 `src/api/index.ts` 中注册。

```
src/api/
├── modules/
│   ├── bo.ts          ← 新增：Bo 后台管理相关接口
│   ├── team.ts        ← 已有：用户管理（复用）
│   ├── project.ts     ← 已有：项目管理（复用 + 扩展字段）
│   ├── genResult.ts
│   ├── modelInfo.ts
│   └── order.ts
└── index.ts           ← 注册 bo 模块导出
```
