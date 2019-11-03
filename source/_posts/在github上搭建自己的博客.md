---
title: 在github上搭建自己的博客
date: 2019-11-03 00:04:24
categories:
  - tutorial
tags: 
  - hexo
  - github pages
---

将搭建博客的步骤记录下来分享给大家

### 安装Node.js

我选择的操作系统是fedora, 选择的博客框架为hexo

```bash
# 安装nodejs
sudo dnf install nodejs
# 设置淘宝镜像源
npm config set registry https://registry.npm.taobao.org
```

### 安装hexo与主题

```bash
# 安装hexo
sudo npm install -g hexo-cli
# 创建一个工作文件夹
hexo init souhup.github.io
# 通过npm初始化
npm install
# 将工作目录交由git管理
git init
# 在github上对theme-next/hexo-theme-next进行fork
git submodule add https://github.com/souhup/hexo-theme-next.git themes/next
cd themes/next
git remote add upstream https://github.com/theme-next/hexo-theme-next.git
# 将开发部分保存到source分支
git checkout -b source
git remote add origin https://github.com/souhup/souhup.github.io.git
```

### 定制博客

#### 修改网站名称

修改修改`souhup.github.io/_config.yml`
```yaml
# Site
title: 美好的每一天
subtitle: ''
description: 'blog'
keywords:
author: souhup
language: zh-CN
timezone: ''
```

#### 修改网站图标和个人图标

向`souhup.github.io/themes/next/source/images`添加图标

修改`souhup.github.io/themes/next/_config.yml`

```yaml
favicon:
  small: /images/favicon-16x16.ico
  medium: /images/favicon-32x32.ico
  apple_touch_icon: /images/apple-touch-icon-next.png
  safari_pinned_tab: /images/logo.svg
  #android_manifest: /images/manifest.json
  #ms_browserconfig: /images/browserconfig.xml
  
# Sidebar Avatar
avatar:
  # Replace the default image and set the url here.
  url: /images/avatar.jpg
  # If true, the avatar would be dispalyed in circle.
  rounded: true
  # If true, the avatar would be rotated with the cursor.
  rotated: true
```

#### 添加tags和categories页面

执行如下命令
```bash
hexo new page categories
hexo new page tags
```
修改`souhup.github.io/sources/{categories|tags}/index.md`
```
---
title: 文章分类|标签
date: 2019-11-04 00:00:00
type: {categories|tags}
---
```
修改`souhup.github.io/scaffolds/post.md`
```
---
title: {{ title }}
date: {{ date }}
tags:
categories:
---
```

修改`souhup.github.io/themes/next/_config.yml`

```yaml
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```

#### 显示阅读进度

修改`souhup.github.io/themes/next/_config.yml`

```yaml
back2top:
  enable: true
  # Back to top in sidebar.
  sidebar: false
  # Scroll percent label in b2t button.
  scrollpercent: true
```

#### 在右上角显示github链接

修改`souhup.github.io/themes/next/_config.yml`

```yaml
# `Follow me on GitHub` banner in the top-right corner.
github_banner:
  enable: true
  permalink: https://github.com/souhup
  title: Follow me on GitHub
```


#### 添加本地搜索

执行以下命令
```bash
npm install hexo-generator-search --save
```
修改`souhup.github.io/themes/next/_config.yml`
```yaml
# Local Search
# Dependencies: https://github.com/theme-next/hexo-generator-searchdb
local_search:
  enable: true
  # If auto, trigger search by changing input.
  # If manual, trigger search by pressing enter key or search button.
  trigger: auto
  # Show top n results per article, show all results by setting to -1
  top_n_per_article: 1
  # Unescape html strings to the readable one.
  unescape: false
  # Preload the search data when the page loads.
  preload: false

```

#### 添加字数统计

执行以下命令
```bash
npm install hexo-symbols-count-time --save
```
修改`souhup.github.io/_config.yml`,在最后添加如下内容
```yaml
# workcount
symbols_count_time:
  symbols: true
  time: true
  total_symbols: true
  total_time: true
```
修改`souhup.github.io/themes/next/_config.yml`
```yaml
symbols_count_time:
  separated_meta: true
  item_text_post: true
  item_text_total: false
  awl: 2
  wpm: 300
```
随后需要执行`hexo clean`

### 向git提交

执行如下命令

```bash
npm install hexo-deployer-git --save
```

修改`souhup.github.io/_config.yml`

```yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/souhup/souhup.github.io.git
  branch: master
```

在github创建一个名为`souhup.github.io`的新项目

```bash
# 将自己的博客发布到github上
hexo d
# 提交工作内容
git add .
git commit
# 推送工作内容
git push -u origin source
```
