# 博客维护手册

线上站点：<https://innoseon.github.io/>
当前技术栈：**Hexo 7 + Icarus 6 主题 + GitHub Pages**
本地仓库：`~/codebase/innoseon.github.io/`

---

## 一、协作写作流程（Claude 起草 → 你审查 → 你发布）

这是最核心的一节。**Claude 只写草稿，不会自动把文章发出去**。

### Step 1 — Claude 起草

Claude 把文章放到 `source/_drafts/<slug>.md`。frontmatter 大致长这样：

```yaml
---
title: 中文标题
date: 2026-04-18
tags:
  - 标签1
  - 标签2
---
```

草稿不会出现在线上，也不会被 CI 构建进发布产物。

### Step 2 — 你本地预览

```bash
cd ~/codebase/innoseon.github.io
npx hexo server --drafts
# 浏览器打开 http://localhost:4000/
# 草稿也会显示（因为 --drafts）
# Ctrl-C 停止
```

### Step 3 — 你审查、修改

直接编辑 `source/_drafts/<slug>.md`。
如果要换标题，保留 frontmatter 里的 `title`，文件名 `<slug>` 可以不改。

### Step 4 — 你发布（一条命令三步）

假设草稿文件是 `source/_drafts/kai-ge-bo-ke.md`，slug 就是 `kai-ge-bo-ke`：

```bash
cd ~/codebase/innoseon.github.io

npx hexo publish kai-ge-bo-ke        # 草稿 → _posts，自动渲染
git add .
git commit -m "post: 开个博客"
git push
```

push 到 `main` 后 CI 自动构建并部署，大约 1 分钟上线。

### Step 5 — 确认上线

```bash
# 看 CI
gh run list -R innoseon/innoseon.github.io --limit 3
gh run watch -R innoseon/innoseon.github.io

# 打开线上
open https://innoseon.github.io/
```

---

## 二、日常命令速查

| 场景 | 命令 |
|------|------|
| 本地预览（只含已发布） | `npx hexo server` |
| 本地预览（含草稿） | `npx hexo server --drafts` |
| 清缓存 | `npx hexo clean` |
| 本地生成静态文件 | `npx hexo generate` |
| 新建草稿（你自己动手写） | `npx hexo new draft "标题"` |
| 新建发布文章（跳过草稿阶段） | `npx hexo new "标题"` |
| 发布某个草稿 | `npx hexo publish <slug>` |
| 查看 CI 运行 | `gh run list -R innoseon/innoseon.github.io --limit 5` |
| 看某次运行详情 | `gh run view <run-id> -R innoseon/innoseon.github.io` |
| 手动触发重新部署 | `gh workflow run pages.yml -R innoseon/innoseon.github.io` |

---

## 三、撤回 / 修改已发布文章

### 撤回为草稿

```bash
cd ~/codebase/innoseon.github.io
mv source/_posts/<slug>.md source/_drafts/<slug>.md
git add -A
git commit -m "draft: <标题>"
git push
```

### 直接修改已发布文章

```bash
# 编辑 source/_posts/<slug>.md
git add .
git commit -m "update: <简述改动>"
git push
```

### 彻底删除文章

```bash
git rm source/_posts/<slug>.md
git commit -m "remove: <标题>"
git push
```

---

## 四、配置与主题

| 文件 | 作用 |
|------|------|
| `_config.yml` | Hexo 主配置：站点标题、作者、语言、URL |
| `_config.icarus.yml` | Icarus 主题配置：logo、profile widget、侧栏组件等 |
| `source/_drafts/` | 草稿（**不会发布**） |
| `source/_posts/` | 已发布文章 |
| `source/about/index.md` | 关于页 |
| `themes/` | 主题文件（Icarus 已安装） |
| `.github/workflows/pages.yml` | CI：push 到 main 自动部署到 GitHub Pages |

**修改 profile 作者名、头像、社交链接**：编辑 `_config.icarus.yml` 里 `type: profile` 那一段（约 227 行起）。

---

## 五、环境与依赖

```bash
# 首次克隆后安装依赖
cd ~/codebase/innoseon.github.io
npm install

# 升级 Hexo（偶尔）
npm update hexo hexo-theme-icarus
```

Node 版本：CI 用 **Node 20**（见 `.github/workflows/pages.yml`）。本地用 Node 20+ 即可。

---

## 六、已下线：Astro 试验版（归档记录）

**仓库**：`innoseon/blog`（private，保留代码作为归档）
**状态**：GitHub Pages 已于 2026-04-18 关闭。CDN 缓存约 1 小时内自动失效。

### 若要彻底删掉这个仓库

```bash
gh auth refresh -h github.com -s delete_repo   # 给 token 加删除权限（会开浏览器）
gh repo delete innoseon/blog --yes
```

### 当初建 Astro 版用到的命令（仅归档，不需要再跑）

```bash
# ---------- 1. 建目录、初始化 package.json ----------
mkdir -p ~/codebase/blog && cd ~/codebase/blog
# （手写 package.json，依赖 astro / @astrojs/mdx / @astrojs/sitemap / @astrojs/rss）
pnpm install
pnpm approve-builds --all       # 让 esbuild / sharp 跑 postinstall

# ---------- 2. 本地开发 ----------
pnpm dev                        # http://localhost:4321
pnpm build                      # 输出到 dist/
pnpm preview

# ---------- 3. 初始化 git + 推送到 GitHub（private） ----------
git init
git add .
git commit -m "feat: initial blog scaffold"
gh repo create innoseon/blog --private --source=. --remote=origin --push

# ---------- 4. 启用 GitHub Pages（workflow 模式） ----------
gh api -X POST repos/innoseon/blog/pages \
  -f "build_type=workflow" \
  -f "source[branch]=main" \
  -f "source[path]=/"

# ---------- 5. 后来关闭 Pages ----------
gh api -X DELETE repos/innoseon/blog/pages
```

---

## 七、故障排查

- **push 后网站没更新**：`gh run list -R innoseon/innoseon.github.io --limit 3` 看 CI 是否红。红了点进去看日志。
- **本地 `hexo server` 起不来**：先 `npx hexo clean`，再 `rm -rf node_modules && npm install`。
- **草稿里图片引用不出来**：Hexo 配置了 `post_asset_folder: true`，图片要放在跟文件同名的文件夹里。详见 [Hexo 资源文件夹](https://hexo.io/docs/asset-folders.html)。
- **CI 报 `Node.js 20 deprecated` 警告**：非错误，可以忽略；等 2026 年下半年升 `actions/*@v5` 或设 `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`。

---

## 八、当前待审草稿

- `source/_drafts/kai-ge-bo-ke.md` — 《开个博客》。Claude 起草，等你审查后发布。
