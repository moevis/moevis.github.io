---
layout: post
categories: cheatsheet
published: true 
---
开始写 python 了，jupyter notebook 虽然方便，但是没有语法检查，有时候脑残起来没下限，各种错。而用 spyder 又经常用出 bug 来，很气。

实际上 vscode 体验是挺好的，就是在 repl 上体验不佳，虽然可以整合 jupyter 的插件。所以我现在转回了 spacemacs，所以这时候记忆快捷键就很重要了（虽然可以随手查）。

## 窗口

移动和创建窗口都是以 w 开头：

- `SPC w h` 或者 `SPC w -` 横向窗口
- `SPC w v` 或者 `SPC w /` 垂直窗口

- `SPC w U/J/H/K` 将当前窗口移至最左/上/下/右
- `SPC w w` 切换窗口
- `SPC w c` 关闭窗口
- `SPC w =` 让窗口长宽协调

## 项目

项目命令以 p 开头

- `SPC p f` 打开/查找文件
- `SPC p '` 打开命令行
- `SPC p $t` 在根目录打开命令行
- `SPC p h` 最近编辑文件
- `SPC p t` 打开文件树
- `SPC p p/l` 切换项目

## 文件

文件命令以 f 开头

- `SPC f y` 显示当前文件路径并复制

## 注释

注释命令以 c 开头

- `SPC c l` 注释行
- `SPC c p` 注释一段

## emacs

emacs 标准键

- `C-c C-k` 关闭文件
- `[e` 将选中行向上移
- `]e` 将选中行向下移

## vim 常用

- visual-selection 模式下按 o 在选取两端切换光标
- visual-selection 模式下输入 :'<,>'sort 进行行排序
- visual-selection 模式下按 s' 对选区两端加单引号


- `v50G` 从当前开始选择到第 50 行
- `v6k` 从当前开始向下选择 6 行
- `vat` 选择 html tag
- `v/foo` 选择到下一个 foo 之前
- `v?foo` 选择到上一个 foo 之后
- `viw` 选中当前词
- `~` 转换大小写
- `A` 在行末插入
- `J` 将下一行接到本行末尾
- `.` 重复刚才的命令
- `ea` 在当前单词末尾插入
- `E` 停在下一个空白符前
- `ctrl-r` 重做
- `mx` 设定一个叫 x 的记号
- `'x` 或者 `\`x` 回到 x 记号位置或者回到 x 记号所在行
- `guu` 整行变小写
- `gUU` 整行变大写

## 国内源

添加下面的代码到 .spacemacs 的 dotspacemacs/user-init()

```
  (setq configuration-layer--elpa-archives
      '(("melpa-cn" . "http://elpa.emacs-china.org/melpa/")
        ("org-cn"   . "http://elpa.emacs-china.org/org/")
        ("gnu-cn"   . "http://elpa.emacs-china.org/gnu/")))
```

这样下载插件就快了。
