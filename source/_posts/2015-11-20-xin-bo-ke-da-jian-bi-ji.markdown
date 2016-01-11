---
layout: post
title: "基于 Octopress 新博客搭建笔记 - Mac 端"
date: 2015-11-20 17:29:23 +0800
comments: true
tags: "博客"
categories: "装逼指南"
---

其实很早就想搭建自己的博客，但是无奈每次想要实施的时候总是被中途的一些东西挫败，其中不乏遇到各种配置环境错误等等，搞得非常烦。上个礼拜终于狠下心研究了一番，搞出了这个还算能看的博客。</br>

下面就来记录一下整个博客配置的过程，也许能对你带来一些帮助。</br>

<!--more-->

#### 概要：
首先大概讲一下这个博客搭建的流程，博客的搭建其实是利用一个叫做 `octopress` 的项目，这个项目开源在 GitHub 上，我们将这个项目 clone 到本地，进行一些自定义设定：比如更换主题，更改布局等，当然前提是你需要会写 `html` `css` 等；然后再用 `markdown` 写一些文章， 最后利用这个项目把这些东西生成静态网页，Push 到 GitHub 的 master 上，然后访问响应的域名，就能看到自己的博客了。具体方法会在下面详细讲解。

#### 必备工具：
`Git` `Ruby` 如果你还没有安装这两个东西，请提前安装好。Mac 是自带 Git 的，所以只需要安装 Ruby。 这两个的安装教程不在本篇的范围内，请自己 Google 。
申请一个 GitHub 账号，因为我们博客项目最终要托管到 Github 上。</br>

#### 第一部分：博客的配置部署

##### Clone 和 Install `octopress` 项目
打开 `Terminal` ，进到桌面目录，然后输入以下命令：

```
git clone git://github.com/imathis/octopress.git blog
```
从 github 的服务器拉取 `octopress` 这个项目保存到桌面 `blog` 文件夹内。

打开 `cd blog` 进入 `blog` 文件夹，安装 `bundle`：

