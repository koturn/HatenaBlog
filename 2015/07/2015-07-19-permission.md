Vimで作成する新規ファイルのパーミッションを自動的に変更する
===========================================================

僕はWindowsを用いる場合，[KaoriYa](http://www.kaoriya.net/)のgvimと，Cygwinのvimを用いている．
そして，Cygwinのgitを利用している．
Cygwinであれば，新規に作成されるファイルのパーミッションは644であるが，Windowsだとパーミッションが700となるのが気にくわなかった．
というのも，gitはファイルのパーミッションも含めて管理するし，GitHub上では実行可能なテキストファイル等（例えばLICENSE）は，シェルスクリプトのシンタックスハイライトが適用されるからだ．

無意味な実行権限の付与は避けたいので，Windows環境でVimで新規ファイルを作成するときに，Cygwin等のchmodを呼び出し，ファイルのパーミッションを変更する処理を加えることにした．
ただし，ファイル先頭がshebang行であるならば，パーミッションを755にする．

```vim
function! s:system(cmd)
  try
    call vimproc#cmd#system(a:cmd)
  catch /^Vim\%((\a\+)\)\=:E\%(117\): .\+: vimproc#cmd#system$/
    call system(a:cmd)
  endtry
endfunction

if executable('chmod')
  augroup Permission
    autocmd!
  augroup END
  if has('win95') || has('win16') || has('win32') || has('win64')
    autocmd Permission BufNewFile *
          \ autocmd Permission BufWritePost <buffer>  call s:permission_644()
    function! s:permission_644()
      autocmd! Permission BufWritePost <buffer>
      silent! call s:system((getline(1)[0 : 1] ==# '#!' ? 'chmod 755 ' : 'chmod 644 ')
            \ . shellescape(expand('%')))
    endfunction
  else
    autocmd Permission BufNewFile *
          \ autocmd Permission BufWritePost <buffer>  call s:add_permission_x()
    function! s:add_permission_x()
      autocmd! Permission BufWritePost <buffer>
      if getline(1)[0 : 1] ==# '#!'
        silent! call s:system('chmod a+x ' . shellescape(expand('%')))
      endif
    endfunction
    autocmd MyAutoCmd BufWritePost * call s:add_permission_x()
  endif
endif
```

元ネタは，shebang付きファイルに実行権限を与える設定である．
以下の記事あたりが参考例だろう．

- [Vimで shebang 付ファイルを保存時に実行権限を自動で付加する - spiritlooseのはてなダイアリー](http://d.hatena.ne.jp/spiritloose/20060519/1147970872)
- [ぼくの.vimrcをさらしてみる - Studio3104::BLOG.new](http://studio3104.hatenablog.com/entry/20130220/1361349310)

しかし，既存のファイルの実行権限を勝手に変更してしまうのもどうかと思ったので（例えshebang付きであっても），「新規ファイル作成時のみ」そのパーミッションを変更するようにした．
仕組みとしては以下のようになる．

1. 存在しないファイルの編集を始めたとき（ ```BufNewFile``` が発火したとき）に，バッファローカルでバッファ全体をファイルに書き込んだ後に発火するイベント（ ```BufWritePost``` ）に対し，パーミッションを変更する関数を呼び出すアクションを設定する
2. パーミッションを変更する関数では，最初に ```BufWritePost``` に対して設定された自動コマンドを消去し，パーミッションを変更する外部コマンドを ```system()``` を通じて呼び出す．

```BufNewFile``` 時に ```BufWritePost``` の自動コマンドを設定しているのがポイントだ．
こうすることで，新規ファイルの保存時のみパーミッションを変更することができる．

ただし， ```writefile()``` 等でのファイルのパーミッションの変更には対応していない．
これらは，個別にスクリプト中やコマンドを実行して対応すべきだろう．
（[vital.vim](https://github.com/vim-jp/vital.vim)の ```:Vitalize``` をWindowsのVimでやると，実行権限が付いてしまうので，Cygwinのvimで ```:Vitalize``` しよう）


```s:system()``` については，組み込み関数の ```system()``` でも構わないが，Windowsだと一瞬コマンドプロンプトのウインドウが開くのが気になるので，[vimproc.vim](https://github.com/Shougo/vimproc.vim)がインストールされているならば，その ```system()``` を用いることにする．
```vimproc#system()``` と ```vimproc#cmd#system()``` の違いについては，vimproc.vimのソースコードを参照するか，以下を参照して欲しい．

- [Windows の Vim で system() を高速化してみた - C++でゲームプログラミング](http://d.hatena.ne.jp/osyo-manga/20130611/1370950114)
- [vimproc.vim に vimproc#cmd#system が実装された - C++でゲームプログラミング](http://d.hatena.ne.jp/osyo-manga/20130616/1371388900)


## 参考

- [Vimで shebang 付ファイルを保存時に実行権限を自動で付加する - spiritlooseのはてなダイアリー](http://d.hatena.ne.jp/spiritloose/20060519/1147970872)
- [ぼくの.vimrcをさらしてみる - Studio3104::BLOG.new](http://studio3104.hatenablog.com/entry/20130220/1361349310)
- [Windows の Vim で system() を高速化してみた - C++でゲームプログラミング](http://d.hatena.ne.jp/osyo-manga/20130611/1370950114)
- [vimproc.vim に vimproc#cmd#system が実装された - C++でゲームプログラミング](http://d.hatena.ne.jp/osyo-manga/20130616/1371388900)
