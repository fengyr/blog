---
layout: post
title: Ubuntu--Vim-Ctags-Cscope-Taglist
categories:
- programmer
tags:
- tools
---


本文简单介绍在 Ubuntu 下，使用 vim 中的几个插件(Ctags、Cscope和TagList）实现一个比较好用的编辑器

##一 软件安装
Ubuntu 给我们安装软件提供了很大的便利，直接可以使用 apt-get install software_name		
如下安装 vim, ctags, cscope

	sudo apt-get install vim
	sudo apt-get install ctags
	sudo apt-get install cscope

当然，如果你愿意，可以下载源码安装

	vim			http://www.vim.org/download.php
	ctags		http://ctags.sourceforge.net/
	cscope		http://cscope.sourceforge.net/

对于 Taglist 使用这个方法安装不了,我们需要先下载，然后安装完成		
首先上网下载 Taglist 插件，下载完成后解压，再将文件下的 taglist.vim 使用 cp 命令拷贝到HOME/.vim/plugin文件夹下

	http://www.vim.org/scripts/script.php?script_id=273

这样，vim, taglist, cscope, ctags 四种工具我们是安装好了


##二 Vim 简介及使用
Vim是从vi发展出来的一个文本编辑器。代码补完、编译及错误跳转等方便编程的功能特别丰富，在程序员中被广泛使用。
和Emacs并列成为类Unix系统用户最喜欢的编辑器。

基本使用，请参考 Vim 自带的 vimtutor。	
只需要在终端中输入vimtutor (有的系统中可能是vim-tutor).		

Vim 的配置配置文件，在 HOME/.vimrc， 本文后面有参考配置


##三 Ctags 简介及使用
Ctags是一个用于从程序源代码树产生索引文件（或tag文件)，从而便于文本编辑器来实现快速定位的实用工具。		
在产生的tag文件中，每一个tag的入口指向了一个编程语言的对象。这个对象可以是变量定义、函数、类或其他的物件。

常用的 ctags 命令		

	1	ctags –R *		
		“-R”表示递归创建，也就包括源代码根目录（当前目录）下的所有子目录。“*”表示所有文件		
		这条命令会在当前目录下产生一个“tags”文件		
	2	Ctrl + ] 跳到光标所在函数或者结构体的定义处		
	3	Ctrl + T 返回查找或跳转


##四 Cscope 简介及使用
cscope 是一支面向屏幕的（与面向行相对）交互式C源代码浏览程序。它可以对C程序源代码的元素（例如各种标号：变量，
宏以及函数调用等）进行索引，提供简单的字符查询界面，用户可根据【元素名】对其定义及引用进行查询查看

cscope的用法很简单，首先需要为你的代码生成一个cscope数据库。在你的项目根目录运行下面的命令：

	cscope -Rbkq

这个命令会生成三个文件：
	cscope.out, cscope.in.out, cscope.po.out
其中cscope.out是基本的符号索引，后两个文件是使用”-q“选项生成的，可以加快cscope的索引速度。

Cscope缺省只解析C文件(.c和.h)、lex文件(.l)和yacc文件(.y)，虽然它也可以支持C++以及Java，但它在扫描目录时会跳过C++及Java后缀的文件。
如果你希望cscope解析C++或Java文件，需要把这些文件的名字和路径保存在一个名为cscope.files的文件。当cscope发现在当前目录中存在cscope.files时，
就会为cscope.files中列出的所有文件生成索引数据库。通常我们使用find来生成cscope.files文件，以vim 7.0的源代码为例：

	cd ~/src/vim70 
	find . -type f > cscope.files
	cscope -bkq 

	or

	find . -name "*.h" -o -name "*.c" -o -name "*.cpp" -o -name "*.java" \
		-o -name "*.cc" > cscope.files
	cscope -bkq -i cscope.files 

这条命令把~src/vim70目录下的所有普通文件都加入了cscope.files，这样，cscope会解析该目录下的每一个文件。
上面的cscope命令并没有使用”-R“参数递归查找子目录，因为在cscope.files中已经包含了子目录中的文件。

下表中列出了cscope的常用选项：

	-R: 在生成索引文件时，搜索子目录树中的代码
	-b: 只生成索引文件，不进入cscope的界面
	-q: 生成cscope.in.out和cscope.po.out文件，加快cscope的索引速度
	-k: 在生成索引文件时，不搜索/usr/include目录
	-i: 如果保存文件列表的文件名不是cscope.files时，需要加此选项告诉cscope到哪儿去找源文件列表。
		可以使用”-“，表示由标准输入获得文件列表。
	-Idir: 在-I选项指出的目录中查找头文件
	-u: 扫描所有文件，重新生成交叉索引文件
	-C: 在搜索时忽略大小写
	-Ppath: 在以相对路径表示的文件前加上的path，这样，你不用切换到你数据库文件所在的目录也可以使用它了。


