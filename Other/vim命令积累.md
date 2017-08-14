# VIM 命令积累

ctrl+f 下翻一屏

ctrl+b 上翻一屏

G 移动到最后一行

gg 移动到第一行

num G 移动到第 num 行

x 删除当前光标所在位置的字符

dw 删除当前光标所在位置的单词

d$ 删除当前光标所在位置至行尾的内容

J 删除当前光标所在行行位的换行符（拼接行）

A 在当前光标所在行行尾追加数据

r char 用 char 替换当前光标位置的单个字符

R text 用 text 替换当前光标位置的数据，直到按下 ESC 键

可视模式 v

vim 删除数据时将数据保存在一个单独寄存器中，可以用 p 取回数据。

y 复制

yw 复制单词

y$ 复制到行尾

/ text 查找 text，/ 继续查找同一个，n 下一个

:s/old/new 用 new 替换 old 第一次出现的地方

:s/old/new/g 一行命令替换所有 old

:n,ms/old/new/g 替换行号 n 和 m 之间所有 old

:%s/old/new/g 替换整个文件中所有 old

:%s/old/new/gc 替换整个文件所有 old，但在每次出现时出现提示
