---
layout: post
tag: Language_Study
date: '\[2012-05-10 4 13:05\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Practical Common Lisp 学习笔记
---

绪言: 为什么是 Lisp
===================

为什么是 Lisp
-------------

Common Lisp 的描述: "`可编程的编程语言`", 意味着使用 CL 的时候,
不会出现语言里 刚好缺乏某些令程序更容易编写的特性.

Lisp 的诞生
-----------

CL 是由 McCarthy(2012驾鹤西去)发明的 Lisp 的现代版本.

曾经 Lisp 居然是系统编程语言, 主要在 Lisp 机上.

所以 Lisp 现在的情况是:

-   计算机领域的 `经典语言` 之一, 其中体现了z各种计算机科学的思想

-   完全是现代语言, 其设计尽可能的求解各种问题.

-   唯一的缺点是, 很多人对他的认识过于片面, 例如 Lisp 只能被解释执行
    很慢, 等等

周而复始: REPL 简介
===================

REPL: read-eval-print loop

使用构建在 Emacs 上的开发环境(Good), 见 [搭建 Lisp
开发环境](%20Build_Lisp_Development_Environment.org)

Hello World
-----------

创建名为 hello.lisp 的文件, 输入:

``` elisp
(defun hello-world ()
  (format t "Hello, world"))
```

加载一个文件:

-   可以用 转到文件, 用 C-c C-c(加载函数)
-   可以在 REPL 里面用 (load "filename")
-   或者用 compile-file 编译成字节码文件, 再用 load 加载该文件

```{=html}
<!-- -->
```
``` bash
(load (compile-file "filename"))
```

-   在源码文件利用 C-c C-l 加载

几个 tips:

C-c C-c
:   启动 slime-compile-defun 将当前定义发给 lisp 求值并编译.

C-c C-z
:   回到 REPL 模式

退出 lisp
:   在 REPL 中输入 ",", 在 minibuffer 中输入 quit

退出调试信息
:   当发生错误时, 会创建调试窗口并转到该 buffer, 按 q 回到 REPL

保存工作成果
------------

实践: 简单的数据库
==================

CD 和录入
---------

属性表:

``` elisp
(list :a 1 :b 2 :c 3)
(getf (list :a 1 :b 2 :c 3) :a) ; 获取符号的值
```

录入 CD
-------

查看数据库的内容
----------------

改进用户交互
------------

保存和加载数据库
----------------

查询数据库
----------

更新已有的记录: WHERE 再战江湖
------------------------------

消除重复, 获益良多
------------------

总结
----

语法和语义
==========

打开黑箱
--------

其它语言里的黑箱

:   一系列表示程序文本的字符被送进黑箱, 然后
    它执行预想行为(解释器)或者产生一个编译版本的程序并在运行时执行这些
    行为(编译器).

CL 里面的黑箱

:   CL 定义了两个黑箱, 一个将文本转化为 Lisp 对象, 而另一个运用这些对象
    实现语言的语义. 分别被称为读取器和求值器.

读取器定义了字符串如何被转化为名为 S- 表达式的 Lisp 对象, 求值器随后定义
了构建在 S- 表达式上的 Lisp 形式的语法.

S- 表达式
---------

S- 表达式的基本元素是 `列表` 和 `原子`. 列表由括号包围, 包含任意数量的由
空格分隔的元素. 原子是所有其他内容

常用的原子包括: 数字, 字符串, 名字

### 数字

任何数位的序列被当成数字

### 字符串

字符串由双引号包围, 反斜杠可以转义任意字符. 在字符串里必须被转移的字符是
'"' 和 '\\'.

### 名字

如 FORMAT, hello-world, **db** 等, 由称为符号的对象表示.

读取器不需要知道名字的用途, 只是简单的读取字符序列并构造此名代表的对象.

特征:

-   读取名字时, 将名字中所有未转义的字符转换成他们的大写形式, 也就是说,
    读取器不区分大小写.

例子:

``` example
X               ; 符号 X
()              ; 空列表
(1 2 3)         ; 三个数字组成的列表
("foo" "bar")   ; 两个字符串组成的列表
(x y z)         ; 三个符号组成的列表
(x 1 "foo")     ; 符号, 字符, 字符串组成的列表
(defun hello ()
(format t "hello")) ; 两个符号, 空列表, 以及另一个列表组成的列表
```

作为 Lisp 形式的 S- 表达式
--------------------------

任何原子(非列表或空列表)都是合法的 Lisp 形式, 但普通列表不一定是, 如:

``` example
(foo 1 2)   ; Lisp 形式
("foo" 1 2) ; 非 Lisp形式
```

