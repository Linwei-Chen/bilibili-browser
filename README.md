# bilibili-browser

一个面向 AI Agent 的极简技能（skill）：以最短路径获取 B 站（bilibili）视频信息与全文字幕，并支持在 B 站搜索视频。

## 能做什么

- **基本信息**（无需登录）：标题、UP 主、时长、简介、分章节
- **全文字幕/AI 文稿**（需浏览器内 B 站登录态）：带时间轴的完整转写
- **搜索视频**：关键词搜索，返回 bvid / 标题 / UP 主 / 时长 / 播放量
- **无字幕兜底**：yt-dlp 下载音频 + whisper 本地转写

全部路径均在 macOS 实测通过，skill 里同时记录了已踩过的坑（curl 直调 API 触发 412 风控、Chrome cookie 无法被 yt-dlp 解密、存盘双重 JSON 编码等），让模型照着做即可，不走弯路。

## 安装

将本目录放入 agent 的技能目录即可，例如：

```bash
git clone https://github.com/Linwei-Chen/bilibili-browser.git ~/.agents/skills/bilibili-browser
```

## 依赖

- [yt-dlp](https://github.com/yt-dlp/yt-dlp)（路径 A / D）
- 会话内浏览器工具（Playwright MCP 或 browser-use 等，路径 B / C 需要能在页面内执行 JS）
- 可选：whisper / faster-whisper（路径 D）

## 使用

把 `SKILL.md` 的 description 作为触发条件：用户发来 `bilibili.com/video/BV…` 链接要求提取内容/字幕/梳理，或要求在 B 站搜索视频时，agent 会自动加载本技能并按四条路径执行。
