> [!IMPORTANT]
> **本项目已合并至 [wechat2knowledge](https://github.com/bonboruyau-dev/wechat2knowledge)**（公众号文章转知识库：Markdown + 飞书二合一）。本仓库已归档，仅作历史参考，请前往新仓库获取最新版本。

# HTML → 飞书文档 · html-to-feishu-doc

> 把任意 HTML 网页或 `.html` 文件，转换成一篇飞书云文档：表格保留、图片自动上传、大标题层级还原、含完整性回查与定点修复。可与「微信公众号转 Markdown」串联，形成「抓取 → 飞书存档」闭环。
>
> Convert any HTML page or file into a Feishu (Lark) cloud document: tables preserved, images auto-uploaded, heading hierarchy restored, with integrity check & surgical fix.

[English version](#english)

---

## ✨ 功能特性 / Features

- 🌐 **HTML / URL 双输入**：直接喂网页链接或本地 `.html` 文件
- 📊 **表格 → 飞书原生表格**，合并单元格自动展开（AI / RAG 友好）
- 🖼️ **图片下载 + 上传**：本地 ASCII 命名 `img_NNN.png`，文档创建时由飞书自动上传
- 🅷 **大标题层级还原**：识别「一、背景」「结语」等样式伪装标题，提升为 `##`
- 🔧 **图片格式归一化**：按 magic bytes 纠错扩展名，SVG 用 Chromium 渲染成真 PNG（飞书按内容校验，否则整篇创建失败）
- ✅ **完整性回查（Step 5）**：原文 vs 文档逐行比对，防止整段丢失
- 🩹 **定点修复（Step 6）**：用 `docs +update` 外科手术式修补，**绝不重建**（飞书无删除文档命令，重建会留废稿）

## 🚀 快速开始 / Quick Start

> 转换脚本（`html_to_md.py` / `normalize_images.py` / `verify_doc.py`）纯 Python，三端通用；仅「创建飞书文档」这步依赖 `lark-cli`（WorkBuddy 内置，Claude Code / Codex 需自装）。下方为 WorkBuddy 内的标准调用链。

```bash
PY="<你的 WorkBuddy 隔离 venv python>"
NODE="<你的 WorkBuddy node>"
RUNJS="<lark-cli 入口 run.js>"

# Step 1-2：HTML → Markdown（产出 _build/doc.md + _build/images/）
"$PY" scripts/html_to_md.py "<HTML文件或URL>" ./_build

# Step 2.5：图片格式归一化（必做，否则 create 整篇失败）
"$PY" scripts/normalize_images.py ./_build          # 纠错扩展名，列出 SVG
NODE_PATH=... "$NODE" scripts/svg2png.js ./_build/images   # SVG→PNG
"$PY" scripts/normalize_images.py ./_build          # 退出码 0 才继续

# Step 3：创建飞书文档（先 cd 进 _build）
cd ./_build
"$NODE" "$RUNJS" docs +create --doc-format markdown \
  --content "@./doc.md" --title "<标题>" \
  --parent-position my_library        # 或 --parent-token <token>
```

**零依赖本地预览**：只想看 Markdown 产物，不落飞书？只跑 Step 1-2 即可，`doc.md` 可直接在任何 Markdown 阅读器打开。

## 🧩 跨平台安装 / Multi-Platform Setup

| 平台 | 转换脚本（HTML → MD） | 落盘飞书（Step 3+） |
|---|---|---|
| **WorkBuddy** | 内置 venv，直接调用 | 飞书连接器已提供 `lark-cli` |
| **Claude Code** | `pip install -r requirements.txt` | 自装 `lark-cli`（`npm i -g @larksuite/cli`）+ 飞书应用凭证 |
| **Codex** | `pip install -r requirements.txt` | 自装 `lark-cli` + 飞书应用凭证 |

> 装到各自 skills 目录即可被识别：WorkBuddy `~/.workbuddy/skills/`、Claude Code `~/.claude/skills/`、Codex `~/.codex/skills/`。只想转 Markdown 不落飞书，三端零额外依赖。

## 📖 典型工作流 / Workflow

1. **准备** 独立 `_build/` 工作目录（`doc.md` 与 `images/` 必须同处）
2. **转换** `html_to_md.py` → GFM Markdown
3. **归一化** `normalize_images.py` + `svg2png.js`（SVG 渲染为 PNG）
4. **创建** `lark-cli docs +create`，`new_blocks` 中 `image` 数量 = 上传图片数
5. **回查** `verify_doc.py source.html _verify.json`，确认 `缺失行数: 0`
6. **修复** 出问题用 `docs +update` 定点修补，不重建

## 📂 目录结构 / Structure

```
html-to-feishu-doc/
├── SKILL.md                 # 技能元数据 + 完整 6 步工作流（WorkBuddy 读取）
├── README.md                # 本文件
├── requirements.txt         # requests + beautifulsoup4
├── LICENSE                  # MIT
└── scripts/
    ├── html_to_md.py        # HTML → GFM Markdown 转换器
    ├── normalize_images.py  # magic bytes 纠错扩展名 + 列出 SVG
    ├── svg2png.js           # Playwright/Chromium 把 SVG 渲染成 PNG
    └── verify_doc.py        # 原文 vs 飞书文档 完整性比对
```

## ⚠️ 注意事项 / Caveats

- **图片必须 ASCII 命名**：中文/含空格的图片路径在部分环境会导致 `@./images/` 上传静默失败。脚本默认 `img_NNN.png`，**不要改回中文**。
- **扩展名 ≠ 真实格式**：飞书按内容校验并**整篇创建失败**（不是跳过）。务必先跑 Step 2.5。
- **Git Bash 下 `lark-cli` launcher 路径 bug**：Windows + Git Bash 会把 `/c/Users` 拼成 `c:\c\Users`，必须用绝对路径的 `node + run.js` 调用。
- **无法删除文档**：飞书当前无 delete 命令，创建失败/重复的旧文档需手动删除，所以一律走 Step 6 定点修复。

## 🔗 配套 / Related

- **微信公众号文章转 Markdown（wechat-article-to-md）**：先抓公众号文章为 HTML/Markdown，再用本技能落盘飞书。

## 🤝 贡献 / Contributing

欢迎提交 Issue 与 Pull Request。PR 请保证：

1. `python -m py_compile scripts/*.py` 与 `node --check scripts/svg2png.js` 通过
2. 工作流变更同步更新 `SKILL.md` 的对应 Step
3. README 与 SKILL.md 保持一致

## 📄 许可证 / License

[MIT](./LICENSE)

---

## English

Convert HTML pages/files into Feishu (Lark) cloud documents, preserving tables and uploading images.

**Highlights**

- HTML file or URL input
- Tables → native Feishu tables (merged cells expanded)
- Images downloaded + auto-uploaded on doc create
- Styled fake headings promoted to `##`
- Image format normalization (magic bytes + SVG→PNG via Chromium)
- Integrity check + surgical `docs +update` fix (never recreate)

**Usage**

Depends on the Feishu connector's `lark-cli`. See `SKILL.md` for the full 6-step pipeline.

**Keywords**: html, feishu, lark, convert, markdown-to-doc, wechat, archive, knowledge-base, rag, skill, agent-skills, workbuddy, claude-code, codex
