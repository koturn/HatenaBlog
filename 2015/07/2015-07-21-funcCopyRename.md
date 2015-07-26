Vim scriptにおける関数のコピーと削除
====================================

実用的ではない小ネタを紹介しようと思う．
それは，Vim Scriptにおける関数のコピーとリネームである．

Vimでは， ```function [関数名]``` というコマンドを実行することで，関数の定義を行番号付きで見ることができる．
従って，これを ```redir``` すれば関数定義を文字列として得られるというわけだ．
そこから，邪魔な行番号を除去すればよい．

```vim
function! s:redir(cmd) abort
  let [verbose, verbosefile] = [&verbose, &verbosefile]
  set verbose=0 verbosefile=
  redir => str
  execute 'silent!' a:cmd
  redir END
  let [&verbose, &verbosefile] = [verbose, verbosefile]
  return str
endfunction

function! s:get_funcdef(funcname, ...) abort
  let list_state = &l:list
  setlocal nolist
  let is_bang = a:0 > 0 ? a:1 : 1
  let funcdef = s:redir('function ' . a:funcname)
  if is_bang
    let funcdef = substitute(funcdef, 'function', 'function!', '')
  endif
  let funcdef = join(map(split(funcdef, "\n"), 'substitute(v:val, "^\\%(\\d\\{3,}\\|[0-9 ]\\{3}\\)", "", "")'), "\n")
  let &l:list = list_state
  return funcdef
endfunction

" これで関数定義（の文字列）が得られる
let s:funcdef = s:get_funcdef('s:get_funcdef')
" 関数定義を表示
function s:funcdef
" 関数を消去しても...
delfunction s:get_funcdef
" 再び定義することが可能
execute s:funcdef
```

途中の正規表現は，行番号を消去し，元々定義された通りのインデントにするためのものだ．（別にインデントも消去しても良いかもしれない）
また， ```setlocal nolist``` とオプションをセットしているのは， ```redir``` でlistcharsに設定した行末文字ごと取得していたためである．

さて，関数定義を文字列として取得できたのならば，関数のコピーやリネームはたやすい．

```vim
function! s:copy_function(funcname, newname) abort
  execute substitute(s:get_funcdef(a:funcname), 'function! \zs\(.\{-}\)\ze(', a:newname, '')
endfunction

function! s:rename_function(funcname, newname) abort
  call s:copy_function(a:funcname, a:newname)
  execute 'delfunction' a:funcname
endfunction
```

あとは，適当なコマンドを定義し，

```vim
command! -bar -nargs=+ -complete=function CopyFunction  call s:copy_function(<f-args>)
command! -bar -nargs=+ -complete=function RenameFunction  call s:rename_function(<f-args>)
```

以下のようにコマンドを実行すればよいだけだ．

```vim
CopyFunction s:copy_function A
RenameFunction s:rename_function B
```

関数定義内のコメントやインデント等を削除すれば，しょっぱいパフォーマンスの向上ができるかもしれない...．
