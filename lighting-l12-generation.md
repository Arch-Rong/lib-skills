# 打光 L12：生成按钮、结果节点与防重提交

打光直接使用**普通图片模型**生成，不再需要专用服务契约（`LightingServiceContract` 仅保留给显式适配，生产路径不传）。面板底部的“生成”按钮已接通。

## 提示词模板

`utils/lightingGeneration.ts` 的 `buildPrompt` 把面板点位带入模板：

```
保持原图内容不变，只调整光照。保留 @图片1 原有细节。在@图片1 正前方打RGB颜色（#0000ff）光，主光强度为80%；轮廓光在背后上方打紫光（RGB颜色#a855f7）。光源位置必须严格按上述方位，不要改变光源所在的位置和方向。保持原图宽高比（1600:900）和原有构图，不裁切、不扩图。
```

- 主光方向：六个标准方向用中文方位；非标准点位用“正前方偏下方”“右前方偏上方”这类位置词（单轴分量超过 0.3 才计入），并附方位角与仰角（精确位置以角度为准）。
- 位置强调：主光或轮廓光至少有一个生效时，追加“光源位置必须严格按上述方位，不要改变光源所在的位置和方向。”；主光强度为 0 且无轮廓光时不追加。
- 轮廓光是主体背后的灯，有独立灯位（`rimDirection`，缺省为背后上方 45°）。开关打开时追加“轮廓光在<灯位>打紫光”；灯位仍为默认值时只写“背后上方”，被拖动后追加精确方位角/仰角。
- 智能模式的光效描述、光照参考图（图片2）沿用原有拼接。

## 调用链

新增 `hooks/useLightingGeneration.ts`，由 `WorkflowBuilder` 调用并以 `generation` 传给 `LightingPanel`：

1. 面板内的 `ModelListComponent` 选择图片模型（`modelType=2`）。
2. 点击“生成”时同步加锁（`submittingRef`），并复核来源图未变化（`sourceKey`）。
3. 冻结快照（`LightingTaskSnapshot`）→ `buildLightingGenerationOrder` 构造 V2 订单，快照写入 `clientParams.lightingTask`。
4. 在来源节点右侧创建普通图片结果占位节点（id 即请求 ID）和来源连线。
5. 调用 L11 的 `submitPreparedImageOrder`（登记恢复记录 → `preserveGenerationNode` 保全草稿 → 发单），之后由结果节点自身的原轮询回填结果。
6. 已受理或待确认：加入来源所在分组、关闭面板、取预估耗时；明确拒绝或未发送：移除空占位节点和连线并提示错误。

## 防重

- `submittingRef` 同步锁 + `submitting` 状态：提交中按钮禁用，连点只发一次订单。
- 固定请求 ID 沿用 `submitPreparedImageOrder`，重试不会重复计费。
- 提交期间来源图、草稿的改动不影响已冻结的订单。

## 验证

- `useLightingGeneration.test.tsx`（6 项）：结果节点与连线、单次发单、连点防重、拒绝回滚、待确认保留、来源变化拦截。
- `lightingGeneration.test.ts`：新增普通图片模型无契约路径与提示词模板断言，并修正过期用例。
- 未运行构建；未发起真实订单。`WorkflowBuilder.graph.test.tsx` 的右键菜单用例与 `WorkflowBuilder.retry.test.tsx` 的视口用例共 10 项在本次改动前即失败（移除本次改动后复测结果相同），与打光无关。

## 面板位置

点击「打光」后，面板通过 `LightingPanelContext` + `LightingPanelSlot` 嵌入源节点（图片 / 上传节点）下方的提示词输入区域，替代该节点的提示词输入框，不再以固定定位浮在屏幕底部。面板宽度为 840（未开启智能模式时收窄为 640），受画布可视区域限制。

## 面板样式

面板样式按设计稿调整：左侧灯位球卡片（顶部为透视/正面切换），中间全局参数，开启智能模式后右侧为描述输入、参考图入口和 4 列预设缩略图；底部为重置、费用与仅图标的生成按钮。原来的「图片预览（近似效果）」不再在面板内显示，`LightingPreview` 组件和 `useLightingPreview` 保留未删。

智能模式开启时为三栏：左侧灯位球加宽加高，右侧智能模式栏收窄，预设为 3 列；描述输入和参考图入口高度均为 64，参考图为 64×64 方块，不提供素材库和「重新上传」入口，说明文字仅对读屏可见（`LightingReferenceInput` 的 `minimal` 默认 false，只在打光面板传 true；更换参考图需先删除再添加）。

## 灯位与实时预览补充

- 灯位球分“透视 / 正面”两种视角；主灯（白）和轮廓光灯（紫，主体背后）都可在灯位球上用鼠标拖动。
- 主灯正在被拖动（灯位球或图片上）时，轮廓光开关与灯点不可用，松开后恢复（状态由 `WorkflowBuilder` 的 `lightDragging` 经 `LightingPanelContext` 传递）。
- 被打光的图片节点内显示主灯位点（`LightingImageHandle`），可直接在图片上拖动；图片水平/垂直方向对应灯位 x/y 分量，与预览着色一致，背后的灯显示为虚线空心点。
- 画布图片节点内的光影由 `useLightingPreview` 实时重绘（`LightingPanel` 内调用），参数变化即更新；轮廓光在预览里仅为边缘紫色光晕近似，最终效果以模型生成为准。
