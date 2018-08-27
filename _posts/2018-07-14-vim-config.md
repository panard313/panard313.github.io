---
layout: article
title: vim 配置参数列表
key: 20180714
tags:
  - vim
lang: zh-Hans
---

# vim 配置参数列表
```json
"""""""""""""""""""""""""""""""""""""""""
" Author   : lucasysfeng
" Email    : lucasysfeng@gmail.com
" Blog     : http://cnblogs.com/lucasysfeng
" ReadMe   : README.md
" Version  : 1.0 - Jan. 16,2014
" Contents : -> 基础配置设置
"            -> 自定义快捷键
"            -> 主题颜色显示
"            -> 其它杂项配置
"            -> 插件管理配置
"            -> 自定义的函数
"""""""""""""""""""""""""""""""""""""""""

"""""""""""""""""""""""""""""""""""""""""
" 基础配置设置
"""""""""""""""""""""""""""""""""""""""""
filetype on                      "检测文件类型
filetype indent on               "针对不同的文件类型采用不同的缩进格式
filetype plugin on               "允许插件
filetype plugin indent on        "启动自动补全

set nobackup                     "不自动保存
"set paste
set relativenumber number        "相对行号，可用Ctrl+n在相对/绝对行号间切换
set history=2000                 "history存储长度
set nocompatible                 "非兼容vi模式,避免以前版本的一些bug和局限
set autoread                     "文件修改之后自动载入
set shortmess=atI                "启动的时候不显示那个援助索马里儿童的提示
set t_ti= t_te=                  "退出vim后，内容显示在终端屏幕
set title                        "change the terminal's title
set novisualbell                 "don't beep
set noerrorbells                 "don't beep
set t_vb=
set tm=500
set mat=2                        "Blink times every second when matching brackets
set showmatch                    "括号配对情况
set hidden                       "A buffer becomes hidden when it is abandoned
set wildmode=list:longest
set ttyfast
set wildignore=*.swp,*.,*.pyc,*.class
set scrolloff=7                  "至少有7行在光标所在行上下
set mouse=a                      "为所有模式启用鼠标
set selection=old
set selectmode=mouse,key
set viminfo^=%                   "Remember info about open buffers on close
set magic                        "正则表达式匹配形式
set backspace=eol,start,indent   "Configure backspace so it acts as it should act
set whichwrap+=<,>,h,l

" 搜索
"""""""""""""""""""""""""""""""""""""""""
set hlsearch                     "高亮search命中的文本。
set ignorecase                   "搜索时忽略大小写
set incsearch                    "随着键入即时搜索
set smartcase                    "有一个或以上大写字母时仍大小写敏感

" 折叠
"""""""""""""""""""""""""""""""""""""""""
set foldenable
set foldmethod=indent
set foldlevel=99

" 缩进
"""""""""""""""""""""""""""""""""""""""""
set smartindent                  "智能缩进
set autoindent                   "总是自动缩进
set tabstop=4                    "设置Tab键的宽度(等同的空格个数)
set shiftwidth=4                 "自动对齐的空格数
set softtabstop=4                "按退格键时可以一次删掉4个空格
set smarttab                     "insert tabs on the start of a line according to shiftwidth, not tabstop
set expandtab                    "将Tab自动转化成空格(需要输入真正的Tab键时,使用Ctrl+V+Tab)
set shiftround                   "Use multiple of shiftwidth when indenting with '<' and '>'

" 编码
"""""""""""""""""""""""""""""""""""""""""
set encoding=utf-8
set fileencodings=ucs-bom,utf-8,cp936,gb18030,chinese,big5,euc-jp,euc-kr,latin1
set fileencoding=utf-8
set helplang=cn
set termencoding=utf-8           "这句只影响普通模式 (非图形界面) 下的Vim
set ffs=unix,dos,mac             "Use Unix as the standard file type
set formatoptions+=m             "如遇Unicode值大于255的文本，不必等到空格再折行。
set formatoptions+=B             "合并两行中文时，不在中间加空格：

" 语法
"""""""""""""""""""""""""""""""""""""""""
syntax enable                    "打开语法高亮
syntax on

" 撤销
"""""""""""""""""""""""""""""""""""""""""
set undolevels=1000              "How many undos
set undoreload=10000             "number of lines to save for undo
if v:version >= 730
    set undofile                 "keep a persistent backup file
    set undodir=~/bak/vimundo/
endif

" 状态栏
"""""""""""""""""""""""""""""""""""""""""
set ruler                        "显示当前的行号列号
set showcmd                      "在状态栏显示正在输入的命令
set showmode
set statusline=%<%f\ %h%m%r%=%k[%{(&fenc==\"\")?&enc:&fenc}%{(&bomb?\",BOM\":\"\")}]\ %-14.(%l,%c%V%)\ %P
set laststatus=2                 "命令行（在状态行下）的高度，默认为1，这里是2

" 相对绝对行号
"""""""""""""""""""""""""""""""""""""""""
"autocmd FocusLost * :set norelativenumber number
autocmd FocusGained * :set relativenumber
autocmd InsertEnter * :set norelativenumber number
autocmd InsertLeave * :set relativenumber

"set cursorline                  "突出显示当前行，可用Ctrl+k切换是否显示
"set cursorcolumn                "突出显示当前列，可用Ctrl+k切换是否显示
"set backup                       "备份
" 退出插入模式时自动保存
"autocmd InsertLeave * :w!<ESC>

"""""""""""""""""""""""""""""""""""""""""
" 自定义快捷键
"""""""""""""""""""""""""""""""""""""""""
" 前导符号
"""""""""""""""""""""""""""""""""""""""""
let mapleader = ','
let g:mapleader = ','

" 常用快捷键
"""""""""""""""""""""""""""""""""""""""""
" 到行首行尾
noremap H ^
noremap L $

" 到光标所在行第一个非空字符
"map 0 ^

" 从光标处复制到行尾，不包括行尾结束符
map Y y$

" t/T增加空行
nmap t o<ESC>
nmap T O<ESC>

" 恢复撤销
nnoremap U <C-r>

" 保存
map <leader>w :w!<cr>

" 保存并退出当前窗口
map <leader>q :wq!<CR>

" 保存并挂起
map <leader>z :w!<cr><C-z>

" 全选
map <C-a> ggVG

" 相对/绝对行号转换
noremap <C-n> :call NumberToggle()<cr>
cnoremap <C-n> :call NumberToggle()<cr>:

" 突出显示当前行/列开关
map <leader>k :call CusorCulLineToggle()<cr>

" 打开/关闭全部折叠
map <leader>o zR
map <leader>c zM

" 连续按jf退出插入/命令行/可视模式
inoremap jk <ESC>l
cnoremap jk <ESC>l
vnoremap jf <ESC>l

" 选中状态下 Ctrl+c 复制
vmap <C-c> "+y

" 按;直接进入命令行模式
nnoremap ; :

" 将光标所在行上移或下移一行,^[是按ctrl+v出来的,[k是按Alt+k出来的.
"nmap <S-k> :m-2<cr>
"nmap <S-j> :m+1<cr>
"vmap <S-k> :m-2<cr>
"vmap <S-j> :m+1<cr>
"nmap k :m-2<cr>
"nmap j :m+1<cr>
"vmap k :m-2<cr>
"vmap j :m+1<cr>
nmap <S-j> yyp
nmap <S-k> yyp

" 搜索查找快捷键
"""""""""""""""""""""""""""""""""""""""""
" 替换,匹配模式采用very magic
map \ :%s/\v

" 查找,匹配模式采用very magic
nnoremap / /\v
vnoremap / /\v

" 去掉查找后的高亮显示
noremap <silent><leader>/ :nohls<CR>

" 搜索的内容总是在屏幕中心显示
nnoremap <silent> n nzz
nnoremap <silent> N Nzz
nnoremap <silent> * *zz
nnoremap <silent> # #zz
nnoremap <silent> g* g*zz

" 普通模式(Nomal Mode)下快捷键
"""""""""""""""""""""""""""""""""""""""""
" 快速编辑重vimrc
nmap <silent> <leader>ev :e $MYVIMRC<CR>
nmap <silent> <leader>sv :so $MYVIMRC<CR>

" 快速滚屏
nnoremap <C-e> 2<C-e>
nnoremap <C-y> 2<C-y>

" 命令行模式(Command Line Mode)下快捷键
"""""""""""""""""""""""""""""""""""""""""
cnoremap <C-j> <t_kd>
cnoremap <C-k> <t_ku>
cnoremap <C-a> <Home>
cnoremap <C-e> <End>

" 功能键
"""""""""""""""""""""""""""""""""""""""""
" F6语法高亮快捷键
nnoremap <F6> :exec exists('syntax_on') ? 'syn off' : 'syn on'<CR>
" F7标签导航
"nnoremap <silent> <F7> :TlistToggle<CR>
nnoremap <leader>m :TlistToggle<CR>
" F8标签导航
"nmap <F8> :TagbarToggle<CR>

" 多窗口标签快捷键
"""""""""""""""""""""""""""""""""""""""""
" 在多个窗口间切换
map <C-j> <C-W>j
map <C-k> <C-W>k
map <C-h> <C-W>h
map <C-l> <C-W>l

" Disbale paste mode when leaving insert mode
map <leader>tn :tabnew<cr>
map <leader>to :tabonly<cr>
map <leader>tc :tabclose<cr>
map <leader>tm :tabmove<cr>

" Opens a new tab with the current buffer's path
" Super useful when editing files in the same directory
map <leader>te :tabedit <C-r>=expand("%:p:h")<cr>/

" 杂项
"""""""""""""""""""""""""""""""""""""""""
" Disbale paste mode when leaving insert mode
autocmd InsertLeave * set nopaste

" w!! to sudo & write a file
cmap w!! w !sudo tee >/dev/null %

" Swap implementations of ` and ' jump to markers
" By default, ' jumps to the marked line, ` jumps to the marked line and
" column, so swap them
nnoremap ' `
nnoremap ` '

" Automatically reload vimrc when it's saved
autocmd BufWritePost .vimrc so ~/.vimrc

" ; can repeat fx/tx. so do not map it
nnoremap <leader>v V`}

