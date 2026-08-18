---
name: xhs-fav-export
description: 把小红书 Web 端当前登录用户「收藏」tab 的收藏笔记批量导出为本地 Markdown（每篇含标题、原文链接、正文、图片/视频 URL），支持 offset/count 分批断点续导。当用户需要导出/备份/检索自己的小红书收藏内容时唤起。
---

# xhs-fav-export

把小红书 Web 端**当前登录用户**自己主页「收藏」tab 里的收藏笔记，批量导出为本地 Markdown 文件。每篇一个 md，包含标题、原文链接（可跳转原文）、正文文字、图片/视频 URL（只存 URL 不下载），最后生成 `summary.json` 汇总。

## 前置条件

- wc3-chrome 扩展 + Relay（`http://127.0.0.1:3459`）已启动（`extensionConnected` 为 true）
- 已在 Chrome 中登录小红书（收藏 tab 依赖登录态；未登录会明确报错）
- 本 skill 全程确定性代码（Extension Relay 操作），**不依赖**本地 pipeline 子会话服务
- 需要 Node.js 22+

## 使用方式

```bash
node skill.mjs [input.json]
```

`input.json`：

```json
{
  "offset": 0,
  "count": 10,
  "outputDir": "/absolute/path/to/output"
}
```

- `offset`（number，默认 0）：从第几篇收藏笔记开始导出（收藏 tab 默认按收藏时间倒序，最新在前，脚本按页面实际顺序取数）
- `count`（number，默认 10）：本次导出的收藏笔记数量上限
- `outputDir`（string，默认当前目录下 `xhs-fav-export-output/`）：md 输出目录
- `profileUrl`（string，可选）：当前登录用户主页 URL（如 `https://www.xiaohongshu.com/user/profile/<uid>`）。不传时脚本自动探测；探测失败或拿到空收藏时传它最稳

不传 `input.json` 时按默认值执行；传了 `input.json` 路径则从中读取。

## 流程说明

1. 打开小红书首页，自动探测当前登录用户主页——收藏 tab 必须是自己主页
2. 打开 `<profile>?tab=collection`，校验收藏视图（无卡片 = 未登录 / 非自己主页，明确报错）
3. 若卡片不足 `offset+count`，增量滚动触发懒加载（分步 `scrollBy(0,800)`，不一次性滚到底；后台 tab 懒加载可能受限，一次运行不一定加载全量收藏）
4. 缓存卡片列表，**串行逐篇**：点击 `a.cover` 打开详情弹层 → 提取标题/正文/图片/视频 → 立即写 md 文件 → `history.back()` 关闭弹层
   - 串行逐篇是硬约束：多 tab 并行会触发反爬
   - 逐篇立即落盘：中途扩展断连只丢当前一篇，重建 tab 后自动继续
5. 写 `summary.json`

## 输出

`<outputDir>/` 下：

- `<序号>_<笔记名>.md`——每篇一个，序号从 offset 延续递增，笔记名 = 标题去非法字符、截断 50 字符、空则 `untitled`；仅标题+链接（正文/图片/视频全缺）的文件名追加 ` (partial)`
- `summary.json`——`{ total_exported, offset, count, skipped_partial: [note_id...] }`

单篇 md 格式：

```
# <标题>
原文链接: <https://www.xiaohongshu.com/explore/<note_id>?xsec_token=...>
收藏于: <收藏列表顺序序号>
---
<正文文字>
---
图片: <图片URL1> <图片URL2> ...
视频: <视频URL>（若有）
```

stdout 只输出一行 JSON：`{ status: "success"|"partial"|"failed", summary, output_dir }`。

## 注意事项

- 收藏 tab 懒加载依赖真实可见滚动，一次运行可能加载不全全部收藏；用 `offset/count` 分批、断点续导即可覆盖全部
- 媒体只存 URL（图片/视频 CDN 裸请求可访问），不下载本地
- 个别篇提取失败会自动重试后跳过并计入 summary 的 `failures`；`status` 分级：全成功 `success`、有失败或 partial 笔记 `partial`、零导出 `failed`