在vim中使用cscope非常简单，首先调用”cscope add“命令添加一个cscope数据库，然后就可以调用”cscope find“命令进行查找了。
Vim 支持8种cscope的查询功能，如下：

	c:Find functions calling this function//查找调用本函数的函数
	d:Find functions called by this function //查找本函数调用的函数
	e:Find this egrep pattern//查找egrep模式，相当于egrep功能，但查找速度快多了
	f:Find this file //查找并打开文件，类似vim的find功能
	g:Find this definition//查找函数、宏、枚举等定义的位置，类似ctags的功能
	i:Find files #including this file //查找包含本文件的文件
	s:Find this C symbol //查找C语言符号，即查找函数名、宏、枚举值等出现的地方
	t:Find assignments to //查找指定的字符串

例如，我们想在代码中查找调用work()函数的函数，我们可以输入：“:cs f c work”


##五 Taglist 简介及使用
TagList插件,是一款基于ctags,在vim代码窗口旁以分割窗口形式显示当前的代码结构概览,增加代码浏览的便利程度的vim插件

常用的 Taglist 命令		

	1	TlistOpen(直接Tlist也可）打开并将输入焦点至于标签列表窗口
	2	TlistClose关闭标签列表窗口
	3	TlistToggle切换标签列表窗口状态(打开←→关闭)，标签列表窗口是否获得焦点取决于其他配置
	4	o:新建一个窗口,跳到标记定义处
	5	p:预览标记定义(仍然在taglist窗口)
	6	空格:显示标记的原型(如函数原型)
	7	u:更新标记列表(比如源文件新增了一个函数,并在保存后,可在taglist窗口按u)
	8	s:选择排序字段(暂时我也不知道什么意思)
	9	x:放大/缩小taglist窗口


##六 初次使用
1	安装好 vim, ctags, cscope, taglist		
	参考上面说明	
2	设置好 vim 配置文件 HOME/.vimrc		
	主要是配置好 ctags, cscope, taglist		
3	切换到项目的源码目录下		
	cd PROJECT/src		
	find . -type f > cscope.files		
	cscope -bkq -i cscope.files		
	ctags -R *		
4	打开一个文件查看	
	vim xxx.java	

	taglist 使用
		打开 taglist 窗口
		:Tlist
		其他命令参考上面说明

	ctags 使用
		Ctrl + ] 跳到光标所在函数或者结构体的定义处
		Ctrl + T 返回查找或跳转
		其他命令参考上面说明

	cscope 使用
		查找调用本函数的函数
		:cs f c function_name
		其他命令参考上面说明

		或者快捷键
		nmap <C-@>c :cs find c <C-R>=expand("<cword>")<CR><CR>

		光标停在函数上
		ctrl + super + c
	

##七 Vim 智能补全
后续


##八 参考资料
Ubuntu下创建vim+Taglist+cscope+ctags组合编辑器		
http://blog.csdn.net/longerzone/article/details/7789581

vi/vim使用进阶: 程序员的利器 – cscope		
http://easwy.com/blog/archives/advanced-vim-skills-cscope/

vim中taglist使用(比较详细的)		
http://blog.csdn.net/vaqeteart/article/details/4146618

Vim中自动加载cscope.out		
http://blog.csdn.net/mci2004/article/details/7944074

vim+cscope		
http://blog.163.com/sunshine_linting/blog/static/4489332320116234512574/

使用Cscope阅读Linux源码			
http://www.360doc.com/content/11/0117/15/1921373_87127589.shtml