"""""""""""""""""""""""""""""""""""""""""
" 主题颜色显示
"""""""""""""""""""""""""""""""""""""""""
" Set extra options when running in GUI mode
if has("gui_running")
    set guifont=Monaco:h14
    set guioptions-=T
    set guioptions+=e
    set guioptions-=r
    set guioptions-=L
    set guitablabel=%M\ %t
    set showtabline=1
    set linespace=2
    set noimd
    set t_Co=256
endif

" 修改主题和颜色展示
" colorscheme solarized
set background=dark
set t_Co=256
" colorscheme molokai
" colorscheme desert

" 设置标记一列的背景颜色和数字一行颜色一致
hi! link SignColumn   LineNr
hi! link ShowMarksHLl DiffAdd
hi! link ShowMarksHLu DiffChange

" 防止错误整行标红导致看不清
highlight clear SpellBad
highlight SpellBad term=standout ctermfg=1 term=underline cterm=underline
highlight clear SpellCap
highlight SpellCap term=underline cterm=underline
highlight clear SpellRare
highlight SpellRare term=underline cterm=underline
highlight clear SpellLocal
highlight SpellLocal term=underline cterm=underline

" settings for kien/rainbow_parentheses.vim
autocmd VimEnter * RainbowParenthesesToggle
autocmd Syntax * RainbowParenthesesLoadRound
autocmd Syntax * RainbowParenthesesLoadSquare
autocmd Syntax * RainbowParenthesesLoadBraces

"""""""""""""""""""""""""""""""""""""""""
" 其它杂项配置
"""""""""""""""""""""""""""""""""""""""""
autocmd! bufwritepost _vimrc source %     "vimrc文件修改之后自动加载。 windows。
autocmd! bufwritepost .vimrc source %     "vimrc文件修改之后自动加载。 linux。
set completeopt=longest,menu              "自动补全配置,让Vim的补全菜单行为与一般IDE一致(参考VimTip1228)

" 离开插入模式后自动关闭预览窗口
autocmd InsertLeave * if pumvisible() == 0|pclose|endif

" 回车即选中当前项
inoremap <expr> <CR>       pumvisible() ? "\<C-y>" : "\<CR>"

" 增强模式中的命令行自动完成操作
set wildmenu

" Ignore compiled files
" set wildignore=*.o,*~,*.pyc,*.class

" Python 文件的一般设置，比如不要tab等
autocmd FileType python set tabstop=4 shiftwidth=4 expandtab ai
" 如果不起作用，确定.viminfo是否可写
if has("autocmd")
  au BufReadPost * if line("'\"") > 1 && line("'\"") <= line("$") | exe "normal! g'\"" | endif
endif

" 删除多余空格
func! DeleteTrailingWS()
  exe "normal mz"
  %s/\s\+$//ge
  exe "normal `z"
endfunc
autocmd BufWrite *.py :call DeleteTrailingWS()

"""""""""""""""""""""""""""""""""""""""""
" 插件管理配置开始
"""""""""""""""""""""""""""""""""""""""""
" package dependent:  ctags
" python dependent:  pep8, pyflake

filetype off
set rtp+=~/.vim/bundle/vundle/
call vundle#rc()
Bundle 'gmarik/vundle'

" vim plugin bundle control, command model
" :BundleInstall     install
" :BundleInstall!    update
" :BundleClean       remove plugin not in list

" 插件：目录导航等
"""""""""""""""""""""""""""""""""""""""""
Bundle 'scrooloose/nerdtree'
map <leader>n :NERDTreeToggle<CR>
let NERDTreeHighlightCursorline=1
let NERDTreeIgnore=[ '\.pyc$', '\.pyo$', '\.obj$', '\.o$', '\.so$', '\.egg$', '^\.git$', '^\.svn$', '^\.hg$' ]
let g:netrw_home='~/'

" close vim if the only window left open is a NERDTree
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTreeType") && b:NERDTreeType == "primary") | q | end

" for minibufferexpl
Bundle 'fholgado/minibufexpl.vim'
let g:miniBufExplMapWindowNavVim = 1
let g:miniBufExplMapWindowNavArrows = 1
let g:miniBufExplMapCTabSwitchBufs = 1
let g:miniBufExplModSelTarget = 1
let g:miniBufExplForceSyntaxEnable = 1
let g:miniBufExplorerMoreThanOne=2
let g:miniBufExplCycleArround=1

" 默认方向键左右可以切换buffer
nnoremap <TAB> :MBEbn<CR>
noremap <leader>bn :MBEbn<CR>
noremap <leader>bp :MBEbp<CR>
noremap <leader>bd :MBEbd<CR>

" 插件：标签导航等
"""""""""""""""""""""""""""""""""""""""""
Bundle 'majutsushi/tagbar'
let g:tagbar_autofocus = 1
Bundle 'vim-scripts/taglist.vim'
set tags=tags;/
let Tlist_Ctags_Cmd="/usr/bin/ctags"
let Tlist_Auto_Highlight_Tag = 1
let Tlist_Auto_Open = 0
let Tlist_Auto_Update = 1
let Tlist_Close_On_Select = 0
let Tlist_Compact_Format = 0
let Tlist_Display_Prototype = 0
let Tlist_Display_Tag_Scope = 1
let Tlist_Enable_Fold_Column = 0
let Tlist_Exit_OnlyWindow = 1
let Tlist_File_Fold_Auto_Close = 0
let Tlist_GainFocus_On_ToggleOpen = 1
let Tlist_Hightlight_Tag_On_BufEnter = 1
let Tlist_Inc_Winwidth = 0
let Tlist_Max_Submenu_Items = 1
let Tlist_Max_Tag_Length = 30
let Tlist_Process_File_Always = 0
let Tlist_Show_Menu = 0
let Tlist_Show_One_File = 1
let Tlist_Sort_Type = "order"
let Tlist_Use_Horiz_Window = 0
let Tlist_Use_Right_Window = 0
let Tlist_WinWidth = 25

" 插件：文件搜索
"""""""""""""""""""""""""""""""""""""""""
Bundle 'kien/ctrlp.vim'
let g:ctrlp_map = '<leader>p'
let g:ctrlp_cmd = 'CtrlP'
map <leader>f :CtrlPMRU<CR>

" set wildignore+=*/tmp/*,*.so,*.swp,*.zip     " MacOSX/Linux"
let g:ctrlp_custom_ignore = {
    \ 'dir':  '\v[\/]\.(git|hg|svn|rvm)$',
    \ 'file': '\v\.(exe|so|dll|zip|tar|tar.gz)$',
    \ }

" \ 'link': 'SOME_BAD_SYMBOLIC_LINKS',
let g:ctrlp_working_path_mode=0
let g:ctrlp_match_window_bottom=1
let g:ctrlp_max_height=15
let g:ctrlp_match_window_reversed=0
let g:ctrlp_mruf_max=500
let g:ctrlp_follow_symlinks=1

" 插件：状态栏美观
"""""""""""""""""""""""""""""""""""""""""
Bundle 'Lokaltog/vim-powerline'
" if want to use fancy,need to add font patch -> git clone git://gist.github.com/1630581.git ~/.fonts/ttf-dejavu-powerline
"let g:Powerline_symbols = 'fancy'
"let g:Powerline_symbols = 'unicode'

" 插件：括号显示增强
"""""""""""""""""""""""""""""""""""""""""
Bundle 'kien/rainbow_parentheses.vim'
let g:rbpt_colorpairs = [
    \ ['brown',       'RoyalBlue3'],
    \ ['Darkblue',    'SeaGreen3'],
    \ ['darkgray',    'DarkOrchid3'],
    \ ['darkgreen',   'firebrick3'],
    \ ['darkcyan',    'RoyalBlue3'],
    \ ['darkred',     'SeaGreen3'],
    \ ['darkmagenta', 'DarkOrchid3'],
    \ ['brown',       'firebrick3'],
    \ ['gray',        'RoyalBlue3'],
    \ ['black',       'SeaGreen3'],
    \ ['darkmagenta', 'DarkOrchid3'],
    \ ['Darkblue',    'firebrick3'],
    \ ['darkgreen',   'RoyalBlue3'],
    \ ['darkcyan',    'SeaGreen3'],
    \ ['darkred',     'DarkOrchid3'],
    \ ['red',         'firebrick3'],
    \ ]
let g:rbpt_max = 40
let g:rbpt_loadcmd_toggle = 0

" 插件：将每行无效的空格标红（,空格按键去掉末尾空格）
"""""""""""""""""""""""""""""""""""""""""
Bundle 'bronson/vim-trailing-whitespace'
map <leader><space> :FixWhitespace<cr>

