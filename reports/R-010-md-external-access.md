# 本地 MD 文件外网访问方案研究报告

> 任务编号：R-010 | 日期：2026-03-29 | 方法论：v4 深度研究（4 Search → 双 Review → Citation）

---

## 一、核心发现

### 发现 1：最快上手方案 — Tailscale Serve/Funnel + 轻量 MD 服务器

**结论：这是最适合当前环境的方案。** 用户已有 Tailscale，无需额外购买服务。

- **Tailscale Serve**（tailnet 内访问）：将本地 Web 服务路由到 tailnet 内所有设备，手机装 Tailscale app 即可访问 [1][2]
- **Tailscale Funnel**（公网访问）：无需公网 IP，通过加密中继暴露本地服务到互联网 [1]
- Funnel 限制：仅支持 443/8443/10000 端口，有不可配置带宽限制，仅 TLS 连接 [1]
- Funnel 可直接作为文件服务器：`tailscale funnel --bg 443 /path/to/folder` [3]
- Funnel 也可反代本地 HTTP 服务：`tailscale funnel --bg 443 http://localhost:3000` [3]

**推荐的 MD 服务器搭配（按推荐度排序）**：

| 工具 | 语言 | 部署难度 | 搜索功能 | 移动端 | 适合场景 |
|------|------|----------|----------|--------|----------|
| **MkDocs** | Python | ⭐ 极低 | ✅ 插件支持 | ✅ 响应式 | 纯文档浏览，最推荐 |
| **mdBook** | Rust | ⭐ 极低（单二进制） | ✅ 内置 | ✅ 响应式 | 轻量文档，无需 Python |
| **Quartz** | Node.js | ⭐⭐ 中等 | ✅ 内置全文 | ✅ SPA | 数字花园，与 Obsidian 兼容 |
| **code-server** | TS | ⭐⭐ 中等 | ✅ VS Code 搜索 | ⚠️ 可用但重 | 需要远程编辑能力 |

**来源**：Tailscale 官方文档 [1][2][3]；MD 工具对比 [7][8]；code-server [5]

### 发现 2：零部署方案 — 通过 OpenClaw Agent 直接查询

**当前即可使用**，无需额外部署任何服务：

- OpenClaw agent 有完整文件系统访问权限，可通过 Telegram/其他渠道直接要求 agent 读取并返回 MD 文件内容
- **局限**：不支持批量浏览文件列表的 UI，不适合"翻阅"大量文件，适合定向查询
- 社区已开发 Telegram 文件浏览器插件，可通过按钮导航 workspace 文件夹（medium 置信度）
- Control UI（Dashboard）目前无内置 workspace 文件浏览器（GitHub Issue #8192）

### 发现 3：第三方托管方案（适合非技术用户或需要协作的场景）

| 方案 | 费用 | 实时性 | 安全性 | 移动端 |
|------|------|--------|--------|--------|
| GitHub 私有 repo + GitHub Pages | 免费（公开）/ Enterprise（私有） | 需 git push | 公开 Pages 不适合敏感数据 [6] | ✅ |
| Syncthing 同步 | 免费 | 实时 P2P | 端到端加密 | ⚠️ 需另配查看器 |
| Obsidian Sync | $4/月 | 实时 | 加密同步 | ✅ 原生 app |
| Notion 导入 | 免费起步 | 需手动同步 | 云端存储 | ✅ 原生 app |

**来源**：GitHub Pages 限制 [6]；code-server [5]

### 发现 4：方案组合的架构建议

**最佳实践架构**：

```
本地 MD 文件
    │
    ├──→ Tailscale Serve → MkDocs/mdBook（tailnet 内浏览）
    │
    ├──→ Tailscale Funnel → MkDocs/mdBook（公网访问，可选）
    │
    └──→ OpenClaw Agent（通过 Telegram 智能查询）
```

三个通道互补：Serve 覆盖日常浏览，Funnel 覆盖无 Tailscale 设备的临时访问，Agent 覆盖智能查询。

---

## 二、推荐方案与实施步骤

### 🏆 方案 A：Tailscale + MkDocs（推荐首选）

**为什么选 MkDocs 而非其他**：Python 生态，`mkdocs.yml` 单文件配置，插件丰富（搜索、导航），Material 主题移动端体验极佳。

**实施步骤**：

