---
name: html-to-feishu-doc
description: "将 HTML 网页或 HTML 文件转换为飞书云文档的技能。当用户给出网页 URL 或 .html 文件、希望把它（含标题/正文/列表/表格/图片）存档成一篇飞书文档时使用；通常与 wechat-article-to-md 技能串联，形成「抓取 → 飞书存档」闭环。依赖 lark-doc 技能提供的 lark-cli 来创建文档。"
agent_created: true
slug: html-to-feishu-doc
displayName: HTML 转飞书文档
version: 1.0.0
summary: 将 HTML 网页/文件转成飞书云文档，支持表格、大标题层级、图片下载上传与格式归一化、完整性回查。
license: MIT
---

# HTML → 飞书文档（html-to-feishu-doc）

## Overview

把任意 HTML 网页或 HTML 文件，转换成一篇飞书云文档。流程为：HTML → GFM Markdown（**ASCII 命名的本地图片**）→ `lark-cli docs +create` 创建到指定飞书文件夹/知识库或"我的空间"。表格保留为飞书表格，图片在文档创建时由飞书自动上传。

含 **Step 5 完整性回查** 与 **Step 6 定点修复**：交付前必须拿原文逐行比对，发现问题用 `docs +update` 外科手术式修补，**不要重建文档**（lark-cli 删不了文档，重建会留废稿）。

## When to use

- 用户给出网页 URL 或本地 `.html` 文件，要求"转成飞书文档""存到飞书""发到飞书文档"。
- 与 `wechat-article-to-md` 串联：先抓公众号文章为 Markdown/HTML，再用本技能落盘飞书。
- 需要把网页内容（文章、文档页、博客）归档进团队/个人飞书空间。

## Prerequisites

- Python 环境（已装 `requests` + `beautifulsoup4`，复用隔离 venv）：
  `C:/Users/Jiazi/.workbuddy/binaries/python/envs/default/Scripts/python.exe`
- 飞书连接器已连接；`lark-cli` 在 WorkBuddy 连接器层可用。**在 Git Bash 沙箱里 `lark-cli` launcher 有 POSIX 路径 bug（`/c/`→`c:\c\`），必须用 node 直接调用其入口脚本**，详见 Step 3。
- **非 WorkBuddy 平台（Claude Code / Codex）**：转换脚本三端通用（`pip install -r requirements.txt` 即可）；仅「创建飞书文档」这步需自装 `lark-cli`（`npm i -g @larksuite/cli`）并配置飞书应用凭证（app_id / app_secret）。SVG→PNG 的 `svg2png.js` 依赖 playwright（`npm i playwright`）。
- **图片文件名必须 ASCII**（如 `img_001.png`），脚本已默认按 `img_{index:03d}.{ext}` 命名。中文图片路径在 Git Bash 环境下会导致 lark-cli 找不到文件、`@./images/` 上传静默失败——**绝对不要改成中文/含空格命名**。
- 落点二选一：
  - 用户提供了目标文件夹/知识库的 `parent-token` → 用 `--parent-token <token>`
  - 用户没给 token 或要落到默认位置 → 用 `--parent-position my_library`（无需 token，落入"我的空间"）

## Workflow

### Step 1：准备构建目录

在用户指定的输出根目录下新建独立工作目录（如 `_build/`），后续所有产物都放在里面。`doc.md` 与 `images/` 必须同处该目录——因为 `lark-cli` 以当前工作目录为基准解析 `@./` 本地引用。

### Step 2：HTML → Markdown

用技能脚本转换：

```bash
PY="C:/Users/Jiazi/.workbuddy/binaries/python/envs/default/Scripts/python.exe"
"$PY" "<skill_dir>/scripts/html_to_md.py" "<HTML文件或URL>" "<_build目录>"
```

脚本行为：
- 正文容器按优先级提取：`div#js_content` / `div.rich_media_content`（公众号）→ `article` → `.markdown-body` → `main` → `.content` 等 → 回退 `body`。
- 标题/段落/列表/引用/代码块/分隔线正常转换；**表格转 GFM 管道表格**，单元格内加粗/斜体/图片保留。
- 图片下载到 `_build/images/`，文件名 `img_NNN.{ext}`（ASCII），Markdown 中写为 `![alt](@./images/img_001.png)`。下载失败则回退为原图 URL。
- **大标题提升**：公众号大标题常是「样式伪装的文本」（彩色渐变背景条 / 加粗）而非真正的 `<h1>/<h2>` 标签，脚本会识别「中文序号（一、二、…）/ 阿拉伯数字序号（01、02、…）+ 结语/附/总结」的短文本块并提升为 `##`，使层级对齐（`#` 文章标题 → `##` 大标题 → `###` 1.1 → `####` 2.3.1）。
- 产出 `_build/doc.md`，首行是 `# 标题`。

