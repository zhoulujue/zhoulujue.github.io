# 个人博客系统全面优化实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 系统化清理、修复、优化并扩展基于 Jekyll 的个人博客，提升站点可靠性、SEO 表现、移动端及代码阅读体验、全站搜索能力与工程化标准。

**Architecture:** 保持 Jekyll + GitHub Pages 的轻量静态架构不变，重构 `_config.yml`、模板组件 (`_includes/`、`_layouts/`) 与 Sass 样式 (`_sass/`)，移除冗余废弃文件，添加前端搜索 (Simple Search) 与长文 TOC、代码复制及深色模式支持。

**Tech Stack:** Jekyll, Liquid, Kramdown, Sass/SCSS, Vanilla JavaScript, GitHub Pages.

---

### Task 1: 规范 `_config.yml` 配置与废弃服务清理

**Files:**
- Modify: `_config.yml`
- Modify: `_includes/analytics.html`
- Modify: `_includes/svg-icons.html`
- Modify: `_includes/disqus.html`

- [x] **Step 1: 修正 `_config.yml` 配置**
  - 移除重复的 `highlighter: rouge`。
  - 清理废弃的 `googleplus`。
  - 修正 `disqus` 配置为正确 shortname `blog-zhoulujue-com`。
  - 升级 `google_analytics` 结构支持 GA4（`gtag.js`），并保留向后兼容。
  - 规范 `rss: true`。

- [x] **Step 2: 升级 `_includes/analytics.html` 至 GA4**
  - 替换过时的 `analytics.js` (Universal Analytics) 为现代 Google Tag (`gtag.js`)。

- [x] **Step 3: 更新 `_includes/disqus.html` 动态绑定**
  - 使用 `{{ site.disqus }}` 变量替代写死的 script 路径。

- [x] **Step 4: 清理 `_includes/svg-icons.html`**
  - 移除已下线的 Google+ 图标判断。

---

### Task 2: 清理废弃代码与冗余静态文件

**Files:**
- Delete: `_includes/duoshuo.html`
- Delete: `stylesheets/github-dark.css`, `stylesheets/print.css`, `stylesheets/stylesheet.css`
- Delete: `javascripts/main.js`
- Delete: `_posts/images/` 整个目录及其下重复图片

- [x] **Step 1: 删除废弃多说模板 `_includes/duoshuo.html`**
- [x] **Step 2: 删除无引用的 `stylesheets/` 与 `javascripts/main.js`**
- [x] **Step 3: 删除 `_posts/images/` 冗余图片目录（所有图片在 `/images/` 下已有完整备份）**

---

### Task 3: 优化基础模板与死链修复

**Files:**
- Modify: `_layouts/default.html`
- Modify: `_layouts/post.html`

- [x] **Step 1: 优化 `_layouts/default.html`**
  - 移除已失效且非 HTTPS 的 `html5shiv.googlecode.com` 脚本。
  - 补全 `html lang="zh-CN"` 语言属性。
  - 规范化容器与结构语义。

- [x] **Step 2: 优化 `_layouts/post.html`**
  - 修正微信二维码引用路径为 `{{ site.baseurl }}/images/Wechat_qrcode.jpeg`。
  - 添加文章上下篇导航（Previous/Next Post）提升连贯阅读体验。

---

### Task 4: 规范文章图片路径与排版细节

**Files:**
- Modify: `_posts/2019-12-31-Google-Rising-Star-Meetup.md`
- Modify: `about.md`

- [x] **Step 1: 修复 `2019-12-31-Google-Rising-Star-Meetup.md` 中的 `//` 路径拼写错误**
- [x] **Step 2: 规范 `about.md` 中的图片引用路径**

---

### Task 5: SEO 与 OpenGraph / Viewport 优化

**Files:**
- Modify: `_includes/meta.html`

- [x] **Step 1: 优化 Viewport 标签**
  - 去除 `maximum-scale=1.0`，符合现代 Web 无障碍可缩放标准。

- [x] **Step 2: 补充完整 SEO 元数据**
  - 添加 `canonical` 链接。
  - 添加 OpenGraph 标签 (`og:site_name`, `og:type`, `og:url`, `og:image`)。
  - 添加 Twitter Card 标签 (`twitter:card`, `twitter:site`, `twitter:creator`)。

---

### Task 6: 头像资源本地化与防失效

**Files:**
- Create: `images/avatar.png`
- Modify: `_config.yml`

- [x] **Step 1: 下载并保存远程图床头像至 `images/avatar.png`**
- [x] **Step 2: 将 `_config.yml` 中 `avatar` 更新为本地路径 `/images/avatar.png`**

---

### Task 7: 工程化支持（Gemfile、.gitignore 与 README）

**Files:**
- Create: `Gemfile`
- Modify: `.gitignore`
- Modify: `README.md`

- [x] **Step 1: 创建标准 `Gemfile`，声明 `github-pages` 与相关插件 gem**
- [x] **Step 2: 更新 `.gitignore`，规范忽略规则**
- [x] **Step 3: 重写 `README.md` 为专属博客介绍、本地运行指南与目录指引**

---

### Task 8: 代码块阅读体验与复制功能增强

**Files:**
- Modify: `_sass/_highlights.scss`
- Modify: `style.scss`
- Create: `_includes/code-copy.html`
- Modify: `_layouts/default.html`

- [x] **Step 1: 优化代码块水平滚动机制**
  - 调整 `_highlights.scss` 中的 `pre` 样式，在保持整洁的同时支持代码水平滚动条，防止缩进被强制折行破坏。
  - 优化行内代码 `code.highlighter-rouge` 的边距与圆角。

- [x] **Step 2: 增加代码一键复制组件**
  - 编写原生轻量 JS 实现代码块右上角浮动“复制”按钮，带复制成功提示交互。
  - 在 `_layouts/default.html` 中引入。

---

### Task 9: 全站即时搜索功能 (Client-side Search)

**Files:**
- Create: `search.json`
- Create: `_includes/search.html`
- Modify: `_layouts/default.html`
- Modify: `style.scss`

- [x] **Step 1: 创建 `search.json` 索引模板**
  - 自动将所有文章的标题、链接、日期、标签与正文摘要生成为静态 JSON 索引。

- [x] **Step 2: 创建搜索交互组件 `_includes/search.html`**
  - 包含轻量搜索弹窗/输入框与纯前端实时匹配渲染逻辑。

- [x] **Step 3: 在导航栏中加入“搜索”入口及对应样式**

---

### Task 10: 长文目录 (TOC) 与体验优化

**Files:**
- Modify: `_layouts/post.html`
- Modify: `style.scss`

- [x] **Step 1: 为技术长文增加自动目录 (Table of Contents)**
  - 在 `_layouts/post.html` 中加入目录结构与折叠/展开样式，提升长篇技术博客查阅体验。

---

### Task 11: 深色模式 (Dark Mode) 支持

**Files:**
- Modify: `_sass/_variables.scss`
- Modify: `style.scss`
- Modify: `_sass/_highlights.scss`

- [x] **Step 1: 使用 CSS 自定义属性（Variables）重构颜色管理**
- [x] **Step 2: 适配 `@media (prefers-color-scheme: dark)`，为背景、文字、边框、代码块提供优雅的深色主题**
