# Agent.md

> 给在本仓库工作的 AI Agent（以及作者本人）的**实操指南**。基础信息（项目结构、命令、配置说明）见 `CLAUDE.md`；本文件聚焦「怎么把事做成、怎么避坑」，提炼自 2026-07-28 发布 `learn-from-video` 一文的实战经验。

---

## 1. 发布一篇普通文章

1. 在 `content/posts/` 下新建 `<slug>.md`，front matter 参考 `content/posts/java-multithreading-handmade-1.md`：`title` / `date`（带 `+08:00`）/ `draft: false` / `tags` / `categories` / `summary`。
2. 本地预览：`hugo server`（需 Hugo extended，本机未装见 §4）。
3. `git add` → `git commit` → `git push origin main`，推送即触发自动部署（见 §3）。

文章 URL：`/posts/<slug>/`（pretty URL，带尾斜杠；`hugo.toml` 无 permalink 改写）。

---

## 2. ⭐ 发布「带独立 HTML / 交互资源」的文章

场景：文章要引用一个自包含 HTML（动画流程图、scholar-notes、ppt-animation、flowchart 产物等），希望它能**单独打开、正常渲染**。

### 坑：leaf bundle 不会发布 `.html`

把 `index.md` 和 `xxx.html` 放进同一目录（leaf bundle）**行不通**——实测：

- Hugo **不会**把 `.html` 作为 page resource 发布，即使 front matter 加 `_build: publishResources: true`；
- `relref` / `ref` shortcode 对 `.html` 直接报 `REF_NOT_FOUND`；
- 普通相对链接 `[x](./xxx.html)` 能渲染出来，但目标文件根本没输出 → **线上 404**。

### 正确做法：放进 `static/`

`static/` 是 Hugo 唯一保证「逐字发布」的位置。按文章 URL 镜像目录结构摆放：

```
content/posts/<slug>.md              # 文章本体（普通文件，不要用 bundle 目录）
static/posts/<slug>/<file>.html      # 独立 HTML 资源
```

构建后两者都落在 `/posts/<slug>/` 下：

- 文章：`https://newbie-id.github.io/posts/<slug>/`
- HTML：`https://newbie-id.github.io/posts/<slug>/<file>.html`

文章里用**绝对路径**链接（最稳）：

```markdown
> 完整工作流图：[查看动画版](/posts/<slug>/<file>.html)
```

PDF、zip 等任意静态资源同理。`learn-from-video` 一文即此法，已验证线上可访问。

---

## 3. 部署管线与时机

```
git push origin main
  → .github/workflows/gh-pages.yml 触发（ubuntu runner）
  → checkout(submodules: true) → setup hugo extended → hugo --minify
  → peaceiris/actions-gh-pages 把 public/ 推到 gh-pages 分支
  → GitHub Pages 对外提供服务
```

- 主题 `blowfish` 是 submodule；CI 自动拉取，`module.toml` 的 `path = "blowfish"` 解析到本地 `themes/blowfish`，**不需要 Go**。
- ⚠️ **CDN 传播延迟**：CI 变绿 ≠ 立刻可访问。`gh-pages` 推送后 GitHub Pages 还要约 **1 分钟**才生效，期间新 URL 会返回博客自带的 404 页。线上验证遇到 404 先等一会再试。

---

## 4. 本地构建 / 验证技巧（网络受限环境）

- 本机可能没装 Hugo，而 Hugo **extended** 是必须的（SCSS）。GitHub 直连慢时，用博客自带的镜像下二进制：

  ```bash
  curl -L "https://gh-proxy.com/https://github.com/gohugoio/hugo/releases/download/v0.139.0/hugo_extended_0.139.0_windows-amd64.zip" -o hugo.zip
  unzip hugo.zip && ./hugo.exe version
  ```

- **主题 submodule 在慢网下很难拉**（仓库大，且梯子常不代理 git 协议）。好消息：§2 的发布机制**与主题无关**，做一个无主题的隔离构建即可验证 HTML 是否被发布：

  ```bash
  hugo new site test && cd test
  mkdir -p layouts/_default && printf '%s\n' '{{ .Content }}' > layouts/_default/single.html
  # 放入 content/posts/<slug>.md + static/posts/<slug>/<file>.html
  hugo --minify
  find public -name "<file>.html"   # 出现 = 会被发布；没有 = 不会发布
  ```

- 要跑完整主题构建，需先 `git submodule update --init --recursive`（慢网耐心等）。

---

## 5. 验证一次发布是否真的成功

```bash
# 1. 看 CI 跑完且成功
gh run list --workflow=gh-pages.yml --limit 1
gh run watch <run-id> --exit-status

# 2. 确认 gh-pages 分支里文件真的在
gh api "repos/newbie-ID/newbie-id.github.io/contents/posts/<slug>?ref=gh-pages" -q '.[].name'

# 3. 等约 1 分钟后 curl 线上（200 = 生效）
curl -s -o /dev/null -w "%{http_code}\n" "https://newbie-id.github.io/posts/<slug>/<file>.html"

# 4. 想确认「能正常展示」：浏览器打开 + 全页截图（动画类页面尤其推荐）
```

---

## 6. 网络备忘

- 慢 GitHub 区域：博客推荐的镜像 `https://gh-proxy.com/` 可代理 release / raw 文件下载。
- `git push` 鉴权：本项目用 `gh repo clone` 拉取，`gh` 的 credential helper 已配好，直接 `git push` 即可。
- 梯子常代理 HTTP 而不代理 git 协议——所以 `git submodule` / `git clone` 可能仍卡，而 `gh` API / curl / 浏览器走 HTTPS 一般没问题。
