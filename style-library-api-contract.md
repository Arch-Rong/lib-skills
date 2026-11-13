# 团队风格库前后端联调契约

本仓库实现风格库界面、接口客户端、画布和创作页的提示词处理；对应服务端接口、团队隔离和事务逻辑已在同级 `Aigcbuilder` 工程实现。下列内容为双方接口契约，沿用现有 RPC、`replyHeader`、游标分页、上传接口及画布保存和协作机制。

后端相关 66 项测试和 Maven 打包检查已通过。`online` 已于 2026-10-03 16:39（Asia/Shanghai）显式执行 `V37__team_style_library.sql`，16:41 独立回读确认；154 个团队分类已初始化，3 个原有个人风格内容完整保留。其他环境部署前须核对 V37，先上线后端再发布新版前端；本次未部署应用，真实联调尚未执行。后端实现及迁移说明见 `Aigcbuilder/docs/style-library-team.md`，执行记录见 `Aigcbuilder/docs/style-library-online-20261003.md`。

## 数据与授权

- 风格 `scope: 0` 为个人，必须记录 `ownerId`，`categoryId: null`；`scope: 1` 为团队共享，必须记录 `tId` 和本团队公共 `categoryId`。个人风格不得绑定公共分类。
- 列表和详情补充 `tId`、`ownerId`、`hidden`、`owned`。`owned` 只表示创建关系，不能作为共享风格的编辑授权；上传者也不能从用户接口更新或删除共享风格。
- 分类包含 `categoryId`、`categoryName`、`categoryCode`、`sortWeight`、`tId`、`isSystem`、`styleCount`。后台数量必须包含隐藏项。每个团队提供固定“未分类”（`isSystem: true`、`categoryCode: "uncategorized"`），不允许重命名、删除。
- 用户接口根据登录身份确定本团队与个人所有者，不信任客户端传入的所有者或团队。用户可读取本团队未隐藏共享风格和自己的个人风格，可新建个人或上传本团队共享风格，只能更新、删除自己的个人风格。
- `team_admin` 只能管理本团队共享风格和分类。同团队管理员共同维护；其他团队请求必须拒绝。
- `admin` 的目标团队必须出现在服务端授权列表中，包括受限管理员。后台始终排除个人风格，管理员也不得通过后台接口维护他人的个人风格。
- 新建、更新校验：名称去除首尾空白后必填且不超过 30；提示词必填且不超过 500；参考图可选，最多一个 JPG、PNG、WebP。参考图只用于预览，不参与模型生成。新建不要求 `modelCode`，旧字段可兼容返回。
- 分类名称去除首尾空白后必填，在同团队内唯一。服务端再次校验所有权限、长度、图像格式和系统分类保护规则。

## 用户接口

客户端继续使用现有 `StyleTemplateApi`，HTTP 路径由现有请求层增加 `/v2` 前缀，均为 POST。

| RPC 后缀 | 请求 | 返回及行为 |
| --- | --- | --- |
| `ListCategory` | `{}` | `list`：本团队公共分类，同一份数据供筛选和保存位置使用 |
| `List` | `{ tab, categoryId, keyword, nextIndex }` | `{ list, nextIndex, last }`；关键词按风格名称筛选；`all` 返回本团队可见共享与本人个人；`discover` 返回共享；`mine` 返回本人个人；保留原收藏、最近使用调用兼容 |
| `Get` | `{ styleId }` | `{ data }`；包含 `prompt` 和上述权限字段；隐藏共享项不得用于新选择 |
| `Add` | `{ styleName, prompt, cover, scope, categoryId }` | `{ styleId }`；个人 `categoryId: null`；共享分类必须属于登录团队；共享保存成功后上传者立即失去编辑和删除权限 |
| `Update` | 上述字段 + `styleId` | 仅自己的个人风格，保存位置保持个人；共享修改走后台接口 |
| `Delete` | `{ styleId }` | 仅自己的个人风格 |
| `Use` | `{ styleId }` | 重新校验风格可见性和团队权限；记录使用统计；失败不得改变当前客户端选择 |

`categoryId: ""` 表示全部；`nextIndex: 0` 表示首页；最后一页返回 `last: true`。共享隐藏、删除和越权记录不得通过用户列表或详情暴露。`Favorite`、`Unfavorite` 保留现有接口兼容。

## 后台接口

