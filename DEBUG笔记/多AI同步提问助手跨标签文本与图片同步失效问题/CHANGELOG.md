# Changelog

## 2026-03-28 (迭代次轮)
- 本次升级至 v3.2-dev 及 v3.3-dev，解决由于盲目依赖 `dispatchEvent` 致误判和图片/常规文件策略混用的问题。
- 新增两阶段注入（注水 -> 等待 -> 核对 DOM DOM附件快照），不轻信返回值。
- 分离策略通道（图片独享 `enablePasteForImage` 等；非图片启用 `enableFileInputForNonImage`）。
- 增加了针对“发后无回显需点按钮”等场景特化的 `discoverFileInputViaTrigger` 逻辑。
- 确立 `README.md` 内的“最小用例 A/B/C”以便针对双站（通义、元宝）打磨验证。

## 2026-03-28 (初代修复构建)
- 新建调试归档目录与主复盘文档。
- 记录两轮修复后的最终方案：
  - 文本：网络拦截 + 本地发送事件兜底。
  - 资产：图片先 paste 再 file input 回退。
  - 状态锁：文本/资产双锁拆分，避免互斥丢任务。
- 新增归档 README 索引，规范后续迭代入口。
