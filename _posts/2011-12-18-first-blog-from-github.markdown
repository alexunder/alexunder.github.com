---
layout: post
title: Github & Jekyll 我的新旅程
category: blog
---

# 前言

虽然我已经使用了一阵子github了，但是知道她有部署blog的功能，还是最近几天。于是迫不及待，马上折腾。很久没有自己的blog，还是免费的，很是期待，以前弄过一个独立空间的，无奈最终输给了时间和自己的懒惰，关闭了。

# 开始折腾

于是就按照文档开始配置，开启新的git都好说，可是一到配置Ruby的环境的时候，总出问题：刚开始的时候想用 [Octopress](http://octopress.org) ，需要Ruby1.9.2的环境，可是这个1.9.2死活装不上，Gem是装好了，用Gem装Ruby1.9.2时，总是提示安装成功了，可是当运行：

{% highlight bash %}
ruby -v
{% endhighlight %}

得到的信息总是：查不到此程序，建议安装。后来没辙了用apt命令安装：

{% highlight bash %}
sudo apt-get install ruby
{% endhighlight %}

但是我想要的Ruby1.9.2，始终没有，能装的只有1.8和1.9.1，我对OctoPress绝望了。

# 希望背后的失望

后来了解到了Jekyll，我看到一丝希望的微光，她对Ruby的版本没那么多要求，于是乎，继续弄。但是事实告诉我，Ruby这个关卡之于我就像山海关之于奴尔哈赤，出现无数艰难险阻，第一步安装Jekyll就过不去，运行文档中的命令：

{% highlight bash %}
gem install Jekyll
{% endhighlight %}

总是失败，我后来一怒之下，心生一计：本地安装。无外乎Google之，下载之，一安装：

{% highlight bash %}
gem install --local ~/Download/jekyll.0.4.0.gem
{% endhighlight %}

又提示出需要其他库依赖，我继续google之，下载之，安装之，我一口气安装了好多gem：

```bash
bundler (0.7.0)
classifier (1.3.1)
diff-lcs (1.1.3)
directory_watcher (1.4.1)
jekyll (0.4.1)
json (1.6.3)
kramdown (0.13.4)
liquid (2.3.0, 2.0.0)
maruku (0.6.0)
open4 (1.3.0)
Platform (0.4.0)
rake (0.8.7)
rdiscount (1.6.8)
RedCloth (4.2.9, 4.2.2)
redgreen (1.2.2)
rr (1.0.4)
stemmer (1.0.1)
syntax (1.0.0)
term-ansicolor (1.0.7)
```

我以为我快大功告成了，而且运行jekyll -v还有输出，我在本地的jekyll的git下编译，有很多编译错误，我又安装了ruby-1.9-dev，可是错误依旧。我仰天长啸，Ruby怎么这么坑爹？正当我心灰意冷时，我做了一个决定，我决定把这个困难绕过去。

# 柳暗花明

我直接clone了Jekyll开发者兼github联合创始人mojombo的blog数据，我将index.html,post.html等文件一顿乱改，又删除了丫以前的文章，然后git push上去，页面果然出来了。我虽然不能在本地预览我的文章，但是我可以通过不停的git push来预览，仍然可以装逼的Bloging like a hacker.

# 其他

可以免费的使用酷逼了的github&Jekyll提供的blog服务，自然很高兴。还有，这几个月来我经历了一次我个人使用计算机的革命：我删除了Windows XP, 彻底投向了Ubuntu阵营，目前来看，有时还需virtualbox里的WinXP的小帮助。但是我已经决意和Microsoft Windows说拜拜了。