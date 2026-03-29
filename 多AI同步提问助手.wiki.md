# 多AI同步提问助手 - 集成设计 Wiki

## 1. 现状对比与问题分析

### 脚本A：AI 对话助手(一键同步多模型).js
- 优势：文本同步链路稳定。
- 机制：通过拦截 `XMLHttpRequest` / `fetch` 请求体提取用户问题，再通过 GM 共享存储广播给目标标签页。
- 结论：文本同步“触发源”可靠（基于实际发送请求），比监听键盘/鼠标事件更稳。

### 脚本B：多模型同时回答 & 目录导航.js
- 优势：已实现图片同步（监听粘贴图片并跨标签页分发，再在目标页模拟粘贴）。
- 文本不同步主要原因（根因）
  1. 文本触发依赖 UI 事件推断（`keydown/mousedown/mouseup/input`），站点改版后极易失效。
  2. 输入框被框架频繁替换，监听器容易丢失，虽然有重挂载兜底，但仍存在窗口期。
  3. 发送行为判断依赖“输入框是否清空”等间接信号，不如网络层“已发请求”可靠。
  4. 不同站点 Enter/组合键/按钮发送逻辑差异大，DOM 事件方案维护成本高。

> 结论：文本同步应保留脚本A的网络拦截方案；图片/文件同步能力从脚本B迁移并增强。

---

## 2. 集成目标

以 `AI 对话助手(一键同步多模型).js` 为主干，新增“图片/文件同步”模块，生成新脚本：

- 新文件名：`多AI同步提问助手.js`
- 不改动两个源文件。
- 功能目标：
  1. 文本同步（沿用脚本A稳定链路）
  2. 图片同步（粘贴图跨标签页同步）
  3. 文件同步（文件选择/拖拽跨标签页同步，优先走原生 `<input type=file>` 注入）

---

## 3. 整合方案（实现级）

### 3.1 新增共享消息通道
在脚本A的 GM Key 中新增：
- `SHARED_ASSET`: 统一承载图片/文件同步消息

消息结构建议：
```json
{
  "timestamp": 0,
  "sourceId": "CHATGPT",
  "targetIds": ["GEMINI", "CLAUDE"],
  "asset": {
    "name": "xxx.png",
    "mimeType": "image/png",
    "size": 12345,
    "dataUrl": "data:image/png;base64,...",
    "origin": "paste|drop|file-input"
  }
}
```

### 3.2 本地捕获（发送端）
新增三个捕获入口：
1. `paste`：抓取 `clipboardData.items` 里的文件（重点是图片）
2. `drop`：抓取拖拽上传的文件
3. `change`：抓取 `<input type=file>` 选择文件

捕获后统一转 DataURL，写入 `SHARED_ASSET`。

### 3.3 远端注入（接收端）
- 监听 `SHARED_ASSET` 变化，按 `targetIds` 与 freshness 校验后消费。
- 注入顺序：
  1. 优先找到可用 `input[type=file]`，通过 `DataTransfer` 设置 `files` 并触发 `change`。
  2. 若为图片且 file input 不可用，回退到对输入框触发 `ClipboardEvent('paste')`。

### 3.4 循环广播防护
新增状态位：
- `isApplyingRemoteAsset`：接收端执行注入期间置为 `true`。

并在本地事件捕获时跳过：
- `!event.isTrusted` 的事件（脚本模拟事件）
- `isApplyingRemoteAsset === true`

避免“收到远端注入后再次广播”造成回环。

---

## 4. 风险与兼容性说明

1. 部分站点上传仅接受原生用户手势，脚本注入可能受限（浏览器安全策略）。
2. DataURL 体积较大时 GM 存储写入耗时增加（大文件建议限制大小）。
3. 不同站点 DOM 改版可能导致 file input 选择器变化，需保留 fallback。

---

## 5. 本次改造步骤（执行顺序）

1. 生成本 Wiki（当前已完成）
2. 复制 `AI 对话助手(一键同步多模型).js` 为 `多AI同步提问助手.js`
3. 在新文件中新增“图片/文件同步模块”与对应初始化
4. 快速自检语法与关键逻辑
5. 输出改造结果与后续验证建议