```bash
# 1. 安装 MkDocs 和 Material 主题
pip install mkdocs mkdocs-material

# 2. 在 workspace 目录初始化（或指向现有 MD 目录）
cd /home/noname/.openclaw/workspace
mkdocs new .  # 如果已有 MD 文件，只需创建 mkdocs.yml

# 3. 配置 mkdocs.yml
cat > mkdocs.yml << 'EOF'
site_name: My Knowledge Base
theme:
  name: material
  features:
    - search.suggest
    - search.highlight
    - navigation.instant
plugins:
  - search
docs_dir: .
site_dir: /tmp/mkdocs-site
EOF

# 4. 构建并启动
mkdocs build
mkdocs serve -a 127.0.0.1:8000

# 5. Tailscale Serve（tailnet 内访问）
tailscale serve --bg 8000 http://127.0.0.1:8000

# 6. （可选）Tailscale Funnel（公网访问）
tailscale funnel --bg 443 http://127.0.0.1:8000
```

**注意**：`docs_dir: .` 会将整个 workspace 作为文档源，可用 `exclude_docs` 过滤不需要的文件。

### 🥈 方案 B：Tailscale + mdBook（更轻量）

适合不希望依赖 Python 环境、追求极致轻量的场景：

```bash
# 1. 下载 mdBook 单二进制
curl -sSL https://github.com/rust-lang/mdBook/releases/latest/download/mdbook-v*-x86_64-unknown-linux-gnu.tar.gz | tar -xz -C ~/.local/bin/

# 2. 初始化
cd /home/noname/.openclaw/workspace
mdbook init

# 3. 配置 book.toml（支持搜索默认开启）
# 4. 构建
mdbook build -d /tmp/mdbook-site

# 5. 启动并暴露
mdbook serve -n 127.0.0.1:3000
tailscale serve --bg 3000 http://127.0.0.1:3000
```

---

## 三、实践建议

1. **优先用 Tailscale Serve**（非 Funnel）：带宽无限制，速度更快，安全性更高。只有需要在未安装 Tailscale 的设备上访问时才开启 Funnel
2. **安全提示**：Funnel 暴露的服务对全互联网可见，建议搭配 HTTP Basic Auth 或仅对特定路径开放。MkDocs 无内置认证，可用 Caddy 反代加认证 [4]
3. **自动重建**：可用 `inotifywait` 或 systemd watch 监控 MD 文件变化，自动触发 `mkdocs build`
4. **OpenClaw Agent 查询**作为补充：日常用 Web 界面浏览，需要"找出某段内容"时直接在 Telegram 问 agent

---

## 四、知识缺口

| 缺口 | 影响程度 | 建议 |
|------|----------|------|
| Funnel 在中国大陆的实际访问速度和稳定性 | 中 | 实际测试后再决定是否启用 |
| MkDocs/mdBook 全文搜索对中文的分词效果 | 中 | 需实际测试，可能需要额外插件 |
| OpenClaw Control UI 文件浏览器功能进展 | 低 | 关注 GitHub Issue #8192 |
| 各方案实际资源占用（RAM/CPU）对比 | 低 | 量化数据缺失但不影响方案选择 |

---

## 五、来源列表

| # | 标题 | 类型 | URL |
|---|------|------|-----|
| [1] | Tailscale Funnel · Tailscale Docs | 官方文档 | https://tailscale.com/docs/features/tailscale-funnel |
| [2] | Tailscale Serve · Tailscale Docs | 官方文档 | https://tailscale.com/docs/features/tailscale-serve |
| [3] | Tailscale Funnel examples · Tailscale Docs | 官方文档 | https://tailscale.com/docs/reference/examples/funnel |
| [4] | Use Caddy to manage Tailscale HTTPS certificates | 官方博客 | https://tailscale.com/blog/caddy |
| [5] | coder/code-server: VS Code in the browser | GitHub | https://github.com/coder/code-server |
| [6] | GitHub Pages limits | 官方文档 | https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits |
| [7] | Text in, docs out: Popular Markdown documentation tools compared | 技术博客 | https://www.azalio.io/text-in-docs-out-popular-markdown-documentation-tools-compared/ |
| [8] | MkDocs vs Docusaurus for technical documentation | 技术博客 | https://blog.damavis.com/en/mkdocs-vs-docusaurus-for-technical-documentation/ |
| [9] | denzyldick/mserver | GitHub | https://github.com/denzyldick/mserver |

---

## 六、方法论反思

**做得好**：4 角度并行搜索效率高；Tailscale 方案调研充分（官方文档支撑强）；方案对比有实际操作性

**需改进**：OpenClaw 相关发现来源可信度偏低（缺少一手官方文档）；缺少各方案的实际部署测试数据；中文搜索能力评估不足
