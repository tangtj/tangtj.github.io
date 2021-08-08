---
layout: post
title: Github + OSS + CDN 搭建静态博客
category: 学习
tags: 
  - github action
  - blog
typora-root-url: ..
id: 6
---

github page 支持放置静态html 搭建自己的网站。**免费**，这对我太有吸引力了，我正好想使用 github page 搭建一个自己博客。

通过搜索不难发现，现在已经有很多成熟的静态博客生成的框架。例如：[hexo](https://pages.github.com)、[hugo](https://www.gohugo.org)、[jekyll](http://jekyllcn.com)。但是他们都有个问题：都需要在本地安装环境，使用程序生成静态的 html ，然后 push 到 github ，比较繁琐，本地环境的管理也是个麻烦事。

通过阅读 github page 文档，我最后选择了使用 jekyll 。因为 github page 原生支持 jekyll 。把 repo 项目配置好 github page ，只要把博客网站文件 push 到 github ，就会自动启动一个默认 github action 来打包和部署网站，就十分方便。



### 搭建博客

有个很简单的办法，fork 我的项目：[传送门](https://github.com/tangtj/tangtj.github.io) 。

然后到fork的项目里。

1. Setting -> Options 把 Repository name 改成 xxx.github.io（xxx 指你的 github username ），如下图所示。

![image-6-1](/file/image/6-1.png)



2. Setting -> Pages 配置成如下所示。

   ![image-6-2](/file/image/6-2.png)

3. 正常情况下。这个时候访问 xxx.github.io（xxx 指你的 github username ），就能看到博客了。

在 Setting -> Pages Source 配置的 branch ，每当这个分支有 push 的时候就会有一个默认 github action 启动，来尝试打包部署。

具体情况可以如下图所示，点 commit hash 旁边的绿色的勾勾或者红色的叉叉看执行情况。

![image-6-3](/file/image/6-3.png)

绿色购够代表执行成功，红色叉叉代表执行失败。



**目前应为某种不可抗力的情况，你会发现访问特别慢、特别慢，还经常打不开**



### 能不能访问快点

**可以**，上CDN。



我选择的是 [腾讯云CDN](https://cloud.tencent.com/product/cdn]) 。

- 每个月免费的50G的CDN流量包，持续半年（七牛云免费流量里头不支持HTTPS）。

- 回源时，不会跳转到回源域名。（这个还挺影响体验的。用自己的域名，总是跳转到 xxx.github.io）。

- ~~我的域名在腾讯云，搞起来方便~~。

**前置条件：一个备案了的域名(不讨论海外地区)**

cdn 里源站配置如下：

![image-6-4](/file/image/6-4.png)

其中红箭头处填写：xxx.github.io（xxx 指你的github username）。



确认提交，做好dns解析。过几分钟后，用你的域名访问。可以访问了，快多了。

但是，你会发现。

过一段时间，首次访问每个页面，会很慢，虽然第二次都是快的。因为缓存过期了，cdn又回源去源站取数据了。

可以调整一下 CDN 缓存配置，把静态页面统统都给缓存个十天半个月。可是这样，每次有博客有更新，cdn 又没更新，可以手动去刷新缓存来解决。

我的缓存配置：（从上到下优先级排列）

1. 首页、index.html 缓存一天（使用腾讯云cdn的插件定时任务，一个小时预热一次首页）。
2.  /post 路径下所有文件，缓存7天。
3.  css 、image 、js 基本不怎么变动的 html ，缓存30天。
4. 其他遵循源站

虽然，有点不舒服。资源没缓存的时候首次加载需要 2秒 左右。但是好歹比之前快多了，基本都能访问了。



### 能不能再快一点

**能**，上OSS。



慢是回源忙，github 服务器在境外加不可抗力，很慢是正常的。而 jekyll 无非就是生成 html 文件，访问的都是生成好的静态文件，那静态文件能放在 github 上那就能放在其他地方。那放在境内的服务器上就应该快很多了。

然后我就放 腾讯云OSS 上了，配合 腾讯云CDN。（放自己服务器上当作静态页面也得，可是我选择 jekyll 就是为了白嫖（或者很低成本的白嫖））

现在的问题就是，第一步我们用决定 jekyll 就是应为 github page 原生支持它，免去我们自己去折腾打包和部署。现在想要更快的体验就需要我们自己解决打包和部署了。

这就不得不提到 **github action**，我写一个简单 CI 脚本，借助 github action 来白嫖服务器打包部署过程。

以下是打包和部署 jekyll 的 CI脚本，可以直接用。

```yaml
name: Jekyll site build

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build the site in the jekyll/builder container
        run: |
          docker run \
          -v ${{ github.workspace }}:/srv/jekyll -v ${{ github.workspace }}/_site:/srv/jekyll/_site \
          jekyll/builder:latest /bin/bash -c "chmod -R 777 /srv/jekyll && gem install jekyll-relative-links && jekyll build --future"
          ls ${{ github.workspace }}/_site
      - uses: actions/setup-python@v2.2.2
      - name: Upload site
        run: |
          ls
          pip install coscmd
          echo "[common]
                secret_id = ${{secrets.COS_SECRET_ID}}
                secret_key = ${{secrets.COS_SECRET_KEY}}
                bucket = ${{secrets.COS_BUCKET}}
                region = ap-chongqing
                max_thread = 5
                part_size = 1
                retry = 5
                timeout = 60
                schema = https
                verify = md5
                anonymous = False" >> cos.conf
          echo y | coscmd -c ./cos.conf upload -rs --delete ${{ github.workspace }}/_site/ /
```



在第二步部署阶段，我用了 [**coscmd**](https://cloud.tencent.com/document/product/436/10976) 腾讯云提供的，oss cli 工具。用他来把打包完成的静态文件上传到OSS。在其中，有三个 secret，需要配置一下。

COS_SECRET_ID ： 腾讯云api secret_id

COS_SECRET_KEY ： 腾讯云api secret_key

COS_BUCKET ：腾讯云cos bucket名

api secret id/key 获取方式：[点我](https://console.cloud.tencent.com/cam/capi)

配置路径在：Setting -> Secrets -> New Repository secret

配置完，运行一次 action。正常的话，oss里面就已经有文件了。然后就需要简单配置一下。

- OSS 配置

  ![image-6-5](/file/image/6-5.png)

- CDN 回源配置

  ![image-6-6](/file/image/6-6.png)

然然后就等几分钟就完事了。



**使用 CDN + OSS 可能会产生费用，请自己斟酌一下再使用。** 



最后讲个段子：

我在查阅 github action 中文文档的时候，发现了一个翻译错误。点下方的反馈是提示 fork + pr 。当时我就在想 提个 fix : typo 的 pr ，我马上就要成为全球顶流公司项目的贡献者了，就有点小激动。仔细阅读完 pr 要求，发现暂不接受翻译相关的 pr。害，小遗憾。



资料参考

- [**github page 文档**](https://docs.github.com/en/pages)

- [**github action 文档**](https://docs.github.com/en/actions)

- [**腾讯云 coscmd 文档**](https://cloud.tencent.com/document/product/436/10976)