" 插件：主题solarized
Bundle 'altercation/vim-colors-solarized'
" let g:solarized_termcolors=256
let g:solarized_termtrans=1
let g:solarized_contrast="normal"
let g:solarized_visibility="normal"

" 插件：主题molokai
Bundle 'tomasr/molokai'
" let g:molokai_original = 1

" 插件：快速移动
"""""""""""""""""""""""""""""""""""""""""
" 更高效的移动 ,, + w/fx
Bundle 'Lokaltog/vim-easymotion'
Bundle 'vim-scripts/matchit.zip'

" 插件：迄今为止用到的最好的自动VIM自动补全插件
"""""""""""""""""""""""""""""""""""""""""
" Bundle 'Valloric/YouCompleteMe'
" youcompleteme  默认tab  s-tab 和自动补全冲突
" let g:ycm_key_list_select_completion=['<C-n>']
" let g:ycm_key_list_select_completion = ['<Down>']
" let g:ycm_key_list_previous_completion=['<C-p>']
" let g:ycm_key_list_previous_completion = ['<Up>']
" 在注释输入中也能补全
" let g:ycm_complete_in_comments = 1
" 在字符串输入中也能补全
" let g:ycm_complete_in_strings = 1
" 注释和字符串中的文字也会被收入补全
" let g:ycm_collect_identifiers_from_comments_and_strings = 0

" 插件：快速插入代码片段
"""""""""""""""""""""""""""""""""""""""""
Bundle 'SirVer/ultisnips'
let g:UltiSnipsExpandTrigger = "<tab>"
let g:UltiSnipsJumpForwardTrigger = "<tab>"
" 定义存放代码片段的文件夹 .vim/snippets下，使用自定义和默认的，将会的到全局，有冲突的会提示
let g:UltiSnipsSnippetDirectories=["snippets", "bundle/ultisnips/UltiSnips"]

" 插件：快速加/减注释(选中后,按,cc加上注释,按,cu解开注释)
"""""""""""""""""""""""""""""""""""""""""
Bundle 'scrooloose/nerdcommenter'

" 插件：用双引号/单引号包裹字符串
"""""""""""""""""""""""""""""""""""""""""
" cs" '
" Hello world!" -> 'Hello world!'
" ds"
" " Hello world!" -> Hello world!
" ysiw"
" Hello -> " Hello"
Bundle 'tpope/vim-surround'
Bundle 'tpope/vim-repeat'

" 插件：自动补全单引号，双引号等
"""""""""""""""""""""""""""""""""""""""""
Bundle 'Raimondi/delimitMate'
" for python docstring " ,优化输入
autocmd FileType python let b:delimitMate_nesting_quotes = ['"']

" 自动补全html/xml标签
Bundle 'docunext/closetag.vim'
let g:closetag_html_style=1

" 插件：代码格式化
"""""""""""""""""""""""""""""""""""""""""
Bundle 'godlygeek/tabular'
nmap <Leader>a= :Tabularize /=<CR>
vmap <Leader>a= :Tabularize /=<CR>
nmap <Leader>a: :Tabularize /:\zs<CR>
vmap <Leader>a: :Tabularize /:\zs<CR>

" for visual selection
Bundle 'terryma/vim-expand-region'
map = <Plug>(expand_region_expand)
map - <Plug>(expand_region_shrink)

" 插件：多光标批量操作
"""""""""""""""""""""""""""""""""""""""""
" Bundle 'terryma/vim-multiple-cursors'
" let g:multi_cursor_use_default_mapping=0
" Default mapping
" let g:multi_cursor_next_key='<C-m>'
" let g:multi_cursor_prev_key='<C-p>'
" let g:multi_cursor_skip_key='<C-x>'
" let g:multi_cursor_quit_key='<ESC>'

" 插件：语法检查
"""""""""""""""""""""""""""""""""""""""""
" 编辑时自动语法检查标红, vim-flake8目前还不支持,所以多装一个
" 使用pyflakes,速度比pylint快
Bundle 'scrooloose/syntastic'
let g:syntastic_error_symbol='>>'
let g:syntastic_warning_symbol='>'
let g:syntastic_check_on_open=1
let g:syntastic_enable_highlighting = 0
let g:syntastic_python_checkers=['pyflakes']
highlight SyntasticErrorSign guifg=white guibg=black

" python fly check, 弥补syntastic只能打开和保存才检查语法的不足
Bundle 'kevinw/pyflakes-vim'
let g:pyflakes_use_quickfix = 0

" 插件：具体语言语法高亮
"""""""""""""""""""""""""""""""""""""""""
" for python.vim syntax highlight
Bundle 'hdima/python-syntax'
let python_highlight_all = 1

" for golang
Bundle 'jnwhiteh/vim-golang'
Bundle 'Blackrush/vim-gocode'

" for markdown
Bundle 'plasticboy/vim-markdown'
let g:vim_markdown_folding_disabled=1

" for javascript
Bundle "pangloss/vim-javascript"
let g:html_indent_inctags = "html,body,head,tbody"
let g:html_indent_script1 = "inc"
let g:html_indent_style1 = "inc"

" for jquery
Bundle 'nono/jquery.vim'

" for jinja2 highlight
Bundle 'Glench/Vim-Jinja2-Syntax'

" for nginx conf file highlight.   need to confirm it works
Bundle 'thiderman/nginx-vim-syntax'

