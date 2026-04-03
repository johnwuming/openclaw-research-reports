# R-016：本地 Markdown 报告网页实时展示方案设计

> 日期：2026-03-30 | 类型：技术方案 | 状态：已完成

## 一、需求概述

将 `/home/noname/.openclaw/workspace/shared/results/` 中的 Markdown 报告通过局域网网页展示，供手机端快速浏览。

核心要求：只读、手机友好、实时更新、中文渲染、轻量部署。

## 二、方案对比

### 方案 A：自写 Node.js 单文件服务（⭐ 推荐）

**技术栈**：Node.js + Express + marked + highlight.js + chokidar + SSE

| 维度 | 评价 |
|------|------|
| 复杂度 | 极低（单文件 ~150 行 JS） |
| 依赖 | express, marked, highlight.js, chokidar（4 个 npm 包） |
| 实时性 | ✅ 通过 SSE（Server-Sent Events）推送文件变更，无需手动刷新 |
| 手机适配 | ✅ 内嵌响应式 CSS，自定义阅读样式 |
| 中文支持 | ✅ marked 原生支持 UTF-8 |
| 构建步骤 | 无，直接运行 |
| 可定制性 | 最高——完全可控 |

**工作原理**：
1. Express 提供路由：`/` 列出文件列表，`/view/:file` 渲染单个文件
2. `marked` 将 Markdown 转 HTML，`highlight.js` 处理代码高亮
3. `chokidar` 监听目录变更，通过 SSE 推送通知浏览器自动刷新
4. 内嵌 CSS 适配移动端（大字号、合适行距、表格横向滚动）

### 方案 B：md-serve（npm 包）

**技术栈**：npm 全局包 `md-serve`

| 维度 | 评价 |
|------|------|
| 复杂度 | 最低（零配置，一条命令启动） |
| 实时性 | ⚠️ 部分版本支持 live reload，需确认 |
| 手机适配 | ⚠️ 基础渲染，无专门移动端优化 |
| 可定制性 | 低 |
| 目录列表 | ⚠️ 需确认是否支持 |

```bash
npx md-serve /path/to/results -p 3000
```

**优点**：开箱即用，零配置。**缺点**：功能受限，实时性和移动端体验不可控。

### 方案 C：Python + Flask/FastAPI

**技术栈**：Python3 + Flask + markdown + watchdog

| 维度 | 评价 |
|------|------|
| 复杂度 | 中等 |
| 实时性 | ✅ watchdog + SSE 可实现 |
| 手机适配 | ✅ 可自定义 |
| 可定制性 | 高 |
| 部署 | 无需 npm，但 Python 虚拟环境管理稍繁琐 |

**评价**：功能与方案 A 等价，但 Node.js 与已有环境（OpenClaw）更契合。

### 对比总结

| | 方案 A（自写 Node） | 方案 B（md-serve） | 方案 C（Python） |
|---|---|---|---|
| 部署难度 | 中（需写代码） | 低（一条命令） | 中 |
| 实时性 | ✅ SSE | ⚠️ | ✅ |
| 移动端 | ✅ 优化 | ❌ | ✅ 优化 |
| 可控性 | 最高 | 最低 | 高 |
| 维护成本 | 低 | 最低 | 低 |
| **推荐度** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |

## 三、推荐方案详细设计（方案 A）

### 3.1 目录结构

```
~/.openclaw/md-viewer/
├── server.js          # 主服务文件
├── package.json       # 依赖声明
└── public/
    └── style.css      # 可选：分离样式
```

### 3.2 安装步骤

```bash
# 1. 创建目录
mkdir -p ~/.openclaw/md-viewer
cd ~/.openclaw/md-viewer

# 2. 初始化并安装依赖
npm init -y
npm install express marked highlight.js chokidar

# 3. 创建 server.js（见下方代码）
```

### 3.3 server.js 核心代码

