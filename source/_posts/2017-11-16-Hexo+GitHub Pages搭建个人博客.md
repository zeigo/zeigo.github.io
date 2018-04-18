---
title: Hexo+GitHub Pages搭建个人博客
date: 2017-11-16 20:28:02
tags: [Hexo] 
category: 工具
description: Hexo+GitHub Pages搭建个人博客
---
心血来潮想搭个博客玩玩，搞域名、服务器什么的太麻烦，于是找了些教程学习用Hexo+GitHub Pages搭建个人博客，完全是静态文件。第一篇博客就记录下搭建的过程以及一些坑，以供以后参考，之后也可能继续更新。
<!--more-->
环境及版本：Mac, hexo 3.4.1, node 8.7.0，有人提到过`Windows`的`cmd`会有些问题，推荐`git bash`，`Hexo`版本差异较大，之前一些教程有些地方已经过时了，一切以[Hexo官方文档](https://hexo.io/docs/)为准。

## Git

### 本地git

电脑先下载个`git`

### GitHub账号

得有个[github](https://github.com)账号，假设你的用户名是example，之后所有内容都依据这个example，新建个example.github.io的仓库。这个时候就能直接访问[https://example.github.io](https://example.github.io)了（根据别人的教程貌似还要等十来分钟才能生效？不过我直接就能访问了），只显示一两行文字，因为这个repo里面还是空的。而要往repo里面放html，js等静态文件就需要`Hexo`的帮助了。

## Hexo基本

### 安装Node

`Hexo`是基于[Node.js](https://nodejs.org/en/)的，所以需要Node.js环境，关于Node.js的安装就不谈了，网上教程都有。

### 安装Hexo

```bash
npm install -g hexo-cli
```

### init

找个位置创建`Hexo`（名字随意）文件夹用来存储博客相关的东西，像我的是`~/Desktop/Hexo`

```bash
mkdir Hexo
cd Hexo
hexo init
```

或者直接在`~/Desktop`目录下

```bash
hexo init Hexo
cd Hexo
```

此时目录结构为

```file
.
├── _config.yml
├── package.json
|-- package-lock.json
├── scaffolds
├── node_modules
├── source
|   └── _posts
└── themes
```

似乎`hexo init`时会自动运行`npm install`，即会有`node_modules`和`package-lock.json`，如果没有的话记得手动

```bash
npm install
```

接着执行

```bash
hexo g # generate的缩写
```

`Hexo`会将source里的md文件处理为html，放到新建的public里面，提交到github的内容即public里的内容。这里如果想提交一个`README.md`，要放到source里面，同时为了避免被默认转为html，需修改`_config.yml`。

```yml
skip_render: README.md
```

要想先看看本地效果，可执行

```bash
hexo s # server的缩写，CTRL+C退出
```

接着访问[localhost:4000](localhost:4000)就可以看到初始化的`hello word`内容（端口`4000`可能被占用，注意一下），这样就完成了一个本地的静态网站，接下来就是将本地同步到GitHub上。

## 部署

其实本质就是把public里的js html css等文件`push`到之前创建的GitHub仓库里，以达到发布到线上和版本管理的目的。

### ssh key

要提交到GitHub上需要有权限才行，我们使用`ssh key`来解决本地和服务器的连接问题。这个用过GitHub的一般都有。

```bash
ssh-keygen -t rsa -C "邮件地址"
```

找到`.ssh\id_rsa.pub`复制到GitHub里`Setting`->`SSH and GPG keys`里，不详细写了。

### _config.yml

配置`_config.yml`文件

```yml
deploy:
  type: git
  repository: git@github.com:example/example.github.io
  branch: master
  message: "Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }}"
```

有教程写的是`type: github`，现在不行了！！

### 安装插件

```bash
npm install hexo-deployer-git --save
```

### hexo d

```bash
hexo d # deploy缩写，直接部署，
hexo d -m "commit msg" # 可选，自定义commit的msg
```

deploy这一步是把public里面的文件复制到`.deploy_git`，`.deploy_git`这个文件夹才是真正同步到GitHub上的本地仓库，部署时直接将GitHub上的内容完全覆盖。直接`hexo d`的commit msg会根据`_config.yml`里`message`确定，默认是`Site updated:`加时间这种，想要自定义的话`hexo d -m "commit msg"`

## 完善博客

### 相关配置

待补充，`_config.yml`可配置的太多。

### 新建文章

```bash
hexo new [layout] "my-new-post"
INFO  Created: ~/Desktop/demo/source/_posts/my-new-post.md
```

`layout`是可选参数，默认post，可在scaffolds里查看已有的layout，如`scaffolds/post.md`

```markdown
---
title: {{ title }}
date: {{ date }}
tags:
---
```

想使新的post含有category， description等，就要修改`post.md`

Hexo会在`source/_posts`下生成`my-new-post.md`，打开这个文件就可用Markdown格式写博文了。完整格式如下

```markdown
---
title: postName #文章页面上的显示名称，一般是中文
date: 2013-12-02 15:30:16 #文章生成时间，一般不改，当然也可以任意修改
category: 默认分类 #分类
tags: [tag1,tag2,tag3] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 附加一段文章摘要，字数最好在140字以内，会出现在meta的description里面
---

以下是正文
```

默认情况下，生成的博文目录会显示全部的文章内容，如何设置文章摘要的长度呢？此时在合适的位置加上`<!--more-->`即可。

### 新建页面

```bash
$ hexo new page "second-page"
INFO  Created: ~/Desktop/demo/source/second-page/index.md
```

部署时生成`hexo\public\second-page\index.html`

点`/second-page`链接可访问

### 404页面

GitHub Pages 自定义404页面非常容易，直接在根目录下创建自己的404.html就可以。但是自定义404页面仅对绑定顶级域名的项目才起作用，GitHub默认分配的二级域名是不起作用的。（这里没怎么明白，之后再补充）

## 主题

[官方主题](https://hexo.io/themes/)
找回你喜欢的主题，下载，比如我用的[cafe](http://cafe.giscafer.com/)（感觉跟阮一峰大神比较像，手动滑稽）

```bash
 git clone https://github.com/litten/hexo-theme-cafe.git themes/cafe
```

可以看到themes下有landscape（默认主题）cafe两个文件夹

修改`_config.yml`中的`theme: landscape`为`theme: cafe`，执行`hexo g`重新生成一下，有问题可以`hexo clean`清空public的内容再重新生成。

具体主题内的配置看主题的官方介绍，修改对应theme下的`_config.yml`。

## 补充

1. 默认[example.github.io](example.github.io)访问，想更个性化的可以绑定一个新的域名，这里我偷懒没有用。
1. 图床的话推荐[七牛](portal.qiniu.com)
1. 挺多部分只是简单的写了一下，因为实在有太多相关的教程了，不想再重复，本身这个也不是用来手把手教的（汗）。

## 参考

参考的教程挺多，最详细的应该是[http://ibruce.info/2013/11/22/hexo-your-blog/](http://ibruce.info/2013/11/22/hexo-your-blog/)，入门和进阶的内容都有。
