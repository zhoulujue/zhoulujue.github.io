# Michael's Blog (blog.zhoulujue.com)

个人技术与生活博客，记录 Android / Flutter / 前端 / 移动端架构演进以及生活日常。

- **线上地址**：[https://blog.zhoulujue.com](https://blog.zhoulujue.com)
- **技术选型**：Jekyll + GitHub Pages 静态生成与托管
- **主题特点**：极简清爽、响应式适配、支持代码高亮与一键复制、全站即时搜索、深色模式支持

---

## 🛠 本地开发与调试

### 前置要求
- Ruby (>= 2.7) 与 Bundler

### 运行步骤
1. 安装依赖：
   ```bash
   bundle install
   ```

2. 启动本地开发服务（支持热重载）：
   ```bash
   bundle exec jekyll serve
   ```

3. 浏览器访问：
   ```
   http://127.0.0.1:4000
   ```

---

## 📝 编写与发布文章

1. 在 `_posts/` 目录下新建 Markdown 文件，命名格式为 `YYYY-MM-DD-your-title.md`。
2. 文件顶部配置 Front-matter：
   ```yaml
   ---
   layout: post
   title: "文章标题"
   date: 2026-08-19
   ---
   ```
3. 正文使用标准 Markdown 编写，图片推荐存放于 `/images/` 目录下并在文章中使用 `/images/your-image.png` 进行引用。
4. 提交并推送到 GitHub `master` 分支，GitHub Pages 将自动触发构建与发布。

---

## 📂 目录结构

```
.
├── _config.yml          # 站点全局配置文件
├── _includes/           # 可复用组件（SEO Meta、代码复制、搜索弹窗、评论、统计）
├── _layouts/            # 页面骨架模板（default, post, page）
├── _posts/              # 博客文章源文件
├── _sass/               # 样式模块（变量、代码高亮、基础重置）
├── docs/                # 计划与设计文档
├── images/              # 文章插图、头像与静态媒体
├── index.html           # 博客首页
├── search.json          # 全站搜索数据源模板
├── about.md             # 关于我页面
└── 404.md               # 404 页面
```