### Step 2.5：图片格式归一化（**必做，否则 create 会直接报错**）

公众号文章里大量出现两类「扩展名骗人」的图片，lark-cli 按**内容**校验，遇到就整篇创建失败：

```
{"ok":false,"error":{"message":"local image #2: file is not a supported BMP, GIF, JPEG, PNG, TIFF, or WebP image"}}
```

- **内容是 SVG、扩展名是 .png/.gif**：公众号常把流程图/架构图以内联 `<svg>` 形式吐出来，被脚本按 `img_NNN.png` 存下。
- **内容是 GIF、扩展名是 .png**：动图同理。

处理顺序（两个脚本都在 `scripts/`）：

```bash
PY="C:/Users/Jiazi/.workbuddy/binaries/python/envs/default/Scripts/python.exe"
NODE="C:/Users/Jiazi/.workbuddy/binaries/node/versions/22.22.2/node.exe"
WS="C:/Users/Jiazi/.workbuddy/binaries/node/workspace/node_modules"

# 1) 按 magic bytes 纠错扩展名 + 列出 SVG（退出码 2 表示还有 SVG 待处理）
"$PY" "<skill_dir>/scripts/normalize_images.py" "<_build目录>"

# 2) 把 SVG 用 Chromium 渲染成真 PNG（原地覆盖同名 .png，doc.md 引用不变）
NODE_PATH="$WS" "$NODE" "<skill_dir>/scripts/svg2png.js" "<_build目录>/images"

# 3) 再跑一次确认，退出码 0 才继续
"$PY" "<skill_dir>/scripts/normalize_images.py" "<_build目录>"
```

- `svg2png.js` 依赖 **playwright**（托管 node workspace 里已装）；用 `NODE_PATH` 指过去即可，不要全局安装。
- 渲染时 `deviceScaleFactor: 2`，中文走系统 Microsoft YaHei，出来的图清晰不糊。
- 不要用 Pillow/cairosvg 这类方案：Windows 上要 cairo 原生库、且中文字体与 `<foreignObject>` 支持都差。

### Step 3：创建飞书文档

**先 `cd` 进 `_build` 目录**（关键），再调用 lark-cli。

⚠️ **Git Bash 沙箱里不要直接用 `lark-cli` 命令**——其 launcher 在 Windows + Git Bash 环境下会把 `/c/Users/...` 拼成 `c:\c\Users\...`（双前缀），node 报 `Cannot find module`。绕过方式：**用绝对 Windows 路径直接调 node + run.js**。

```bash
NODE="C:/Users/Jiazi/.workbuddy/binaries/node/versions/22.22.2/node.exe"
RUNJS="C:/Users/Jiazi/.workbuddy/binaries/node/cli-connector-packages/node_modules/@larksuite/cli/scripts/run.js"
cd "<_build目录绝对路径>"
"$NODE" "$RUNJS" docs +create \
  --doc-format markdown \
  --content "@./doc.md" \
  --title "<文档标题>" \
  --parent-token "<token>"   # 二选一
# 或
  --parent-position my_library
```

- `--title` 取 Step 2 打印出的标题。
- `--parent-token` 与 `--parent-position` 互斥；用户没给 token 时用 `my_library` 落到"我的空间"。
- 返回 JSON：`data.document.url` 即新文档地址；`data.document.new_blocks` 里 `block_type == "image"` 的数量 = **成功上传的图片数**（用于 Step 4 校验）。
- `--dry-run` 可在真实创建前预检：会打印完整 body 与 API 请求，不实际写入。认证/语法有问题 dry-run 也会暴露。

### Step 4：校验与交付

**校验图片上传数量**（最关键，常因路径问题静默失败）：

```bash
# 直接看 create 返回的 new_blocks 即可，无需 fetch
"$NODE" "$RUNJS" docs +create ... | grep -c '"block_type": "image"'
```

