# 打光 L10：本地构参准备与服务确认清单

本记录对应 `WorkflowBuilder/utils/lightingGeneration.ts`。用户已选择“暂未确定服务，先完成本地准备”。本步交付纯函数及测试；L10 的真实服务协议、计费映射和效果确认尚未完成，生产面板继续显示“能力待接入”。

## 复用范围与调用方式

只新增本地构参适配及对应测试，没有修改模型接口、普通生图、估价接口或发送流程。复用：

- `lighting.ts` 的有效配置、提交校验及方向／角度换算。
- `getNodePromptBudget` 的引用编号与现有 `serializePrompt`；完整提示词经过 `parsePromptLimit`／`assertPromptWithinLimit`。
- `getMediaReferenceLimitReason` 的总输入数量校验，源图计为一张，有光影参考时总数为两张。
- `buildCostEstimateRequest` 的费用输入，只有已确认按模型版本计费时才派生请求对象；其他计费状态返回 `null`，供原组件显示“—”。

入口：

```ts
getLightingCapabilityReason(model, contract); // undefined 表示契约门禁通过，否则是阻断原因
prepareLightingGeneration({ input, model, contract });
// 返回独立的规范化 input、promptContent、params、costEstimateRequest。
// 不创建 requestId，不读取模型缓存、普通上游、风格或画布尺寸，不发任何请求。

buildLightingGenerationOrder({ snapshot, model, contract });
// 接收已有 schemaVersion:1 / requestId 的快照，输出 V2 订单。
// params/clientParams 为 JSON 字符串，快照在 clientParams.lightingTask。
```

`LightingServiceContract` 是该适配器显式接收的本地确认输入，不是新增服务端字段或另一套模型注册系统。当前没有生产契约实例。`confirmed` 只表示调用方已核对协议，不替代服务及效果证据；不得仅把测试配置改名后用于生产。

已确认契约必须绑定完整 `name/version/type` 且不可原位变更，尤其不能在同一模型版本下改固定轮廓光配置。后续可靠发送／重试必须保留并复用已构造的订单负载，不根据新草稿或变更后的契约重新构造同一 `requestId` 的请求。

## 已落实的本地行为

- 模型必须为图片类型，与输入及契约完整路由一致；契约状态、版本、八项必需能力、轮廓配置、资源定位策略和规格均须有效。
- 六轴使用对应中文方向；非轴方向附方位角和仰角，精确到小数点后六位度数，不使用界面的方向标签容差吸附；快照保留规范化连续向量。
- 亮度 `0` 明确表达“不新增主光贡献”，不回退默认亮度；显式 RGB 和轮廓光 `false` 保留。轮廓方向及颜色从已确认契约读取，不写死方案示例。
- 智能关闭时忽略隐藏描述和参考；开启时描述去首尾空白，参考只占图片2，并说明光照用途、禁止主体／构图迁移、全局主光优先。同 URL 的两种用途仍保留两项引用。
- 仅使用原始源图和有效参考的 HTTP(S) 地址；真实素材编号原样传递。无素材编号只能在契约明确允许空编号时发送空字符串，不以节点 ID 冒充素材 ID。
- 输出须属于契约声明的规格集合，像素为正安全整数；在未确认比例容差时只接受源图精确比例，不裁剪、扩图或套用画布节点尺寸。分辨率标签必须无首尾空白，保证订单与估价值一致。
- 估价和订单共用规范化路由、输出档位、源图存在和数量1。未知计费／额外打光维度计费保持 `null`，不发普通图片报价请求。
- 可选 seed 须由契约明确支持；保留 `0`。当前构参没有实际下单、资源上传、请求编号分配或费用查询。

## 服务字段与效果确认状态

