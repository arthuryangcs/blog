---
title: Vim 实用技巧（一）
date: 2016-12-03 11:06:25
tags: 
  - Vim
  - 技巧
categories: Vim
---

## 普通模式转插入模式

命令 | 功能
-- | --
i | 光标左侧插入
a | 光标右侧插入
o | 光标下方插入新行后插入
O | 光标上方插入新行后插入
I | 当前行第一个非空字符左侧插入
A | 当前行行尾插入
s | 删除光标所在字符后插入
C | 删除光标到行尾字符后插入
S | 删除当前行所有字符后插入

<!-- more -->

## 重复、回退

目的 | 操作 | 重复 | 回退
-- | -- | -- | --
做出一个修改 | {edit} | . | u
在行内查找下一指定字符 | f{char}/t{char} | ; | ,
在行内查找上一指定字符 | F{char}/T{char} | ; | ,
在文档中查找下一处匹配项 | /pattern<CR> | n | N
在文档中查找上一处匹配项 | ?pattern<CR> | n | N
执行替换 | :s/target/replacement | & | u
执行一系列修改 | qx{changes}q | @x | u 

## 插入模式下的删除操作

命令 | 功能
-- | --
&lt;C-h&gt; | 删除前一个字符（同退格键）
&lt;C-w&gt; | 删除前一个单词
&lt;C-u&gt; | 删至行首