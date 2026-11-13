# 打光 L09 预设与素材来源

本记录对应 `WorkflowBuilder/utils/lightingPresets.ts` 的八套 v1 配置。参数来自《打光效果技术方案与实施步骤》6.1 的候选配置；这些配置用于本地交互和后续模型联调，尚未通过 L14 正式生成效果验收。

## 来源与用途

以下文件全部位于 `packages/canvas-main/src/components/WorkflowBuilder/assets/lighting/`。素材由本次开发用几何图形、渐变和解析表面着色原创绘制，可随本项目使用、修改和分发；未使用第三方照片、竞品缩略图、人物肖像或品牌图形。

| 预设 | 卡片缩略图 | 正式生成的光影参考 | 来源 |
| --- | --- | --- | --- |
| 过曝胶片 | `overexposed-film.svg` | 无 | 原创 SVG 球体与展台光照示意 |
| 蓝色逆光 | `blue-backlight.svg` | 无 | 原创 SVG 球体与展台光照示意 |
| 伦勃朗光 | `rembrandt.svg` | 无 | 原创 SVG 球体与展台光照示意 |
| 赛博朋克 | `cyberpunk.svg` | 无 | 原创 SVG 球体、展台与霓虹色示意 |
| 落日迷幻 | `sunset-psychedelic.svg` | `sunset-psychedelic-reference.png` | 缩略图为原创 SVG；参考图为独立绘制的 512×512 RGB PNG 几何静物 |
| 神秘暗调 | `mysterious-dark.svg` | 无 | 原创 SVG 球体、展台与条带光照示意 |
| 黄金时刻 | `golden-hour.svg` | 无 | 原创 SVG 球体与展台光照示意 |
| 诺兰冷灰 | `nolan-gray.svg` | 无 | 原创 SVG 球体与展台光照示意 |

八张 SVG 均为 256×256，用于区分预设风格，不能视作真实图片打光结果或模型效果样例。落日迷幻的 PNG 使用独立的球体、立柱和台座场景，表达暖色侧后光与紫色暗部的光照方向、色彩及氛围；它是光影引导素材，不应把几何物体内容迁移到源图。

## 配置及验收边界

- `thumbnail` 仅显示卡片；`reference` 明确为 `null` 或独立参考资源，二者不互相兜底。
- 每套预设完整定义方向、亮度、显式 RGB、轮廓光、描述与参考。空描述和空参考必须清除上一套的残留值。
- 描述由本次开发原创撰写；三套按方案保持为空（蓝色逆光、伦勃朗光、落日迷幻）。
- 本地近似预览只响应全局方向、亮度与颜色；描述和参考的语义、轮廓光、主体保留及模型对预设的实际响应仍需后续服务与 L14 效果验收。
- 预设参考已通过项目既有 `uploadFile` 上传并确认尺寸，不将开发服务器的本地资源 URL 当作公网输入；正式生成继续使用该已解析参考。

## L09 本地交互交付记录（2026-11-10）

本步沿用 `useLightingPanel` 页面会话，复用 `uploadFile`、`getMediaType`、`getImageDimensions`、`AssetLibraryPanel`、`assertPromptWithinLimit` 和现有弹窗。没有接入订单或新增生成请求。

改动位置及行为：

- `hooks/useLightingPanel.ts`、`utils/lighting.ts`：参考上传、图库选择、移除与重试，按项目／源图／会话／操作版本隔离迟到结果；关闭后保留同源草稿，切换源图使旧操作失效。预设完整替换参数、描述、参考与错误状态；手动改生效参数清预设选中，重置保留智能开关。
- `ui/LightingPanel.tsx`、`LightingReferenceInput.tsx`、`LightingPanel.css`：单张参考的空、加载、失败、就绪状态，进度、大图预览和八预设入口。上传期间仍可替换参考；参考弹窗的 Escape 不会关闭父面板。
- `utils/lightingPresets.ts`、`assets/lighting/`：八套有版本的完整配置和原创素材，来源及正式效果限制见上文。
- `ui/LightingImagePreview.test.tsx`：ResizeObserver 测试替身使用普通只读字段，兼容项目 `erasableSyntaxOnly` 配置。

验证：

- `pnpm exec vitest run` 定向选择 12 个打光测试文件：132 项通过。随后修正测试语法并复跑 `LightingImagePreview.test.tsx`，11 项通过。
- 打光工具、hooks、UI 与入口测试的定向 `pnpm exec eslint ... --max-warnings 0` 通过；修正后的测试文件单独复查通过。
- `pnpm exec tsc --noEmit -p tsconfig.app.json --pretty false` 未通过，剩余诊断与本阶段开始时一致：`@lumina/director-cli` 模块／导出组件类型，以及 `tidyCanvasLayout.test.ts` 中旧 Map 调用。打光文件没有剩余诊断；此结果不代表全仓类型检查通过。
- 已在 `localhost:5174` 的实际画布验证素材库选图、大图预览、同源草稿恢复、预设覆盖与重置保留智能开关；800×700 窗口可滚动到预设区，关闭与底部操作区保持可见，无内容横向溢出。验收后已恢复原窗口尺寸。
- 未运行构建、未提交付费生成。

人工复测：选中有图片的图片／上传节点，点击工具栏“打光”，开启智能模式；从素材库选择一张图片并打开大图，再关闭重开面板检查草稿。切换“黄金时刻”和“蓝色逆光”确认描述及参考完整替换，手动修改亮度确认取消预设选中，最后重置检查智能开关仍开启、参考及描述清空。

尚未通过项与下一步：L09 交付本地交互，预设真实效果与完整打光生成尚未验收。L10 从核对精确模型路由、连续灯向／轮廓光配置、输入图片角色、费用维度与脱敏请求响应开始，再实现纯构参和估价；现有生成按钮保持“能力待接入”。模型描述上限尚未确认，当前不编造固定上限，后续接入已确认值并校验最终完整提示词。
