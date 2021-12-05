---
title: "vimのカスタマイズは.vimrcのみで完結させたい"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "vimplugin"]
published: true
---

# vim のカスタマイズは簡潔に済ませたい

新しい作業環境ができたときに毎回セットアップするのはなかなか面倒。
プラグイン管理入れたり色々設定するのも面倒。
`~/.vimrc`だけで済ませたい = vim 開いたら勝手にセットアップしてほしい。
やりたいことは`~/.vimrc`の記述だけでできるだけ実現してみた。

## カラースキーマ設定

プラグインマネージャーで入れる方法もあると思いますが、なんだか気持ち的にここは切り離したいので切り離しています。
`jellybeans.vim` が存在しない場合に curl で取ってきます。`~/.vim/colors`のディレクトリ事前作成はしなくてよいように curl に`--create-dirs`オプションを付けます。
他のカラースキーマにしたい方は、ファイル名や url を適宜変更してください。
TODO ハイライト設定は jellybeans 固有のカスタマイズ設定なので、他のカラースキーマを選択する方は無視してください。

```vim
if empty(glob('~/.vim/colors/jellybeans.vim'))
  silent !curl -fLo ~/.vim/colors/jellybeans.vim --create-dirs
    \ https://raw.githubusercontent.com/nanotech/jellybeans.vim/master/colors/jellybeans.vim
    \ >/dev/null 2>&1
endif
colorscheme jellybeans
" TODOハイライト設定
let g:jellybeans_overrides = {
\    'Todo': { 'guifg': '303030', 'guibg': 'f0f000',
\              'ctermfg': 'Black', 'ctermbg': 'Yellow',
\              'attr': 'bold' },
\    'Comment': { 'guifg': 'cccccc' },
\}
```

## undo 永続化

履歴保持フォルダがなければ作成して設定。

```vim
" undo 永続化
silent !mkdir ~/.vim/undo -p >/dev/null 2>&1
if has('persistent_undo')
  set undodir=~/.vim/undo
  set undofile
endif
```

## プラグイン

### プラグイン管理

色々ありますが、一番シンプルな`vim-plug`使います。

https://github.com/junegunn/vim-plug

### プラグインマネージャーの導入

カラースキーマ導入の時と同じようなやり方で導入します。

```vim
if empty(glob('~/.vim/autoload/plug.vim'))
  silent !curl -fLo ~/.vim/autoload/plug.vim --create-dirs
    \ https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
    \ >/dev/null 2>&1
  autocmd VimEnter * PlugInstall --sync | source $MYVIMRC
endif
```

### プラグイン設定

`call plug#begin('~/.vim/plugged')`から`call plug#end()`の間にプラグイン設定を記述します。