### 原子求值

1.  符号求值

    符号被视为 Lisp 形式的时候被视为一个变量名, 并被求值为该变量的值.

2.  其他原子求值

    都是自求值对象, 简单的返回自身. 符号也可能是自求值对象, 所命名的对象
    可以被赋值成符号本身的值. 如 T 和 NIL(真值, 假值).

    另外, 关键字符号(以名字冒号开始的符号)也是自求值符号,
    读取器自定义一个 以此命名的常量值并以该符号作为其值.

### 列表求值

合法的列表形式均已一个符号开始.

有三种不同的列表形式, 相应的有三种不同的求值方式: 函数调用形式, 宏形式,
特殊形式

函数调用
--------

所有参数在调用前都会被求值

``` elisp
(function-name argument*)
(+ 1 2)             ; 先对1求值, 在对2求值, 传递到 + 函数里面, 得到3
(* (+ 1 2) (- 3 4)) ; 先对 (+ 1 2) 求值, 再对 (- 3 4) 求值
```

特殊操作符
----------

只有 25个, 第一个元素是特殊操作符所命名的符号,
其余部分按照该操作符的规则 进行求值. 例如:

``` elisp
(if test-form then-form [else-form])    ; if 
(quote (+ 1 2))                         ; 简单的返回, 并不求值, 等价于 '(+ 1 2)
```

宏
--

TODO: 和 C 宏的区别

以 S- 表达式为参数的函数, 返回一个 Lisp 形式.
然后对其求值并用该值取代宏形式.

宏的求值过程:

-   首先, 宏形式的元素不经过求值就被传到宏函数里.
-   其次, 由宏函数所返回的形式(展开式)按照正常的求值规则求值.

真, 假, 和等价
--------------

### 基本

符号 NIL 是唯一的假值, 其他都是真值, T 是标准的真值. nil 即是原子,
又是列表对象, 基于此, T 和 'T, NIL 和 'NIL, (), 和 '()
的求值结果都是相同的.

### 等价谓词

EQ

:   

EQL
:   两个对象相同时才相等

EQUAL

:   

EQUALP

:   

格式化 Lisp 代码
----------------

-   位于同一个嵌入层次的项应当按行对齐
-   主体元素相对于整个形式的开括号缩进两个空格
-   SLIME 中, 将光标放在一个开括号上输入 C-M-q 来缩进整个表达式, 或者在
    函数体内用 C-c M-q 来重新缩进整个函数体

函数
====

定义新函数
----------

``` elisp
(defun foo (parameter*)
  "Function ducomention string"
  body-form*)
```

可选参数
--------

``` elisp
;; 格式 a b 是必要形参, c d 是可选形参

(defun foo (a b &optional c d)
  body-form*)
(foo 1 2)           -> (1 2 NIL NIL)
(foo 1 2 3)         -> (1 2 3 NIL)
(foo 1 2 3 4)       -> (1 2 3 4)

;; 指定可选参数默认值
(defun foo (a &optional (b 10))
(foo 1 2)           -> (1 2)
(foo 1)             -> (1 10)

;; 基于其他参数(必要/可选)选择默认值, 可以用一个变量来获取变量是否
;; 使用的默认值
(defun foo (a &optional (b a b-supplied-p))
  (list a b b-supplied-p))
(foo 1)             -> (1 1 NIL)
(foo 1 2)           -> (1 2 T)
```

剩余形参
--------

``` elisp
(defun foo (a &rest values)
  (list a values))
(foo 1)             -> (1 NIL)
(foo 1 2 3)         -> (1 (2 3))
```

关键字形参
----------

可以调整形参顺序, 在任何必要的 &optional 和 &rest 后面, 加上 &key 即
任意数量的的关键字形参标识符.

``` elisp
(defun foo (a &key b c)
  (list a b c))
(foo 1 :b 2)        -> (1 2 NIL)

;; 复杂的形式
(defun foo (a &key b (c 6 b-supplied-p))
  (list a b c b-supplied-p))
(foo 1 :b 2)        -> (1 2 6 NIL)
(foo 1 :b 2 :c 5)   -> (1 2 5 T)
```

混合不同的形参类型
------------------

基本规则, 按照以下顺序声明参数: (必要形参 可选形参 剩余形参 关键字形参),
一般情况是必要形参和另外一种形参混合使用. 或者是组合的 &optional 和
&rest

函数返回值
----------

可以使用 return-from 立即返回

作为数据的函数-高阶函数
-----------------------

Lisp 中, 函数也是对象,

### 获取函数对象

