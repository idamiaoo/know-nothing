---
title: 利用用 hexo和 github pages 搭建个人博客
date: 2017-07-14 17:21:42
toc: true
category: 工具
tags: 
  - "hexo"
  - "blog"

---

## 准备

在github创建 `<yourname>.github.io` 项目，不要添加 readme 文件

## 安装 hexo

hexo 是 node.js写的工具，首先安装node.js,点击进入 [node.js 官网](https://nodejs.org/zh-cn/)
node.js 安装好后执行

```bash
$npm install hexo-cli -g
```

## 创建博客

### 创建本地 blog 目录

```bash
$hexo init peep
INFO  Cloning hexo-starter to ~/opt/hexo/peep
Cloning into '/Users/anger/opt/hexo/peep'...
```

### 生成 blog 文件

```bash
$hexo g
```

创建了public目录，并将站点的静态文件生成在这个目录下面(只需要把 public 里的文件 push 到 `<yourname>.github.io` 项目的master分支下就可以通过 `https://<yourname>.github.io` 访问你的博客了)。

### 发布到 github pages

#### 安装 hexo-deployer-git

```bash
$npm install hexo-deployer-git --save
```

#### 修改本地 blog目录下的_config.yml 文件

```yaml
url: https://<yourname>.github.io/ 
root: /

deploy:
  type: git
  repo: https://github.com/<yourname>/<yourname>.github.io.git
  branch: master
```

其他配置根据需要修改
你也可以把你的博客放在 git pages的子目录中,比如`https://<yourname>.github.io/blog/`, 首先在 github 上创建 blog 项目,不要 readme 文件，此时修改配置为

```yaml
url: https://<yourname>.github.io/blog/ 
root: /blog/

deploy:
  type: git
  repo: https://github.com/<yourname>/blog.git
  branch: gh-pages
# 发布后通过 https://<yourname>.github.io/blog/ 访问你的博客
```

#### 部署到 github

```bash
$hexo d
```

这个命令会生成 public 里的静态文件，并创建一个 git repo 推送到你指定的库中此时你可以通过 `https://<yourname>.github.io` 访问你的博客了

## 切换主题

选择的主题为 [maupassant](https://github.com/tufu9441/maupassant-hexo),个人非常喜欢，简洁明了,可以按作者提供的[文档](https://www.haomwei.com/technology/maupassant-hexo.html)安装

遇到问题:

1. npm install hexo-renderer-sass 安装报错，npm 使用代理解决
2. self_search - 基于jQuery的本地搜索引擎，需要安装`hexo-generator-search`插件使用。这个很好用，根据关键字搜索本地文章
3. 根据作者设置的语法高亮会报错

```yaml
highlight:
  enable: true
  auto_detect: false #这里要设置成 false 才不会报错
  line_number: true
  tab_replace:
```