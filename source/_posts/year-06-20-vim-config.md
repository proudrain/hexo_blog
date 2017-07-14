---
title: 从零开始配置你的个性化Vim
date: 2017-06-20 10:04:07
tags:
  - Vim
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/vim-config-cover.jpg)
> [*pixiv-ID: 42105534*](https://www.pixiv.net/member_illust.php?mode=medium&illust_id=42105534)

这是[Vim系列教程]()中的一篇，讲述如何将Vim配置为自己喜欢的样子。

<!--more-->

先放一张我的Vim配置后的截图：
![vim-screen](http://oanr6klwj.bkt.clouddn.com/blog/vim-screen.png)

Vim的配置文件位于`~/.vimrc`，文件使用VimScript语法来编写。
有必要为了Vim而专门去学习VimScript吗？我个人的建议是：不需要，你将在日后为了实现自己的想法折腾插件时慢慢了解VimScript。而且现在Vim社区的大量插件足够满足一般人的日常需要了。

所谓授人以鱼不如授人以渔，本文**不会进行大量的插件罗列（仅介绍个人认为必备的），而是教你如何安装自己想要的插件**，为日后的自行折腾铺平道路。

我将.vimrc文件分为三个部分，分别是：

<!-- toc -->

本文将按照顺序进行讲解。

---
# Vundle插件管理部分
一个理想的Vim编辑器总是离不开各种插件的，以前的时候，安装插件都需要手动下载对应的插件仓库，然后再拷贝到`~/.vim`文件夹的对应文件夹中，有的插件甚至需要手动`make`来安装，非常的麻烦。   
Vundle就是为此而生，它是一个全自动的插件管理器，让我们通过维护插件列表的方式管理插件。它为安装、更新、删除插件提供了方便的命令。Vundle也是我们唯一需要手动安装的插件。  
在安装Git的情况下（本文不赘述Git的安装），输入命令：

`$ git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim`

这样我们就把Vundle插件放到了Vim能够读取的地方，然后进入`~/.vimrc`文件中配置Vundle。

`$ vim ~/.vimrc`

打开配置文件，先忽略其他内容，将以下内容粘贴到文件顶部：
```vim
"Vundle Section Start
set nocompatible
filetype off
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
Plugin 'VundleVim/Vundle.vim'
" ADD YOUR PLUGIN
call vundle#end()
filetype plugin indent on
"Vundle Section End
```
我们无需关心这些代码做了什么，只需要知道，接下来如果需要安装插件，只要把插件添加在 `" ADD YOUR PLUGIN`的位置就可以了。插件在该位置的统一格式是:
`Plugin 'path'`
其中，path的格式分为三种：
* 第一种是github仓库中的插件，安装时可以省略github域名。例如[github.com/scrooloose/nerdtree ](https://github.com/scrooloose/nerdtree)，可以写为`'/scrooloose/nerdtree'`。
* 第二种是虽然在github仓库中，却是在非git仓库中的插件，这时就需要传入合适的参数，例如[github.com/rstacruz/sparkup ](https://github.com/rstacruz/sparkup)仓库中，Vim插件在该仓库的vim文件夹中，这时的格式为：`'rstacruz/sparkup', {'rtp': 'vim/'}`。这一功能也可以用来安装不同版本的同一插件，例如`''ascenator/L9', {'name': 'newL9'}'`。
* 第三种是位于[vim官方插件列表](http://vim-scripts.org/vim/scripts.html)中的插件，也就是[github.com/vim-scripts ](https://github.com/vim-scripts)中的插件，这部分可以直接输入插件名。例如[github.com/vim-scripts/L9 ](https://github.com/vim-scripts/L9)，可以直接写为`'L9'`
* 第四种是不在github上的git插件，此时要使用git前缀，并写全仓库名称和地址，例如：`'git://git.example.com/example.git'`
* 第五种是本地插件，此时使用file前缀，并写上绝对路径，例如：`'file:///User/me/path/to/plugin'`

添加好插件列表之后，我们就需要安装插件了。先在任意位置进入Vim`$ vim`，然后输入指令`:PluginInstall`即可。
Vundle内置了一些实用的命令让我们管理插件：
```vim
:PluginList "列出列表中的插件
:PluginInstall "安装插件
:PluginInstall! "更新插件
:PluginUpdate "更新插件
:PluginSearch foo "查找名中含有foo的插件
:PluginSearch! foo "查找前清除本地缓存
:PluginClean "清理不在列表中的插件
:PluginClean! "清理时不需用户同意
```

如果你发现有些插件不再需要了，只需要在插件列表中删除它，然后重启Vim，输入`:PluginClean`，Vundle就会帮我们删除它。

> 颜色主题往往也可以当作插件安装

---
# Vim编辑器配置部分
配置完插件后，我们就要配置一下编辑器了，Vim的配置自由度非常高，比如编辑器是否显示行数，使用空格还是tab等。这部分的配置通过搜索一般都可以找到，推荐[这个小工具](http://vimconfig.com/)，可以模拟Vim，帮你找到自己喜欢的配置。

值得一提的是，由于使用`vimscript`，所以我们可以在配置中加入一些逻辑，例如：
```vim
if strftime('%H') >= 21 || strftime('%H') <= 9
  set background=dark
else
  set background=light
endif
```
这个逻辑判断了当前的系统时间，当系统时间在晚上九点后到早上九点前会自动使用暗色背景，其他时间会使用亮色背景。

再例如在终端中开启256色并模拟GUI中的光标模式：
```vim
if !has('gui_running')
  set t_Co=256
  if has('termguicolors')
    set termguicolors
  end
  let &t_SI = "\<Esc>]50;CursorShape=1\x7"
  let &t_SR = "\<Esc>]50;CursorShape=2\x7"
  let &t_EI = "\<Esc>]50;CursorShape=0\x7"
  set timeoutlen=1000 ttimeoutlen=0
else
```
这样，插入模式的光标会变为竖线，替换模式的光标会变为下划线。

> 需要注意的是，颜色主题往往也定义在这一部分中。

---
# Vim插件配置部分
vim的很多插件都是需要配置的，插件的配置一般分为两种，一种是对插件本身的配置，另一种是对插件快捷键的配置。以著名插件[nerdtree](https://github.com/scrooloose/nerdtree)为例：
```vim
" 插件的配置
let NERDTreeWinSize=20
let NERDTreeWinPos="right"
let NERDTreeIgnore=['\~$', '\.pyc$', '\.swp$']
" 快捷键的配置
nmap <F5> :NERDTreeToggle<CR>
```
在插件的配置中，我们定义了nerdtree窗口的宽度，打开的方向和忽略指定格式的文件。在快捷键的配置中，我们用**映射模式**将`F5`映射为`:NERDTreeToggle`+`enter`来打开nerdtree。

**关于映射模式：**
定义映射模式时，我们可以使用`nmap`、`imap`、`vmap`来定义映射仅在normal、insert、visual模式有效。
一般的映射模式是有副作用的，例如：
```vim
nmap dd O<esc>jddk
```
这一命令想要将`dd`映射为：
* O向上添加一个新行
* esc返回normal模式
* j向下移动到要删除的一行
* dd删除这一行
* k向上移动到新增的一行

然而实际执行中，当你按下`dd`时，文件会无限刷出空行直到你按下<C-c>，这是因为这一命令中jddk中的dd也按照按键映射解读了。因此形成了一个死循环。
命令`noremap`解决了这个问题，每一个`map`命令都对应一个`noremap`命令。包括`noremap`、`nnoremap`、`inoremap`、`vnoremap`，它可以保证映射中的操作都遵循默认的操作。
这次我们使用`noremap`重新定义刚才的映射：
```vim
nnoremap dd O<esc>jddk
```
再次测试，发现不再出现死循环了。

**使用Leader键：**
vim中的组合键是通过按键序列来执行的，例如`qd`这个操作，你只需要先按下`q`再按下`d`就好了，而不需要qd一起按。
由于Vim已经占用了大量的按键，导致我们配置快捷键时处处受限。
由于有些按键在vim的非insert模式中几乎是永远不会用到的（例如逗号`,`），因此我们可以在快捷键的命令前统一加上这个键，方便好记又不会冲突。这个键就称为**Leader键**。
我们可以通过`let mapleader=","`这一命令将逗号设置为leader键（设置其它键的方法同理）。然后对前文中我们定义的映射`dd`做一些修改：
```vim
nnoremap <leader>dd O<esc>jddk
```
这下我们就可以通过`,dd`的组合键来调用映射了。

这就是对Vim基础配置的介绍内容，这里提供[我的配置文件](https://github.com/Saul-Mirone/vim_opt/blob/master/.vimrc)以供各位参考。