| 项目 | 仓库已有约定 | 打光服务状态及待交付证据 |
| --- | --- | --- |
| V2 外层及序列化 | `CreateOrderV2Req`、`params/clientParams` JSON 字符串 | 通用入口已存在；打光脱敏请求／响应未提供 |
| 模型路由 | `modelName/modelVersion/type` | 未确认精确打光路由；不从普通图生图能力或名称推断 |
| 提示词及图像角色 | `prompt`、`referImage`、图片媒体常量、`IMG_TO` | 本地固定图片1源图／图片2参考；服务鉴权及角色效果未确认 |
| 连续灯向 | 向量及角度模板已准备 | 六轴、至少两个非轴点的真实响应未确认 |
| 亮度、颜色 | 0／50／100及实际 RGB 可表达 | 强度定义、显式白色及彩光响应未确认；当前草稿不提供“未指定颜色”模式 |
| 轮廓光 | 开关及契约提供的固定配置 | 实际固定方向／颜色未确认，不能把示例紫光当默认协议 |
| 智能组合 | 描述、参考和主光优先模板已准备 | 空输入、分别输入、组合及冲突响应未确认 |
| 图片数量／素材 ID | 复用 `MaxImage`；资源策略显式输入 | 源图与参考总数、无素材 ID 时的定位／鉴权规则未确认 |
| 输出 | 仅接收精确比例的确认规格 | 合法尺寸、档位、步进及允许比例误差未确认 |
| 费用 | 原估价接口支持模型、档位、数量、是否有图 | 打光是否按模型版本定价未确认；额外智能／参考维度尚无接口映射 |
| 任务可靠性 | 原 V2 下单及请求标识入口已存在 | 打光场景的幂等、查询、响应丢失和计费结果未联调 |

以下是**纯测试配置的参数示例**，不是实际服务请求证据，也不是生产模型路由。它展示源图800×450输出、65%显式白光、关闭智能及轮廓光时的标准 JSON：

```json
{
  "prompt": "保持原图内容不变，只调整光照。保留 图片1 原有细节。在图片1 正前方打RGB颜色（#ffffff）光，主光强度为65%。",
  "negativePrompt": "",
  "classType": 2,
  "referImage": [
    {
      "id": "source-asset",
      "type": 2,
      "title": "图片1",
      "description": "图片1",
      "referImage": "https://assets.test/source.png",
      "referVideo": "",
      "referAudio": ""
    }
  ],
  "referVideo": [],
  "referAudio": [],
  "width": 800,
  "height": 450,
  "resolution": "test-1K"
}
```

## 验证与后续

本步最终验证：

```sh
lighting_test_root=packages/canvas-main/src/components/WorkflowBuilder
pnpm exec vitest run \
  "$lighting_test_root"/utils/{lighting,lightingGeneration,lightingPresets,lightingPreview,lightingPreviewBridge,lightingSource,nodePrompt,mediaReferenceLimits,costEstimate}.test.ts \
  "$lighting_test_root"/hooks/{useLightingPanel,useLightingPreview}.test.tsx \
  "$lighting_test_root"/ui/{LightingPanel,LightingReferenceInput,LightingImagePreview,LightingSphere}.test.tsx \
  "$lighting_test_root"/nodes/view/NodeToolbar.lighting.test.tsx
pnpm exec eslint "$lighting_test_root"/utils/lightingGeneration.ts "$lighting_test_root"/utils/lightingGeneration.test.ts --max-warnings 0
pnpm exec tsc --noEmit -p tsconfig.app.json --pretty false
git diff --check
```

- 16 个定向文件共 **249 项测试通过**，包括新增构参测试78项、原打光测试132项，以及引用／媒体限制／费用回归39项。
- 新增两个 TypeScript 文件的 ESLint 与 diff 检查通过。
- 类型检查仍未通过，诊断与 L09 最终记录完全相同：`@lumina/director-cli` 模块／组件类型错误和 `tidyCanvasLayout.test.ts` 的旧 Map 调用；新增打光文件没有诊断。不代表全仓类型检查通过。
- 独立源码审查未发现未解决的阻断问题；已明确记录 confirmed 契约不可原位变更、重试复用原订单的约束。
- 未运行构建，没有实际生成、估价或付费请求。

手工复测仍从 `localhost:5174` 的图片节点工具栏进入“打光”；本步没有新增可执行生成入口，继续只能调整本地预览和草稿。

后续 L11 是最小提取原单笔可靠发送，需遵守技术方案中对应的原13.2冲突确认要求；L12 再接入冻结任务与结果占位。真实服务及付费生成须等上述协议确认和 L14 的授权效果验收，不能用本步测试契约宣称已接通。