##附 Vim 配置文件
HOME/.vimrc		
===== Begin

	" ###################################################################
	" vimrc 配置
	" Edit By Gene
	" Update At 2014-03-17
	" ###################################################################


	" 基本设置
	"--------------------------------------------------------------------
	set nocompatible        " 关闭 vi 兼容模式 
	syntax on               " 自动语法高亮
	syntax enable           " 语法高亮
	filetype on             " 打开文件类型检测
	"set cursorline         " 突出显示当前行 
	set ruler               " 打开状态栏标尺 
	set shiftwidth=4        " 设定 << 和 >> 命令移动时的宽度为 4 
	set softtabstop=4       " 使得按退格键时可以一次删掉 4 个空格 
	set tabstop=4           " 设定 tab 长度为 4 
	set laststatus=2        " 显示状态栏 (默认值为 1, 无法显示状态栏) 
	set nobackup            " 覆盖文件时不备份
	set noswapfile          " 不需要 .swp 文件
	set incsearch           " 输入搜索内容时就显示搜索结果 
	set hlsearch            " 搜索时高亮显示被找到的文本 
	set noerrorbells        " 关闭错误信息响铃 
	set smartindent         " 开启新行时使用智能自动缩进 
	set pastetoggle=<F9>    " 插入模式下，按F9键就切换自动缩进

	" VIM主题
	" colorscheme darkblue
	colorscheme elflord

	" 记住上次打开的位置
	au BufReadPost * if line("'\"") > 0|if line("'\"") <= line("$")|exe("norm '\"")|else|exe "norm $"|endif|endif

	" 设置在状态行显示的信息 
	set statusline=\ %<%F[%1*%M%*%n%R%H]%=\ %y\ %0(%{&fileformat}\ %{&encoding}\ %c:%l/%L%)\

	" 注释颜色
	hi Comment ctermfg=3

	" 修改声明颜色
	" hi Statement ctermfg=3
	" 修改常量颜色
	" hi Constant ctermfg=3
	" 修改字符串颜色
	" hi String ctermfg=3
	" 修改类型颜色
	" hi Type ctermfg=3
	" 修改数字颜色
	" hi Number ctermfg=3


	" 中文乱码  
	set fileencodings=gb2312,gb18030,utf-8
	set termencoding=utf-8

	" 自动补全
	filetype plugin indent on
	set completeopt=longest,menu

	" 阅读代码设置
	"====================================================================
	" ctags
	"--------------------------------------------------------------------
	" map <F4> :!ctags -R --c++-kinds=+p --fields=+iaS --extra=+q .<CR><CR> 
	set tags=tags;
	set autochdir


	" taglist
	"--------------------------------------------------------------------
	" map <F3> : Tlist<CR>                      " 按下F3就可以呼出了
	let Tlist_Auto_Open = 0                     " 在启动VIM后，自动打开taglist窗口
	" let Tlist_Ctags_Cmd = '/usr/bin/ctags'    " 设定ctags的位置
	let Tlist_Use_Right_Window=0                " 1 为让窗口显示在右边，0 为显示在左边 
	let Tlist_Show_One_File=1                   " 让taglist可以同时展示多个文件的函数列表，设置为1时不同时显示多个文件的tag，只显示当前文件的 
	let Tlist_File_Fold_Auto_Close=1            " 同时显示多个文件中的tag时，taglist只显示当前文件tag，其他文件的函数列表折叠隐藏  
	let Tlist_Exit_OnlyWindow=1                 " 当taglist是最后一个分割窗口时，自动退出vim  
	"let Tlist_Use_SingleClick=1                " 缺省情况下，在双击一个tag时，才会跳到该tag定义的位置
	"let Tlist_Process_File_Always=0            " 是否一直处理tags.1:处理;0:不处理 
	let TList_GainFocus_On_ToggleOpen=0         " 打开时焦点不放在tl窗口中
	let Tlist_WinWidth=50                       " Tlist_WinHeight和Tlist_WinWidth可以设置taglist窗口的高度和宽度


	" cscope
	"--------------------------------------------------------------------
	" set cscopequickfix=s-,c-,d-,i-,t-,e-
	if has("cscope")
		set csprg=/usr/bin/cscope
		set csto=1
		set cst
		set nocsverb
		" add any database in current directory
		if filereadable("cscope.out")
			cs add cscope.out
		" else search cscope.out elsewhere
		else
			let cscope_file=findfile("cscope.out", ".;")
			let cscope_pre=matchstr(cscope_file, ".*/"
			if !empty(cscope_file) && filereadable(cscope_file
				exe "cs add" cscope_file cscope_pre
			endif
		endif
		set csverb
	endif

	nmap <C-@>s :cs find s <C-R>=expand("<cword>")<CR><CR>
	nmap <C-@>g :cs find g <C-R>=expand("<cword>")<CR><CR>
	nmap <C-@>c :cs find c <C-R>=expand("<cword>")<CR><CR>
	nmap <C-@>t :cs find t <C-R>=expand("<cword>")<CR><CR>
	nmap <C-@>e :cs find e <C-R>=expand("<cword>")<CR><CR>
	nmap <C-@>f :cs find f <C-R>=expand("<cfile>")<CR><CR>
	nmap <C-@>i :cs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
	nmap <C-@>d :cs find d <C-R>=expand("<cword>")<CR><CR>


===== End