" 插件：杂项
"""""""""""""""""""""""""""""""""""""""""
" task list
Bundle 'vim-scripts/TaskList.vim'
map <leader>td <Plug>TaskList

" for git 尚未用起来
Bundle 'tpope/vim-fugitive'

" 可以查看/回到某个历史状态
Bundle 'sjl/gundo.vim'
nnoremap <leader>h :GundoToggle<CR>

" end turn on
filetype plugin indent on
"""""""""""""""""""""""""""""""""""""""""
" 插件管理配置结束
"""""""""""""""""""""""""""""""""""""""""

"""""""""""""""""""""""""""""""""""""""""
" 自定义的函数
"""""""""""""""""""""""""""""""""""""""""
" 相对绝对行号转换
function! NumberToggle()
  if(&relativenumber == 1)
    set norelativenumber number
  else
    set relativenumber
  endif
endfunc

" 突出显示当前行列转换
function! CusorCulLineToggle()
  if(&cursorcolumn == 1)
    set nocursorcolumn
    set nocursorline
  else
    set cursorcolumn
    set cursorline
  endif
endfunc

function! GoToFileEnd()
    normal G
endfunc
""定义函数SetTitle，自动插入文件头
autocmd BufNewFile *.py,*.cpp,*.c,*.sh,*.java exec ":call SetTitle()"
function! SetTitle()
    "如果文件类型为.sh文件
    if &filetype == 'sh'
        call setline(1,"\#!/bin/bash")
        call append(line("."), "\#############################################")
        call append(line(".")+1, "\# Copyright(c) Tencent, all rights reserved")
        call append(line(".")+2, "\# @file        : ".expand("%"))
        call append(line(".")+3, "\# @author      : lucasysfeng")
        call append(line(".")+4, "\# @revision    : ".strftime("%Y-%m-%-d %H:%M:%S"))
        call append(line(".")+5, "\# @brief       :")
        call append(line(".")+6, "\#############################################")
    elseif &filetype == 'cpp' || &filetype == 'c'
        call setline(1,"/****************************************************")
        call append(line("."), "\# Copyright(c) Tencent, all rights reserved")
        call append(line(".")+1, "\# @file        : ".expand("%"))
        call append(line(".")+2, "\# @author      : lucasysfeng")
        call append(line(".")+3, "\# @revision    : ".strftime("%Y-%m-%-d %H:%M:%S"))
        call append(line(".")+4, "\# @brief       :")
        call append(line(".")+5, "****************************************************/")
        call append(line(".")+6, "")
        call append(line(".")+7, "#include <iostream>")
        call append(line(".")+8, "#include <string>")
        call append(line(".")+9, "")
        call append(line(".")+10, "using namespace std;")
        call append(line(".")+11, "")
        call append(line(".")+12, "int main()")
        call append(line(".")+13, "{")
        call append(line(".")+14, "")
        call append(line(".")+15, "    return 0;")
        call append(line(".")+16,"}")
    elseif &filetype == 'python'
        call setline(1,"\#!/usr/bin/env python")
        call append(line("."), "\# -*- coding: utf-8 -*-")
        call append(line(".")+1, "\#############################################")
        call append(line(".")+2, "\# Copyright(c) , all rights reserved")
        call append(line(".")+3, "\# @file        : ".expand("%"))
        call append(line(".")+4, "\# @author      : lucasysfeng")
        call append(line(".")+5, "\# @revision    : ".strftime("%Y-%m-%-d %H:%M:%S"))
        call append(line(".")+6, "\# @brief       :")
        call append(line(".")+7, "\#############################################")
    endif
    "新建文件后，自动定位到文件末尾
    "autocmd BufNewFile * exec ":call GoToFileEnd()"
    call GoToFileEnd()
endfunc





" Doxygen注释快捷键
nnoremap <leader>g :Dox<CR>

" DoxygenToolkit.vim
" Brief: Usefull tools for Doxygen (comment, author, license).
" Version: 0.2.13
" Date: 2010/10/16
" Author: Mathias Lorente
"
" TODO: add automatically (option controlled) in/in out flags to function
"       parameters
" TODO: (Python) Check default paramareters defined as list/dictionnary/tuple
"
" Note: Correct insertion position and 'xxx_post' parameters.
"      - Insert position is correct when g:DoxygenToolkit_compactOneLineDoc = "yes"
"        and let g:DoxygenToolkit_commentType = "C++" are set.
"      - When you define:
"              g:DoxygenToolkit_briefTag_pre = "@brief "
"              g:DoxygenToolkit_briefTag_post = "<++>"
"              g:DoxygenToolkit_briefTag_funcName = "yes"
"        Documentation generated with these parameters is something like:
"           /// @brief foo <++>
"        You can configure similarly parameters to get something like:
"           /// @brief foo <++>
"           /// @param bar <++>
"           /// @param baz <++>
"
" Note: Position the cursor at the right position for one line documentation.
"
" Note: Remove trailing blank characters where they are not needed.
"
" Note: 'extern' keyword added in list of values to ignore for return type.
"
" Note: Correct bugs related to templates and add support for throw statement
"       (many thanks to Dennis Lubert):
"   - Template parameter of different type from class and typename are
"     recognized.
"   - Indentation mistake while detecting template.
"   - New option are available: g:DoxygenToolkit_throwTag_pre and
"     g:DoxygenToolkit_throwTag_post
"
" Note: Add support for documentation of template parameters.
"       Thanks to Dennis (plasmahh) and its suggestions.
"   - New option are available: g:DoxygenToolkit_templateParamTag_pre
"     and g:DoxygenToolkit_templateParamTag_post
"
" Note: Solve almost all compatibility problem with c/c++ IDE
"
" Note: Bug correction and improve compatibility with c/c++ IDE
"   - Documentation of function with struct parameters are now allowed.
"   - Comments are written in two steps to avoid conflicts with c/c++ IDE.
"
" Note: Bug correction (thanks to Jhon Do)
"   - DoxygenToolkit_briefTag_funcName and other xxx_xxName parameters
"     should work properly now.
"
" Note: Bug correction (thanks to Anders Bo Rasmussen)
"   - C++: now functions like  void foo(type &bar); are correctly documented.
"          The parameter's name is bar (and no more &bar).
"
" Note: Added @version tag into the DocBlock generated by DoxygenAuthorFunc()
"       (thanks to Dave Walter).
"       The version string can be defines into your .vimrc file with
"       g:DoxygenToolkit_versionString or it will be asked the first time the
"       function is called (same behavior as @author tag). Example:
"                 /// \file foo.cpp
"                 /// \brief
"                 /// \author Dave Walter
"                 /// \version 1.0
"                 /// \date 2009-03-26
"
" Note: Comments are now allowed in function declaration. Example:
"   - C/C++:   void func( int foo, // first param
"                         int bar  /* second param */ );
"
"   - Python:  def func( foo,  # first param
"                        bar ) # second param
"
" Note: Bug correction (many thanks to Alexey Radkov)
"   - C/C++: following function/method are now correctly documented:
"      - operator(),
"      - constructor with initialization parameter(s),
"      - pure virtual method,
"      - const method.
"   - Python:
"      - Single line function are now correctly documented.
"
" Note: The main function has been rewritten (I hope it is cleaner).
"   - There is now support for function pointer as parameter (C/C++).
"   - You can configure the script to get one line documentation (for
"     attribute instance for example, you need to set
"     g:DoxygenToolkit_compactOneLineDoc to "yes").
"
"   - NEW: Support Python scripts:
"      - Function/method are not scanned, so by default they are considered
"        as if they always return something (modify this behavior by defining
"        g:DoxygenToolkit_python_autoFunctionReturn to "no")
"      - self parameter is automatically ignored when scanning function
"        parameters (you can change this behavior by defining
"        g:DoxygenToolkit_python_autoRemoveSelfParam to "no")
"
" Note: Number of lines scanned is now configurable. Default value is still 10
"     lines. (Thanks to Spencer Collyer for this improvement).
"
" Note: Bug correction : function that returns null pointer are correctly
"     documented (Thanks to Ronald WAHL for his report and patch).
"
" Note: Remove header and footer from doxygen documentation
"   - Generated documentation with block header/footer activated (see
"     parameters g:DoxygenToolkit_blockHeader and
"     g:DoxygenToolkit_blockFooter) do not integrate header and footer
"     anymore.
"     Thanks to Justin RANDALL for this.
"     Now comments are as following:
"     /* --- My Header --- */             // --- My Header ---
"     /**                                 /// @brief ...
"      *  @brief ...                or    // --- My Footer ---
"      */
"     /* -- My Footer --- */
"
" Note: Changes to customize cinoptions
"   - New option available for cinoptions : g:DoxygenToolkit_cinoptions
"     (default value is still c1C1)
"     Thanks to Arnaud GODET for this. Now comment can have the following
"     look:
"     /**                      /**
"     *       and not only     *
"     */                       */
" Note: Changes for linux kernel comment style
"   - New option are available for brief tag and parameter tag ! Now there is
"     a pre and a post tag for each of these tag.
"   - You can define 'let g:DoxygenToolkit_briefTag_funcName = "yes"' to add
"     the name of commented function between pre-brief tag and post-brief tag.
"   - With these new features you can get something like:
"     /**
"      * @brief MyFunction -
"      *
"      * @param foo:
"      * @param bar:
"      */
" Note: Changes suggested by Soh Kok Hong:
"   - Fixed indentation in comments
"     ( no more /**               /**
"                 *       but      *
"                 */               */     )
" Note: Changes made by Jason Mills:
"   - Fixed \n bug which resulted in comments being screwed up
"   - Added use of doxygen /// comments.
" Note: Changes made by Mathias Lorente on 05/25/04
"   - Fixed filename bug when including doxygen author comment whereas file
"     has not been open directly on commamd line.
"   - Now /// or /** doxygen comments are correctly integrated (except for
"     license).
" Note: Changes made by Mathias Lorente on 08/02/04
"   - Now include only filename in author comment (no more folder...)
"   - Fixed errors with function with no indentation.
"
"
" Currently five purposes have been defined :
"
" Generates a doxygen license comment.  The tag text is configurable.
"
" Generates a doxygen author skeleton.  The tag text is configurable.
"
" Generates a doxygen comment skeleton for a C, C++ or Python function or class,
" including @brief, @param (for each named argument), and @return.  The tag
" text as well as a comment block header and footer are configurable.
" (Consequently, you can have \brief, etc. if you wish, with little effort.)
"
" Ignore code fragment placed in a block defined by #ifdef ... #endif (C/C++).  The
" block name must be given to the function.  All of the corresponding blocks
" in all the file will be treated and placed in a new block DOX_SKIP_BLOCK (or
" any other name that you have configured).  Then you have to update
" PREDEFINED value in your doxygen configuration file with correct block name.
" You also have to set ENABLE_PREPROCESSING to YES.
"
" Generate a doxygen group (begining and ending). The tag text is
" configurable.
"
" Use:
" - Type of comments (C/C++: /// or /** ... */, Python: ## and # ) :
"   In vim, default C++ comments are : /** ... */. But if you prefer to use ///
"   Doxygen comments just add 'let g:DoxygenToolkit_commentType = "C++"'
"   (without quotes) in your .vimrc file
"
" - License :
"   In vim, place the cursor on the line that will follow doxygen license
"   comment.  Then, execute the command :DoxLic.  This will generate license
"   comment and leave the cursor on the line just after.
"
" - Author :
"   In vim, place the cursor on the line that will follow doxygen author
"   comment.  Then, execute the command :DoxAuthor.  This will generate the
"   skeleton and leave the cursor just after @author tag if no variable
"   define it, or just after the skeleton.
"
" - Function / class comment :
"   In vim, place the cursor on the line of the function header (or returned
"   value of the function) or the class.  Then execute the command :Dox.  This
"   will generate the skeleton and leave the cursor after the @brief tag.
"
" - Ignore code fragment :
"   In vim, if you want to ignore all code fragment placed in a block such as :
"     #ifdef DEBUG
"     ...
"     #endif
"   You only have to execute the command :DoxUndoc(DEBUG) !
"
" - Group :
"   In vim, execute the command :DoxBlock to insert a doxygen block on the
"   following line.
"
" Limitations:
" - Assumes that the function name (and the following opening parenthesis) is
"   at least on the third line after current cursor position.
" - Not able to update a comment block after it's been written.
" - Blocks delimiters (header and footer) are only included for function
"   comment.
" - Assumes that cindent is used.
" - Comments in function parameters (such as void foo(int bar /* ... */, baz))
"   are not yet supported.
"
"
" Example:
" Given:
" int
"   foo(char mychar,
"       int myint,
"       double* myarray,
"       int mask = DEFAULT)
" { //...
" }
"
" Issuing the :Dox command with the cursor on the function declaration would
" generate
"
" /**
"  * @brief
"  *
"  * @param mychar
"  * @param myint
"  * @param myarray
"  * @param mask
"  *
"  * @return
"  */
"
"
" To customize the output of the script, see the g:DoxygenToolkit_*
" variables in the script's source.  These variables can be set in your
" .vimrc.
"
" For example, my .vimrc contains:
" let g:DoxygenToolkit_briefTag_pre="@Synopsis  "
" let g:DoxygenToolkit_paramTag_pre="@Param "
" let g:DoxygenToolkit_returnTag="@Returns   "
" let g:DoxygenToolkit_blockHeader="--------------------------------------------------------------------------"
" let g:DoxygenToolkit_blockFooter="----------------------------------------------------------------------------"
" let g:DoxygenToolkit_authorName="Mathias Lorente"
" let g:DoxygenToolkit_licenseTag="My own license"   <-- Does not end with
" "\<enter>"


" Verify if already loaded
"if exists("loaded_DoxygenToolkit")
" echo 'DoxygenToolkit Already Loaded.'
" finish
"endif
let loaded_DoxygenToolkit = 1
"echo 'Loading DoxygenToolkit...'
let s:licenseTag = "Copyright (C) \<enter>\<enter>"
let s:licenseTag = s:licenseTag . "This program is free software; you can redistribute it and/or\<enter>"
let s:licenseTag = s:licenseTag . "modify it under the terms of the GNU General Public License\<enter>"
let s:licenseTag = s:licenseTag . "as published by the Free Software Foundation; either version 2\<enter>"
let s:licenseTag = s:licenseTag . "of the License, or (at your option) any later version.\<enter>\<enter>"
let s:licenseTag = s:licenseTag . "This program is distributed in the hope that it will be useful,\<enter>"
let s:licenseTag = s:licenseTag . "but WITHOUT ANY WARRANTY; without even the implied warranty of\<enter>"
let s:licenseTag = s:licenseTag . "MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the\<enter>"
let s:licenseTag = s:licenseTag . "GNU General Public License for more details.\<enter>\<enter>"
let s:licenseTag = s:licenseTag . "You should have received a copy of the GNU General Public License\<enter>"
let s:licenseTag = s:licenseTag . "along with this program; if not, write to the Free Software\<enter>"
let s:licenseTag = s:licenseTag . "Foundation, Inc., 59 Temple Place - Suite 330, Boston, MA  02111-1307, USA.\<enter>"

" Common standard constants
if !exists("g:DoxygenToolkit_briefTag_pre")
  let g:DoxygenToolkit_briefTag_pre = "@brief "
endif
if !exists("g:DoxygenToolkit_briefTag_post")
  let g:DoxygenToolkit_briefTag_post = ""
endif
if !exists("g:DoxygenToolkit_templateParamTag_pre")
  let g:DoxygenToolkit_templateParamTag_pre = "@tparam "
endif
if !exists("g:DoxygenToolkit_templateParamTag_post")
  let g:DoxygenToolkit_templateParamTag_post = ""
endif
if !exists("g:DoxygenToolkit_paramTag_pre")
  let g:DoxygenToolkit_paramTag_pre = "@param "
endif
if !exists("g:DoxygenToolkit_paramTag_post")
  let g:DoxygenToolkit_paramTag_post = ""
endif
if !exists("g:DoxygenToolkit_returnTag")
  let g:DoxygenToolkit_returnTag = "@return"
endif
if !exists("g:DoxygenToolkit_throwTag_pre")
  let g:DoxygenToolkit_throwTag_pre = "@throw " " @exception is also valid
endif
if !exists("g:DoxygenToolkit_throwTag_post")
  let g:DoxygenToolkit_throwTag_post = ""
endif
if !exists("g:DoxygenToolkit_blockHeader")
  let g:DoxygenToolkit_blockHeader = ""
endif
if !exists("g:DoxygenToolkit_blockFooter")
  let g:DoxygenToolkit_blockFooter = ""
endif
if !exists("g:DoxygenToolkit_licenseTag")
  let g:DoxygenToolkit_licenseTag = s:licenseTag
endif
if !exists("g:DoxygenToolkit_fileTag")
  let g:DoxygenToolkit_fileTag = "@file "
endif
if !exists("g:DoxygenToolkit_authorTag")
  let g:DoxygenToolkit_authorTag = "@author "
endif
if !exists("g:DoxygenToolkit_dateTag")
  let g:DoxygenToolkit_dateTag = "@date "
endif
if !exists("g:DoxygenToolkit_versionTag")
  let g:DoxygenToolkit_versionTag = "@version "
endif
if !exists("g:DoxygenToolkit_undocTag")
  let g:DoxygenToolkit_undocTag = "DOX_SKIP_BLOCK"
endif
if !exists("g:DoxygenToolkit_blockTag")
  let g:DoxygenToolkit_blockTag = "@name "
endif
if !exists("g:DoxygenToolkit_classTag")
  let g:DoxygenToolkit_classTag = "@class "
endif

if !exists("g:DoxygenToolkit_cinoptions")
    let g:DoxygenToolkit_cinoptions = "c1C1"
endif
if !exists("g:DoxygenToolkit_startCommentTag ")
  let g:DoxygenToolkit_startCommentTag = "/** "
  let g:DoxygenToolkit_startCommentBlock = "/* "
endif
if !exists("g:DoxygenToolkit_interCommentTag ")
  let g:DoxygenToolkit_interCommentTag = "* "
endif
if !exists("g:DoxygenToolkit_interCommentBlock ")
  let g:DoxygenToolkit_interCommentBlock = "* "
endif
if !exists("g:DoxygenToolkit_endCommentTag ")
  let g:DoxygenToolkit_endCommentTag = "*/"
  let g:DoxygenToolkit_endCommentBlock = "*/"
endif
if exists("g:DoxygenToolkit_commentType")
  if ( g:DoxygenToolkit_commentType == "C++" )
    let g:DoxygenToolkit_startCommentTag = "/// "
    let g:DoxygenToolkit_interCommentTag = "/// "
    let g:DoxygenToolkit_endCommentTag = ""
    let g:DoxygenToolkit_startCommentBlock = "// "
    let g:DoxygenToolkit_interCommentBlock = "// "
    let g:DoxygenToolkit_endCommentBlock = ""
  else
    let g:DoxygenToolkit_commentType = "C"
  endif
else
  let g:DoxygenToolkit_commentType = "C"
endif

" Compact documentation
" /**
"  * \brief foo      --->    /** \brief foo */
"  */
if !exists("g:DoxygenToolkit_compactOneLineDoc")
  let g:DoxygenToolkit_compactOneLineDoc = "no"
endif
" /**
"  * \brief foo             /**
"  *                         * \brief foo
"  * \param bar      --->    * \param bar
"  *                         * \return
"  * \return                 */
"  */
if !exists("g:DoxygenToolkit_compactDoc")
  let g:DoxygenToolkit_compactDoc = "no"
endif

" Necessary '\<' and '\>' will be added to each item of the list.
let s:ignoreForReturn = ['template', 'explicit', 'inline', 'static', 'virtual', 'void\([[:blank:]]*\*\)\@!', 'const', 'volatile', 'struct', 'extern']
if !exists("g:DoxygenToolkit_ignoreForReturn")
  let g:DoxygenToolkit_ignoreForReturn = s:ignoreForReturn[:]
else
  let g:DoxygenToolkit_ignoreForReturn += s:ignoreForReturn
endif
unlet s:ignoreForReturn

" Maximum number of lines to check for function parameters
if !exists("g:DoxygenToolkit_maxFunctionProtoLines")
  let g:DoxygenToolkit_maxFunctionProtoLines = 10
endif

" Add name of function/class/struct... after pre brief tag if you want
if !exists("g:DoxygenToolkit_briefTag_className")
  let g:DoxygenToolkit_briefTag_className = "no"
endif
if !exists("g:DoxygenToolkit_briefTag_structName")
  let g:DoxygenToolkit_briefTag_structName = "no"
endif
if !exists("g:DoxygenToolkit_briefTag_enumName")
  let g:DoxygenToolkit_briefTag_enumName = "no"
endif
if !exists("g:DoxygenToolkit_briefTag_namespaceName")
  let g:DoxygenToolkit_briefTag_namespaceName = "no"
endif
if !exists("g:DoxygenToolkit_briefTag_funcName")
  let g:DoxygenToolkit_briefTag_funcName = "no"
endif

" Keep empty line (if any) between comment and function/class/...
if !exists("g:DoxygenToolkit_keepEmptyLineAfterComment")
  let g:DoxygenToolkit_keepEmptyLineAfterComment = "no"
endif

" PYTHON specific
"""""""""""""""""
" Remove automatically self parameter from function to avoid its documantation
if !exists("g:DoxygenToolkit_python_autoRemoveSelfParam")
  let g:DoxygenToolkit_python_autoRemoveSelfParam = "yes"
