---
name: "session-artifacts"
description: "Generates a notes bundle from web articles or session content: full notes MD (with images), quick-reference MD, and interactive markmap HTML, saved under notes/<topic>/ with cross-links. Invoke when user asks to save/organize notes from an article URL, generate a mindmap, or produce review-friendly artifacts from a session."
---

# Session Artifacts（会话产物整理）

把一次会话中产生的内容（通常是网页文章/课程）整理为三件可长期复习的产物，统一存放、互相链接。

## 产物清单（三件套）

| 文件 | 用途 | 命名 |
|---|---|---|
| 完整笔记 MD | 深入学习，保留原文结构、图片、留言区精华 | `<主题>-完整版.md` |
| 速览版 MD | 复习首选，5 分钟过完，只留重点表格和金句 | `<主题>-速览版.md` |
| 交互脑图 HTML | 结构化记忆，markmap 渲染，可折叠展开 | `<主题>-脑图.html` |

存放结构（工作目录为根）：

```
notes/<topic-slug>/
├── <主题>-完整版.md
├── <主题>-速览版.md
├── <主题>-脑图.html
└── images/
    └── 01-xxx.jpg ... (按原文出现顺序编号)
```

## 工作流程

### 第 1 步：抓取正文

1. 先试 `WebFetch`（快）。成功则直接进入第 2 步。
2. WebFetch 失败（付费墙/登录墙如极客时间、需 JS 渲染）→ 用浏览器自动化：
   - `browser_navigate` 打开 URL → `browser_wait_for({time: 3})` → `browser_evaluate` 提取：
   ```js
   JSON.stringify({
     title: document.querySelector('h1')?.textContent?.trim() ?? '',
     article: document.querySelector('article')?.innerText ?? document.body.innerText
   })
   ```
   - 输出截断时，完整内容会写入临时日志文件，可用 Read 工具读取 `Temp\trae\browser-logs\evaluate-*.log`。

### 第 2 步：提取并下载图片

用 `browser_evaluate` 提取图片及其**上下文**（前/后兄弟节点文本），用于确定每张图在笔记中的位置：

```js
JSON.stringify(Array.from(document.querySelectorAll('img'))
  .filter(i => i.src.startsWith('https'))
  .map(i => ({
    s: i.src, w: i.naturalWidth, h: i.naturalHeight,
    prevText: i.parentElement?.previousElementSibling?.innerText?.slice(0, 50) ?? '',
    nextText: i.parentElement?.nextElementSibling?.innerText?.slice(0, 50) ?? ''
  })))
```

过滤规则：
- 排除头像（路径含 `/account/avatar/` 或 `x-oss-process=resize` 且宽高 ≤ 64）
- 排除 data: URI 内嵌图（多为图标/logo）
- 保留宽 ≥ 300 的内容配图

下载（PowerShell，需带 UserAgent 防盗链）：

```powershell
$dir = "notes/<topic-slug>/images"; New-Item -ItemType Directory -Force -Path $dir | Out-Null
Invoke-WebRequest -Uri "<url>" -OutFile (Join-Path $dir "01-name.jpg") -UserAgent "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
```

按原文出现顺序编号：`01-`、`02-`…，文件名用英文语义短词（如 `04-dual-axis-framework.jpg`）。

### 第 3 步：生成完整版 MD

结构模板：

```markdown
# <文章标题>

> 课程/来源 · 作者 · 核心命题一句话 · 原文链接
> 📝 相关笔记：[速览版](xxx-速览版.md) · 🧠 [在浏览器中打开脑图](http://127.0.0.1:8765/notes/<slug>/%E8%84%91%E5%9B%BE.html) · 或双击本地文件 [xxx-脑图.html](xxx-脑图.html)

![配图](images/01-xxx.jpg)   ← 按第 2 步的 prevText/nextText 放回原文位置

## 一、<章节>       ← 忠实原文结构，用表格转化对比性内容
## 二、<章节>
...
## 留言区精华        ← 作者亲自回复的补充价值内容（如有）
## 一句话总结
---
*笔记生成时间：<日期> · 图片版权归属原出处，仅作个人学习收藏*
```