应等于 doc.md 中 `@./images/` 引用的数量。若为 0 或偏少，**100% 是图片路径问题**（中文路径/路径含空格/`@./` 相对解析失败）——回 Step 2 确认图片命名 ASCII。

可选：回查文档完整结构（用 Docx XML 格式，统计 `<table>` 数量等）：

```bash
"$NODE" "$RUNJS" docs +fetch --doc "<document_id 或 URL>" --detail with-ids > _check.json
```

向用户交付文档 URL，并说明：图片上传数、表格数（如有 `<table>` 统计）。

### Step 5：完整性回查（交付前必做；用户问"文档完善吗"就跑这个）

只看 `new_blocks` 数量不够——**转换器可能整段吞掉一个列表**。必须拿原文逐行比对。

```bash
# 原文 HTML（Step 2 没留就重新抓一份存成 source.html）
"$NODE" "$RUNJS" docs +fetch --doc "<doc_id>" --detail with-ids > _verify.json
"$PY" "<skill_dir>/scripts/verify_doc.py" source.html _verify.json
```

健康输出示例：

```
原文  : 图片 23（含无 src 占位 4）、表格 6
文档  : 图片 19、表格 6、h3 35、代码块 5、引用 3
原文纯文本 20321 字 / 文档纯文本 20355 字
缺失行数: 0
```

判读要点：
- **原文图片数 > 文档图片数** 未必是丢图：公众号正文常有 `<img class="rich_pages wxw-img"/>` 这种**无 src 的空占位**，不是内容，脚本已单独统计。
- 文档纯文本比原文多几十字符属正常（表格单元格分隔、标题标记），**少了才是问题**。
- 缺失行若全是代码块片段、`&nbsp;`(`\xa0`) 或加粗标记附近的文本，多为误报——先 `grep` 确认关键词是否真在 `doc.md` 里，再下结论。

### Step 6：定点修复已创建的文档（**绝不要重建**）

发现问题后**不要重新 `docs +create`**：lark-cli 没有 delete 命令，重建会在用户空间留下一篇删不掉的废文档。改用 `docs +update` 做外科手术式修补：

```bash
# 1) 在 Step 5 的 _verify.json 里定位插入点的 block id
# 2) 内容存成 XML 文件（中文/箭头等字符走文件，别走命令行转义）
"$NODE" "$RUNJS" docs +update --doc "<doc_id>" \
  --command block_replace --block-id "<target_block_id>" \
  --content "@./insert.xml" --dry-run        # 先预检
"$NODE" "$RUNJS" docs +update --doc "<doc_id>" \
  --command block_replace --block-id "<target_block_id>" --content "@./insert.xml"
```

- 常用 command：`block_replace`（替换单块）、`block_insert_after`（块后插入）、`append`、`str_replace`、`overwrite`。
- **嵌套子列表**正确写法（子列表放在 `<li>` 内）：
  `<li>主项文字：<ol><li>子项一</li><li>子项二</li></ol></li>`
  这样主列表编号不变（1..5 还是 1..5），子项自动变 a./b./c./d.。
  若用 `block_insert_after` 插 4 个顶层项，会被并入同一个 `<ol>`，把后续编号顶成 6..9，**语义就错了**。
- 改完再跑一次 Step 5，确认 `缺失行数: 0`。

## Key conventions

