# helix

## config.toml

**~/.config/helix/config.toml**:
```
[keys.normal]
"C-c" = "no_op"
"C-f" = ["collapse_selection", "keep_primary_selection"]
"C-v" = "vsplit"
"C-n" = "rotate_view"

[keys.insert]
"C-f" = "normal_mode"

[keys.select]
"C-c" = "no_op"
"C-f" = ["collapse_selection", "keep_primary_selection", "normal_mode"]
"C-v" = "vsplit"
"C-n" = "rotate_view"

[editor]
whitespace.render = "all"
smart-tab.enable = false

[editor.whitespace.characters]
space = "․"
tab = "→"
newline = "↵"
```

- Ctrl+f で操作をリセットしつつノーマルモードに戻す
- Ctrl+v で縦分割
- Ctrl+n で分割したウィンドウを切り替え
- スペース、タブ、改行を表示する
    - tmux を使ってると文字が潰れていた。`export LANG=ja_JP,UTF-8` が必要だった。 
- スマートタブを無効

## ファイルによってスペースの幅を変える

のはそのうちやる
