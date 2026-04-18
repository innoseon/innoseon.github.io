# 博客维护手册

线上站点：<https://innoseon.github.io/>
当前技术栈：**Hexo 7 + Icarus 6 主题 + GitHub Pages**
本地仓库：`~/codebase/innoseon.github.io/`

> **核心约定**（Hexo 的"文件夹即规则"）
>
> - `source/_posts/<slug>.md` → 会上线
> - `source/_drafts/<slug>.md` → **不会上线**（只本地可预览）
> - 平时自己写用「直写法」（第〇节）；和 Claude 协作用「草稿法」（第一节）

---

## 总览：这一整套为什么能跑起来

整个博客从 push 到上线一共三层，理解了这三层，后面每一节就都顺了：

```
  Markdown (source/_posts/*.md)
         │
         │  ① 构建：hexo generate
         ▼
  静态 HTML/CSS/JS (public/)
         │
         │  ② 托管：GitHub Pages
         ▼
  https://innoseon.github.io/
```

- **Hexo 是静态站点生成器（SSG）**。它不是服务器——它在构建时把 Markdown **预先**渲染成 HTML，保存到 `public/`。线上**没有**数据库，**没有**后端进程。用户访问时拿到的是写死的 HTML 文件。
- **GitHub Pages 是哑巴静态文件服务**。它只认"有没有文件在这个路径上"。你把一坨 HTML 扔给它，它就照着服务。
- **GitHub Actions（CI）是这两者的粘合剂**。你 `git push` 之后，CI 按 `.github/workflows/pages.yml` 跑 `npm install && npm run build`，把产物 `public/` 打包上传给 Pages。

这套模型的两个重要推论：

1. **所有内容变更 = 文件变更 + git push**。写文章是在改 `source/_posts/`；调样式是在改 `_config.icarus.yml` 或 `themes/`；加资源是新建文件。没有"后台管理系统"要登录。
2. **线上状态滞后于仓库**。push → CI 跑 → 上线之间有约 1 分钟延迟。如果 push 没上线，99% 是 CI 挂了而不是 Pages 坏了。

---

## 〇、直写法（最简，自己写时用）

### 操作

1. **新建文件** `source/_posts/<slug>.md`（`<slug>` 是 URL 里那一段，用英文/拼音，别用中文）：

   ```bash
   cd ~/codebase/innoseon.github.io
   npx hexo new "我的笔记"               # 自动建 source/_posts/我的笔记.md，含 frontmatter
   # 或手写：vim source/_posts/my-note.md
   ```

2. **顶部 frontmatter 至少写这几行**：

   ```yaml
   ---
   title: 标题（可以中文）
   date: 2026-04-20
   tags: [杂记]
   ---
   ```

3. **本地预览一下**（可选但推荐）：

   ```bash
   npx hexo server
   # 浏览器 http://localhost:4000/
   ```

4. **推上线**：

   ```bash
   git add .
   git commit -m "post: <slug 或中文标题>"
   git push
   ```

push 完 CI 自动构建 + 部署，~1 分钟后 <https://innoseon.github.io/> 上线新文。

### ▶ 原理

- **`_posts/` 是 Hexo 的约定目录**。`hexo-generator-index` 插件扫这个目录生成首页列表；`hexo-generator-archive` / `-tag` / `-category` 根据每篇的 frontmatter 分别生成归档页、标签页、分类页。**文件放对位置，Hexo 就知道把它编进所有相关索引里**——不用在别的地方"注册"。
- **frontmatter 是 YAML 元数据块**，被 Hexo 读取：`title` 做展示、`date` 决定排序和 permalink（`permalink: :year/:month/:day/:title/`）、`tags` 影响标签页和侧栏。`date` 没写会用文件 mtime，但 mtime 在 git 里不稳定，**显式写更安全**。
- **文件名成为 URL 的 `:title` 部分**。中文文件名会被 percent-encode，URL 丑且部分工具（RSS 阅读器、回链工具）处理不好。所以 slug 用 ASCII。`title` 字段可以中文——那是展示层。

---

## 一、协作写作流程（Claude 起草 → 你审查 → 你发布）

### 操作

**Step 1 — Claude 起草**

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

**Step 2 — 你本地预览**

```bash
cd ~/codebase/innoseon.github.io
npx hexo server --drafts
# 浏览器打开 http://localhost:4000/
# 草稿也会显示（因为 --drafts）
# Ctrl-C 停止
```

**Step 3 — 你审查、修改**

