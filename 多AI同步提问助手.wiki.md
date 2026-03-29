# 多AI同步提问助手 - 功能实现跟踪 Wiki

> 更新时间：2026-03-29  
> 对应脚本：`多AI同步提问助手.js`  
> 当前版本：`v3.3-dev`

---

## 1. 文档目的

本 Wiki 用于**跟踪当前代码已实现功能**，不再只描述设计目标。  
每次脚本迭代时，优先更新以下内容：

1. 功能状态矩阵（是否已实现、是否稳定）
2. 关键模块与函数映射
3. 站点规则与差异化行为
4. 已知限制与下一步验证点

---

## 2. 功能状态矩阵（以当前代码为准）

| 模块 | 状态 | 当前实现 | 备注 |
|---|---|---|---|
| 文本同步主链路 | ✅ 已实现 | 网络拦截 `XHR/fetch` 提取 query 后广播 `SHARED_QUERY` | 主路径稳定 |
| 文本同步兜底 | ✅ 已实现 | 本地发送兜底：监听 `keydown/click` 判断发送行为 | 作为主链路补偿 |
| 资产同步入口 | ✅ 已实现 | 监听 `paste/drop` 并广播 `SHARED_ASSET` | 覆盖图片+通用文件 |
| 资产接收与消费 | ✅ 已实现 | 目标页按 `targetIds + freshness` 消费并注入 | 带幂等去重 |
| 资产注入策略（v3.3） | ✅ 已实现 | 站点级策略（paste/drop）+ DOM/网络联合校验 | 可在 UI 单独配置 |
| 推荐策略应用 | ✅ 已实现 | 首次安装自动应用推荐；设置页可一键重应用当前推荐 | 更新脚本默认保留本地存储策略 |
| 选择状态跨标签同步 | ✅ 已实现 | `SELECTION_SYNC_ENABLED` + `SHARED_SELECTION` | 可开关 |
| 活跃标签管理 | ✅ 已实现 | 心跳注册 + 过期清理 + URL 变化重注册 | UI 在线状态依赖该机制 |
| 调试日志体系 | ✅ 已实现 | 普通日志 + 资产链路日志双开关 | 菜单可切换 |
| 自动化回归脚本 | ⚠️ 部分完成 | 已有手工回归用例文档 | 尚未接入自动测试框架 |

---

## 3. 当前支持站点

脚本当前内置站点（`SITES`）：

- ChatGPT
- Claude
- Gemini
- AI Studio
- Qwen / 千问
- 元宝
- 豆包
- DeepSeek
- Kimi
- Grok

说明：站点由 `host + apiPaths + inputSelectors + queryExtractor` 四元组驱动。

---

## 4. 关键实现映射（函数级）

### 4.1 文本链路

- `deployNetworkInterceptor()`
  - 拦截 `XMLHttpRequest` 与 `fetch` 的 POST 请求。
  - 根据站点 `apiPaths` 命中后调用 `queryExtractor` 提取问题文本。

- `deployLocalSendFallback()`
  - 本地发送兜底（键盘 Enter / 点击发送按钮）。
  - 使用 `suppressNextLocalSendCaptureUntil` 防止回放注入时反向触发。

- `handleQueryFound()` / `processSharedQuery()` / `processSubmission()`
  - 负责 query 的广播、消费、输入框注入、发送动作触发。

### 4.2 资产链路

- `deployAssetSyncListeners()`
  - 捕获 `paste/drop` 文件来源。
  - 内容指纹去重：`makeDataUrlFingerprint()` + `recentAssetFingerprints`。

- `handleAssetFound()` / `processSharedAsset()`
  - 广播 `SHARED_ASSET`。
  - 接收端基于 `assetMessageId` 幂等处理，避免重复消费。

- `processAssetSubmission()`（v3.3 核心）
  - 按站点配置执行 `paste` 或 `drop` 注入。
  - 注入后通过 `takeAssetDomSnapshot()` 与 `isAssetLikelyAttached()` 做结果校验。
  - 若 DOM 未确认，则在窗口内等待网络上传信号补充校验。

