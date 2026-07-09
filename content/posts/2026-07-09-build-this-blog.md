---
title: "零成本博客搭建全记录：Hugo + Cloudflare Pages"
date: 2026-07-09T11:07:00+08:00
draft: false
tags: ["Hugo", "PaperMod", "Cloudflare Pages", "折腾笔记"]
categories: ["技术"]
summary: "从装机到发文，全程 0 元，踩了两个坑，都解决了。"
---

> 这篇博客本身，就是用这套流程搭出来的。下面把今天从零到 `git push` 的完整路径原样记录，包括踩过的两个坑。

## 为什么是 Hugo + Cloudflare Pages

我想要的博客非常简单：

- 写作体验像写 Markdown，能用 Vim 快捷键，能离线写
- 不要数据库、不要服务器、不要月租
- 国内访问不能太离谱
- 一次配置，长期受益

筛了一圈，组合落在 **Hugo + PaperMod + Cloudflare Pages**：

| 组件 | 作用 | 费用 |
|---|---|---|
| Hugo Extended | 静态站点生成器 | 免费 |
| PaperMod | 主题（极简、暗色、搜索内置） | 免费 |
| Cloudflare Pages | 全球 CDN 托管 | 免费 |
| 自定义域名（可选） | 短一点好记 | 几十元/年 |

合计 **0 元起步**。

## 第一步：装 Hugo

Windows 下最省事的不是 winget / scoop，而是直接把二进制丢进项目目录：

```powershell
# 1. 拉最新 release 的 Windows amd64 extended 包
$env:HTTPS_PROXY="http://127.0.0.1:10808"
curl -sSL -o hugo.zip `
  "https://github.com/gohugoio/hugo/releases/download/v0.164.0/hugo_extended_0.164.0_windows-amd64.zip"

# 2. 解压到项目 bin/ 目录
Expand-Archive -Force hugo.zip bin
# bin/hugo.exe 65MB，足够
```

为啥不放系统 PATH？这样项目可以整体打包带走，换电脑重装零成本。

```powershell
.\bin\hugo.exe version
# hugo v0.164.0-ce2470e... extended windows/amd64
```

## 第二步：建站 + 装主题

```powershell
cd my-blog
.\bin\hugo.exe new site . --force    # --force 因为目录非空
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
echo "theme = ''PaperMod''" >> hugo.toml
```

## 第三步：写 `hugo.toml`

默认 `hugo.toml` 是英文向，要改三块才像中文博客：

```toml
baseURL = ''https://chanriver.github.io/''
title = ''我的博客''
theme = ''PaperMod''

# 中文 + 按字数分词
defaultContentLanguage = ''zh-cn''
hasCJKLanguage = true

# 摘要取 80 字、暗色模式跟随系统、显示阅读时长
summaryLength = 80

[params]
  defaultTheme = ''auto''
  ShowReadingTime = true
  ShowShareButtons = true

[params.homeInfoParams]
  Title = ''Hi，这里是我的小角落''
  Content = '''
    记录折腾、笔记、和偶尔的碎碎念。
  '''
```

## 第四步：第一篇文章 + 本地预览

```powershell
.\bin\hugo.exe new content content/posts/hello-world.md
.\bin\hugo.exe server   # http://localhost:1313
```

```powershell
$env:HTTPS_PROXY="http://127.0.0.1:10808"
git push -u origin main
```

弹出登录窗口，粘贴 GitHub PAT 即通过。

## 第五步：Cloudflare Pages 接仓库

在 https://dash.cloudflare.com 里：

1. **Workers & Pages → Create application → Pages → Connect to Git**
2. 选 `chanriver/chanriver-blog` → **Begin setup**
3. 配置：
   - Build command：`hugo`
   - Build output directory：`public`
   - Environment variables → Add：`HUGO_VERSION = 0.164.0`
4. **Save and Deploy**

`HUGO_VERSION` 是最容易忘的——不填的话 CF 默认装 0.111，新版 PaperMod 跑不起来，会卡在 `ERROR ... no such template`。

部署完拿到地址 `https://chanriver-blog.pages.dev`，等 1 分钟首次构建。

---

## 踩坑记录

### 坑一：访问 `*.pages.dev` "意外终止连接"