直接编辑 `source/_drafts/<slug>.md`。标题换了，保留 frontmatter 的 `title`，文件名 `<slug>` 可以不改。

**Step 4 — 你发布**

假设草稿文件是 `source/_drafts/kai-ge-bo-ke.md`，slug 就是 `kai-ge-bo-ke`：

```bash
cd ~/codebase/innoseon.github.io

npx hexo publish kai-ge-bo-ke        # 草稿 → _posts，自动渲染
git add .
git commit -m "post: 开个博客"
git push
```

**Step 5 — 确认上线**

```bash
gh run list -R innoseon/innoseon.github.io --limit 3
gh run watch -R innoseon/innoseon.github.io     # 阻塞等 CI 跑完
open https://innoseon.github.io/
```

### ▶ 原理

- **`_drafts/` 是 Hexo 内置的"两阶段"机制**。`hexo generate` 默认跳过这个目录（除非在 `_config.yml` 设 `render_drafts: true` 或命令行加 `--drafts`）。CI 用的是默认 `hexo generate`，所以草稿怎么写、提交多少次都不会上线——**目录就是访问控制**。
- **`hexo publish <slug>` 做的事极简单**：把 `source/_drafts/<slug>.md` 移到 `source/_posts/<slug>.md`，并在 frontmatter 里补上 `date`（如果没写过）。它不碰内容。所以"发布"其实就是一次 `mv`，可逆（第三节教怎么撤回）。
- **为什么这套流程适合 Claude 协作**：Claude 写的东西可能有事实错误、AI 味、口吻不对。让 Claude 只能碰 `_drafts/`，不能动 `_posts/`，相当于**用文件系统实现了一个审批闸**——比任何代码约定都硬，而且不依赖别人自觉。

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

### ▶ 原理

- **`hexo clean` 做的事**：删除 `public/`（上次构建产物）和 `db.json`（Hexo 的增量缓存 DB）。`db.json` 记录"哪个文件上次渲染成什么"，用来避免重复工作。**改了文件但 Hexo 没察觉**，或者切换主题后样式没更新——基本都是它没失效。清掉就干净。CI 每次都从零跑，所以 CI 不需要 clean。
- **`hexo server` vs `hexo generate` 的区别**：`server` 是内存里的 dev 模式，监听文件变化自动刷新，**不写 `public/`**；`generate` 写磁盘。CI 要的是能上传的磁盘文件，所以 CI 用 `hexo generate`（workflow 里叫 `npm run build`，其实就是这个）。
- **`gh run watch` vs `gh run list`**：`list` 是拍快照，`watch` 阻塞直到 CI 结束并告诉你成败。想知道"到底部没部署上"用 `watch` 一次看清楚，比反复敲 `list` 省事。
- **`gh workflow run` 什么时候用**：CI 偶尔抽风（比如 Actions 后端故障），不想靠瞎改一个文件重推来触发时，直接手动跑一遍最快。

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

### ▶ 原理

- **"撤回 = 反向 publish"**。把文件从 `_posts/` 移回 `_drafts/`，下次 CI 构建时这篇不进 `public/`，Pages 把旧页面替换掉。对搜索引擎来说这篇等于消失了，但 **git 历史还留着**，需要的时候可以再 publish 回来。这是为什么推荐它而不是 `git rm`。
- **`git rm` 是物理删除**。CI 下次构建 `public/` 里不含这篇，它的 URL 会 404。旧的外链（知乎引用、Google 索引）全部失效。删之前想清楚"有没有人可能已经引用了这个 URL"。
- **为什么没有"hexo remove"这样的命令**：Hexo 把文件系统当唯一真实来源（single source of truth），增删就是文件操作 + git。它不维护自己的数据库，所以你做什么它都认。

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

### ▶ 原理

- **为什么是两个 config 文件**：Hexo 是"核心 + 主题"双层设计。`_config.yml` 里的字段是 Hexo 自己用的（路由、permalink、插件开关）；`_config.<theme>.yml` 是主题独占的（Icarus 的侧栏 widget、颜色变量、第三方嵌入等）。**主题换掉（比如换回 landscape），icarus 这个文件就没用了**，但 `_config.yml` 不用动。这种分工让"换主题"变成一次 `theme: xxx` 的修改，而不是大改动。
- **`post_asset_folder: true` 的意义**（在 `_config.yml` 里已开启）：每篇文章可以有一个跟文件同名的文件夹存图片等资源。比如 `source/_posts/foo.md` 配 `source/_posts/foo/screenshot.png`，文章里写 `{% asset_img screenshot.png %}` 或相对路径引用。**好处是资源跟文章绑定**，搬家/重命名/归档时一起走，不会出现"图全挂了"的惨案。
- **themes/ 是主题源代码**。极端情况可以直接改模板文件（比如改某个组件的 HTML 结构）。但改了就跟主题上游分叉，未来升级主题要手动合并。非必要不动。

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

