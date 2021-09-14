# 1. Basic Editing - 基本编辑

在这一章，可以学到：

- 4个基本的移动命令（commands）
- 如何插入（<u>i</u>nsert）和删除（<u>d</u>elete）文本
- 如何获取帮助（:help）信息
- 退出编辑器

## 1.1 Before You Start - 开始之前

如果是在 UNIX 环境下运行，事先执行下面命令：

```
touch ~/.vimrc
```

通过创建`~/.vimrc`，是在告诉 Vim 你需要以 Vim 模式使用它。如果该文件不存在，Vim 会以 VI 兼容模式启动，这样你将不能访问许多 Vim 的高级特性。

你可以在任意时刻启用 Vim 的高级特性：`:set nocompatible<ENTER>`。

## 1.2 Running Vim for the First Time - 初次运行 Vim

```
vim file.txt
```

你将会得到一个基于你执行 Vim 命令的环境的空白窗口。

窗口中的`~`行指出不在文件中的行。

### Modes - 模式

Vim 是一个模态编辑器。所处模式不同，编辑器的行为有所不同。

- normal mode，正常模式

  屏幕下方显示文件名或空白

- insert mode，插入模式

  指示器（屏幕下方）显示"--INSERT--"

- visual mode，可视模式

  指示器显示"--VISUAL--"

- command mode，命令模式

  正常模式下，输入`:`进入

## 1.3 Editing for the First Time - 初次编辑

### Inserting Text - 插入文本

输入`i`进入插入模式，屏幕坐下角变为"--INSERT--"。

完成插入后，按<ESC>键，"--INSERT--"指示器消失并回到正常模式。

### Getting Out of Trouble - 摆脱困扰

Vim 初学者会遇到的一个问题是"模式混淆"。原于忘记了所处的模式，或者突然输入了某个命令导致模式切换。记住，不管你所处什么模式，按<ESC>键都可以回到正常模式。

### Moving Around - 移动

正常模式下：

- h/LA(left)
- j/DA(down)
- k/UA(up)
- l/RA(right)

使用 h,j,k,l 代替箭头键来移动更高效。

### Deleting Characters - 删除字符

删除字符，将光标移动到其上方，然后输入`x`。

### Undo and Redo - 撤销与重做

正常模式下:

- 输入`u`可撤销之前的文本编辑.
- `CTRL-R`恢复之前的操作。
- 输入`U`撤销上一编辑行的所有改变，再输入`U`撤销之前的`U`操作。

### Getting Out - 退出

正常模式下，使用`ZZ`命令，写文件并退出 Vim。

### Discarding Changes - 丢弃变更

Vim 有一个"quit-and-throw-things-away"命令，`:q!`:

- `:`，进入命令模式
- `q`命令，告诉编辑器退出
- `!`，覆盖命令修饰符，强制执行退出命令

## 1.4 Other Editing Commands - 其它编辑命令

### Inserting Characters at the End of a Line - 行尾插入字符

正常模式下，输入`a`，可在光标之后插入文本。

### Deleting A Line - 删除一行

使用`dd`命令，可以删除光标所在位置的一行。

### Opening Up New Lines - 开启新的一行

正常模式下：

- 使用`o`命令，在光标所在位置下方开启新的一行
- 使用`O`命令，在光标所在位置上方开启新的一行

### Help - 帮助

使用命令模式命令：`:help`，显示综合帮助信息窗口。Vim 的创造者将 help 窗口设计成了一个普通的编辑窗口。你可以使用所有的 normal 的 vim 命令来浏览帮助信息。

#### 标记/超链接

在帮助文本中，所有被"|"包围着的文本，例如`|:help|`，都表示一个超链接。将光标移动到这样的文本上，然后按下`CTRL+]`（jump to tag），帮助系统会带你去到指向的目标。（在 Vim 中，超链接的术语是"tag"（标记））

`CTRL+T`（pop tag）命令带你回到之前的位置，用 Vim 的术语来说，这叫“从标记堆栈里取出一个标记“。

#### 获取帮助信息

- `:help subject`，获取指定主题的帮助信息。

- `:help x`， 获取命令"x"的帮助信息。
- `:help CTRL-X`，获取控制字符的帮助信息。例如，`CTRL-H`。
- `:help -x`，获取命令行选项的帮助信息。
- `:help 'option'`，获取设置选项的帮助信息。
- `:help <KEY>`，获取特殊按键的帮助信息。例如`:help <Up>`。

#### 模式前缀

默认情况下，help 系统显示正常模式命令的帮助信息。例如`:help CTRL-H`。要指定其它模式，需要使用模式前缀`:helo m_X`，例如插入模式下的`:help i_CTRL_H`。

*模式前缀表*

------

| **What**              | **Prefix**   | **Example**        |
| --------------------- | ------------ | ------------------ |
| Normal-mode commands  | (nothing)    | :help x            |
| Insert-mode commands  | i            | :help i_<Esc>      |
| Visual-mode commands  | v            | :help v_u          |
| Control character     | CTRL-        | :help CTRL-u       |
| Command-ling commands | :            | :help :quit        |
| Command-line editing  | c            | :help c_<Del>      |
| Vim Command arguments | -            | :help -r           |
| Options               | '(both ends) | :help 'text width' |
| Regular Expression    | /            | :help /[           |

### Other Ways to Get Help - 获取帮助的其它方法

- 按<F1>键打开综合帮助窗口。
- 按<Help>（如果有）。

## 1.5 Using a Count to Edit Faster - 使用数字来更快的编辑

可以在所有移动命令前加上一个数字。例如，`9k`可以代替`kkkkkkkkk`，向上移动9行。

也可以用在插入命令前，例如`3a!<Esc>`，代替`a!!!<Esc>`。

也可以用在删除命令前，例如`3x`，代替`xxx`。

## 1.6 The Vim Tutorial - Vim 教程

UNIX 版的 Vim 附带了一个交互式的教程。通过命令`$vimtutor`启动。

如果不是 UNIX 类的系统，查看帮助信息`:help tutor`来获取教程的启动方式。

# 2. Editing a Little Faster - 编辑快一点

本章覆盖了一些额外的命令，可以使你编辑的更快一些，包括：

- 额外的移动命令
- 在单行内快速查找
- 额外的删除和更改命令
- 重复命令
- 键盘宏（记录和重放命令）
- 有向图（Digraphs）