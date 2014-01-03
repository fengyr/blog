---
layout: post
title: github pages构建blog步骤
categories:
- programmer
tags:
- git
---


###前言
喜欢写Blog的人，会经历三个阶段：
- 第一阶段，刚接触Blog，觉得很新鲜，试着选择一个免费空间来写。
- 第二阶段，发现免费空间限制太多，就自己购买域名和空间，搭建独立博客。
- 第三阶段，觉得独立博客的管理太麻烦，最好在保留控制权的前提下，让别人来管，自己只负责写文章。
大多数Blog作者，都停留在第一和第二阶段，因为第三阶段不太容易到达：你很难找到俯首听命、愿意为你管理服务器的人。


示例博客
https://github.com/jekyll/jekyll/wiki/Sites
https://github.com/kejinlu/kejinlu.github.com
git clone https://github.com/kejinlu/kejinlu.github.com.git
http://geeklu.com/

参考网址
http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html


###一	Github Pages 是什么？
github Pages可以被认为是用户编写的、托管在github上的静态网页。
github提供模板，允许站内生成网页，但也允许用户自己编写网页，然后上传。有意思的是，这种上传并不是单纯的上传，而是会经过Jekyll程序的再处理。


###二	Jekyll是什么？
Jekyll 是一个静态站点生成器，它会根据网页源码生成静态文件。它提供了模板、变量、插件等功能，所以实际上可以用来编写整个网站。

####1	Ubuntu下搭建Jekyll环境
其实可以把所有的修改直接上传到github上让服务器帮忙解析网页来观察效果，但是如果想在本地调试的话本地也要装一套jekyll

步骤一：安装ruby环境
	sudo apt-get install ruby ruby-dev

步骤二：更换Gem sources
苦B的天朝码畜们肯定是世界上耐心最好的Coder，各种源都速度奇慢，包括Ruby的Gem sources，好在国内有个不错的加速镜像，taobao的ruby源，
所以在开始安装前，我们需要替换gem sources为淘宝的镜像
	sudo gem sources --remove https://rubygems.org/
	sudo gem sources -a http://ruby.taobao.org/

为了确认下替换是否成功，我们可以检查下
	sudo gem sources -l

如果替换成功，则会看到如下返回：
	*** CURRENT SOURCES ***

	http://ruby.taobao.org

步骤三：安装Jekyll
	sudo gem install jekyll

步骤四：启动jekyll服务
	$ jekyll serve
	Configuration file: /home/peter/develop/github_pages/jekyll_demo2/jekyll_demo/_config.yml
			    Source: /home/peter/develop/github_pages/jekyll_demo2/jekyll_demo
		   Destination: /home/peter/develop/github_pages/jekyll_demo2/jekyll_demo/_site
		  Generating... done.
		Server address: http://0.0.0.0:4000
	  Server running... press ctrl-c to stop.

可以通过 http://0.0.0.0:4000 来访问自己的blog了


####2	Jekyll基本语法
参考网址
http://www.zhanxin.info/jekyll/2013-08-07-jekyll-doc-installation.html


####3	Jekyll Bootstrap
待写


###三	Markdown 是什么？
Markdown 语法说明
http://wowubuntu.com/markdown/


###四	小结
整个思路到这里就很明显了。你先在本地编写符合Jekyll规范的网站源码，然后上传到github，由github生成并托管整个网站。

博客使用 git 管理，Markdown用来写博客，写好后提交到github上，github上使用Jekyll把博客生成静态网页，就可以访问了。
当然，写好的博客在提交前，可以本地通过Jekyll调试，确定有没有语法错误。

这种做法的好处是：、
- 免费，无限流量。
- 享受git的版本管理功能，不用担心文章遗失。
- 你只要用自己喜欢的编辑器写文章就可以了，其他事情一概不用操心，都由github处理。

它的缺点是：
- 有一定技术门槛，你必须要懂一点git和网页开发。
- 它生成的是静态网页，添加动态功能必须使用外部服务，比如评论功能就只能用disqus。
- 它不适合大型网站，因为没有用到数据库，每运行一次都必须遍历全部的文本文件，网站越大，生成时间越长。

但是，综合来看，它不失为搭建中小型Blog或项目主页的最佳选项之一。


###五	一个实例
下面，我举一个实例，演示如何在github上搭建blog，你可以跟着一步步做。为了便于理解，这个blog只有最基本的功能。
在搭建之前，你必须已经安装了git，并且有github账户。

####1	创建项目。
在你的电脑上，建立一个目录，作为项目的主目录。我们假定，它的名称为jekyll_demo。
	$ mkdir jekyll_demo

对该目录进行git初始化。
	$ cd jekyll_demo
	$ git init

然后，创建一个没有父节点的分支gh-pages。因为github规定，只有该分支中的页面，才会生成网页文件。
	$ git symbolic-ref HEAD refs/heads/gh-pages
	$ rm .git/index 
	$ git clean -fdx 
	$ <do work> 
	$ git add your files 
	$ git commit -m "Initial commit"

以下所有动作，都在该分支下完成。