```vim
call plug#begin('~/.vim/plugged')

Plug 'bling/vim-airline'
Plug 'vim-airline/vim-airline-themes'
Plug 'junegunn/vim-easy-align'
Plug 'preservim/nerdtree'
Plug 'sheerun/vim-polyglot'
Plug 'tpope/vim-fugitive'
Plug 'airblade/vim-gitgutter'
Plug 'mhinz/vim-signify'
" Ctrl + p でファイル・バッファをあいまい検索
Plug 'ctrlpvim/ctrlp.vim'
Plug 'jmcantrell/vim-virtualenv'

" theme setting
set t_Co=256 " この設定がないと色が正しく表示されない場合がある
" let g:airline_powerline_fonts = 1
let g:airline_theme = 'luna'   " テーマ指定
" 他テーマを指定したい場合には以下を参考にお好みのものを指定
" https://github.com/vim-airline/vim-airline/wiki/Screenshots
let g:airline#extensions#hunks#enabled = 0
let g:airline#extensions#branch#enabled = 1
let g:airline#extensions#tabline#enabled = 1 " タブラインを表示
set ttimeoutlen=50 " モード変更遅延解消

" Ctrl + e でエクスプローラー表示
nnoremap <silent><C-e> :NERDTreeToggle<CR>
" Ctrl + b Ctrl + n でタブ移動
nmap <C-b> <Plug>AirlineSelectPrevTab
nmap <C-n> <Plug>AirlineSelectNextTab

let g:airline#extensions#tabline#buffer_idx_mode = 1 " タブ番号表示

" symbol
if !exists('g:airline_symbols')
  let g:airline_symbols = {}
endif
" unicode symbols
" let g:airline_left_sep = '»'
" let g:airline_left_sep = '▶'
" let g:airline_right_sep = '«'
let g:airline_right_sep = '◀'
let g:airline_symbols.crypt = '🔒'
" let g:airline_symbols.linenr = '␊'
" let g:airline_symbols.linenr = '␤'
let g:airline_symbols.linenr = '¶'
" let g:airline_symbols.maxlinenr = '☰'
let g:airline_symbols.maxlinenr = ''
let g:airline_symbols.branch = '⎇'
let g:airline_symbols.paste = 'ρ'
" let g:airline_symbols.paste = 'Þ'
" let g:airline_symbols.paste = '∥'
let g:airline_symbols.spell = 'Ꞩ'
let g:airline_symbols.notexists = '∄'
let g:airline_symbols.whitespace = 'Ξ'

call plug#end()
```

### プラグインインストール

`Plug 'PluginName'`で利用したいプラグインを`~/.vimrc`に記述しただけではプラグインはインストールされず、`:PlugInstall`を実行してインストールが必要があります。
これも面倒なので、'~/.vimrc'にインストールしてもらい、何もしなくていいようにします。

```vim
autocmd VimEnter * if len(filter(values(g:plugs), '!isdirectory(v:val.dir)'))
  \| PlugInstall --sync | source $MYVIMRC
```

### powerline フォント

Powerline 用のフォントの設定は面倒なのでやっていません。
Nerd Fonts 等をいれてイケてるアイコンを表示するのを実現しようとすると、
利用しているデバイスそれぞれでフォントのインストールや端末利用フォント設定をしなければいけないので、OS は Windows/Mac/Linux 、エディタは vim/Sublime Text/VScode を行き来する自分としては面倒でモチベーションあがらずといところでした。
`~/.vimrc`だけで実現できるならやってみたくありますが、難しそうなのでパスしました。

### ~/.vimrc

他設定含んだ全量こちら。`~/.vimrc`置くだけで勝手にセットアップされて楽かなと思います。

