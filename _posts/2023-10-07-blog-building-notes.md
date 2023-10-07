---
layout: post
title: 'GitHub Pages 博客搭建记录'
subtitle: 重启 GitHub Pages 博客建设
date: 2023-10-07
author: Mr Chen
cover: '/assets/img/shan.jpg'
#cover: 'https://images.unsplash.com/photo-1653629154302-8687b83825e2'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- GitPages
- blog
---

# 渊源

&emsp;&emsp;国庆放假期间，机缘巧合看到一篇使用了非常漂亮主题的文章，它也是基于 GitHub 的 Pages 进行搭建和部署的，回想起几年前自己搭建的个人博客（因始终不满足自己的审美要求，已废弃），让我一眼便喜欢上了它，立即决定重建我的个人博客。


# 博客构建框架

&emsp;&emsp;搭建个人博客，大部分人都是通过 GitHub 的 Pages 进行白嫖，但是已有的主题都不是非常完美，对于一个从事非前端开发的人来说，优化调整博客主题到自己满意的程度是非常难的，长时间搞不到自己满意的程度，结局大概率都是废弃。
在看到这篇基于[H2O-ac 主题](https://blog.lui8.cn/tech/new-theme-h2o-ac.html) 搭建的的博客后，我决定就是它了，瞬间重拾搭建个人博客的兴趣。

&emsp;&emsp;早些时候，我使用的 GitHub blog 是通过 `Hexo` 构建的，`Hexo` 是基于 `Node.js` 驱动的，而上述博客是基于 `Jekyll` 构建，该框架基于 `Ruby` 驱动，它们都是开源的，通过 GitHub 的 star 和 fork 数量，可以看到 `Jekyll` 的热度要高于 `Hexo`，但是不知道当时自己怎么没有注意到这一点，可能是因为当时是摸着别人过的河，没有了自己的思考。

&emsp;&emsp;由于该主题是基于 `Jekyll` 框架构建的，所以我们就此切换到 `Jekyll`，主题就用 `H2O-ac`。


# 初始化

&emsp;&emsp;首先创建一个自己的专门用户博客的 public 仓库，这里建议直接从 `H2O-ac` 主题作者的 repo [jekyll-theme-H2O-ac](https://github.com/zhonger/jekyll-theme-H2O-ac) 中 fork，在命名新的 repo 时，一定要命名自己的 GitHub `name` + '.github.io'，比如我自己的：gbcpp.github.io，其中 `gbcpp` 便是我的 GitHub name，这样做也是官方建议的，同时 Fork 过去的 repo 才可以直接 Publish，而不用做其它任何链接失效之类的修复，而我因为当初好几年没有再折腾过 GitHub 的博客，早已经忘记了这一点，起了个非此模板的名字：`blog`、 `webindex` 等，部署上的博客有好几处都是无效的链接，需要自行查找并修改，同时博客的 Url 中也要增加子路径。

Fork 后，在该 repo 的 `Settings` 页面，找到 `Pages`，在 `Build and deployment` 中选择 `master` 分支进行发布，目录就选择 `/(root)`, Save 后等待约 3 分钟，即可通过 `xxx.github.io` 进行访问。

![](/assets/img/github_settings.png)


# 个人信息调整

虽然经过上述的简单操作，已经能够访问到自己新搭建的博客，但是还有一些个人信息需要调整，在 `.github/workflow` 中，每次提交，均会触发 GitHub Actions Workflow，重新执行 Jekyll build & deploy CI 操作，所以我们要为自己的 repo 增加一个 `GITHUB_TOKEN`，否则每次 CI 均会报错。如下图所示：

![GitHub Token](/assets/img/github_token.png)


GitHub 的 CI 每次构建完成后会将生成的 `_site` 目录 Push 到 [`gh-pages`](https://jupyterbook.org/en/stable/publish/gh-pages.html) 分支。

在该作者的 repo 中，添加的 GitHub workflow 不止 `GitHub` 一个，还有亚马逊等其它家，用于网站加速，不过，我个人并没有使用这种多个 workflow 的方式，我删除了除了 'jekyll.yml' 以外所有的 workflow，网站加速请继续往下看。


# 部署 CDN

博客初步建立后，直接访问是非常慢的，没有任何的 CDN 托管加速，而 `H2O-ac` 的作者开通了多家国内外 CDN 厂家的托管，访问速度要比直接 xxx.github.io 快很多。

这里我选择了提供全球节点的 `Cloudflare` [官网](https://dash.cloudflare.com/) 进行托管，主要是因为它对于个人站点是完全免费的，并且操作简单，易上手。


## Cloudflare 新建 Pages 

注册并登陆 Cloudflare，进入到个人 `账户主页`，进入 `Worker 和 Pages`，选择 `Pages` 选项，**注意**我们这里选择 `连接到 Git`，选择自己的仓库和分支，自动部署即可，如下图：

[](/assets/img/Cloudflare_GitPages.png)


由于我们仅使用了 `jekyll.yml` 一个 GitHub Action，该 CI 完成后，会将 `_site` 目录自动 push 到 `gp-pages` 分支，所以在 `Cloudflare` 的 `Pages` 配置中，我们选择 **分支** 为 `gh-pages`，同时不要添加任何的构建框架和执行命令，只要选择自动选择更新即可。添加完成后的配置如下，可做参考：
[](/assets/img/Cloudflare_GitPages2.png)


## 自定义域名

上述部署完成，可以生成自己博客在 Cloudflare 的域名，比如：xxx-github-io.pages.dev，执行以下步骤将 xxx.github.io 重定向到 xxx-github-io.pages.dev。


### CNAME
Custom domain 中填入 `xxx-github-io.pages.dev` 
博客仓库中新建 `CNAME` 文件，填充 Cloudflare 生成的 'xxx-github-io.pages.dev' 域名，并提交。


### Pages 自定义域名配置

回到 GitHub 博客仓库的 `Settings` 页面，在 `Pages` 子页面的 `Custom domain` 中填入 `xxx-github-io.pages.dev` 保存。以后每次访问自己的 xxx.github.io 便会自动跳转到 xxx-github-io.pages.dev。

当然这样的白嫖效果也就一般般，访问速度并不能完全达到国内建站的速度。

# 绊脚石

## Jekyll 安装

本人的环境是在 WSL2 下的 Ubuntu Linux 安装的，发现 Jekyll 官方提供的 Installing docs 在 Ubuntu 18.04 及以下由于版本兼容问题已经无法正常的安装了，在 Ubuntu 20.04 中较为顺利，依次执行以下命令不太会遇到太多的问题：

```bash
sudo apt update && sudo apt upgrade -y
sudo apt-get install -y  ruby-full build-essential zlib1g-dev
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 这里 proxychains 为本人配置的 代理，否则容易出现下载失败
sudo proxychains gem install jekyll bundler
sudo proxychains bundle install
```


## H2O-ac Blog 页面异常

基于作者的最新版本 `master` 或者 `tag: v1.2.1`，在 `Blog` 首页会有如下分页显示的问题，该问题在 GitHub 中已经有人反馈给作者，等待作者修复（本人不懂前端，经过几天努力，排查修复失败），不过本人已经排查到该显示问题是从 `v1.1.6` 升级到 `v1.1.7` 时开始引入的，在本地 build 查看时，没有异常，一旦部署到 GitHub 便异常，所以该站使用的是 `v1.1.6` 版本。

[](/assets/img/jekyll_blog_bug.png)


## Cloudflare 部署报错

当 Blog 中存在文件 size ~> 25MB 时，Cloudflare 会报错而终止部署，对此需要删除 repo 中所有 size 过大的文件（一般是图片），根据报错，通过以下命令查找并删除所有相关文件的历史记录。

```bash
# 其中 assets/img/shan.png 为要删除的文件
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch assets/img/shan.png' --prune-empty --tag-name-filter cat -- --all
```

排查结束后，会显示涉及到该文件的分支列表，如下：

```bash
Ref 'refs/heads/gh-pages' was rewritten
Ref 'refs/heads/master' was rewritten
Ref 'refs/remotes/origin/master' was rewritten
Ref 'refs/remotes/origin/dev' was rewritten
Ref 'refs/remotes/origin/gh-pages' was rewritten
WARNING: Ref 'refs/remotes/origin/master' is unchanged
```

将 `rewritten` 的本地分支 push 到 GitHub，然后在 Cloudflare 上重新部署。

```bash
git checkout master
git push -f origin master

git checkout gh-pages
git push -f origin gh-pages
```


