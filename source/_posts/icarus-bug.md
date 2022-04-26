---
title: icarus-bug
date: 2019-04-18 15:56:47
tags: icarus
author: ygdxd
type: 原创
---

##记一次hexo切换主题icarus排版混乱的bug

######起因

第一次使用hexo搭建blog,看到icarus主题漂亮就入了坑，本地运行完美。
仿照国内教程各种详细的配置。

http://blog.kimzing.com/

准备remote上传

{% codeblock %}
sudo hexo d -g
{% endcodeblock %}

一切ok,打开网站却发现左边的介绍去了中间，样式全无。按F12查看控制台发现并没有脚本或者css报错信息，只能google. 搜索内容如下

<!-- more -->

1.查看css文件完整性

打开F12对比发现完全一下

2.修改根目录下的url

设置完发现无用


######结果

在下面helloworld中发现了类似图片404的图片，于是就想着先删除它，在本地rm helloworld.md后 

{% codeblock %}
sudo hexo g
sudo hexo d
{% endcodeblock %}

hellowold 没删掉 排版好了。。。
