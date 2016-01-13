---
title: 博客改用 hexo 托管到 github 爬坑之旅
date: 2016-01-08 13:17:41
tags: "博客" 
categories: "装逼指南"
---

最近又蛋疼了，这个博客原本是用 Octopress 搭建的，搭建过程 **[详见这里](http://maquannene.github.io/2015/11/20/2015-11-20-xin-bo-ke-da-jian-bi-ji/)**。直到最近在知乎上看到 **[这个提问](https://www.zhihu.com/question/24422335)**，我又决定把博客从 Octopress 转为 Hexo 了。

本来想在这篇文章记录一下迁移过来的流程，但网上关于 hexo 博客的搭建技术贴实在太多了，所以考虑之后，不打算写这个流程，毕竟意义也不大，在这里贴一篇我觉得写得比较好的，如果你需要基于 hexo 搭建个人博客 **[详见这里](http://blog.hjtxxx.com/2015/08/13/Hexo-%E4%B8%80-%EF%BC%9A%E5%9C%A8GitHub%E4%B8%8A%E6%90%AD%E5%BB%BA%E9%9D%99%E6%80%81%E5%8D%9A%E5%AE%A2)**

下面最主要就是总结一下我遇到的问题，关于博客托管到 github 的问题。声明一下，我是完全不懂 js 网页这些东西的，所以下面写的东西，也只是我个人层面上的理解，所以如果有什么地方说的有问题，轻喷。

<!--more-->

在讲述我遇到的坑的问题之前，有必要先讲解一下 hexo 工程的文件划分。

hexo 的源码目录大概是这样：

![hexo目录结构](http://ww4.sinaimg.cn/large/65312d9agw1ezxy3ebuqjj20b408twf1.jpg)

**public**

在 clone 了 hexo 项目之后，应该是是没有 `public` 这个文件夹的。当执行 `hexo g` 时（g 是 generate 的缩写，意思是生成），hexo 会生成 `public` 这个文件夹和里面的文件，所以 `/public` 文件夹中放置的文件，自然就是最终网页显示需要的静态网页文件，这个是因每个人博客的内容不同而不同，所以源码中不必要存在这个文件夹，这一点在我们查看 .gitignore 时会发现，这个文件夹是被忽略的，并不需要上传。

当我们写好博客之后，执行 `hexo g` 生成网页文件，然后执行 `hexo d`，就会将 `public` 里面的文件 post 在 `github` 的 username.github.com 这个仓库，之后就能访问你的博客了。

**source**

这个文件夹是用来存放你博客的文本内容。

打开 `/source` 之后，发现有一个 `_post` 文件夹，此文件夹中就是存放你所有博文内容的文件夹，博文是用markdown 来写的，写好之后，放入 `_post` 即可。除此之外 `source` 还可以加入 `about` 等文件夹，用来写 `关于我` 等页面的文本内容。

**theme**

theme 自然就是放置博客主题的文件夹，比如源生的主题：`landscape`。自己通过 clone 安装的三方主题也会出现在这里。

**_config.yml 等**

剩下的这些就是配置文件等，除此之外其他的我也不知道是些什么东西。

在了解了 hexo 整体的目录结构之后，就具体讲一讲让我蛋疼的坑。

我在 clone 完 hexo 这个项目之后，发现主题不是很好看，就想换一个主题，于是乎我去找了个漂亮的第三方主题并且 clone 到了 `/theme` 文件夹下。然后发现这个主题需要做一些配置，所以就在 `/theme/xxx主题/..` 怒改造了一下，之后`hexo g` `hexo s`， 然后预览了一下，嗯~~还不错，于是 `hexo d` 将博客的静态文件 push 到了我的 github 的 maquannene.github.com 仓库的 master 分支上面。最后根据当时使用 Octopress 的经验，我又将 hexo 的源码我也要推到 maquannene.github.com 的另一个分支，比如 hexo。下一次换个新环境写博客时，只用将 clone 一下 hexo 这个分支，然后写写博文，`hexo g` `hexo d` 就ok了。

在确定已经将 hexo 文件 post 到 github 之后，我将本地的 hexo 文件夹直接删掉，准备尝试一下重新拉取自己 github 上 hexo 项目，继续写博客的流程。

如上面所述，clone 自己 github 的 hexo，执行 `hexo g`，这是我发现报了很多 WARNING：</br>
![无主题生成错误](http://ww3.sinaimg.cn/large/65312d9agw1ezy032rvmej20b403zt9g.jpg)

在执行 `hexo s` 后访问：`localhost：4000` 发现什么也没有显示。
于是我检查了一遍 clone 下来的 hexo，发现是我先前安装的的 `yelee` 主题文件夹中什么文件都没有，默认的主题 `landscape` 却存在。我以为是 github clone 出了问题，然后我打开我 github 的 hexo 项目，发现 theme/yelee 文件夹是黑色的：</br>
![无主题文件](http://ww4.sinaimg.cn/large/65312d9agw1ezy032cl8uj20b406ct94.jpg)

这时候我意识到，我在 push hexo 时，并没有将所有的文件 push 上去，于是我默默地从回收站找回来删掉的那个 hexo，仔细查看为什么主题没有被 push 上去。我又尝试在更改了一下 `/yelee` 中的文件，这时，我在 iterminal 中输入 `git diff` ，看到了如下信息：</br>
![gitdiff子repo](http://ww2.sinaimg.cn/large/65312d9agw1ezy0323yb3j20dw03cwf3.jpg)

用了一段时间 git，但是从来没有见过 `Subproject` 这个关键字，于是 google 一下，发现原来是一个被 git 版本管理的 repo 中的某个文件夹被另外一个 git 进行版本管理着，于是我进到 `/theme/yelee` 中，输入 `ls -a` 发现真有一个 `.git` 文件，然后又输入 `git remote -v`，发现当前 `yelee` 主题仓库的 origin 是另外一个远程主机，而不是我自己 github 博客 repo 的主机，恍然大悟，原来在 clone yelee 时，也将 `.git` 文件 clone 了下来，那么这个 repo 自然就被原来的 git 版本管理。所以当我处在外面文件夹的版本控制中进行 add commit push 时，对里面受另外一个版本控制的文件是无效的。

到这里，终于弄明白为什么 yelee 没有被 push 上去了。

于是我在我的 github 中创建了一个主题仓库，并且删除了 yelee 的 origin remote，添加新的 origin 为自己的主题仓库地址，最后将 clone 下来的被自己改过的主题 push 到了我自己的主题 repo 中。

同样我有建立了一个 hexo 的仓库，将 hexo 推到了 `hexo` repo 中，而不是 `xxx.github.com` repo 的 hexo 分支中。

而当执行 `hexo d` 时，静态网页仍然部署到的是 xxx.github.com 仓库的 master 分支。

那么目前为止，我将原来的一个仓库两个分支（ hexo 源码分支，静态网页分支）变成了三个仓库（hexo 源码仓库：`hexo`，静态网页仓库：`xxx.github.com`，主题仓库：`yelee-theme`），每个仓库有自己的主分支。

这样，我下次拉取博客时，不但需要将 hexo repo 克隆下来，并且还要将自己的主题也从主题仓库中克隆下来。当我更改了博文时，就 push 到 `hexo` repo ，改了主题时，就 push 到 `yelee-theme` repo 中，而 `hexo d` 部署时，会自动 push 到 `xxx.github.com` repo 中。

最后这样的结构看来是比原来清晰多了，哎，还是自己将 branch 分支这个概念乱用了，如果是定位不同的文件，就要用 repo 仓库进行区分，而不是推到不同的分支，这是不符合 git 的分支理念的。