- `onSiteInjectionStrategyChange()` / `onApplyRecommendedSiteStrategies()`
  - 支持单站点切换策略与一键应用推荐策略。
  - 写入 `ASSET_SITE_INJECTION_STRATEGIES` 并跨标签同步。

### 4.3 状态与 UI

- `registerTabAsActive()` / `cleanupStaleTabs()`：活跃标签心跳与清理。
- `updatePanelState()` / `updateFabBadge()`：控制面板与目标徽标状态。
- `onSelectionSyncToggleChange()`：选择状态同步开关。

---

## 5. 共享消息结构（当前实现）

### 5.1 文本消息（`SHARED_QUERY`）

```json
{
  "query": "...",
  "timestamp": 0,
  "sourceId": "CHATGPT",
  "targetIds": ["GEMINI", "CLAUDE"]
}
```

### 5.2 资产消息（`SHARED_ASSET`）

```json
{
  "assetMessageId": "SOURCE-TS-RAND",
  "timestamp": 0,
  "sourceId": "TONGYI",
  "targetIds": ["YUANBAO"],
  "assetCount": 1,
  "assets": [
    {
      "name": "example.png",
      "mimeType": "image/png",
      "size": 12345,
      "dataUrl": "data:image/png;base64,...",
      "origin": "paste"
    }
  ],
  "asset": { "...": "兼容单资产结构" }
}
```

---

## 6. 站点注入策略（当前内置）

默认/推荐策略（当前实现）：

- 推荐策略由 `ASSET_SITE_STRATEGY_RECOMMENDED` 与站点列表组合生成。
- 当前内置推荐为：`QWEN: drop`，其余站点默认 `paste`。

生效规则：

- 首次安装（本地无策略存储）时：自动写入并应用当前推荐策略。
- 脚本后续更新时：默认读取并保留本地已存策略，不自动覆盖。
- 点击“应用推荐注入策略”按钮时：按当前脚本版本推荐策略覆盖本地策略。

说明：可在设置页“站点注入策略（资产）”中逐站覆盖为 `paste` 或 `drop`。

---

## 7. 已知限制与风险

1. 部分站点对“非用户手势上传”限制严格，脚本注入可能被浏览器/站点策略拦截。
2. `dataUrl` 传输对大文件不友好，跨标签广播存在性能与延迟开销。
3. DOM 结构变化会影响输入框与上传入口定位（需持续维护 selector）。
4. 目前主要依赖手工回归（尚无自动化测试与 CI 验证）。

---

## 8. 回归清单（建议每次版本都执行）

### A. 文本链路

1. 任意站点发送文本，目标站点可自动注入并发送。
2. 网络请求结构变化场景下，本地兜底可触发同步。

### B. 资产链路

1. 图片 `paste`：目标站点仅新增 1 份附件，无重复。
2. 文件 `drop`：目标站点出现文件附件芯片/上传预览。
3. 站点策略切换：同一目标站点切换 `paste/drop` 后注入方式随之变化。
4. 通义 ↔ 元宝双向验证：图片与非图片均可按预期注入。

### C. 状态与控制面板

1. 目标模型勾选状态在“开启同步”时跨标签一致。
2. 关闭同步后，各标签页选择状态互不影响。
3. 活跃标签在线状态（online）能随页面关闭/超时变化。

---

## 9. 变更记录（简版）

- `v3.1`：资产去重与消费幂等初步完善。
- `v3.2`：升级为内容指纹去重，优化抑制窗口。
- `v3.2-dev`：增加资产链路调试日志。
- `v3.3-dev`：注入统一为 `paste/drop` 策略化执行，支持站点级配置与“一键应用推荐”。

> 详细故障复盘与排障记录请参考 `DEBUG笔记/多AI同步提问助手跨标签文本与图片同步失效问题/` 下文档。