endif
" Consider functions as if they always return something (default: yes)
if !exists("g:DoxygenToolkit_python_autoFunctionReturn")
  let g:DoxygenToolkit_python_autoFunctionReturn = "yes"
endif


""""""""""""""""""""""""""
" Doxygen license comment
""""""""""""""""""""""""""
function! <SID>DoxygenLicenseFunc()
  call s:InitializeParameters()

  " Test authorName variable
  if !exists("g:DoxygenToolkit_authorName")
    let g:DoxygenToolkit_authorName = input("Enter name of the author (generally yours...) : ")
  endif
  mark d
  let l:date = strftime("%Y")
  exec "normal O".strpart( s:startCommentBlock, 0, 1 )
  exec "normal A".strpart( s:startCommentBlock, 1 ).substitute( g:DoxygenToolkit_licenseTag, "\<enter>", "\<enter>".s:interCommentBlock, "g" )
  if( s:endCommentBlock != "" )
    exec "normal o".s:endCommentBlock
  endif
  if( g:DoxygenToolkit_licenseTag == s:licenseTag )
    exec "normal %jA".l:date." - ".g:DoxygenToolkit_authorName
  endif
  exec "normal `d"

  call s:RestoreParameters()
endfunction


""""""""""""""""""""""""""
" Doxygen author comment
""""""""""""""""""""""""""
function! <SID>DoxygenAuthorFunc()
  call s:InitializeParameters()

  " Test authorName variable
  if !exists("g:DoxygenToolkit_authorName")
    let g:DoxygenToolkit_authorName = input("Enter name of the author (generally yours...) : ")
  endif

  " Test versionString variable
  if !exists("g:DoxygenToolkit_versionString")
    let g:DoxygenToolkit_versionString = input("Enter version string : ")
  endif

  " Get file name
  let l:fileName = expand('%:t')

  " Begin to write skeleton
  let l:insertionMode = s:StartDocumentationBlock()
  exec "normal ".l:insertionMode.s:interCommentTag.g:DoxygenToolkit_fileTag.l:fileName
  exec "normal o".s:interCommentTag.g:DoxygenToolkit_briefTag_pre
  mark d
  exec "normal o".s:interCommentTag.g:DoxygenToolkit_authorTag.g:DoxygenToolkit_authorName
  exec "normal o".s:interCommentTag.g:DoxygenToolkit_versionTag.g:DoxygenToolkit_versionString
  let l:date = strftime("%Y-%m-%d")
  exec "normal o".s:interCommentTag.g:DoxygenToolkit_dateTag.l:date
  if ( g:DoxygenToolkit_endCommentTag != "" )
    exec "normal o".s:endCommentTag
  endif

  " Move the cursor to the rigth position
  exec "normal `d"

  call s:RestoreParameters()
  startinsert!