要点：
- 对比型论述（传统 vs Agent、三代模式语言）转成**表格**
- 金句、底层判断用**引用块**突出
- 原文的图必须放回**原文对应段落位置**，并加一句图注说明

### 第 4 步：生成速览版 MD

原则：**一屏一主题，只留不可遗忘的东西**。

- 开头 3 个链接（完整版、脑图、来源）
- "三个核心判断"类开门见山
- 反直觉内容做"你以为 vs 实际"对照表
- 核心框架用 ASCII 图或极简表
- 大清单压缩成"一行一概念"表
- 收尾落在最凝练的纲领句上

### 第 5 步：生成脑图 HTML

用 markmap CDN 模板（无需构建工具）：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title><主题> 脑图</title>
<style>
  body { margin: 0; }
  svg.markmap { width: 100%; height: 100vh; }
  .title { position: fixed; top: 12px; left: 16px; font-size: 14px; color: #666; z-index: 10; background: rgba(255,255,255,.8); padding: 4px 10px; border-radius: 6px; }
</style>
<script src="https://cdn.jsdelivr.net/npm/markmap-autoloader@0.18"></script>
</head>
<body>
<div class="title">…（点击节点可折叠/展开）</div>
<div class="markmap">
<script type="text/template">
---
markmap:
  maxWidth: 320
  initialExpandLevel: 3
  colorFreezeLevel: 2
---

# <根节点>
## <一级分支>
- <要点>
</script>
</div>
</body>
</html>
```

脑图内容组织：按"核心命题 → 问题 → 观点 → 框架 → 结构 → 目标 → 总结"的层次；节点短语化，单个节点不超过 ~20 字；用 emoji 区分分支类型。

### 第 6 步：互链与本地预览

**互链规则（踩过的坑，务必遵守）**：
- 中文文件名在 HTTP URL 中**必须 URL 编码**（如 `%E8%84%91%E5%9B%BE.html`）
- 笔记中的脑图入口必须写成**真正的超链接** `[文字](http://...)`，不要用反引号代码块包 URL（不可点击）
- 同时提供两个入口：HTTP 链接（Trae 内置浏览器）+ 相对路径本地链接（双击打开）

**本地预览服务**（file:// 在 Trae 内置浏览器被拦截，需 HTTP 中转）：

```powershell
python -m http.server 8765 --bind 127.0.0.1   # 工作目录为根，run_in_background
```

验证：`Invoke-WebRequest http://127.0.0.1:8765/notes/<slug>/<URL编码后的文件名>.html` 返回 200。

在 Trae 内置浏览器打开：`browser_navigate({ url: "http://127.0.0.1:8765/notes/<slug>/<urlencoded>.html", newTab: true })`。

若 navigate 未生效（页面停留在旧 tab），加 `newTab: true` 重试。

### 第 7 步：提交并推送到 GitHub

工作目录本身是 git 仓库，remote 为 SSH 地址（公钥已绑定，无需再登录）。

```powershell
git add notes/<topic-slug>/; git commit -m "Add <主题> notes with mindmap and images"; git push
```

要点：
- PowerShell 5 不支持 `&&`，用 `;` 串联命令
- 只 add 本次主题目录 `notes/<topic-slug>/`，避免混入无关文件
- commit message 风格：`Add <主题> notes with mindmap and images`（中英混合即可）
- git 输出中文件名显示为 `\345\274\200...` 是中文转义，**不是错误**
- LF/CRLF warning 无害，忽略即可
- push 认证失败（`Permission denied (publickey)`）→ 提醒用户把 `~/.ssh/id_ed25519.pub` 添加到 GitHub Settings → SSH keys
- 首次在新仓库操作时先 `git status` 确认分支为 `main` 且 remote 指向 `git@github.com:zlzdmv2022/growth.git`

## 完成检查

- [ ] notes/<topic-slug>/ 下三件套齐全
- [ ] images/ 中图片下载成功且被完整版引用（相对路径 images/xx）
- [ ] 完整版 ↔ 速览版 ↔ 脑图 三者头部互链可达
- [ ] 脑图 HTTP 地址返回 200 且渲染出节点
- [ ] 已提交并推送：`git status` 显示 working tree clean，本地与 origin/main 同步
- [ ] 向用户汇报目录树 + 各文件用途一句话