####2	创建设置文件。
在项目根目录下，建立一个名为_config.yml的文本文件。它是jekyll的设置文件，我们在里面填入如下内容，其他设置都可以用默认选项，
具体解释参见官方网页。

	baseurl: /jekyll_demo

目录结构变成：
	/jekyll_demo
		|--　_config.yml


####3	创建模板文件。
在项目根目录下，创建一个_layouts目录，用于存放模板文件。
	$ mkdir _layouts

进入该目录，创建一个default.html文件，作为Blog的默认模板。并在该文件中填入以下内容。

	<!DOCTYPE html>
	<html>
		<head>
			<meta http-equiv="content-type" content="text/html; charset=utf-8" />
			<title>{{ page.title }}</title>
		</head>
		<body>

			{{ content }}

		</body>
	</html>

Jekyll使用Liquid模板语言，{{ page.title }}表示文章标题，{{ content }}表示文章内容，更多模板变量请参考官方文档。
目录结构变成：

	/jekyll_demo
		|--　_config.yml
		|--　_layouts
		|　　　|--　default.html

####4	创建文章。
回到项目根目录，创建一个_posts目录，用于存放blog文章。
	$ mkdir _posts

进入该目录，创建第一篇文章。文章就是普通的文本文件，文件名假定为2012-08-25-hello-world.html。(注意，文件名必须为"年-月-日-文章标题.
后缀名"的格式。如果网页代码采用html格式，后缀名为html；如果采用markdown格式，后缀名为md。）

在该文件中，填入以下内容：（注意，行首不能有空格）

	---
	layout: default
	title: 你好，世界
	---
	<h2>{{ page.title }}</h2>
	<p>我的第一篇文章</p>
	<p>{{ page.date | date_to_string }}</p>

每篇文章的头部，必须有一个yaml文件头，用来设置一些元数据。它用三根短划线"---"，标记开始和结束，里面每一行设置一种元数据。
"layout:default"，表示该文章的模板使用_layouts目录下的default.html文件；
"title: 你好，世界"，表示该文章的标题是"你好，世界"，
如果不设置这个值，默认使用嵌入文件名的标题，即"hello world"。

在yaml文件头后面，就是文章的正式内容，里面可以使用模板变量。{{ page.title }}就是文件头中设置的"你好，世界"，
{{ page.date }}则是嵌入文件名的日期（也可以在文件头重新定义date变量），"| date_to_string"表示将page.date变量转化成人类可读的格式。

目录结构变成：

	/jekyll_demo
	　　　　|--　_config.yml
	　　　　|--　_layouts
	　　　　|　　　|--　default.html 
	　　　　|--　_posts
	　　　　|　　　|--　2012-08-25-hello-world.html


####5	创建首页。
有了文章以后，还需要有一个首页
回到根目录，创建一个index.html文件，填入以下内容。

	---
	layout: default
	title: 我的Blog
	---
	<h2>{{ page.title }}</h2>
	<p>最新文章</p>
	<ul>
		{% for post in site.posts %}
		<li>{{ post.date | date_to_string }} <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
		{% endfor %}
	</ul>

它的Yaml文件头表示，首页使用default模板，标题为"我的Blog"。然后，首页使用了{% raw %}{% for post in site.posts %}{% endraw %}
表示对所有帖子进行一个遍历。这里要注意的是，Liquid模板语言规定，输出内容使用两层大括号，单纯的命令使用一层大括号。
至于{% raw %}{{site.baseurl}}{% endraw %}就是_config.yml中设置的baseurl变量。

目录结构变成：

	/jekyll_demo
	　　　　|--　_config.yml
	　　　　|--　_layouts
	　　　　|　　　|--　default.html 
	　　　　|--　_posts
	　　　　|　　　|--　2012-08-25-hello-world.html
	　　　　|--　index.html


####6	发布内容。
现在，这个简单的Blog就可以发布了。先把所有内容加入本地git库。
	$ git add .
	$ git commit -m "first post"

然后，前往github的网站，在网站上创建一个名为jekyll_demo的库。接着，再将本地内容推送到github上你刚创建的库。
注意，下面命令中的username，要替换成你的username。
	$ git remote add origin https://github.com/username/jekyll_demo.git
	$ git push origin gh-pages

上传成功之后，等10分钟左右，访问http://username.github.com/jekyll_demo/就可以看到Blog已经生成了（将username换成你的用户名）。

	
####7	<p>绑定域名。需要自己购买域名，然后绑定到gitbhub上，
目前没有买域名，不处理。

如果你不想用http://username.github.com/jekyll_demo/这个域名，可以换成自己的域名。

具体方法是在repo的根目录下面，新建一个名为CNAME的文本文件，里面写入你要绑定的域名，比如example.com或者xxx.example.com。

如果绑定的是顶级域名，则DNS要新建一条A记录，指向204.232.175.78。
如果绑定的是二级域名，则DNS要新建一条CNAME记录，指向username.github.com（请将username换成你的用户名）。

此外，别忘了将_config.yml文件中的baseurl改成根目录"/"。
至此，最简单的Blog就算搭建完成了。进一步的完善，请参考Jekyll创始人的示例库，以及其他用Jekyll搭建的blog。



###六	需要处理的一些东西
- 博客评论
- 404.md







