VimでC言語のソースコードを整形したい
====================================

VimでC/C++のソースコードを整形するには，[rhysd/vim-clang-format](https://github.com/rhysd/vim-clang-format)というVimプラグインを用いるのが一番良い．
[作者の紹介記事](http://rhysd.hatenablog.com/entry/2013/08/26/231858)からも，便利さが伝わるはずだ．

しかし，この記事では敢えてVimからindentコマンドを用いて，C言語のソースコードを整形することに挑戦しようと思う．
まずは.vimrcに書いた設定を披露しよう．
以下を.vimrcに記述し，C言語のソースコードで， ```:FormatCProgram``` とコマンドを実行すれば，indentコマンドにより，C言語のソースコードが整形されるはずだ．

```vim
if executable('indent')
  let s:indent_cmd = 'indent -orig -bad -bap -nbbb -nbbo -nbc -bli0 -br -brs -nbs
        \ -c8 -cbiSHIFTWIDTH -cd8 -cdb -cdw -ce -ciSHIFTWIDTH -cliSHIFTWIDTH -cp2 -cs
        \ -d0 -nbfda -nbfde -di0 -nfc1 -nfca -hnl -iSHIFTWIDTH -ipSHIFTWIDTH
        \ -nlp -lps -npcs -piSHIFTWIDTH -nprs -psl -saf -sai -saw -sbi0
        \ -sc -nsob -nss -tsSOFTTABSTOP -ppiSHIFTWIDTH -ip0 -l160 -lc160'
  function! s:format_c_program(has_bang) abort range
    let indent_cmd = substitute(s:indent_cmd, 'SHIFTWIDTH', &sw, 'g')
    let indent_cmd = substitute(indent_cmd, 'SOFTTABSTOP', &sts, 'g')
    let indent_cmd .= &expandtab ? ' -nut' : ' -ut'
    execute 'silent' a:firstline ',' a:lastline '!' indent_cmd
    if !v:shell_error || a:has_bang
      return
    endif
    " 以下，エラーが発生したとき用の処理
    let current_file = expand('%')
    if current_file ==# ''
      let current_file = '[No Name]'
    endif
    " カレントバッファにぶちまけられたエラーメッセージを取得
    let error_lines = filter(getline('1', '$'), 'v:val =~# "^indent: Standard input:\\d\\+: Error:"')
    " 'Standard input'を現在のファイル名に置き換えたり，範囲指定している場合のために，行番号を置き換えたりする
    let error_lines = map(error_lines, 'substitute(v:val, "^indent: \\zsStandard input:\\(\\d\\+\\)\\ze: Error:", "\\=current_file . \":\" . (submatch(1) + a:firstline - 1)", "")')
    let winheight = len(error_lines) > 10 ? 10 : len(error_lines)
    " カレントバッファをエラーメッセージがぶちまけられる前の状態に戻す
    undo
    " カレントバッファの下に新たにウィンドウを作り，エラーメッセージを表示するバッファを作成する
    execute 'botright' winheight 'split [INDENT_ERROR]'
    setlocal nobuflisted bufhidden=unload buftype=nofile
    call setline(1, error_lines)
    " エラー表示用のundo履歴を消去する（誤ってundoでエラー情報を消去しないため）
    let save_undolevels = &l:undolevels
    setlocal undolevels=-1
    execute "normal! a \<BS>\<Esc>"
    setlocal nomodified
    let &l:undolevels = save_undolevels
    " エラーメッセージ用バッファは読み取り専用にしておく
    setlocal readonly
  endfunction
  augroup CFormat
    autocmd!
    autocmd FileType c,cpp
          \ command! -bar -bang -range=% -buffer FormatCProgram
          \ <line1>,<line2>call s:format_c_program(<bang>0)
  augroup END
endif
```

[f:id:koturn:20150726070155g:plain]

まず最初に，indentコマンドが利用可能かどうかを確認している．
ほとんどのLinux環境では利用可能だろうが，Windowsだと標準では利用できないためである．

```s:indent_cmd``` には，indentコマンドとそのオプションの文字列を納めている．
このindentコマンドのオプション設定は個人の好みに応じて変更するとよい．
上記のように1つ1つ設定しなくても，コーディングスタイルを指定するオプション ```-gnu``` ， ```-orig``` ， ```-kr``` のいずれかを設定し，気にいらない部分だけ設定し直すと良い．
ところどころ出現する ```SHIFTWIDTH``` と ```SOFTTABSTOP``` はそれぞれ，Vimのオプション ```shiftwidth``` と ```softtabstop``` の値に置換する．

コマンドを実行し，indentコマンドがエラーを吐いた場合，ソースコードを元に戻し，下にウィンドウを開き，indentコマンドのエラーを表示するようにした．

また， ```:1,22FormatCProgram``` やビジュアルモードで範囲を選択して実行，などのように範囲を指定すれば，その範囲でのみindentコマンドを実行できる．
ただし，中途半端な範囲を指定してコマンドを実行すると，indentコマンドがエラーを吐くので，関数単位で行うなどすべきだ．

あとはオートコマンドで， ```filetype``` が ```c``` ，もしくは ```cpp``` である場合のみ，利用できるようにした．

実は当初，オプション ```equalprg``` にindentコマンドを指定する方法にしようかと思っていたのだが，indentコマンドは文法に誤りがあればエラーを吐き，カレントバッファにエラーの文字列をぶちまけてしまう．
つまり，中途半端な範囲のみでインデントすると，エラーとなってしまうのだ．
これは，デフォルトの ```==``` 等の動作と比べると，小回りが効かず，使いづらいと感じた．
よって，コマンドとして定義することにしたわけである．

最後にもう一度言うが，今回の記事では「敢えて」indentコマンドを利用するVim scriptを書いたわけだ．
現代のC/C++を書く人間ならば，Clangを導入し，clang-formatを利用するプラグイン：[rhysd/vim-clang-format](https://github.com/rhysd/vim-clang-format)を用いて，コード整形を行うべきだ．


## 参考

- [C や C++ のコードを自動で整形する clang-format を Vim で - sorry, uninuplemented:](http://rhysd.hatenablog.com/entry/2013/08/26/231858)
