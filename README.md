# 我的博客

Hugo + PaperMod 搭建的零成本静态博客。

## 本地预览

> 需要 Hugo Extended（v0.164+，本仓库 `bin/hugo.exe` 已自带）。

```powershell
# 1) 直接用仓库自带的二进制（不需要装 Hugo）
.\bin\hugo.exe server

# 2) 浏览器打开 http://localhost:1313
```

想用系统装的 Hugo 只需 `hugo server`。

## 写一篇文章

```powershell
.\bin\hugo.exe new content content/posts/my-new-post.md
# 编辑文件，把 draft: false 改为 draft: false（默认值就是 false）
git add .
git commit -m "post: my-new-post"
git push
```

## 部署到 Cloudflare Pages

1. 把代码推到 GitHub：
   ```powershell
   gh repo create my-blog --public --source . --remote origin --push
   # 没装 gh 就网页端建空 repo，然后：
   # git remote add origin https://github.com/<user>/my-blog.git
   # git branch -M main && git push -u origin main
   ```

2. Cloudflare 控制台 → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git** → 选仓库
3. 配置：
   - Production branch: `main`
   - Build command: `hugo`
   - Build output: `public`
   - Environment variable: `HUGO_VERSION = 0.164.0`
4. Save and Deploy → 1 分钟左右拿到 `https://my-blog.pages.dev`

之后每次 `git push` 自动重新部署。

## 目录结构

```
my-blog/
├── bin/hugo.exe         Hugo Extended 二进制（不入 git）
├── content/             Markdown 文章
│   ├── posts/           博客正文
│   └── about.md         关于页
├── themes/PaperMod/     主题（git submodule）
├── hugo.toml            站点配置
├── static/              直接拷贝到根目录的静态资源
└── README.md
```

## 常用配置项

`hugo.toml` 关键开关：

- `defaultTheme = 'auto'` → 跟随系统亮/暗模式
- `ShowReadingTime = true` → 显示阅读时长
- `ShowShareButtons = true` → 显示分享按钮
- `summaryLength = 80` → 首页摘要字数

## 进阶

- 评论：用 [Giscus](https://giscus.app/)（基于 GitHub Discussions，零成本）
- 图床：Cloudflare R2（10GB 免费）配 PicGo
- 分析：Cloudflare Web Analytics（隐私友好、零成本）
