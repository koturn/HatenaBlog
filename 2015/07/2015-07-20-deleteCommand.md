Vimで実行に失敗したら，自身を削除するコマンドを定義する
=======================================================

実行に失敗したら，自身を削除するコマンドを定義する

自作のVimプラグインで，[ctrlp.vim](https://github.com/ctrlpvim/ctrlp.vim)の拡張を作成したとき， ```plugin/hoge.vim``` にて，

```vim
command! CtrlPHoge  call ctrlp#init(ctrlp#hoge#id())
```

のような定義をする．
しかし，ctrlp.vimの拡張はオプショナルであり，必須ではない．
ユーザの中には，CtrlPを導入していない人もいるだろう．
なので，ctrlp.vimが利用可能な環境でのみ，ctrlp.vimを利用するコマンドを定義することにしたい．

```vim
if globpath(&rtp, 'autoload/vimproc.vim') !=# ''
  command! CtrlPHoge  call ctrlp#init(ctrlp#hoge#id())
endif
```

ただし，この方法には問題がある．
```ctrlp.vim``` が[neobundle.vim](https://github.com/Shougo/neobundle.vim)で管理されており，Lazy読み込み設定がなされている場合， ```runtimepath``` から発見することができない．
かといって， ```ctrlp#init()``` を呼び出して確認するのは，autoloadの読み込みが発生するので，あまり好ましくない．
（そもそも，ここまで神経質になる必要は無いのだが）

ということで，最初から ```:CtrlPHoge``` を定義しないことは諦めて， ```:CtrlPHoge``` の実行に失敗したら， ```:CtrlPHoge``` 自身を消去しようと思う．

```vim
function! s:ctrlp_hook() abort
  try
    call ctrlp#init(ctrlp#hoge#id())
    command! CtrlPHoge  call ctrlp#init(ctrlp#hoge#id())
  catch /^Vim\%((\a\+)\)\=:E\%(117\): .\+: ctrlp#init$/
    delcommand CtrlPHoge
    echoerr 'ctrlpvim/ctrlp.vim is not installed.'
  endtry
endfunction

command! CtrlPHoge  call s:ctrlp_hook() | delfunction s:crtlp_hook
```

やっていることは単純．
最初に仮の関数を呼び出すように定義し， ```ctrlp#init()``` が呼び出せない，すなわちctrlp.vimが利用できないならば，コマンドと仮の関数を消去するだけである．
逆に， ```ctrlp#init()``` が呼び出せるならば，コマンドを上書きする．

上記コードにおいて， ```catch /^Vim\%((\a\+)\)\=:E\%(117\): .\+: ctrlp#init$/``` と長ったらしく記述しているが，単純に ```catch``` で全例外を捕捉しても問題は無いと思う．
（そもそも，ctrlp.vimが存在しないならば， ```ctrlp#hoge#id()``` の呼び出しで ```g:ctrlp_builtins``` が定義されていないというエラーが排出されるので，このエラーキャッチはやや不正確）

さて，このような「実行に失敗したら，自身を消去するコマンドを定義する関数」を書いてみた．
ぶっちゃけ，遊びで作ったこの関数を紹介するために，この記事を書いたのである．
実用性は無い！

```vim
scriptencoding utf-8
" 仮コマンドの実行部分にある<bang>, <line1>, <line2>, <count>, <args>, <q-args>, <f-args>, <lt> を置換する
" action:  与えられたコマンドのアクション部分
" bang:    <bang>の展開結果
" line1:   <line1>の展開結果
" line2:   <line2>の展開結果
" count:   <count>の展開結果
" va_args: 仮コマンドに与えられた引数
function! s:expand_args(action, bang, line1, line2, count, va_args) abort
  let va_args = copy(a:va_args)
  let action = substitute(a:action, '<bang>', a:bang, 'g')
  let action = substitute(action, '<line1>', a:line1, 'g')
  let action = substitute(action, '<line2>', a:line2, 'g')
  let action = substitute(action, '<count>', a:count, 'g')
  let action = substitute(action, '<args>', join(va_args, ' '), 'g')
  let action = substitute(action, '<q-args>', string(join(va_args, ', ')), 'g')
  let action = substitute(action, '<f-args>', join(map(va_args, 'string(v:val)'), ', '), 'g')
  return substitute(action, '<lt>', '<', 'g')
endfunction

" 仮コマンドを定義する
" 仮コマンドの実行に成功したならば，実際のコマンドを定義し，失敗したらならば，この仮コマンドを消去する
" cmdname:  コマンド名
" cmdattr:  コマンドの属性: -bar, -bang, -range など
" action:   コマンドのアクション部: echo 'foo' など
" (errmsg): エラーキャッチ時のメッセージ（省略可能）
" (errptn): キャッチするエラーのパターン（省略可能）
function! s:define_mock_command(cmdname, cmdattr, action, ...) abort
  let tmpfunc = 's:' . (has('cryptv') ? sha256(reltimestr(reltime()))[: 15] : substitute(tempname(), '[\.:/\\]', '_', 'g'))
  let errmsg = a:0 > 0 ? string(a:1) : ''
  let errptn = a:0 > 1 ? ('/' . a:2 . '/') : ''
  execute 'function!' tmpfunc . "(bang, line1, line2, count, ...) abort\n"
        \   "try\n"
        \     'execute s:expand_args(' string(a:action) ", a:bang, a:line1, a:line2, a:count, a:000)\n"
        \     'command!' a:cmdattr a:cmdname a:action . "\n"
        \   'catch' errptn "\n"
        \     'delcommand' a:cmdname "\n"
        \     'echoerr' errmsg "\n"
        \   "endtry\n"
        \ 'endfunction'
  execute 'command!' a:cmdattr a:cmdname 'call' tmpfunc
        \ . '("<bang>", <line1>, <line2>, <count>, <f-args>) | delfunction' tmpfunc
endfunction

function! s:test(...) abort
  echo a:0 ':' a:000
endfunction

" Success example
call s:define_mock_command('Foo', '', 'echo "Hello World"')
call s:define_mock_command('Bar', '-bar', 'call s:test(1, 2, 3)')
call s:define_mock_command('Baz', '-bar -bang', 'call s:test("<bang>", 1, 2, 3)')
call s:define_mock_command('Qux', '-bar -bang -range=% -nargs=*', 'call s:test(<bang>0, <line1>, <line2>, <count>, "<lt>count>", <f-args>)', 'ERROR')

echo '[Foo]'
command Foo
Foo
command Foo
Foo

echo '[Bar]'
command Bar
Bar
command Bar
Bar

echo '[Baz]'
command Baz
Baz!
command Baz
Baz!

echo '[Qux]'
command Qux
1,1Qux! 1 2 3 4
command Qux
1,1Qux! 1 2 3 4


" Failed example
call s:define_mock_command('Quux', '-bar', 'call s:test01()', 'function: s:test01() is not defined')
call s:define_mock_command('FooBar', '-bar', 'call s:test02()', 'function: s:test02() is not defined', '^Vim\%((\a\+)\)\=:E117: .\+: s:test02$')

echo '[Quux]'
command Quux
Quux
command Quux
Quux

echo '[FooBar]'
command FooBar
FooBar
command FooBar
FooBar
```

個人的に面白いと思っているポイントは，一時的なコマンドが呼び出す関数名である．
以下のようにして，一時的に利用する関数名を決定している．

```vim
let tmpfunc = 's:' . (has('cryptv') ? sha256(reltimestr(reltime()))[: 15] : substitute(tempname(), '[\.:/\\]', '_', 'g'))
```

これは， ```sha256``` が利用できる環境ならば ```sha256()``` を，そうでないならば， ```tempname()``` の識別子に使用できない文字を ```_``` に置換して，一時的に利用する関数の名前を作成している．
ちなみに， ```<bang>``` 等を用いないのであれば，以下のように単純に定義するだけでよい．

```vim
" 仮コマンドを定義するが，<bang>等の置換を行わない．<bang>等を用いないコマンドの定義に用いる
" 仮コマンドの実行に成功したならば，実際のコマンドを定義し，失敗したらならば，この仮コマンドを消去する
" cmdname:  コマンド名
" cmdattr:  コマンドの属性: -bar, -bang, -range など
" action:   コマンドのアクション部: echo 'foo' など
" (errmsg): エラーキャッチ時のメッセージ（省略可能）
" (errptn): キャッチするエラーのパターン（省略可能）
function! s:define_mock_command_easy(cmdname, cmdattr, action, ...) abort
  let tmpfunc = 's:' . (has('cryptv') ? sha256(reltimestr(reltime()))[: 15] : substitute(tempname(), '[\.:/\\]', '_', 'g'))
  let errmsg = a:0 > 0 ? string(a:1) : ''
  let errptn = a:0 > 1 ? ('/' . a:2 . '/') : ''
  execute 'function!' tmpfunc . "(...) abort\n"
        \   "try\n"
        \     a:action "\n"
        \     'command!' a:cmdattr a:cmdname a:action . "\n"
        \   'catch' errptn "\n"
        \     'delcommand' a:cmdname "\n"
        \     'echoerr' errmsg "\n"
        \   "endtry\n"
        \ 'endfunction'
  execute 'command!' a:cmdattr a:cmdname 'call' tmpfunc . '(<f-args>) | delfunction' tmpfunc
endfunction
```