endfunction


""""""""""""""""""""""""""
" Doxygen undocument function
" C/C++ only!
""""""""""""""""""""""""""
function! <SID>DoxygenUndocumentFunc(blockTag)
  call s:InitializeParameters()
  let l:search = "#ifdef " . a:blockTag
  " Save cursor position and go to the begining of the file
  mark d
  exec "normal gg"

  while ( search(l:search, 'W') != 0 )
    exec "normal O#ifndef " . g:DoxygenToolkit_undocTag
    exec "normal j^%"
    if ( g:DoxygenToolkit_endCommentTag == "" )
      exec "normal o#endif // " . g:DoxygenToolkit_undocTag
    else
      exec "normal o#endif /* " . g:DoxygenToolkit_undocTag . " */"
    endif
  endwhile

  exec "normal `d"
  call s:RestoreParameters()
endfunction



""""""""""""""""""""""""""
" DoxygenBlockFunc
""""""""""""""""""""""""""
function! <SID>DoxygenBlockFunc()
  call s:InitializeParameters()

  let l:insertionMode = s:StartDocumentationBlock()
  exec "normal ".l:insertionMode.s:interCommentTag.g:DoxygenToolkit_blockTag
  mark d
  exec "normal o".s:interCommentTag."@{ ".s:endCommentTag
  exec "normal o".strpart( s:startCommentTag, 0, 1 )
  exec "normal A".strpart( s:startCommentTag, 1 )." @} ".s:endCommentTag
  exec "normal `d"

  call s:RestoreParameters()
  startinsert!
endfunction