页面入口 `/bo/styleLibrary`。以下路径为完整路径，均使用现有后台请求层，必须校验目标 `tId`。

| 路径 | 请求 | 业务响应字段 |
| --- | --- | --- |
| `/bo/v1/StyleTemplateApi.ListManagedTeams` | `{}` | `list: [{ tId, name }]`；`currentTeamId` 为本团队 ID，供 `team_admin` 固定使用；列表仅包含服务端授权团队 |
| `/bo/v1/StyleTemplateApi.ListCategory` | `{ tId }` | `list`：分类，包含 `tId`、系统保留标记及包括隐藏项的 `styleCount` |
| `/bo/v1/StyleTemplateApi.List` | `{ tId, categoryId, keyword, nextIndex }` | `list, nextIndex, last`；仅目标团队共享风格，隐藏项仍可查看与搜索；按当前分类搜索，空分类 ID 表示全部 |
| `/bo/v1/StyleTemplateApi.Get` | `{ tId, styleId }` | `data`：该团队共享风格详情 |
| `/bo/v1/StyleTemplateApi.Add` | `{ tId, styleName, prompt, cover, scope: 1, categoryId }` | `styleId` |
| `/bo/v1/StyleTemplateApi.Update` | 上述字段 + `styleId` | 标准成功响应，更新 `updateTime` |
| `/bo/v1/StyleTemplateApi.Delete` | `{ tId, styleId }` | 标准成功响应，只删除该团队共享风格 |
| `/bo/v1/StyleTemplateApi.SetHidden` | `{ tId, styleId, hidden }` | 标准成功响应，隐藏可逆，更新 `updateTime` |
| `/bo/v1/StyleTemplateApi.AddCategory` | `{ tId, categoryName }` | `categoryId` |
| `/bo/v1/StyleTemplateApi.UpdateCategory` | `{ tId, categoryId, categoryName }` | `categoryId`，分类 ID 不变 |
| `/bo/v1/StyleTemplateApi.DeleteCategory` | `{ tId, categoryId }` | `movedCount, uncategorizedCategoryId`；在同一事务内将全部风格（包含隐藏项）迁移到固定“未分类”再删除分类，返回实际迁移数量，不删除风格 |

所有响应沿用 `replyHeader`，成功码为 `12000`；业务错误通过既有请求层抛出，不能用空列表掩盖越权。后台时间字段 `updateTime` 沿用毫秒时间戳。

## 提示词、快照与历史兼容

- 用户输入与风格分别维护，最终请求 `params.prompt` 为 `用户输入 + "\n" + 风格提示词`；任一方为空时省略分隔符。“无风格”对应 `appliedStyle: null`。
- 前端按实际发送字符串的 JavaScript `length` 计算 `MaxText`，包括引用标签与换行。重绘的固定指令也占用额度。服务端须校验最终完整字符串，保持现有 `MaxText` 口径。
- 无有效 `MaxText` 时前端显示未知，不猜测上限；由服务端最终校验。切换风格或模型产生超限时保留原输入并禁止生成。新增输入和粘贴超过额度时拒绝整次增加，已有超限内容允许缩短。
- 节点参数、面板缓存及订单 `clientParams` 记录 `appliedStyle` 快照与 `stylePromptVersion: 2`；订单 `clientParams.prompt` 保留原基础输入，`clientParams.genAgainParams.prompt` 记录最终提示词。再次生成直接复用最终字符串。
- 快照和版本仅属于客户端元数据，不能传到模型动态参数。参考图也不能被风格功能附加到生成引用中。
- 历史已混入输入的风格文字原样保留，原选择直接迁移成独立快照，补版本标记；允许重复，重复部分计入实际长度，不尝试拆分或去重。
- 公共风格修改、隐藏、删除不会改写已选节点或历史订单中的快照；后续新选择必须使用最新可见数据。
- 删除个人风格成功后，当前页面中的节点与当前用户面板缓存清除此风格引用。服务端不需删除历史订单快照。

## 联调验收

重点覆盖跨团队拒绝、同团队共同管理、普通上传者失权、隐藏筛选和快照继续使用、分类重复与系统分类保护、迁移数量一致、保存成功后应用失败只重试应用、无目标仅保存、纯风格生成、额度边界与粘贴拒绝、刷新和克隆不额外拼接、再次生成精确复用最终提示词。接口未在后端上线前，后台会显示加载错误和重试入口，不模拟真实数据或放宽权限。