- **本地图片用 `@./` 前缀 + ASCII 文件名**：`![alt](@./images/img_001.png)` 是 `lark-cli` 约定，路径必须位于 `lark-cli` 运行时 cwd 内。中文文件名会导致上传静默失败。
- **cwd 即一切**：`doc.md` 与 `images/` 同目录，且 lark-cli 从该目录执行。
- **Markdown 转义**（脚本已处理，了解即可）：反斜杠 `\`→`\\`；单元格内竖线 `|`→`\|`；字面量 `<`→`\<`（避免被当 XML 标签）。

## Caveats

- ~~不支持合并单元格~~（已支持）：含 `colspan`/`rowspan` 的表格自动展开成规则 GFM 表格（跨行列单元格重复填充），对 AI/RAG 友好。
- **Git Bash 下 `lark-cli` launcher 路径 bug**（`c:\c\Users\...`）：必须用绝对路径的 `node + run.js` 调用（见 Step 3）。WorkBuddy 连接器层正常调用不受此影响。
- **中文图片路径**：在 Git Bash 沙箱里会导致 `@./images/中文.png` 找不到文件、图片上传静默失败，文档里保留 `![](@./images/中文.png)` 文本。**始终用 ASCII 命名**。
- **网络图片**：若 HTML 中图片是防盗链/过期链接，下载会失败并回退为原始 URL，飞书创建时可能因无法取图而留空。
- **图片扩展名 ≠ 真实格式**：lark-cli 按内容校验并**整篇创建失败**（不是跳过）。必须先跑 Step 2.5 的 `normalize_images.py` + `svg2png.js`。这是公众号文章最高频的失败点。
- **删除文档**：lark-cli 当前没有 delete document 命令。创建失败/重复的旧文档需用户手动在飞书里删除。**所以发现内容问题一律走 Step 6 定点修复，不要重建。**
- **公众号的「列表套列表」布局容器**：编辑器会吐出 `<ol style="list-style:none"><ol>…</ol></ol>`，外层没有直接 `<li>`。朴素实现（只取 `find_all('li', recursive=False)`）会把这整段列表静默吞掉——`html_to_md.py` 已用递归 `process_list` 修好，含嵌套缩进与「li 与子 ul 同层混排」（ul 直接子 = `[li, ul]`）两种形态。
- **批量图片上传限流（图片 >~20 张）**：一次 `docs +create` 本地上传超过约 20 张图时，第 3 张起稳定报 `correlation_failed / invalid_response`（压缩体积、等待重试均无效；前 2 张总成功）。**不要重试硬刚**，改用「占位符方案」：
  1. 把 doc.md 里每个图片引用替换为唯一占位符文本（如 `IMG-PLACEHOLDER-003`）；
  2. `docs +create` 创建（纯文本必成功）；
  3. `docs +fetch --detail with-ids` 定位每个占位符的 block id；
  4. 逐张 `docs +update --command block_insert_after --block-id <id> --content '<img path="@./images/img_NNN.jpg"/>'`，再 `block_delete` 占位符块（每张图一次独立上传，绕开限流）。
  注意：`block_replace` **不允许** text→image 类型转换（报 no document changes）；str_replace 不支持资源替换。网络 URL 图片方案不可用——飞书服务端拉不动微信 CDN 防盗链（img 块为 0）。
- **代码围栏错位吞内容**：原文（尤其 Prompt 模板）常残留孤立 ```` ``` ```` 或行内 ```` ```json{…} ````——按 CommonMark，带 info string 的 ```` ```json ```` **不能闭合**围栏，会把后续大段正文+表格吞进代码块。转换后必须校验：模拟配对（开启 = 任意 ```` ``` ```` 行，闭合 = 仅 ```` ``` ```` 的行），发现跨度异常大的代码块即围栏错位，回 doc.md 删除孤立残留行。
- **微信长链风控**：带 `poc_token` 等参数的长链（`/s?__biz=…`）易被风控验证页拦截（标题 untitled、正文为空即中招），换 UA/Referer/真实浏览器均无效——**改用 `/s/xxxxxx` 短链**。
- **大文档**：极长 HTML 生成的 Markdown 若超飞书单次限制，需分段创建后拼接（本技能 v1 不内置分页）。

## Resources

### scripts/

- `html_to_md.py` — HTML（文件/URL）→ GFM Markdown 转换器，输出 `doc.md`（引用 `images/img_NNN.{ext}`）。
- `normalize_images.py` — 按 magic bytes 纠正图片扩展名并同步 `doc.md`；列出 SVG 待转（退出码 2）。
- `svg2png.js` — 用 Playwright/Chromium 把 SVG 渲染成 PNG（原地覆盖，2x 缩放）。
- `verify_doc.py` — 原文 HTML vs 飞书文档的完整性比对（结构统计 + 逐行缺失检测，退出码 3 = 有缺失）。

### 常用常量

```
PY   = C:/Users/Jiazi/.workbuddy/binaries/python/envs/default/Scripts/python.exe
NODE = C:/Users/Jiazi/.workbuddy/binaries/node/versions/22.22.2/node.exe
RUNJS= C:/Users/Jiazi/.workbuddy/binaries/node/cli-connector-packages/node_modules/@larksuite/cli/scripts/run.js
WS   = C:/Users/Jiazi/.workbuddy/binaries/node/workspace/node_modules   # playwright 在这
```