### ▶ 原理

- **为什么锁 Node 20**：`hexo@7.3` 的 `engines.node` 要求 18+，而 CI 显式写 Node 20 是为了**可重现构建**——你本地和 CI 跑同一个 Node，才能避免 "works on my machine" 的怪事。CI 是权威环境，本地对齐它。
- **为什么是 npm 不是 pnpm**：这个仓库的 `.github/workflows/pages.yml` 写的是 `npm install && npm run build`，`package-lock.json` 锁的也是 npm 的依赖图。用 pnpm 会生成不同的锁文件、解析出不同的依赖树，破坏可重现性。要换包管理器就得同步改 CI + 重新生成锁文件——值不值得看你。我之前那个 Astro 项目用 pnpm 是另一个独立的项目，和这里没关系。
- **`npm update` 是小升级**，只在 semver 允许范围内升（比如 `^7.3.0` 最多升到 `7.x.x` 的最新）。大版本升级要改 `package.json` 再 `npm install`，而且升 Hexo 大版本偶尔会触发主题兼容性问题——遇到了再处理。

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

### ▶ 原理（这段踩坑值得记住）

- **为什么这一版被抛弃**：Astro 出来的视觉是"编辑部风"，文风也过度包装（面试官话术、排比"不是 X 而是 Y"、强结构化 rules）。你明确反馈两点都不像自己——于是全部回退到已有的 Hexo Icarus，口味和工作流都现成。
- **Pages 可以在 private repo 上开**（以前免费计划需要 public，GitHub 后来改了）。但 Pages **serve 出来的 HTML 是公开的**，不管仓库私有——不要把敏感信息 commit 到 `source/` 再以为仓库私就没事。
- **GitHub CDN 缓存**：关掉 Pages 后，旧 URL 可能还能访问 20–60 分钟，是 Fastly 那层的缓存没过期。等着就行，不能手动刷。

---

## 七、故障排查

- **push 后网站没更新** — `gh run list -R innoseon/innoseon.github.io --limit 3`。CI 红了就点进去看日志。常见原因：npm install 失败（依赖源抖动）、Hexo 构建报错（某篇 frontmatter 写坏了）、artifact 上传失败（GH 后端）。
  *为什么*：线上状态 = CI 最近一次成功构建的产物。CI 没成功，线上就是上一版。
- **本地 `hexo server` 起不来** — 先 `npx hexo clean`，再 `rm -rf node_modules && npm install`。
  *为什么*：90% 是 `db.json` 或 `node_modules` 状态不一致；10% 是 Node 版本被换掉了（比如 nvm 切过版本）。这两步一起解决大多数情况。
- **草稿里图片引用不出来** — 配置了 `post_asset_folder: true`，图片要放在跟 md 文件同名的子文件夹里。详见 [Hexo 资源文件夹](https://hexo.io/docs/asset-folders.html)。
  *为什么*：Hexo 的 asset 系统是"就近挂载"——文章 `foo.md` 的资源默认在 `foo/` 子目录下找，不在就渲染不出来。
- **CI 报 `Node.js 20 deprecated` 警告** — 非错误，可以忽略；等 2026 年下半年升 `actions/*@v5` 或设 `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`。
  *为什么*：GitHub Actions 内嵌的 Node runtime 要迁到 24。v4 系列 action 的二进制用的是 20，会显示 deprecation notice 但不影响运行。升 v5 是长期正解。
- **看起来某篇文章没被索引到首页** — 检查 frontmatter 的 `date` 是不是未来时间（Hexo 默认 `future: true` 会展示未来文章，但有些主题会按日期过滤），以及 `draft` / `published` 字段是否手动设了 false。
  *为什么*：索引页是构建时生成的 HTML 列表，内容由 frontmatter 过滤规则决定。元数据错一个字段，文章就"隐身"。

---

## 八、当前待审草稿

- `source/_drafts/kai-ge-bo-ke.md` — 《开个博客》。Claude 起草，等你审查后发布。