```vim
" color
if empty(glob('~/.vim/colors/jellybeans.vim'))
  silent !curl -fLo ~/.vim/colors/jellybeans.vim --create-dirs
    \ https://raw.githubusercontent.com/nanotech/jellybeans.vim/master/colors/jellybeans.vim
    \ >/dev/null 2>&1
endif
colorscheme jellybeans
" TODOハイライト設定
let g:jellybeans_overrides = {
\    'Todo': { 'guifg': '303030', 'guibg': 'f0f000',
\              'ctermfg': 'Black', 'ctermbg': 'Yellow',
\              'attr': 'bold' },
\    'Comment': { 'guifg': 'cccccc' },
\}

set encoding=utf-8
set fileencodings=utf-8,cp932
set autoread
set number
set tabstop=4
set shiftwidth=4
set smartindent
syntax enable
set hlsearch
set incsearch
set ignorecase
set smartcase
set wrapscan
set wildmenu
set history=5000
" 不可視文字(e.g. \t)の表示設定
set list listchars=tab:\▸\-
" 行末の1文字先までカーソルを移動できるように
set virtualedit=onemore
" 記号表記で崩れないように
set ambiwidth=double

" undo 永続化
silent !mkdir ~/.vim/undo -p >/dev/null 2>&1
if has('persistent_undo')
  set undodir=~/.vim/undo
  set undofile
endif

" 補完
inoremap ( ()<LEFT>
inoremap " ""<LEFT>
inoremap ' ''<LEFT>
inoremap [ []<LEFT>
inoremap { {}<LEFT>
inoremap { {}<LEFT>
inoremap < <><LEFT>
" インデント
inoremap {<Enter> {}<Left><CR><CR><BS><Up><Right>

" To use fzf in Vim, add the following line to your .vimrc:
set rtp+=/usr/local/opt/fzf

" vim-plug
" install vim-plug if not exists
if empty(glob('~/.vim/autoload/plug.vim'))
  silent !curl -fLo ~/.vim/autoload/plug.vim --create-dirs
    \ https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
    \ >/dev/null 2>&1
  autocmd VimEnter * PlugInstall --sync | source $MYVIMRC
endif
" auto install plugin
autocmd VimEnter * if len(filter(values(g:plugs), '!isdirectory(v:val.dir)'))
  \| PlugInstall --sync | source $MYVIMRC
\| endif

call plug#begin('~/.vim/plugged')

Plug 'bling/vim-airline'
Plug 'vim-airline/vim-airline-themes'
Plug 'junegunn/vim-easy-align'
Plug 'preservim/nerdtree'
Plug 'sheerun/vim-polyglot'
Plug 'tpope/vim-fugitive'
Plug 'airblade/vim-gitgutter'
Plug 'mhinz/vim-signify'
Plug 'ctrlpvim/ctrlp.vim'
Plug 'jmcantrell/vim-virtualenv'

" theme screenshots
" https://github.com/vim-airline/vim-airline/wiki/Screenshots

" theme setting
set t_Co=256 " この設定がないと色が正しく表示されない場合がある
" let g:airline_powerline_fonts = 1
let g:airline_theme = 'luna'   " テーマ指定
let g:airline#extensions#hunks#enabled = 0
let g:airline#extensions#branch#enabled = 1
let g:airline#extensions#tabline#enabled = 1 " タブラインを表示
set ttimeoutlen=50 " モード変更遅延解消

nnoremap <silent><C-e> :NERDTreeToggle<CR>
" タブの切り替えショートカット設定
nmap <C-b> <Plug>AirlineSelectPrevTab
nmap <C-n> <Plug>AirlineSelectNextTab

let g:airline#extensions#tabline#buffer_idx_mode = 1 " タブ番号表示

" symbol
if !exists('g:airline_symbols')
  let g:airline_symbols = {}
endif
" unicode symbols
" let g:airline_left_sep = '»'
" let g:airline_left_sep = '▶'
" let g:airline_right_sep = '«'
let g:airline_right_sep = '◀'
let g:airline_symbols.crypt = '🔒'
" let g:airline_symbols.linenr = '␊'
" let g:airline_symbols.linenr = '␤'
let g:airline_symbols.linenr = '¶'
" let g:airline_symbols.maxlinenr = '☰'
let g:airline_symbols.maxlinenr = ''
let g:airline_symbols.branch = '⎇'
let g:airline_symbols.paste = 'ρ'
" let g:airline_symbols.paste = 'Þ'
" let g:airline_symbols.paste = '∥'
let g:airline_symbols.spell = 'Ꞩ'
let g:airline_symbols.notexists = '∄'
let g:airline_symbols.whitespace = 'Ξ'

call plug#end()
```

### やりたいけどできてないこと

Github CLI/ghq/fzf あたりを用いて cli 上で実現する git 便利操作を vim で実現できてない。
良いプラグインであったり`~/.vimrc`の書き方あればなあとは思っているものの、サクッとみつからないのでそういう作業したいときは他エディタだったりターミナル上での作業だったりに移行してしまいます。
vim にハマりきってないのは、git 便利操作がなかなかうまく実現できないからというところはあるので、いい感じのプラグインインストール一発解決できたらいいなあと思っています。(プラグインは既にあって知らないだけかも)
