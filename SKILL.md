---
name: bilibili-browser
description: 快速获取 B 站（bilibili）视频信息与全文字幕/文稿，以及在 B 站搜索视频。凡是用户发来 B 站链接（bilibili.com/video/BV…）并要求"提取内容/梳理这个视频/拿字幕/转文稿/总结视频"，或要求"在 B 站搜索某类视频"时使用。这是实测最快最省步骤的路径，照做即可，不要另想方案。
---

# B 站视频信息快速提取

## 实测坑（先记住，少走弯路）

- **不要**用 curl/shell 直接调 `api.bilibili.com`——必触发 412 风控。所有 API 都要在浏览器页面内 `fetch(..., {credentials:'include'})`。
- `yt-dlp --cookies-from-browser chrome` 在本机解不开 Chrome cookie，别试。
- AI 字幕需要 B 站登录态；**搜索接口在 B 站域内页面里 fetch，未登录也能出结果**。
- 浏览器工具的 `filename` 存盘功能会把结果双重 JSON 编码：读文件后要再 `json.loads` 一次。

## 路径 A：基本信息（无需登录，几秒）

```bash
yt-dlp -J --no-download "https://www.bilibili.com/video/BVxxxx" > /tmp/bv.json
python3 -c "import json;d=json.load(open('/tmp/bv.json'));print(d['title'],d.get('uploader'),d.get('duration'),'秒');print(d.get('description','')[:800]);[print(round(c['start']),c['title']) for c in d.get('chapters') or []]"
```

可拿到：标题、UP 主、时长、简介、分章节、封面。无需任何凭证。

## 路径 B：全文字幕（需登录态，用会话内浏览器工具）

第 1 步：打开视频页，在页面执行 JS 拿字幕地址——

```js
const st = window.__INITIAL_STATE__;
const j = await (await fetch(`https://api.bilibili.com/x/player/wbi/v2?bvid=${st.bvid}&cid=${st.cid}`, {credentials:'include'})).json();
return (j.data?.subtitle?.subtitles || []).map(s => ({lan: s.lan, url: s.subtitle_url.startsWith('//') ? 'https:' + s.subtitle_url : s.subtitle_url}));
```

第 2 步：在页面 `fetch(字幕url)` → 返回 `j.body`，为 `[{from, to, content}]`（秒 + 文本）。拼成带时间轴文稿：

```js
return JSON.stringify((j.body||[]).map(x => [Math.round(x.from), x.content]));
```

登录态检查：页面内 `fetch('https://api.bilibili.com/x/web-interface/nav',{credentials:'include'})` → `data.isLogin`。为 false 时请用户在浏览器里扫码登录，或降级用路径 A 交付。

## 路径 C：搜索视频

在任意 B 站页面内执行（未登录可用）：

```js
const j = await (await fetch(`https://api.bilibili.com/x/web-interface/search/all/v2?keyword=${encodeURIComponent('关键词')}&page=1`, {credentials:'include'})).json();
const vids = j.data.result.flatMap(g => g.data || g.result || []).filter(x => x.bvid);
return vids.slice(0,10).map(x => ({bvid:x.bvid, title:x.title.replace(/<[^>]+>/g,''), author:x.author, dur:x.duration, play:x.play}));
```

结果按分区分组，视频组字段含 bvid/title/author/play/duration；`title` 带 `<em>` 高亮标签，记得剥掉。

## 路径 D：无字幕兜底

个别视频没有 AI 字幕（路径 B 第 1 步返回空数组）：`yt-dlp -x --audio-format m4a <url>` 下载音频，再用 whisper/faster-whisper 本地转写（本机未装则先装 small 模型）。

## 交付建议

字幕到手后按用户需求整理；文稿全文值得存档时，连同整理稿一起写入工作目录子文件夹，便于追溯。