"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Main comment function for class, attribute, function...
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! <SID>DoxygenCommentFunc()

  " Initialize default templates.
  " Assure compatibility with Python for classes (cf. endDocPattern).
  let l:emptyLinePattern = '^[[:blank:]]*$'
  let l:someNamePattern  = '[_[:alpha:]][_[:alnum:]]*'

  if( s:CheckFileType() == "cpp" )
    let l:someNameWithNamespacePattern  = l:someNamePattern.'\%(::'.l:someNamePattern.'\)*'
    let l:endDocPattern    = ';\|{\|\%([^:]\zs:\ze\%([^:]\|$\)\)'
    let l:commentPattern   = '\%(/*\)\|\%(//\)\'
    let l:templateParameterPattern = "<[^<>]*>"
    let l:throwPattern = '.*\<throw\>[[:blank:]]*(\([^()]*\)).*' "available only for 'cpp' type

    let l:classPattern     = '\<class\>[[:blank:]]\+\zs'.l:someNameWithNamespacePattern.'\ze.*\%('.l:endDocPattern.'\)'
    let l:structPattern    = '\<struct\>[[:blank:]]\+\zs'.l:someNameWithNamespacePattern.'\ze[^(),]*\%('.l:endDocPattern.'\)'
    let l:enumPattern      = '\<enum\>\%(\%([[:blank:]]\+\zs'.l:someNamePattern.'\ze[[:blank:]]*\)\|\%(\zs\ze[[:blank:]]*\)\)\%('.l:endDocPattern.'\)'
    let l:namespacePattern = '\<namespace\>[[:blank:]]\+\zs'.l:someNamePattern.'\ze[[:blank:]]*\%('.l:endDocPattern.'\)'

    let l:types = { "class": l:classPattern, "struct": l:structPattern, "enum": l:enumPattern, "namespace": l:namespacePattern }
  else
    let l:commentPattern   = '#\|^[[:blank:]]*"""'

    let l:classPattern     = '\<class\>[[:blank:]]\+\zs'.l:someNamePattern.'\ze.*:'
    let l:functionPattern  = '\<def\>[[:blank:]]\+\zs'.l:someNamePattern.'\ze.*:'

    let l:endDocPattern    = '\%(\<class\>\|\<def\>[^:]*\)\@<!$'

    let l:types = { "class": l:classPattern, "function": l:functionPattern }
  endif

  let l:lineBuffer       = getline( line( "." ) )
  let l:count            = 1
  let l:endDocFound      = 0

  let l:doc = { "type": "", "name": "None", "params": [], "returns": "" , "templates": [], "throws": [] }

  " Mark current line for future use
  mark d

  " Look for function/method/... to document
  " We look only on the first three lines!
  while( match( l:lineBuffer, l:emptyLinePattern ) != -1 && l:count < 4 )
    exec "normal j"
    let l:lineBuffer = l:lineBuffer.' '.getline( line( "." ) )
    let l:count = l:count + 1
  endwhile
  " Error message when the buffer is still empty.
  if( match( l:lineBuffer, l:emptyLinePattern ) != -1 )
    call s:WarnMsg( "Nothing to document here!" )
    exec "normal `d"
    return
  endif

  " Remove unwanted lines (ie: jump to the first significant line)
  if( g:DoxygenToolkit_keepEmptyLineAfterComment == "no" )
    " This erase previous mark
    mark d
  endif

  " Look for the end of the function/class/... to document
  " TODO does not work when function/class/... is commented out!
  let l:readError = "Cannot reach end of function/class/... declaration!"
  let l:count = 0
  let l:throwCompleted = 0
  let l:endReadPattern = l:endDocPattern
  while( l:endDocFound == 0 && l:count < g:DoxygenToolkit_maxFunctionProtoLines )
    let l:lineBuffer = s:RemoveComments( l:lineBuffer )
    " Valid only for cpp. For Python it must be 'class ...:' or 'def ...:' or
    " '... EOL'.
    if( match( l:lineBuffer, l:endReadPattern ) != -1 )
      " Look for throw statement at the end
      if( s:CheckFileType() == "cpp" && l:throwCompleted == 0 )
        " throw statement can have already been read or can be on next line
        if( match( l:lineBuffer.' '.getline( line ( "." ) + 1 ), '.*\<throw\>.*' ) != -1 )
          let l:endReadPattern = l:throwPattern
          let l:throwCompleted = 1
          let l:readError = "Cannot reach end of throw statement"
        else
          let l:endDocFound = 1
        endif
      else
        let l:endDocFound = 1
      endif
      continue
    endif
    exec "normal j"
    let l:lineBuffer = l:lineBuffer.' '.getline( line( "." ))
    let l:count = l:count + 1
  endwhile
  " Error message when the end of the function(/...) has not been found
  if( l:endDocFound == 0 )
    if( match( l:lineBuffer, l:emptyLinePattern ) != -1 )
      " Fall here when only comments have been found.
      call s:WarnMsg( "Nothing to document here!" )
    else
      call s:WarnMsg( l:readError )
    endif
    exec "normal `d"
    return
  endif

  " Trim the buffer
  let l:lineBuffer = substitute( l:lineBuffer, "^[[:blank:]]*\|[[:blank:]]*$", "", "g" )

  " Check whether it is a template definition
  call s:ParseFunctionTemplateParameters( l:lineBuffer, l:doc )
  " Remove any template parameter.
  if( s:CheckFileType() == "cpp" )
    while( match( l:lineBuffer, l:templateParameterPattern ) != -1 )
      let l:lineBuffer = substitute( l:lineBuffer, l:templateParameterPattern, "", "g" )
    endwhile
  endif

  " Look for the type
  for key in keys( l:types )
    "call s:WarnMsg( "[DEBUG] buffer:_".l:lineBuffer."_, test:_".l:types[key] )
    let l:name = matchstr( l:lineBuffer, l:types[key] )
    if( l:name != "" )
      let l:doc.type = key
      let l:doc.name = l:name

      " Python only. Functions are detected differently for C/C++.
      if( key == "function" )
        "call s:WarnMsg( "HERE !!!".l:lineBuffer )
        call s:ParseFunctionParameters( l:lineBuffer, l:doc )
      endif
      break
    endif
  endfor

  if( l:doc.type == "" )
    " Should be a function/method (cpp only) or an attribute.
    " (cpp only) Can also be an unnamed enum/namespace... (or something else ?)
    if( s:CheckFileType() == "cpp" )
      if( match( l:lineBuffer, '(' ) == -1 )
        if( match( l:lineBuffer, '\<enum\>' ) != -1 )
          let l:doc.type = 'enum'
        elseif( match( l:lineBuffer, '\<namespace\>' ) != -1 )
          let l:doc.type = 'namespace'
        else
          " TODO here we get a class attribute of something like that.
          "      We probably just need a \brief statement...
          let l:doc.type = 'attribute'
          " TODO Retrieve the name of the attribute.
          "      Do we really need it? I'm not sure for the moment.
        endif
      else
        let l:doc.type = 'function'
        call s:ParseFunctionParameters( l:lineBuffer, l:doc )
        if( l:throwCompleted == 1 )
          call s:ParseThrowParameters( l:lineBuffer, l:doc, l:throwPattern )
        endif
      endif

    " This is an attribute for Python
    else
      let l:doc.type = 'attribute'
    endif
  endif

  " Remove the function/class/... name when it is not necessary
  if( ( l:doc.type == "class" && g:DoxygenToolkit_briefTag_className != "yes" ) || ( l:doc.type == "struct" && g:DoxygenToolkit_briefTag_structName != "yes" ) || ( l:doc.type == "enum" && g:DoxygenToolkit_briefTag_enumName != "yes" ) || ( l:doc.type == "namespace" && g:DoxygenToolkit_briefTag_namespaceName != "yes" ) || ( l:doc.type == "function" && g:DoxygenToolkit_briefTag_funcName != "yes" ) )
    let l:doc.name = "None"

  " Remove namespace from the name of the class/function...
  elseif( s:CheckFileType() == "cpp" )
    let l:doc.name = substitute( l:doc.name, '\%('.l:someNamePattern.'::\)', '', 'g' )
  endif

  " Below, write what we have found
  """""""""""""""""""""""""""""""""

  call s:InitializeParameters()
  if( s:CheckFileType() == "python" && l:doc.type == "function" && g:DoxygenToolkit_python_autoFunctionReturn == "yes" )
    let l:doc.returns = "yes"
  endif

  " Header
  exec "normal `d"
  if( g:DoxygenToolkit_blockHeader != "" )
    exec "normal O".strpart( s:startCommentBlock, 0, 1 )
    exec "normal A".strpart( s:startCommentBlock, 1 ).g:DoxygenToolkit_blockHeader.s:endCommentBlock
    exec "normal `d"
  endif

  " Brief
  if( g:DoxygenToolkit_compactOneLineDoc =~ "yes" && l:doc.returns != "yes" && len( l:doc.params ) == 0 )
    let s:compactOneLineDoc = "yes"
    exec "normal O".strpart( s:startCommentTag, 0, 1 )
    exec "normal A".strpart( s:startCommentTag, 1 ).g:DoxygenToolkit_briefTag_pre
  else
    let s:compactOneLineDoc = "no"
    let l:insertionMode = s:StartDocumentationBlock()
    exec "normal ".l:insertionMode.s:interCommentTag.g:DoxygenToolkit_briefTag_pre
  endif
  if( l:doc.name != "None" )
    exec "normal A".l:doc.name." "
  endif
  exec "normal A".g:DoxygenToolkit_briefTag_post

  " Mark the line where the cursor will be positionned.
  mark d

  " Arguments/parameters
  if( g:DoxygenToolkit_compactDoc =~ "yes" )
    let s:insertEmptyLine = 0
  else
    let s:insertEmptyLine = 1
  endif
  for param in l:doc.templates
    if( s:insertEmptyLine == 1 )
      exec "normal o".substitute( s:interCommentTag, "[[:blank:]]*$", "", "" )
      let s:insertEmptyLine = 0
    endif
    exec "normal o".s:interCommentTag.g:DoxygenToolkit_templateParamTag_pre.param.g:DoxygenToolkit_templateParamTag_post
  endfor
  for param in l:doc.params
    if( s:insertEmptyLine == 1 )
      exec "normal o".substitute( s:interCommentTag, "[[:blank:]]*$", "", "" )
      let s:insertEmptyLine = 0
    endif
    exec "normal o".s:interCommentTag.g:DoxygenToolkit_paramTag_pre.param.g:DoxygenToolkit_paramTag_post
  endfor

  " Returned value
  if( l:doc.returns == "yes" )
    if( g:DoxygenToolkit_compactDoc != "yes" )
      exec "normal o".substitute( s:interCommentTag, "[[:blank:]]*$", "", "" )
    endif
    exec "normal o".s:interCommentTag.g:DoxygenToolkit_returnTag
  endif

  " Exception (throw) values (cpp only)
  if( len( l:doc.throws ) > 0 )
    if( g:DoxygenToolkit_compactDoc =~ "yes" )
      let s:insertEmptyLine = 0
    else
      let s:insertEmptyLine = 1
    endif
    for param in l:doc.throws
      if( s:insertEmptyLine == 1 )
        exec "normal o".substitute( s:interCommentTag, "[[:blank:]]*$", "", "" )
        let s:insertEmptyLine = 0
      endif
      exec "normal o".s:interCommentTag.g:DoxygenToolkit_throwTag_pre.param.g:DoxygenToolkit_throwTag_post
    endfor
  endif

  " End (if any) of documentation block.
  if( s:endCommentTag != "" )
    if( s:compactOneLineDoc =~ "yes" )
      let s:execCommand = "A"
      exec "normal A "
      exec "normal $md"
    else
      let s:execCommand = "o"
    endif
    exec "normal ".s:execCommand.s:endCommentTag
  endif

  " Footer
  if ( g:DoxygenToolkit_blockFooter != "" )
    exec "normal o".strpart( s:startCommentBlock, 0, 1 )
    exec "normal A".strpart( s:startCommentBlock, 1 ).g:DoxygenToolkit_blockFooter.s:endCommentBlock
  endif
  exec "normal `d"

  call s:RestoreParameters()
  if( s:compactOneLineDoc =~ "yes" && s:endCommentTag != "" )
    startinsert
  else
    startinsert!
  endif

  " DEBUG purpose only
  "call s:WarnMsg( "Found a ".l:doc.type." named ".l:doc.name." (env: ".s:CheckFileType().")." )
  "if( l:doc.type == "function" )
  "  let l:funcReturn = "returns something."
  "  if( l:doc.returns == "" )
  "    let l:funcReturn = "doesn't return anything."
  "  endif
  "  call s:WarnMsg( " - which ".l:funcReturn )
  "  call s:WarnMsg( " - which has following parameter(s):" )
  "  for param in l:doc.params
  "    call s:WarnMsg( "   - ".param )
  "  endfor
  "endif

endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Write the beginning of the documentation block:
" - C and Python format: insert '/**' and '##' respectively then a linefeed,
" - C++ insert '///' and continue on the same line
"
" This function return the insertion mode which should be used for the next
" call to 'normal'.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:StartDocumentationBlock()
  " For C++ documentation format we do not need first empty line
  if( s:startCommentTag != s:interCommentTag )
    "exec "normal O".s:startCommentTag
    exec "normal O".strpart( s:startCommentTag, 0, 1 )
    exec "normal A".substitute( strpart( s:startCommentTag, 1 ), "[[:blank:]]*$", "", "" )
    let l:insertionMode = "o"
  else
    let l:insertionMode = "O"
  endif
  return l:insertionMode
endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Remove comments from the given buffer.
" - Remove everything after '//' or '#'.
" - Remove everything between '/*' and '*/' or keep '/*' if '*/' is not present.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:RemoveComments( lineBuffer )
  if( s:CheckFileType() == "cpp" )
    " Remove C++ (//) comment.
    let l:lineBuffer = substitute( a:lineBuffer, '[[:blank:]]*\/\/.*$', '', '')
    " Remove partial C (/* ...) comment: /* foo bar   -->   /*
    " '/*' is preserved until corresponding '*/' is found. Other part of the
    " comment is discarded to prevent the case where it contains characters
    " corresponding to the endDoc string.
    let l:lineBuffer = substitute( l:lineBuffer, '\%(\/\*\zs.*\ze\)\&\%(\%(\/\*.*\*\/\)\@!\)', '', '')
    " Remove C (/* ... */) comment.
    let l:lineBuffer = substitute( l:lineBuffer, '\/\*.\{-}\*\/', '', 'g')
  else
    let l:lineBuffer = substitute( a:lineBuffer, '[[:blank:]]*#.*$', '', '')
  endif
  return l:lineBuffer
endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Retrieve file type.
" - Default type is still 'cpp'.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:CheckFileType()
  if( &filetype == "python" )
    let l:fileType       = "python"
  else
    let l:fileType       = "cpp"
  endif
  return l:fileType
endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Parse the buffer and set the doc parameter.
" - Functions which return pointer to function are not supported.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:ParseFunctionParameters( lineBuffer, doc )
  "call s:WarnMsg( 'IN__'.a:lineBuffer )
  let l:paramPosition = matchend( a:lineBuffer, 'operator[[:blank:]]*([[:blank:]]*)' )
  if ( l:paramPosition == -1 )
    let l:paramPosition = stridx( a:lineBuffer, '(' )
  else
    let l:paramPosition = stridx( a:lineBuffer, '(', l:paramPosition )
  endif


  " (cpp only) First deal with function name and returned value.
  " Function name has already been retrieved for Python and we need to parse
  " all the function definition to know whether a value is returned or not.
  if( s:CheckFileType() == "cpp" )
    let l:functionBuffer = strpart( a:lineBuffer, 0, l:paramPosition )
    " Remove unnecessary elements
    for ignored in g:DoxygenToolkit_ignoreForReturn
      let l:functionBuffer = substitute( l:functionBuffer, '\<'.ignored.'\>', '', 'g' )
    endfor
    let l:functionReturnAndName = split( l:functionBuffer, '[[:blank:]*]' )
    if( len( l:functionReturnAndName ) > 1 )
      let a:doc.returns = 'yes'
    endif
    let a:doc.name = l:functionReturnAndName[-1]
  endif

  " Work on parameters.
  let l:parametersBuffer = strpart( a:lineBuffer, l:paramPosition + 1 )
  " Remove trailing closing bracket and everything that follows and trim.
  if( s:CheckFileType() == "cpp" )
    let l:parametersBuffer = substitute( l:parametersBuffer, ')[^)]*\%(;\|{\|\%([^:]:\%([^:]\|$\)\)\|\%(\<throw\>\)\).*', '', '' )
  else
    let l:parametersBuffer = substitute( l:parametersBuffer, ')[^)]*:.*', '', '' )
  endif
  let l:parametersBuffer = substitute( l:parametersBuffer, '^[[:blank:]]*\|[[:blank:]]*$', '', '' )

  " Remove default parameter values (if any).
  let l:index = stridx( l:parametersBuffer, '=' )
  let l:startIndex = l:index
  while( l:index != -1 )
    " Look for the next colon...
    let l:colonIndex = stridx( l:parametersBuffer, ',', l:startIndex )
    if( l:colonIndex == -1 )
      let l:colonIndex = strlen( l:parametersBuffer )
    endif
    let l:paramBuffer = strpart( l:parametersBuffer, l:index, l:colonIndex - l:index )
    if( s:CountBrackets( l:paramBuffer ) == 0 )
      " Everything in [l:index, l:colonIndex[ can be removed.
      let l:parametersBuffer = substitute( l:parametersBuffer, l:paramBuffer, '', '' )
      let l:index = stridx( l:parametersBuffer, '=' )
      let l:startIndex = l:index
    else
      " Parameter initialization contains brakets and colons...
      let l:startIndex = l:colonIndex + 1
    endif
  endwhile

  "call s:WarnMsg( "[DEBUG]: ".l:parametersBuffer )
  " Now, work on each parameter.
  let l:params = []
  let l:index = stridx( l:parametersBuffer, ',' )
  while( l:index != -1 )
    let l:paramBuffer = strpart( l:parametersBuffer, 0, l:index )
    if( s:CountBrackets( l:paramBuffer ) == 0 )
      let l:params = add( l:params, s:ParseParameter( l:paramBuffer ) )
      let l:parametersBuffer = strpart( l:parametersBuffer, l:index + 1 )
      let l:index = stridx( l:parametersBuffer, ',' )
    else
      let l:index = stridx( l:parametersBuffer, ',', l:index + 1 )
    endif
  endwhile
  if( strlen( l:parametersBuffer ) != 0 )
    let l:params = add( l:params, s:ParseParameter( l:parametersBuffer ) )
  endif

  if( s:CheckFileType() == "cpp" )
    call filter( l:params, 'v:val !~ "void"' )
  else
    if( g:DoxygenToolkit_python_autoRemoveSelfParam == "yes" )
      call filter( l:params, 'v:val !~ "self"' )
    endif
  endif

  for param in l:params
    call add( a:doc.params, param )
    "call s:WarnMsg( '[DEBUG]:OUT_'.param )
  endfor
endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Parse given parameter and return its name.
" It is easy to do unless you use function's pointers...
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:ParseParameter( param )
  let l:paramName = "Unknown"
  let l:firstIndex = stridx( a:param, '(' )

  if( l:firstIndex == -1 )
    let l:paramName =  split( a:param, '[[:blank:]*&]' )[-1]
  else
    if( l:firstIndex != 0 )
      let l:startIndex = 0
    else
      let l:startIndex = stridx( a:param, ')' )
      if( l:startIndex == -1 ) " Argggg...
        let l:paramName =  a:param
      else
        let l:startIndex += 1
        while( s:CountBrackets( strpart( a:param, 0, l:startIndex ) ) != 0 )
          let l:startIndex = stridx( a:param, ')', l:startIndex + 1 ) + 1
          if( l:startIndex == -1) " Argggg...
            let l:paramName =  a:param
          endif
        endwhile
      endif
    endif

    if( l:startIndex != -1 )
      let l:startIndex = stridx( a:param, '(', l:startIndex ) + 1
      let l:endIndex = stridx( a:param, ')', l:startIndex + 1 )
      let l:param = strpart( a:param, l:startIndex, l:endIndex - l:startIndex )
      let l:paramName =  substitute( l:param, '^[[:blank:]*]*\|[[:blank:]*]*$', '', '' )
    else
      " Something really wrong has happened.
      let l:paramName =  a:param
    endif
  endif

  return l:paramName
endfunction

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Extract template parameter name for function/class/method
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:ParseFunctionTemplateParameters( lineBuffer, doc )
  if( match( a:lineBuffer, '^[[:blank:]]*template' ) == 0 )
    let l:firstIndex = stridx( a:lineBuffer, '<' )
    if( l:firstIndex != -1 )
      let l:lastIndex = stridx( a:lineBuffer, '>', l:firstIndex + 1 )
      if( l:lastIndex != -1 )
        " Keep only template parameters
        let l:parameters = strpart( a:lineBuffer, l:firstIndex + 1, l:lastIndex - l:firstIndex - 1)
        " Split on separator (,)
        let l:params = split( l:parameters, '\,' )
        for param in l:params
          " Extract template parameter name
          let l:paramName = split( split( param, '=' )[0], '[[:blank:]]' )[-1]
          call add( a:doc.templates, l:paramName )
        endfor
      endif
    endif
  endif
endfunction

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Extract throw parameter name
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:ParseThrowParameters( lineBuffer, doc, throwPattern )
  let l:throwParams = substitute( a:lineBuffer, a:throwPattern, '\1', "" )
  for param in split( l:throwParams, "," )
    call add( a:doc.throws, substitute( param, '[[:blank:]]', '', "" ) )
  endfor
endfunction

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Define start/end documentation format and backup generic parameters.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:InitializeParameters()
  if( s:CheckFileType() == "cpp" )
    let s:startCommentTag   = g:DoxygenToolkit_startCommentTag
    let s:interCommentTag   = g:DoxygenToolkit_interCommentTag
    let s:endCommentTag     = g:DoxygenToolkit_endCommentTag
    let s:startCommentBlock = g:DoxygenToolkit_startCommentBlock
    let s:interCommentBlock = g:DoxygenToolkit_interCommentBlock
    let s:endCommentBlock   = g:DoxygenToolkit_endCommentBlock
  else
    let s:startCommentTag   = "## "
    let s:interCommentTag   = "# "
    let s:endCommentTag     = ""
    let s:startCommentBlock = "# "
    let s:interCommentBlock = "# "
    let s:endCommentBlock   = ""
  endif

  " Backup standard comment expension and indentation
  let s:commentsBackup = &comments
  let &comments        = ""
  let s:cinoptionsBackup = &cinoptions
  let &cinoptions        = g:DoxygenToolkit_cinoptions
  " Compatibility with c/c++ IDE plugin
  let s:timeoutlenBackup = &timeoutlen
  let &timeoutlen = 0
endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Restore previously backuped parameters.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:RestoreParameters()
  " Restore standard comment expension and indentation
  let &comments = s:commentsBackup
  let &cinoptions = s:cinoptionsBackup
  " Compatibility with c/c++ IDE plugin
  let &timeoutlen = s:timeoutlenBackup
endfunction


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Count opened/closed brackets in the given buffer.
" Each opened bracket increase the counter by 1.
" Each closed bracket decrease the counter by 1.
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
function! s:CountBrackets( buffer )
  let l:count =  len( split( a:buffer, '(', 1 ) )
  let l:count -= len( split( a:buffer, ')', 1 ) )
  return l:count
endfunction


"""""""""""""""""""""""""""""""""""
" Simple warning message function
"""""""""""""""""""""""""""""""""""
function! s:WarnMsg( msg )
  echohl WarningMsg
  echo a:msg
  echohl None
  return
endfunction

""""""""""""""""""""""""""
" Shortcuts...
""""""""""""""""""""""""""
command! -nargs=0 Dox :call <SID>DoxygenCommentFunc()
command! -nargs=0 DoxLic :call <SID>DoxygenLicenseFunc()
command! -nargs=0 DoxAuthor :call <SID>DoxygenAuthorFunc()
command! -nargs=1 DoxUndoc :call <SID>DoxygenUndocumentFunc(<q-args>)
command! -nargs=0 DoxBlock :call <SID>DoxygenBlockFunc()
```
