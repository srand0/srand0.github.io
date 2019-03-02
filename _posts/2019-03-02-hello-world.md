---
layout: post
title:  'Hello World!'
subtitle:  '搭建Jekyll博客过程.'
date:    '2019-03-02 22:10:28'
background:  /image/post1.jpg
---
研究了一天，终于搞定了Jekyll博客的搭建，Jekyll可以将markdown内容生成静态页面，再加上Github的支持，可以免费部署一个简单的博客服务。

搭建的过程大致分为四个步骤：
- 安装环境
- 搭建博客
- 使用主题
- 使用中文
<br/><br/>

### 1.安装环境

环境的安装主要参照Jekyllrb的[官方教程](https://jekyllrb.com/docs/installation/)，Jekyll是基于Ruby语言的，所以需要安装Ruby的dev安装包。
不同的系统环境安装方式不同，所以不一一说明，我在安装的时候主要问题在于Windows下的完整Ruby-dev安装包可能需要翻墙下载。

### 2.搭建博客

在安装完Ruby环境后，可以按照官方给出的[教程](https://jekyllrb.com/docs/)搭建最基础的博客，搭建的过程非常简单，只需要四个命令：


{% highlight ruby %}
#Install Jekyll and bundler gems
gem install jekyll bundler

#Create a new Jekyll site at ./myblog
jekyll new myblog

#Change into your new directory
cd myblog

#Build the site and make it available on a local server
bundle exec jekyll serve
{% endhighlight %}

执行完成后，下一步就是启动服务，服务默认使用的是4000端口，如果4000端口被占用，可以使用其他端口，只需要在_config.yml文件中添加配置：
{% highlight ruby %}
port:  8089
{% endhighlight %}
然后打开浏览器:localhost:8089即可看到搭建好的博客。

### 3.使用主题
上面搭建完成的博客只是Jekyll博客系统默认的主题，如果需要使用不同的主题美化博客，需要按照[官方教程](https://jekyllrb.com/docs/themes/)操作。
首先到[官方推荐网站](https://rubygems.org/search?utf8=%E2%9C%93&query=jekyll-theme)找到基于gem的主题，一般是名称类似jekyll-theme-xxx的主题。
安装主题分为四个步骤：

1) 编辑Gemfile，修改主题名称，默认主题名称为minima，修改为新主题名称xxx
> gem "minima", "~> 2.0"&emsp;&emsp;==>&emsp;&emsp;gem "xxx"

1) 安装，运行命令
> bundle install

3) 添加_config.yml配置，将以下内容添加到_config.yml文件中，xxx为主题名称
> theme: xxx

4) 编译页面
> bundle exec jekyll serve

最后需要根据新的主题，配置相关的页面。

### 4.使用中文
官网中的主题一般默认只支持英文，如果要添加中文支持，需要在_config.yml文件中设置文件编码，添加以下配置：
> encoding: utf-8

### 5.部署博客
使用了主题之后，部署的时候会有文件引用的问题，因为在本地试运行的时候，ruby会链接ruby安装根目录下的主题项目文件一起编译，
如果只提交项目中的文件会导致在github中无法编译，有两种方法可以解决这个问题：

第一种：将_config.yml中的theme参数修改为远程仓库的地址
> theme: xxx&emsp;&emsp;==>&emsp;&emsp;remote_theme: ref

ref即该主题github项目的ref，具体可以到主题项目查看。

第二种：将所有用到的文件拷贝到项目中一起提交，先找到主题项目目录：
> ruby根目录/lib/ruby/gems 2.5.0/gems/主题名称

然后将该目录下的所有文件，包括_includes、_layouts、_sass、assets等文件一起拷贝到项目中，然后一起提交即可。