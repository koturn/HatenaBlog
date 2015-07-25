シェル上で普通の言語のようにVim scriptを実行したい
==================================================

元ネタは[Wandbox](http://melpon.org/wandbox/)に[Vim scriptが追加されたこと](https://twitter.com/melponn/status/602465969181577217)である．
Wandboxが実行するコマンドラインをパクれば，シェル上でVim scriptが実行できそうだと考えた．
ただ，オプションの数が多く，いちいち打ち込むのは面倒なので，[シェルスクリプトとバッチファイル](https://github.com/koturn/vs)を作った．
vsというネーミングは，正直微妙な気もするが，Javascriptのエンジンのひとつである[SpiderMonkey](https://developer.mozilla.org/ja/docs/SpiderMonkey)のバイナリ名を真似た結果だ．
それに，名前が短い方が打ちやすいという利点があると思う．

作成するにあたって，少々苦労した点もある．
Vimの出力が全て標準エラー出力だったり，コマンドプロンプトはワイルドカードを展開しない，という点だ．

```sh
#!/bin/sh

vim=vim

if [ $# -lt 1 ]; then
  echo 'Invalid arguments' 1>&2
  echo '[USAGE]'
  echo '  vs SRC [ARGS...]'
  exit 1
fi

if [ -f $1 ]; then
  $vim -X -N -u NONE -i NONE -e --cmd "source $1 | qall!" $@ 2>&1
else
  echo "File not found: $1" 1>&2
fi
```

```sh
@echo off
setlocal ENABLEDELAYEDEXPANSION

set vim=vim
set src=%1
set argv=%1
shift

:LOOPSTART
  if "%~1" == "" (
    goto LOOPEND
  )
  for %%i in (%1) do (
    set argv=!argv! %%i
  )
  shift
  goto LOOPSTART
:LOOPEND

if "%src%" == "" (
  echo Invalid arguments 1>&2
  echo [USAGE]
  echo   vs SRC [ARGS...]
) else (
  if exist %src% (
    %vim% -X -N -u NONE -i NONE -e --cmd "source %src% | qall!" %argv% 2>&1
  ) else (
    echo File not found: %src% 1>&2
  )
)
```

最初の問題は，```2>&1```として，標準エラー出力を標準入力に流すことで簡単に解決した．
後者の問題は，頑張ってループを回すことで，ワイルドカードを展開した．
バッチファイルの文法や，ループ内での代入はクセがあって，なかなか苦労したものだ...．
（バッチファイルについては全く自信が無いので，ツッコミどころは満載だと思う）


## 使い方

[GitHubのリポジトリ](https://github.com/koturn/vs)に書いてある通り，パスの通ったところに，```vs```，もしくは```vs.bat```を置いて．実行したいVim scriptを指定するだけだ．
例えば，以下のようなVim scriptを用意する．

```vim
" src.vim
echo 'argc:' argc()
echo 'argv:' argv()
echo 'Hello World!'
```

そして，以下のように実行する．

```sh
$ vs src.vim
argc: 1
argv: ['src.vim']
Hello World!
$ vs src.vim apple banana cake
4 個のファイルが編集を控えています
argc: 4
argv: ['src.vim', 'apple', 'banana', 'cake']
Hello World!
```

ただ，いくつかの問題もある．

まず，コマンドライン引数をとれるようにしてみたものの，Vimに引渡しているので，ファイル名として渡っていることになる．
そのため，どうしても「4 個のファイルが編集を控えています」のようなメッセージが出るところや，ファイル名として認識できない文字列（例えば，```+```など）を渡すことができない点が問題だ．

また，出力をファイルにリダイレクトしたときに，端末制御用と思われる文字列も出力されてしまうのも問題だ．

最後に，Vim scriptで実装したBrainfuck処理系を披露しよう．
[この記事](http://mattn.kaoriya.net/software/vim/20100311235228.htm)の実装に少し手を加えただけのものだ．
連続する```><+-```をひとつにまとめたり，```[-]```を「ゼロ代入命令」に置き換えるという軽度の最適化を行うようにしてある．

```vim
let s:Brainfuck = {
      \ 'pc': 0,
      \ 'dc': 0,
      \ 'bytecode': [],
      \ 'buf': repeat([0], 65536)
      \}

function! s:Brainfuck.add(val) abort dict
  let self.buf[self.dc] += a:val
endfunction

function! s:Brainfuck.sub(val) abort dict
  let self.buf[self.dc] -= a:val
endfunction

function! s:Brainfuck.next(val) abort dict
  let self.dc += a:val
endfunction

function! s:Brainfuck.prev(val) abort dict
  let self.dc -= a:val
endfunction

function! s:Brainfuck.putchar(val) abort dict
  echon nr2char(self.buf[self.dc])
endfunction

function! s:Brainfuck.getchar(val) abort dict
  let self.buf[self.dc] = getchar()
endfunction

function! s:Brainfuck.loop_start(val) abort dict
  if self.buf[self.dc] != 0 | return | endif
  let self.pc += 1
  let c = 0
  while c > 0 || self.bytecode[self.pc].func !=# 'loop_end'
    if self.bytecode[self.pc].func ==# 'loop_start' | let c += 1
    elseif self.bytecode[self.pc].func ==# 'loop_end' | let c -= 1 | endif
    let self.pc += 1
  endwhile
endfunction

function! s:Brainfuck.assign(val) abort dict
  let self.buf[self.dc] = a:val
endfunction

function! s:Brainfuck.loop_end(val) abort dict
  let self.pc -= 1
  let c = 0
  while c > 0 || self.bytecode[self.pc].func !=# 'loop_start'
    if self.bytecode[self.pc].func ==# 'loop_end' | let c += 1
    elseif self.bytecode[self.pc].func ==# 'loop_start' | let c -= 1 | endif
    let self.pc -= 1
  endwhile
  let self.pc -= 1
endfunction

function! s:Brainfuck.compile(src) abort dict
  let inst_table = {
        \ '+': 'add',
        \ '-': 'sub',
        \ '>': 'next',
        \ '<': 'prev',
        \ '.': 'putchar',
        \ ',': 'getchar',
        \ '[': 'loop_start',
        \ ']': 'loop_end',
        \ '[-]': 'assign',
        \}
  let src = substitute(a:src, '[^+-><\[\]\.,]', '', 'g')
  let src = substitute(src, '\[-\]', '\=printf("{\"func\": \"%s\"# \"val\": 0}# ", inst_table[submatch(0)])', 'g')
  let src = substitute(src, '\%(+\+\|-\+\|>\+\|<\+\)', '\=printf("{\"func\": \"%s\"# \"val\": %d}# ", inst_table[submatch(0)[0]], len(submatch(0)))', 'g')
  let src = substitute(src, '\%(\[\|\]\|\.\|,\)', '\=printf("{\"func\": \"%s\"# \"val\": 0}# ", inst_table[submatch(0)])', 'g')
  let src = substitute(src, '#', ',', 'g')
  let self.bytecode = eval('[' . src . ']')
endfunction

function! s:Brainfuck.run() abort dict
  let b = self.bytecode
  let l = len(self.bytecode)
  while self.pc < l
    call self[b[self.pc].func](b[self.pc].val)
    let self.pc += 1
  endwhile
endfunction

function! s:main() abort
  let start_time = reltime()
  call s:Brainfuck.compile(
        \   '+++++++++[>++++++++>+++++++++++>+++++<<<-]>.>++.+++++++..+++.>-.'
        \ . '------------.<++++++++.--------.+++.------.--------.>+.'
        \)
  call s:Brainfuck.run()
  echomsg '[Brainfuck] execute time:' reltimestr(reltime(start_time))
endfunction
call s:main()
```

これを実行すると，無事Hello, World!が表示される．

```sh
$ vs brainfuck.vim
Hello, world!
[Brainfuck] execute time:   0.002001
```

このシェルスクリプトとバッチファイルは，「Emacsユーザだけど，Vim scriptを書いて実行したい」という人の役に立つはずだ（？）
また，競技プログラミングやちょっとしたプログラミングの問題（例えば，少し前に話題になった[「ソフトウェアエンジニアならば1時間以内に解けなければいけない5つの問題」](http://www.softantenna.com/wp/software/5-programming-problems/)）などをVim scriptで実装し，実行するのも楽になるだろう（？？？）


## 参考

- [[Wandbox]三へ( へ՞ਊ ՞)へ ﾊｯﾊｯ](http://melpon.org/wandbox/)
- [Big Sky :: VimでFizzBuzz...いやBrainfuck](http://mattn.kaoriya.net/software/vim/20100311235228.htm)
