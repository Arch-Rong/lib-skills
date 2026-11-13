# 打光 L11：复用单笔可靠提交

本步完成单笔提交的最小提取，普通生图已使用同一函数，供后续打光提交复用。用户已明确同意技术方案 §1.2／L11 涉及的调用关系提取。打光的真实模型及计费协议仍未确认，生产面板继续显示“能力待接入”；本步没有接通打光生成按钮。

## 改动范围

- `WorkflowBuilder/utils/imageGeneration.ts` 新增 `submitPreparedImageOrder`：接收已构造的订单、固定请求 ID、批次已提交 ID、owner、开始时间及节点读取／同步更新／保全依赖。
- `nodes/ImageNodeWithBaseNode.tsx` 的普通生图逐笔调用该函数，继续在原入口处理输入校验、普通构参、上一批保存、批量累计、通知、轮询和倒计时。
- `utils/generationOrder.ts` 补齐可靠业务码判断：缺失、非数值或非有限业务码不再被误判为明确拒绝。图片、视频和剧本生成仍复用原发送函数及原固定 ID 重试，没有另一层重试。
- 新增 `utils/imageGenerationSubmission.test.ts`，扩展 `nodes/GenerationSubmission.test.tsx`，修正 `hooks/useNodePolling.test.tsx` 中已有单笔测试的请求列表夹具。

没有修改恢复存储、轮询、租约、协作保存算法或快捷键。

## 调用与结果

```ts
const result = await submitPreparedImageOrder({
  projectId, nodeId, params, requestId, submittedRequestIds,
  ownerClientId, createTime, getNode,
  updateNodesSync: update => flushSync(() => setNodes(update)),
  preserveGenerationNode,
});
```

调用层负责生成请求 ID、构造订单及同步更新节点；工具层不读取 Context、不调用 hook、不创建节点、不启动轮询。每次只登记此前已提交的批次前缀与本笔请求，不提前登记后续请求。异步保全结束后，再检查完整请求列表及 owner，匹配规则与原图片轮询的精确列表判断一致。

| 状态 | 含义及处理 |
| --- | --- |
| `notSent` | 登记、保全或发送前复核失败，没有发送本笔订单。登记成功且节点／owner／请求仍匹配时，沿原失败登记清理本笔；前批已提交请求继续保留。 |
| `rejected` | 服务以可靠业务码明确拒绝，沿相同归属保护处理本笔失败。 |
| `unconfirmed` | 已尝试发送但没有可靠受理答复，保留原请求、owner 和恢复信息，由原轮询继续查询。 |
| `accepted` | 服务已答复受理，交由原轮询等待结果，不在此删除恢复记录。 |

结果始终携带原 `requestId`；`requestIds` 是调用层应继续跟进的本批请求。任务被删除、替换或接管后，本步不会覆盖新节点状态，也不在不匹配时改写恢复记录。跨会话接管后的刷新及未发送旧请求处理仍需在 L13 按原恢复链路模拟验收，不能把本步单元测试当作完整跨会话验收。

## 基线与验证

提取前发现两个旧测试夹具问题，并先修正基线：

- 普通提交测试的 9 个视频用例因漏模拟 `useNodesData` 而无法渲染；补齐既有 ReactFlow mock 后，原三个基线文件 48／48 通过。
- 图片轮询有 6 个单笔用例登记了两笔、实际只查一笔，不符合现有精确匹配保护；对齐单笔夹具后 25／25 通过，没有放宽生产 hook 的保护。

新增回归覆盖：登记和同步任务状态先于保全／发送、登记失败不发送、保全失败刷新不恢复为在途、缺失／非法业务码保留原请求、保全期间 owner／请求变化及节点删除、迟到响应不覆盖新任务、完整批次逐笔发送，以及第二笔失败时保留首笔且不登记第三笔。缺业务码、保全期间缩减请求列表和入口前缀变化均验证过先失败、修复后通过。

最终验证命令：

```sh
lighting_l11_test_root=packages/canvas-main/src/components/WorkflowBuilder
pnpm exec vitest run \
  "$lighting_l11_test_root"/utils/{lighting,lightingGeneration,lightingPresets,lightingPreview,lightingPreviewBridge,lightingSource,nodePrompt,mediaReferenceLimits,costEstimate,generationRecovery,generateScriptAssets,imageGenerationSubmission}.test.ts \
  "$lighting_l11_test_root"/hooks/{useLightingPanel,useLightingPreview,useNodePolling}.test.tsx \
  "$lighting_l11_test_root"/ui/{LightingPanel,LightingReferenceInput,LightingImagePreview,LightingSphere}.test.tsx \
  "$lighting_l11_test_root"/nodes/{GenerationSubmission,ImageNodeWithBaseNode.mainImage}.test.tsx \
  "$lighting_l11_test_root"/nodes/view/NodeToolbar.lighting.test.tsx \
  packages/core/src/api/{orderRetry,request,responseValidation}.test.ts
pnpm exec eslint \
  "$lighting_l11_test_root"/utils/{imageGeneration,generationOrder,imageGenerationSubmission.test}.ts \
  "$lighting_l11_test_root"/nodes/{ImageNodeWithBaseNode,GenerationSubmission.test}.tsx \
  "$lighting_l11_test_root"/hooks/useNodePolling.test.tsx --max-warnings 0
pnpm exec tsc --noEmit -p tsconfig.app.json --pretty false
git diff --check
```

- 25 个定向测试文件，395 项全部通过，包括图片／视频提交、批量、恢复、轮询、固定请求 ID 重试及此前打光功能。
- 本步修改的 TypeScript 文件 ESLint 通过；独立只读代码审查没有未解决的阻断项，diff 检查通过。
- 全项目类型检查仍失败，诊断与提取前完全一致：`@lumina/director-cli` 缺失及相关组件类型、`tidyCanvasLayout.test.ts` 的旧 Map 调用。本步文件没有新增类型诊断，不代表全仓类型通过。
- 未运行构建、未发起真实订单或付费请求，也未进行真实模型效果验收。

## 操作复测与下一步

本步是提交链路复用，没有新增可见操作。普通图片生成的单张／批量、故障和防重已由上述隔离 API 测试复测。`localhost:5174` 的图片节点仍可打开打光面板并操作本地预览，生成按钮保持能力门禁；不能用测试契约解锁生产生成。

下一步 L12：冻结打光任务，创建普通图片结果占位及来源连线，调用本步共享函数，再由原节点轮询结果。可以继续完成模拟接口下的显式提交、防重和快照隔离；真实生成仍需 L10 的服务协议确认及 L14 的效果联调。
