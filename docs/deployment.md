# 部署与运行原理

本文说明这个应用是如何发布到网上的，以及数据到底存在哪里。

如果你只是想改代码然后看到效果，看 [本地开发](#本地开发) 就够了。如果想理解"为什么断网也能用""为什么换手机数据就没了"，看 [架构](#架构应用在哪里运行)。

---

## 一句话概括

> 推送代码到 `main` 分支 → GitHub 的机器人开一台临时电脑，把仓库目录**原样打包**传到全球 CDN → 用户从最近的节点下载 → **应用在自己的浏览器里运行**。

没有构建步骤、没有服务器、没有数据库、没有月费。

---

## 架构：应用在哪里运行

这是最容易搞混的一点：

```
GitHub CDN   →  只负责把文件传到浏览器（相当于快递）
用户浏览器   →  真正运行应用的地方
```

`index.html` 里的 JavaScript **不在 GitHub 上执行**，而是在用户的手机或电脑上执行。GitHub 只是"文件仓库 + 快递员"。

由此可以解释几个现象：

| 现象 | 原因 |
|---|---|
| 断网后 App 还能打开、数据还在 | 文件已经下载到本地，计算也在本地 |
| 数据存在 `localStorage` | 存在**那台设备**的浏览器里，不在 GitHub |
| 换手机数据就没了 | GitHub 上只有代码，没有任何用户数据 |
| 清浏览器数据 / 卸载 App 后数据丢失 | 同上，`localStorage` 随实例销毁 |
| 不同设备之间数据不同步 | 没有任何后端在做同步 |

**这就是备份码存在的原因** —— 整个架构里没有任何环节在保存用户数据，跨设备迁移只能靠用户自己搬运。

---

## 技术栈

应用是一个**纯静态单文件**：

- 一个 `index.html`（HTML + CSS + JavaScript 全在里面）
- 零依赖、无构建工具（没有 npm、webpack、vite）
- 图表用手写 SVG 绘制
- 数据存在 `localStorage`，键名 `savingsGoalTracker.v1`
- 唯一的外部文件是 `vendor/pako.min.js`（gzip 压缩库，本地引入，不用 CDN）

因为"打开就是成品"，所以可以直接托管，不需要任何加工。

---

## 部署流程

### 触发条件

部署由 [.github/workflows/deploy.yml](../.github/workflows/deploy.yml) 定义，两种方式触发：

1. **推送代码到 `main` 分支**（自动）
2. **在 GitHub 网页上手动点 "Run workflow"**（`workflow_dispatch`）

### 执行步骤

GitHub 会开一台**临时的云端 Ubuntu 电脑**，依次执行四步：

| 步骤 | 动作 | 相当于 |
|---|---|---|
| 1. Checkout | 把仓库代码下载到这台临时电脑 | `git clone` |
| 2. Setup Pages | 登记"这次要发布 Pages" | — |
| 3. Upload artifact | 把整个目录**原样打包**上传 | 压缩文件夹传到网盘 |
| 4. Deploy to GitHub Pages | 把包解开，放到 CDN 上 | 发布 |

**整个 workflow 里没有任何构建命令**（没有 `npm run build`、没有编译、没有打包）。第 3 步的 `path: .` 就是"整个仓库目录原样上传"。

干完活，那台临时电脑**立刻被销毁**。公开仓库的 GitHub Actions 是免费的，所以这套部署零成本。

### 权限配置

```yaml
permissions:
  contents: read      # 读取代码
  pages: write        # 发布到 Pages
  id-token: write     # 向 GitHub 证明"我是这个仓库的正规 workflow"
```

`id-token: write` 用的是 **OIDC**：workflow 运行时 GitHub 现场发一张**一次性身份证**，用它换一个**几分钟就过期的临时令牌**。令牌过期自动作废，泄露也没用，所以不需要在仓库里存长期密钥。

> **前置条件**：仓库 Settings → Actions → General → Workflow permissions 必须勾 **Read and write permissions**。不勾的话 GitHub 连 `id-token` 都不发，第 4 步会直接失败。

### 并发控制

```yaml
concurrency:
  group: pages
  cancel-in-progress: true
```

防止连续推两次代码时两个部署任务互相覆盖。含义是：同一 `pages` 组里，新一轮开始时取消上一轮还没跑完的任务。

---

## 线上基础设施

访问地址格式：

```
https://<用户名>.github.io/<仓库名>/
      ↑ linjack7      ↑ savings-goal
```

线上响应头能看出背后的结构：

```
Server: GitHub.com
Via: 1.1 varnish                          ← 缓存层
X-Served-By: cache-fra-etou8220041-FRA   ← 具体节点
x-github-edge-region: switzerland north  ← 区域
X-Cache: HIT                             ← 命中缓存，未回源
Cache-Control: max-age=600               ← 浏览器可缓存 10 分钟
```

GitHub 在全球部署了大量缓存节点，用户从最近的节点取文件。

### 静态托管的两个行为特征

理解这两点对排查问题很有用：

- **文件存在就返回 200，不存在就 404** —— CDN 不会像传统服务器那样"找不到就回退到 `index.html`"。所以访问 `/任意路径` 会得到 404，而不是应用界面。
- **`/.nojekyll` 返回 200** —— 因为它是真实存在的文件，被一起上传了。

---

## `.nojekyll` 的作用

GitHub Pages 最初为**博客**设计，默认用 Jekyll（一个 Ruby 静态网站生成器）处理文件。Jekyll 有个规则：**以 `_` 开头的文件和目录会被当成模板源文件，不对外发布**。

本项目目前没有 `_` 开头的文件，加不加不影响。但加上可以防止以后踩坑（比如新增了 `_data/` 目录却在线上消失）。

---

## 本地开发

项目已配置 [.claude/launch.json](../.claude/launch.json)，用 `preview_start` 即可启动：

```bash
python -m http.server 8765
```

然后访问 `http://localhost:8765`。

> **注意**：本地用 `http://localhost` 时，Safari 的"添加到主屏幕"不会出现——那是 iOS 对 PWA 的要求，必须是 `https://`。所以 PWA 相关功能只能在线上验证。

---

## 修改代码后怎么发布

```bash
git add -A
git commit -m "说明你改了什么"
git push origin main
```

推送后 GitHub Actions 自动跑一遍上面的四步，大约 30 秒到 1 分钟生效。

查看部署状态：仓库页面 → Actions 标签页。

> 如果 workflow 失败，常见原因是 `permissions` 配置（见上）或 `id-token` 相关设置。

---

## 安装到 iPhone

1. Safari 打开 `https://linjack7.github.io/savings-goal/`
2. 点底部分享按钮
3. 选择 **「添加到主屏幕」** → 「添加」

装好后是一个无地址栏的全屏独立 App。

**注意**：装到主屏幕后，数据存在**那一个 App 实例**里。换机或重装前，请先在 设置 → 备份码 里复制备份码。

---

## 排查清单

线上有问题时，按这个顺序查：

1. **Actions 有没有跑成功** —— 仓库 → Actions，看最近一次 run 是不是绿色
2. **文件是不是真的传上去了** —— 直接访问 `https://linjack7.github.io/savings-goal/manifest.webmanifest`，404 说明没传成功
3. **是不是缓存** —— 响应头里 `X-Cache: HIT` 表示命中缓存，可能还是旧版本；等 10 分钟或强制刷新
4. **本地和线上是否一致** —— 对比 SHA256：
   ```bash
   curl -s https://linjack7.github.io/savings-goal/ | sha256sum
   sha256sum index.html
   ```
5. **PWA 功能不生效** —— 确认是通过 `https://` 访问的，`http://localhost` 下"添加到主屏幕"不会出现

---

## 相关文档

- [ADR 0001 · 计算口径](adr/0001-calculation-model.md)
- [ADR 0002 · 财务自由本金](adr/0002-freedom-principal.md)
- [术语表](../GLOSSARY.md)