```
sudo gem install bundler
sudo bundle install
```
**注意：**</br>
1.这里用加 sudo 的原因是因为有些文件可能存在没有写权限的问题，需要通过 `chomd` 更改权限，如果后续出现权限问题，建议加 `sudo`。</br>
2.如果 `gem` 安装出现了如下错误，请将 Ruby 的源更换为淘宝镜像，具体请怼：[**这里**](https://ruby.taobao.org)</br>
![here](http://ww2.sinaimg.cn/large/65312d9agw1ezoh8xa4k9j20oh00uaab.jpg)

安装 `octooress`：

```
rake install
```
这一步会安装包括博客主题等一些东西，如果不想用默认的主题可以下载别的主题进行更换。</br>

完成了上面的四步，博客就算安装完成了，此时就可以小小的预览一下生成的博客了，方法如下：

生成网页的静态页面，并且开启预览端口：

```
rake generate
rake preview
```
此时用浏览器打开4000端口就能本地预览了：
```
http://localhost:4000
```

**Tips:更换主题**
`rake install` 之后，就已经安装了默认的主题，样式就是[唐巧的博客](http://blog.devtang.com)的样式
当然也可以换成别的样式，比如我用的整个主题：

```
git clone https://github.com/shashankmehta/greyshade.git .themes/greyshade
rake install['greyshade']
rake generate
```
更多用法可以参考他的 [**Github**](https://github.com/shashankmehta/greyshade)

##### Deploy Blog 项目到 Github
首先我们需要在 Github 新建一个 Repo 用来存放博客的源码和文章，将这个仓库命名必须以以下格式：

```
<username>.github.com
```
**注意：**`username` 必须和你 Github 的 `登陆账号` 一致，否则博客部署到 Github 之后，也不能通过 `https://<username>.github.io` 来访问。
我在这里入过坑，所以在此提醒大家。比如：我 Github 的登陆账号是`maquannene` ，那么我新建的这个 Repo 仓库的名字就必须是 `maquannene.github.com`。

接着对本地仓库进行设置，终端输入：

```
rake setup_github_pages
```
这个指令应该是主要做了这几件事情：</br>
1.git remote add origin git@github.com:username/username.github.com.git</br>
增加远程源名为 origin </br>
2.git branch -m master source </br>
将本地的分支名字 `master` 更改为 `source` </br>

此时终端会出现以下内容：

```
Enter the read/write url for your repository:
```
意思是需要写入你的仓库 url ，继续输入如下：

```
git@github.com:<username>/<username>.github.com.git
```
此时需要设置的东西就差不多了，接着就是将工程推到 Github 上:

```
rake generate                             //	生成需要推到 master 的静态网页到_deploy文件夹中
git add .                                 //	将本地source分支更改的所有文件加入提交目录
git commit -am "First deploy to github."  //	提交，并且写入提交日志 "First deploy to github."
git push origin source                    //	将add的全部push到远程主机 origin 的 source 分支上
rake deploy                               //	部署_deploy文件夹里的东西到 origin 的 master 分支
```
此时整个博客的就已经部署到 Github 上了，就像 `概要` 中所说的，博客的静态网页 push 到了 origin 的 master 分支，博客的源码和博文 push 到了 origin 的 source 分支。</br>
输入`git branch`显示本地分支

```
*source
```
输入`cd _deploy` 进入 deploy 文件夹后，输入`git branch`显示本地分支

```
*master
```
从_deploy中退出来，输入`git branch -r`就可以看到远程的分支

```
origin/master
origin/source
```
部署完成完成之后大概等上几分钟就可以通过以下域名访问你的博客了：

```
http://<username>.github.io
```

#### 第二部分：开始写博客

部署完成就可以开始写博客了：

```
rake new_post["New Post"]             //	新建一个markdown格式的文件，博文就写在这个里面
rake generate                         //	生成静态网页
git add .                             //	添加source下的文件，比如刚才添加新的博文文件
git commit -m "Some comment here."    //	提交
git push origin source                //	推到source分支
rake deploy                           //	自动将静态网页部署到master
```

这里你应该发现了，推送博客其实和部署一样，只是更新一下 `source`分支下的一些博客源文件，然后推到 `origin/source` 分支，`rake generate` 生成的静态网页通过 `rake deploy` 推到 `origin/master` 分支，就这么简单。

这里说明一下，new_post 新建的 md 文章 和原来的 md 文章全都在 `/source/_posts` 下，需要更改直接用 md 编辑器编辑即可，编辑器有很多：Mou、MacDown等等...</br>

#### 第三部分：换个Mac继续写

这里主要讲一下，如果换了一个新的环境，怎么 clone 下自己的博客项目，继续写博客：</br>

首先将项目的 source 和 master 分支分别从远程 clone 下来 </br>
命令格式`git clone -b <远程分支名称> (仓库url) [本地文件夹名称]`
代码如下:</br>

```
git clone -b source git@github.com:username/username.github.com.git blog
cd blog
// 这里注意，master clone 到 _deploy文件夹下面
git clone -b master git@github.com:username/username.github.com.git _deploy
```

然后继续安装bundle:

```
gem install bundler
bundle install
```
到这里，项目就被 clone 下来并且安装好了，还是同原来一样，博客的源码放在 source 分支，静态网页放在 _deploy 文件夹下的 master 分支，接着就是重复第二部分，可以继续写博客了。</br>

**Tips：(如果是临时使用的电脑，就可以忽略下面了**

如果是自己以后需要长期使用的办公环境，建议生成新的 SSH 并且添加到 GitHub 上，以后提交代码就比较方便了。

#### 第四部分：部署到GitCafe

国内访问 GitHub 速度目前来说还是不是很理想，所以自然访问我们部署在 GitHub 上的博客速度也不是非常理想，我 ping 了一下，每次访问 GitCafe 的博客时间花费大约在 100ms 左右，而部署在 GitCafe 速度大约 30ms。而且这还是 GitHub 不抽风的状态，有时候如果 github 稍微抽一下疯，那就不知道多久能上去了。所以我在搭建部署在 GitHub 之后又在 GitCafe 部署了一次。</br>

部署在 GitCafe 上的过程就比较简单了，大致过程其实和 GitHub 是一样的：</br>
首先：在注册了 GitHub 的账户之后，建立一个仓库，项目名称必须和你注册的用户名一致，这样 GitCafe 才能将这个项目识别成 Page。

由于我其实只是使用 GitCafe 的 Page，所以不需要将源码托管在这里，当然如果你愿意也可以，我个人只是在这里上传一下静态网页。

在我们执行完 deploy 之后，进到 _deploy 文件夹，然后添加一下 GitCafe 的远程源：

```
git remote add gitcafe git@gitcafe.com:username/username.git >> /dev/null 2>&1
```
接着就能 push 到 GitCafe 上面了：

```
git push gitcafe master:gitcafe-pages
```
上面的意思是：推送本地分支 `master` 到 `gitcafe` 远程源的 `gitcafe-pages` 分支。</br>
**注意**：个人测试，推送的远程分支必须叫 `gitcafe-pages`，否则不能显示网页。

接着，输入一下：`git branch -r`</br>
发现多了一个远程分支：`gitcafe/gitcafe-pages`，即对应的 GitCafe 远程源的gitcafe-pages分支。

对了，记得也设置一下 GitCafe 上的 SSH，以后提交就比较方便了。

#### 写在最后
如果有什么问题并且我能帮到你，请点击左边的`微博`按钮私信我，或者在下面给我留言，工作日我会以非常快的速度秒回你。