部署完成、地址也对，但浏览器访问报错 `ERR_CONNECTION_RESET`。

排查过程：

1. **猜疑网络**：本机访问其他 `*.pages.dev` 也断——不是网络
2. **看 Cloudflare 日志**：Deployments 标签 → 最新那次 → Build log 最后是 `ERROR ... theme ''PaperMod'' not found`
3. **查本地仓库**：`ls themes/PaperMod` 是空的！

**根因**：PaperMod 当时是用 `git submodule add` 装的，子模块在 GitHub 仓库里只是个指针，**实际文件没上传**。Cloudflare 拉到的是一个空目录 → Hugo 找不到主题 → 构建失败 → 部署出空壳 → 你访问任何 `pages.dev` 子域都会被 CF 重置。

**修复**：

```powershell
# 1. 拆掉子模块
git submodule deinit -f themes/PaperMod
git rm -rf themes/PaperMod

# 2. 重新下载主题源码（这次当普通目录）
curl -sSL -o tmp/papermod.zip `
  https://codeload.github.com/adityatelange/hugo-PaperMod/zip/refs/heads/master
Expand-Archive tmp/papermod.zip tmp
Move-Item tmp/hugo-PaperMod-master themes/PaperMod

# 3. 删 .gitmodules，放开 .gitignore
Remove-Item .gitmodules
# .gitignore 里把 themes/*/ 忽略规则去掉

# 4. 本地验证
.\bin\hugo.exe --gc
# Pages | 19 | Total in 287 ms ← OK

# 5. 提交推送
git add .
git commit -m "fix: include PaperMod theme files (drop submodule)"
git push
```

CF 收到新的 commit 自动重新部署，1 分钟后 `chanriver-blog.pages.dev` 终于能打开了。

**教训**：

- Hugo 主题如果不是上游会同步的 fork，就直接当目录提交，省事
- 真要用 submodule，仓库读者必须 `git submodule update --init --recursive`，很多托管平台（包括 CF Pages）默认行为不同

### 坑二：GitHub PAT 没 "Read and Write" 选项

第一次 push 报错 `Authentication failed`。

去 https://github.com/settings/personal-access-tokens/new 申请 Fine-grained token 时：

- **Repository access** 必须选 `Only select repositories`（或 `All repositories`）
- 选了 `Public Repositories (read-only)` 的话**不会**出现 `Repository permissions`

最简单是直接生成 **Classic PAT**，只勾 `repo` 一个 scope 就够推送代码用。

---

## 日常写作流

```powershell
cd E:\codex-project\workspace\my-blog
.\bin\hugo.exe server               # 边写边看
.\bin\hugo.exe new content content/posts/YYYY-MM-DD-title.md
# 编辑文件，把 draft 改 false
git add .
git commit -m "post: title"
git push                             # CF 自动重新部署
```

`server` 会监听文件变动，保存 Markdown 就自动刷新浏览器，写作体验接近 Notion。

## 一些配置备忘

`hugo.toml` 关键开关速查：

```toml
defaultTheme = ''auto''             # 跟随系统亮/暗
ShowReadingTime = true              # 显示阅读时长
summaryLength = 80                  # 首页摘要字数
hasCJKLanguage = true               # 中文按字数分词
paginate = 10                       # 每页文章数
```

要加菜单、加关于页社交账号，全部在 `hugo.toml` 的 `[[menu.main]]` / `[[params.socialIcons]]` 段改。

---

## 还可以加的东西

- **Giscus 评论**：基于 GitHub Discussions，零成本
- **Cloudflare Web Analytics**：CF 自家统计，零配置、隐私友好
- **自定义域名**：在 CF 控制台加 CNAME，30 秒搞定
- **图床**：Cloudflare R2 免费 10GB，配 PicGo 上传

以后慢慢加。

## 小结

这次博客从立项到发文，实际花了大概 40 分钟（含踩坑）。最值钱的两步：

1. **Hugo Extended 二进制直接放项目里**——跨机器迁移零成本
2. **PaperMod 不要用 submodule**——直接当目录提交

剩下的就是 Cloudflare Pages 那套标准动作，照着抄就行。

---

写完这篇，又多了个发博客的理由。

