Vimプラグインの動作確認をしたい
===============================


Vimプラグインを書いていると，新規にVimを起動して，動作を確認したいことがある．
そこで，c0hamaさんの[この記事](http://qiita.com/c0hama/items/4ab505ddebdcfd842e25)を参考にして，.vimrcに

```vim
function! s:plugin_test(use_gvim, ex_command) abort
  let cmd = a:use_gvim ? 'gvim' : 'vim'
  let ex_command = empty(a:ex_command) ? '' : ' -c "au VimEnter * ' . a:ex_command . '"'
  execute '!' . cmd . ' -u ~/.vim/.min.vimrc -U ~/.vim/.min.gvimrc -i NONE -n -N --cmd "set rtp+=' . getcwd() . '"' . ex_command
endfunction
command! -bar -bang -nargs=* PluginTest  call s:plugin_test(<bang>0, <q-args>)
```

と書き， ```~/.vim/.min.vimrc``` に，最低限の設定として，

```vim
syntax on
filetype plugin indent on
```

と記述していた．
（ ```~/.vim/.min.gvimrc``` は，フォントサイズやカラースキーム等の見た目の設定のみ）

後は記事通りにプラグインのルートディレクトリ（plugin/やautoload/があるディレクトリ）で， ```:PluginTest``` とするだけだ．

ただし，このままだと，Windowsでの動作確認がやや不便（gvimの立ち上げで，空のコマンドプロンプトウィンドウが表示されてしまう）なので，僕は以下のように変更した．

```vim
function! s:plugin_test(use_gvim, ex_command) abort
  let cmd = escape((a:use_gvim ? 'gvim' : 'vim')
        \ . ' -u ~/.vim/.min.vimrc'
        \ . ' -U ~/.vim/.min.gvimrc'
        \ . ' -i NONE'
        \ . ' -n'
        \ . ' -N'
        \ . printf(' --cmd "set rtp+=%s"', getcwd())
        \ . (empty(a:ex_command) ? '' : printf('-c "au VimEnter * %s"', a:ex_command)), '!')
  if has('win95') || has('win16') || has('win32') || has('win64')
    if a:use_gvim
      execute 'silent !start' cmd
    else
      if has('gui_running')
        execute 'silent !start cmd /c call' cmd
      else
        execute 'silent !' cmd
      endif
    endif
  else
    execute 'silent !' cmd
    redraw!
  endif
endfunction
command! -bar -bang -nargs=* PluginTest  call s:plugin_test(<bang>0, <q-args>)
```

Windowsでは， ```!start COMMAND``` とすることで，空のコマンドプロンプトの立ち上がりを防ぐことができる．（ ```:h :!start*``` 参照）
また，CUIのvimからCUIのvimを立ち上げるときは，Windows以外の環境での動作と合わせて，新規コマンドプロンプトの立ち上げはせず，同一のウィンドウ内で立ち上げることにした．

Windows以外の環境であれば，gvimよりVimを利用する場合がほとんどなので，gvimについては考慮していない．
ただ，僕の環境では，起動したVimを終了したときに，元のVimの再描画が行われないという問題があったので， ```redraw!``` を追加した．

```escape(cmd, '!')``` のように ``` '!'``` をエスケープしているのは， ```!``` で外部コマンドを実行する際，外部コマンドの引数に ```!``` を渡すにはエスケープが必要なためだ．


## 参考

- [作成中の Vim プラグインの動作確認をしたいときのためのの便利コマンド - Qiita](http://qiita.com/c0hama/items/4ab505ddebdcfd842e25)
