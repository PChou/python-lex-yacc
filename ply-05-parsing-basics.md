## 5 语法分析基础

yacc.py 用来对语言进行语法分析。在给出例子之前，必须提一些重要的背景知识。首先，‘语法’通常用 BNF 范式来表达。例如，如果想要分析简单的算术表达式，你应该首先写下无二义的文法：

```
expression : expression + term
           | expression - term
           | term

term       : term * factor
           | term / factor
           | factor

factor     : NUMBER
           | ( expression )
```
在这个文法中，像`NUMBER`,`+`,`-`,`*`,`/`的符号被称为终结符，对应原始的输入。类似`term`，`factor`等称为非终结符，它们由一系列终结符或其他规则的符号组成，用来指代语法规则。

通常使用一种叫语法制导翻译的技术来指定某种语言的语义。在语法制导翻译中，符号及其属性出现在每个语法规则后面的动作中。每当一个语法被识别，动作就能够描述需要做什么。比如，对于上面给定的文法，想要实现一个简单的计算器，应该写成下面这样：

```
Grammar                             Action
--------------------------------    -------------------------------------------- 
expression0 : expression1 + term    expression0.val = expression1.val + term.val
            | expression1 - term    expression0.val = expression1.val - term.val
            | term                  expression0.val = term.val

term0       : term1 * factor        term0.val = term1.val * factor.val
            | term1 / factor        term0.val = term1.val / factor.val
            | factor                term0.val = factor.val

factor      : NUMBER                factor.val = int(NUMBER.lexval)
            | ( expression )        factor.val = expression.val
```

一种理解语法指导翻译的好方法是将符号看成对象。与符号相关的值代表了符号的“状态”（比如上面的 val 属性），语义行为用一组操作符号及符号值的函数或者方法来表达。

Yacc 用的分析技术是著名的 LR 分析法或者叫移进-归约分析法。LR 分析法是一种自下而上的技术：首先尝试识别右部的语法规则，每当右部得到满足，相应的行为代码将被触发执行，当前右边的语法符号将被替换为左边的语法符号。（归约）

LR 分析法一般这样实现：将下一个符号进栈，然后结合栈顶的符号和后继符号（译者注：下一个将要输入符号），与文法中的某种规则相比较。具体的算法可以在编译器的手册中查到，下面的例子展现了如果通过上面定义的文法，来分析 3 + 5 * ( 10 - 20 ) 这个表达式，$ 用来表示输入结束

```
Step Symbol Stack           Input Tokens            Action
---- ---------------------  ---------------------   -------------------------------
1                           3 + 5 * ( 10 - 20 )$    Shift 3
2    3                        + 5 * ( 10 - 20 )$    Reduce factor : NUMBER
3    factor                   + 5 * ( 10 - 20 )$    Reduce term   : factor
4    term                     + 5 * ( 10 - 20 )$    Reduce expr : term
5    expr                     + 5 * ( 10 - 20 )$    Shift +
6    expr +                     5 * ( 10 - 20 )$    Shift 5
7    expr + 5                     * ( 10 - 20 )$    Reduce factor : NUMBER
8    expr + factor                * ( 10 - 20 )$    Reduce term   : factor
9    expr + term                  * ( 10 - 20 )$    Shift *
10   expr + term *                  ( 10 - 20 )$    Shift (
11   expr + term * (                  10 - 20 )$    Shift 10
12   expr + term * ( 10                  - 20 )$    Reduce factor : NUMBER
13   expr + term * ( factor              - 20 )$    Reduce term : factor
14   expr + term * ( term                - 20 )$    Reduce expr : term
15   expr + term * ( expr                - 20 )$    Shift -
16   expr + term * ( expr -                20 )$    Shift 20
17   expr + term * ( expr - 20                )$    Reduce factor : NUMBER
18   expr + term * ( expr - factor            )$    Reduce term : factor
19   expr + term * ( expr - term              )$    Reduce expr : expr - term
20   expr + term * ( expr                     )$    Shift )
21   expr + term * ( expr )                    $    Reduce factor : (expr)
22   expr + term * factor                      $    Reduce term : term * factor
23   expr + term                               $    Reduce expr : expr + term
24   expr                                      $    Reduce expr
25                                             $    Success!
```

（译者注：action 里面的 Shift 就是进栈动作，简称移进；Reduce 是归约）

在分析表达式的过程中，一个相关的自动状态机和后继符号决定了下一步应该做什么。如果下一个标记看起来是一个有效语法（产生式）的一部分（通过栈上的其他项判断这一点），那么这个标记应该进栈。如果栈顶的项可以组成一个完整的右部语法规则，一般就可以进行“归约”，用产生式左边的符号代替这一组符号。当归约发生时，相应的行为动作就会执行。如果输入标记既不能移进也不能归约的话，就会发生语法错误，分析器必须进行相应的错误恢复。分析器直到栈空并且没有另外的输入标记时，才算成功。
需要注意的是，这是基于一个有限自动机实现的，有限自动器被转化成分析表。分析表的构建比较复杂，超出了本文的讨论范围。不过，这构建过程的微妙细节能够解释为什么在上面的例子中，解析器选择在步骤 9 将标记转移到堆栈中，而不是按照规则 expr : expr + term 做归约。