```javascript
const express = require('express');
const fs = require('fs');
const path = require('path');
const { marked } = require('marked');
const hljs = require('highlight.js');
const chokidar = require('chokidar');

const app = express();
const PORT = 3000;
const MD_DIR = '/home/noname/.openclaw/workspace/shared/results';

// SSE clients
const clients = new Set();

// Configure marked with highlight.js
marked.setOptions({
  highlight: function(code, lang) {
    if (lang && hljs.getLanguage(lang)) {
      return hljs.highlight(code, { language: lang }).value;
    }
    return hljs.highlightAuto(code).value;
  },
  gfm: true,
  breaks: true
});

// File watcher
chokidar.watch(MD_DIR, { ignoreInitial: true }).on('all', (event, filePath) => {
  const rel = path.relative(MD_DIR, filePath);
  const msg = `event: ${event}\ndata: ${JSON.stringify({ file: rel, event })}\n\n`;
  clients.forEach(res => res.write(msg));
});

// SSE endpoint
app.get('/sse', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });
  clients.add(res);
  req.on('close', () => clients.delete(res));
});

// List files
app.get('/', (req, res) => {
  const files = fs.readdirSync(MD_DIR)
    .filter(f => f.endsWith('.md'))
    .sort()
    .reverse(); // newest first

  res.send(`<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>研究报告</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body { font-family: -apple-system, "PingFang SC", "Microsoft YaHei", sans-serif;
         background: #f5f5f5; color: #333; padding: 16px; max-width: 800px; margin: 0 auto; }
  h1 { font-size: 1.4em; margin-bottom: 16px; padding-bottom: 8px; border-bottom: 2px solid #007aff; }
  .file-list { list-style: none; }
  .file-list li { margin-bottom: 8px; }
  .file-list a { display:block; padding:12px 16px; background:#fff; border-radius:8px;
                 text-decoration:none; color:#007aff; font-size:1em;
                 box-shadow: 0 1px 3px rgba(0,0,0,0.08); }
  .file-list a:active { background:#e8f0fe; }
  .file-list .meta { color:#999; font-size:0.85em; margin-top:4px; }
</style>
</head>
<body>
<h1>📄 研究报告</h1>
<ul class="file-list">
${files.map(f => {
  const stat = fs.statSync(path.join(MD_DIR, f));
  const size = (stat.size / 1024).toFixed(1);
  const date = stat.mtime.toLocaleDateString('zh-CN');
  return `<li><a href="/view/${encodeURIComponent(f)}">${f}
    <div class="meta">${date} · ${size}KB</div></a></li>`;
}).join('')}
</ul>
<script>
  const es = new EventSource('/sse');
  es.onmessage = (e) => { /* new file added, refresh list */ location.reload(); };
  es.addEventListener('add', () => location.reload());
</script>
</body></html>`);
});

// View single file
app.get('/view/:name', (req, res) => {
  const filePath = path.join(MD_DIR, req.params.name);
  if (!filePath.startsWith(MD_DIR) || !fs.existsSync(filePath)) {
    return res.status(404).send('文件不存在');
  }
  const md = fs.readFileSync(filePath, 'utf-8');
  const html = marked.parse(md);
  const fileName = req.params.name;

  res.send(`<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>${fileName}</title>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css">
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body { font-family: -apple-system, "PingFang SC", "Microsoft YaHei", sans-serif;
         background:#fff; color:#333; padding:16px; max-width:800px; margin:0 auto;
         font-size:16px; line-height:1.8; }
  .nav { margin-bottom:16px; padding-bottom:12px; border-bottom:1px solid #eee; }
  .nav a { color:#007aff; text-decoration:none; font-size:0.95em; }
  h1,h2,h3 { margin:1.2em 0 0.6em; line-height:1.3; }
  h1 { font-size:1.5em; } h2 { font-size:1.3em; } h3 { font-size:1.1em; }
  p { margin-bottom:0.8em; }
  code { background:#f0f0f0; padding:2px 6px; border-radius:3px; font-size:0.9em; }
  pre { background:#f6f8fa; padding:12px; border-radius:6px; overflow-x:auto;
        margin-bottom:1em; font-size:0.9em; line-height:1.5; }
  pre code { background:none; padding:0; }
  table { border-collapse:collapse; width:100%; margin-bottom:1em; font-size:0.9em; }
  th,td { border:1px solid #ddd; padding:8px 10px; text-align:left; }
  th { background:#f6f8fa; }
  blockquote { border-left:3px solid #007aff; padding-left:12px; color:#666; margin-bottom:1em; }
  img { max-width:100%; }
  a { color:#007aff; }
  .toast { position:fixed; bottom:20px; left:50%; transform:translateX(-50%);
           background:#333; color:#fff; padding:8px 20px; border-radius:20px;
           font-size:0.9em; opacity:0; transition:opacity 0.3s; }
  .toast.show { opacity:1; }
</style>
</head>
<body>
<div class="nav"><a href="/">← 返回目录</a> · ${fileName}</div>
${html}
<div class="toast" id="toast">文件已更新，点击刷新</div>
<script>
  const toast = document.getElementById('toast');
  const es = new EventSource('/sse');
  const currentFile = ${JSON.stringify(fileName)};
  es.addEventListener('change', (e) => {
    const data = JSON.parse(e.data);
    if (data.file === currentFile) {
      toast.classList.add('show');
      toast.onclick = () => location.reload();
    }
  });
  es.addEventListener('add', (e) => {
    const data = JSON.parse(e.data);
    if (data.file === currentFile) location.reload();
  });
</script>
</body></html>`);
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Markdown viewer running at http://0.0.0.0:${PORT}`);
});
```

### 3.4 启动与测试

```bash
cd ~/.openclaw/md-viewer
node server.js
# 手机浏览器访问 http://192.168.3.x:3000
```

### 3.5 设置为 systemd 用户服务（开机自启）

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/md-viewer.service << 'EOF'
[Unit]
Description=Markdown Report Viewer
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/noname/.openclaw/md-viewer
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=default.target
EOF

# 启用并启动
systemctl --user daemon-reload
systemctl --user enable md-viewer.service
systemctl --user start md-viewer.service

# 检查状态
systemctl --user status md-viewer.service
```

> **注意**：确保 `loginctl enable-linger noname` 已执行，否则用户服务在注销后不会保持运行。

### 3.6 手机访问

1. 手机连接同一 WiFi（192.168.3.x 网络）
2. 浏览器访问 `http://<服务器IP>:3000`
3. 建议添加到手机主屏幕（"添加到主屏幕"），体验类似原生 App

### 3.7 安全考虑

**当前方案（局域网可信环境）**：
- UFW 未启用，但局域网隔离，风险可接受
- 只读服务，无写入/上传功能
- 路径遍历已防护（检查路径前缀）

**如需增强（可选）**：
- 在 Express 中添加简单的 Bearer Token 认证：
  ```javascript
  app.use((req, res, next) => {
    if (req.query.token !== 'your-secret') return res.status(401).send('Unauthorized');
    next();
  });
  ```
- 手机访问时带上 `?token=your-secret`

### 3.8 集成到 OpenClaw

**方式一：init.sh 添加**
```bash
# 在 init.sh 中添加
if ! systemctl --user is-active --quiet md-viewer.service 2>/dev/null; then
  systemctl --user start md-viewer.service
fi
```

**方式二：仅手动管理**
由于已设置 systemd 用户服务开机自启，无需额外集成。

## 四、备选方案（方案 B：md-serve 快速启动）

如需最快上手，不关心实时性和移动端优化：

```bash
# 一键启动
npx md-serve /home/noname/.openclaw/workspace/shared/results -p 3000
```

适合快速验证，后续可迁移到方案 A。

## 五、实施清单

- [ ] 创建 `~/.openclaw/md-viewer/` 目录
- [ ] `npm install express marked highlight.js chokidar`
- [ ] 创建 `server.js`
- [ ] 测试访问 `http://localhost:3000`
- [ ] 获取服务器局域网 IP（`ip addr | grep 192.168`）
- [ ] 手机测试访问
- [ ] 配置 systemd 用户服务
- [ ] （可选）添加到手机主屏幕

## 六、总结

推荐 **方案 A（自写 Node.js 服务）**：4 个 npm 依赖，单文件约 150 行，支持 SSE 实时推送、移动端优化、中文渲染、代码高亮。配合 systemd 用户服务实现开机自启，手机通过局域网 IP + 端口即可访问。整体部署时间约 10 分钟。
