Vim scriptのinputlist()で&lt;Esc&gt;を検出する方法
==================================================

以前書いた記事：[Vim scriptのinput()で&lt;Esc&gt;を検出する方法](http://koturn.hatenablog.com/entry/2015/07/18/101510)にて， ```inputlist()``` で ```<Esc>``` を検知できない（数値0の入力と区別できない）と書いた．
しかし，改めて考えてみると， ```inputlist()``` を ```getchar()``` を用いて再実装すれば，簡単に ```<Esc>``` を検知できる ```inputlist()``` ができあがると考えた．

まずは，再実装した ```inputlist()``` を見ていただこう．
```<Esc>``` が入力されると数値-1を， ```<C-c>``` が入力されると数値-2を返すようにしてある．

```vim
function! s:show_candidates(list, str) abort
  echo "\n"
  for e in a:list
    echo e "\n"
  endfor
  " このメッセージをロケール依存にできればいいな
  echon 'Type number and <Enter> (empty cancels): '
  echon a:str
endfunction

function! s:inputlist(list) abort
  let keycode = {
        \ 'esc': char2nr("\<Esc>"),
        \ 'cr': char2nr("\<CR>"),
        \ '0': char2nr('0'),
        \ '9':  char2nr('9')
        \}
  call s:show_candidates(a:list, '')
  let nr = 0
  let ch = 0
  let str = ''
  " 再描画の際に -- More -- と表示されるのを防ぐ
  let save_more = &more
  set nomore
  try
    while ch != keycode.cr
      let ch = getchar()
      if ch == keycode.esc
        return -1
      elseif ch ==# "\<BS>" || ch ==# "\<DEL>"
        let len = strlen(str)
        let str = len > 1 ? str[: len - 2] : ''
        redraw
        call s:show_candidates(a:list, str)
      elseif keycode.0 <= ch && ch <= keycode.9
        let nr = nr2char(ch)
        let str .= nr
        echon nr
      endif
    endwhile
  catch /^Vim:Interrupt$/
    return -2
  finally
    redraw
    let &more = save_more
  endtry
  return str2nr(str)
endfunction
```

```inputlist()``` の基本的な仕様は以下の通りだ．

1. 引数のリストの要素を，ひとつひとう改行して表示し，最後に入力を促すメッセージ（ロケール依存）を表示する
2. 数字，マウスクリック， ```<CR>``` ， ```<BS>``` ， ```<Del>``` 以外の入力を全て受け付けない．カーソルキーでカーソルを移動することはできない．
3. 数字が入力されれば，エコーバックを行う．
4. マウスクリックを行うと，その選択候補のインデックスを返す．
5. ```<CR>```  が入力されると，入力された数字列を数値に変換する（オーバーフローあり）
6. ```<BS>``` ， ```<Del>``` を入力すると，入力された最後の数字を削除し，カーソルをひとつ左に戻す（ ```<BS>``` と ```<Del>``` の動作は同じ）．
7. 入力欄が改行されていた場合は， ```<BS>``` ， ```<Del>``` を用いても前の行に戻れず，前の行は表示されたままになる（バグ？）が，内部的には入力は削除されている．
8. ```<C-c>``` が入力されると，例外 ```'Vim:Interrupt'``` を排出する．

今回実装した  ```inputlist()``` を模倣した関数では，以下のことができない．

1. ロケールに応じた入力を促すメッセージの表示
2. マウスクリックの取得（ ```getchar()``` でマウスクリックは取得できるが，それは ```getchar()``` の入力ウィンドウ以外でのみ可能）
3. 入力欄を前の行に戻せないこと（これは，元の ```inputlist()``` より優れているはずだ）
4. チラつきなく入力数字列を ```<BS>``` ， ```<Del>``` を削除すること．

やはり完全な模倣はできなかったが，主要な使用は模倣しているので，大きな問題は無いだろう．
文字を消去した際に，チラつきが発生するのが一番の問題だが，うまい回避方法を思いつかなかった．
3番の入力欄を前の行に戻せない点については，ウィンドウの幅を取得して，うまくやれば再現できるだろうが， ```inputlist()``` の不便な部分をわざわざ再現することも無いと思う．

以下，動作比較をしたgifアニメだ．

- 通常の ```inputlist()```

[f:id:koturn:20150724063752g:plain]

- 模倣して実装した ```inputlist()```

[f:id:koturn:20150724063825g:plain]