``` elisp
(function function-name)
(function +) -> #<FUNCTION +>
;; 可用 #' 语法糖实现
#' + == (function +)
```

### 调用函数对象

-   funcall
-   apply 接受一个列表作为函数实参

```{=html}
<!-- -->
```
``` elisp
(apply #'function-name list)
;; 也可以传递孤立的实参
(apply #'function-name var list)
```

匿名函数
--------

lambda

变量
====

Lisp 支持两种类型的变量, 词法变量和动态变量

变量的基础知识
--------------

引入新变量的两种方式:

-   函数形参
-   使用 LET 操作符号

格式为:

``` elisp
;; variable* 是一个含有变量名和初始值的列表, 或者是一个简单的变量名
(let (variable*)
  body-form*)

(let ((a 1) (b 2) c)
  (print a)
  (print b)
  (print c))
```

let 执行结束后, 这些变量名将重新引用在执行 let 之前所引用的内容

let\* 的用法:

let\* 中的每一个变量的初始值形式, 可以引用到在变量列表早已引入的变量,
如:

``` elisp
;; y 可以直接引用 x
(let* ((x 10) (y (+ x 10)))
  (list x y))

;; 下面的是错误的
(let ((x 10) (y (+ x 10)))
  (list x y))
```

词法变量和闭包
--------------

### 此法作用域

默认情况下, lisp 中所有绑定形式都将引入词法作用域变量, 类似与 C
中的局部变量.

### 闭包

``` elisp
(defparameter *fn* (let ((count 0))
  #'(lambda () (setf count (+ 1 count)))))

;; 执行结果
(funcall *fn*) -> 1
(funcall *fn*) -> 2
(funcall *fn*) -> 3
```

上面的代码, 根据此法作用域的规则, lambda 里面对 count 的引用是合法的,
这个 匿名函数函数将被返回, 可能被不在 let 作用域内的代码访问,
这个匿名函数称为一个 闭包(封闭包装了 let 创建的绑定)

闭包的关键: 被捕捉的是绑定而不是变量的值.

动态变量
--------

Common Lisp 提供了两种创建全局变量的方式:

defvar
:   只有变量未定义时才赋值

defparameter
:   总是将初始值赋予命名的变量

格式:

``` elisp
(defvar *count* 0
  "Count document")
(defparameter *count* 0
  "Count document")
```

`特点` 一个给定的动态变量的每个新绑定都将被推到一个用于该变量的绑定栈中,
而对该变量的引用总是使用最近的绑定. 绑定形式返回时,
创建的绑定会自动从栈上弹出, 暴露前一个绑定.

常量
----

具有全局效果, 不能被用作函数形参或被任何其他绑定形式重新绑定

``` elisp
(defconstant name inital-value-form [documention-string])
```

赋值
----

setf 是宏, 自动检查赋值的 place 的形式, place 是变量时, 展开成对特殊操作
符 setq 的调用.

``` elisp
(setf PLACE VAL PLACE VAL ...)
;; 展开成对 (setq x 10)
(setf x 10)
```

广义赋值
--------

其他修改位置的方式
------------------

``` elisp
;; 以下三个是修改宏, 建立在 setf 上的宏
(incf x) === (setf x (+ x 1))
(decf x) === (setf x (- x 1))
(incf x 10) === (setf x (+ x 10))

;; 在位置间轮换他们的值
(rotatef a b)

;; 将值左移, 从后往前移, 第一个参数简单的被返回
(shiftf a b 10)
```

宏: 标准控制构造
================

when 和 unless
--------------

``` elisp
(if condition then-form [else-form])
;; IF 的扩展形式, 当 .. 时, 执行什么
(when condition body)
;; 条件为假时, 才求值
(unless condition body)
```

COND
----

主体中的每一个元素代表一个条件分支, 该分支由一个列表组成(如 (test-1
form\*)) 该列表含有一个条件形式, 以及0或者多个该分支将被求值的形式.

``` elisp
(cond
  (test-1 form*)
  .
  .
  .
  (test-N form*))

;; e.g.
(defun foo(p)
  (cond
    ((integerp p) (format t "Integer is a integer") '(1 2 3))
    ((stringp p) (format t "Argument is a string") '(4 5 6))))

(foo 1) -> Integer is a integer
           (1 2 3)
(foo "1") -> "Argument is a string"
           (4 5 6)
```

如何自定义宏
============

实践: 建立单元测试框架
======================

tmp
===

<http://blog.csdn.net/poechant/article/details/7423293>
<http://www.liyaos.com/blog/common-lisp-notes-0/>
