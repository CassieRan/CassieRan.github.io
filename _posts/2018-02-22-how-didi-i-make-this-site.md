---
layout: post
title: 我是如何制作该网站的
tags: [jekyll, github pages, site]
comments: true
---


​	正式从事前端工作已足半年，一直心心念念的个人博客却因为懒惰而滞后，此间虽有尝试过纯手写前后端或用 vue 搭建，却由于这这那那的原因最终半途而废。直到年关将至，回想过去一年的碌碌无为，才想起来，该做点什么了……

​	由于自己没有购买服务器，且不想花过多时间在搭建上面，所以选择了 Jekyll + Github 来搭建我的第一版个人博客。

### 环境准备

- 系统 : MacOS


- Ruby : jekyll 是基于 ruby 开发的，需要安装 >=2.2.5 版本，Mac 自带了2.0版本的 ruby，故仅需使用 rvm 进行升级就好了。在升级过程中遇到很多问题，只需根据报错提示执行即可。

  安装 rvm

  ```bash
  $ \curl -sSL https://get.rvm.io | bash -s stable
  $ source ~/.bashrc
  $ source ~/.bash_profile
  $ rvm -v
  ```

  安装 ruby

  ​	列出已知的 ruby 版本

  ```bash
  $ rvm list known
  ```

  ​	安装指定版本的 ruby

  ```bash
  $ rvm install 2.4.1
  ```

  ​	切换 ruby 版本

  ```bash
  $ rvm use 2.4.1
  ```

  ​	检查 ruby 版本

  ```bash
  $ ruby -v
  ```

- RubyGems : ruby 的包管理器，它是随 ruby 一起安装的，无需独立安装。

  ​	检查 gem 版本

  ```bash
  $ gem -v
  ```

- Xcode、Command-Line Tools : 下载方式 Preferences → Downloads → Components

### 安装 Jekyll

以上一切准备就绪后，便可以安装 jekyll 了。

1. 安装 jekyll

```bash
$ gem install jekyll
```

2. 安装 bundler

```bash
$ gem install bundler
```

3. 创建一个 jekyll 项目

```bash
$ jekyll new my-first-site
```

4. 如果有依赖其他的 gem 包，则需执行 bundle install

```bash
$ bundle install
```

5. 构建并运行

```bash
$ bundle exec jekyll serve
```

此时即可打开 http://localhost:4000 查看网站内容。

### 项目结构

基本的 jekyll 站点目录如下：

```
.
├── _config.yml
├── _data
|   └── members.yml
├── _drafts
|   ├── begin-with-the-crazy-ideas.md
|   └── on-simplicity-in-technology.md
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts
|   ├── 2007-10-29-why-every-programmer-should-play-nethack.md
|   └── 2009-04-26-barcamp-boston-4-roundup.md
├── _sass
|   ├── _base.scss
|   └── _layout.scss
├── _site
├── .jekyll-metadata
└── index.html
```

通过`jekyll new`创建的站点目录相对较简洁，那是因为它默认使用了 minima 主题，`_layouts`, `_includes` and `_sass`默认包含在 minima 主题包里面

```
.
├── _posts
|   └── 2018-02-22-welcome-to-jekyll.md
├── _site
├── Gemfile
├── _config.yml
├── Gemfile.lock
├── 404.html
├── about.md
└── index.md
```

- _config.yml

  用于存储配置数据

- _drafts

  用于存放草稿，即待发表的文章

- _includes

  可以通过 _layouts 和 _posts 进行混合和匹配，以促进复用。

- _layouts

  这些是套用在文章上的模版

- _posts

  这些是你需要发表的文章，命名必须遵循以下格式：YEAR-MONTH-DAY-title.md

- _sass

- _data

- _site

- .jekyll-metadata

### 编写文章

​	了解了这些之后就可以开始写文章了，你可以选用你喜欢的文档格式，例如markdown、html、textitle 等，在此注意在正式编写文章内容时，必须以 YAML 头信息开始。头信息遵循以下格式：

```yaml
---
layout: post
title: How I made this site
---
```

### 发布文章

​	当某篇文章完成后，就可以发布在 Github Pages 上面了。

​	首先，你需要在 github 上新建一个 Github Pages 的项目，具体参考[官网教程](https://pages.github.com/)。然后，将你的本地仓库仓库推送到远程仓库就部署成功了，你可以通过 https://[username].github.io 直接访问你的站点。



​	因为我也是刚刚接触 jekyll，所以写得有些粗浅，后续还想针对个性化写一篇详细的文章，敬请期待！