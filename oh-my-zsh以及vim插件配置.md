# oh-my-zsh

1. `echo $SHELL`
2. `sudo apt install zsh`
3. `chsh -s /usr/zsh`
4. 重启
5. `echo $SHELL`
6. 安装：`sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"`
7. 修改主题为agnoster
8. 安装字体：
   1. `git clone https://github.com/powerline/fonts.git --depth=1`
   2. `cd fonts`
   3. `./install.sh`
   4. `rm -rf fonts`
9. 设置字体
10. 隐藏用户名，添加`prompt_context(){}`到`~/.zshrc`

# vim插件

1. `vim --version`
2. 安装Vundle.vim：`git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim`
3. 新建`.vimrc`文件，内容：

```
set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" The following are examples of different formats supported.
" Keep Plugin commands between vundle#begin/end.
" plugin on GitHub repo
Plugin 'tpope/vim-fugitive'
" plugin from http://vim-scripts.org/vim/scripts.html
" Plugin 'L9'
" Git plugin not hosted on GitHub
Plugin 'git://git.wincent.com/command-t.git'
" git repos on your local machine (i.e. when working on your own plugin)
Plugin 'file:///home/gmarik/path/to/plugin'
" The sparkup vim script is in a subdirectory of this repo called vim.
" Pass the path to set the runtimepath properly.
Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
" Install L9 and avoid a Naming conflict if you've already installed a
" different version somewhere else.
" Plugin 'ascenator/L9', {'name': 'newL9'}

" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
```

4. 运行`vim`，然后运行 `:PluginInstall` 安装

## 自动补全

`Plugin 'Valloric/YouCompleteMe'`

### 安装

1. `sudo apt install build-essential cmake python3-dev`
2. `cd ~/.vim/bundle/YouCompleteMe`
3. `python3 install.py --java-completer`

### 配置



## 项目树

`Plugin 'scrooloose/nerdtree'`

### 配置

```
" NERDTree config
" NERDTree 配置
" open a NERDTree automatically when vim starts up
" 自动打开NERDTree
autocmd vimenter * NERDTree
"open a NERDTree automatically when vim starts up if no files were specified
" 直接输入vim会打开NERDTree
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 0 && !exists("s:std_in") | NERDTree | endif
"open NERDTree automatically when vim starts up on opening a directory
" 打开一个目录也会打开NERDTree
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 1 && isdirectory(argv()[0]) && !exists("s:std_in") | exe 'NERDTree' argv()[0] | wincmd p | ene | endif
"map F2 to open NERDTree
map <F2> :NERDTreeToggle<CR>
"close vim if the only window left open is a NERDTree
" 当文件都关闭只有NERDTree时自动退出
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree") && b:NERDTree.isTabTree()) | q | endif
```



## Git 增删状态

`Plugin 'Xuyuanp/nerdtree-git-plugin'`

## .vimrc文件

```
set nocompatible              " be iMproved, required
filetype off                  " required
set encoding=utf-8

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" The following are examples of different formats supported.
" Keep Plugin commands between vundle#begin/end.
" plugin on GitHub repo
Plugin 'tpope/vim-fugitive'
" plugin from http://vim-scripts.org/vim/scripts.html
" Plugin 'L9'
" Git plugin not hosted on GitHub
" Plugin 'git://git.wincent.com/command-t.git'
" git repos on your local machine (i.e. when working on your own plugin)
" Plugin 'file:///home/gmarik/path/to/plugin'
" The sparkup vim script is in a subdirectory of this repo called vim.
" Pass the path to set the runtimepath properly.
Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
" Install L9 and avoid a Naming conflict if you've already installed a
" different version somewhere else.
" Plugin 'ascenator/L9', {'name': 'newL9'}
" 自动补全
Plugin 'Valloric/YouCompleteMe'
" 项目树
Plugin 'scrooloose/nerdtree'
" Git增删状态
Plugin 'Xuyuanp/nerdtree-git-plugin'
" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
" NERDTree config
" NERDTree 配置
" open a NERDTree automatically when vim starts up
" 自动打开NERDTree
autocmd vimenter * NERDTree
"open a NERDTree automatically when vim starts up if no files were specified
" 直接输入vim会打开NERDTree
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 0 && !exists("s:std_in") | NERDTree | endif
"open NERDTree automatically when vim starts up on opening a directory
" 打开一个目录也会打开NERDTree
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 1 && isdirectory(argv()[0]) && !exists("s:std_in") | exe 'NERDTree' argv()[0] | wincmd p | ene | endif
"map F2 to open NERDTree
map <F2> :NERDTreeToggle<CR>
"close vim if the only window left open is a NERDTree
" 当文件都关闭只有NERDTree时自动退出
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree") && b:NERDTree.isTabTree()) | q | endif
```

