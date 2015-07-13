## 6 Yacc

ply.yacc 模块实现了 PLY 的分析功能，‘yacc’是‘Yet Another Compiler Compiler’的缩写并保留了其作为 Unix 工具的名字。

### 6.1 一个例子

假设你希望实现上面的简单算术表达式的语法分析，代码如下：

```
# Yacc example

import ply.yacc as yacc

# Get the token map from the lexer.  This is required.
from calclex import tokens

def p_expression_plus(p):
    'expression : expression PLUS term'
    p[0] = p[1] + p[3]

def p_expression_minus(p):
    'expression : expression MINUS term'
    p[0] = p[1] - p[3]

def p_expression_term(p):
    'expression : term'
    p[0] = p[1]

def p_term_times(p):
    'term : term TIMES factor'
    p[0] = p[1] * p[3]

def p_term_div(p):
    'term : term DIVIDE factor'
    p[0] = p[1] / p[3]

def p_term_factor(p):
    'term : factor'
    p[0] = p[1]

def p_factor_num(p):
    'factor : NUMBER'
    p[0] = p[1]

def p_factor_expr(p):
    'factor : LPAREN expression RPAREN'
    p[0] = p[2]

# Error rule for syntax errors
def p_error(p):
    print "Syntax error in input!"

# Build the parser
parser = yacc.yacc()

while True:
   try:
       s = raw_input('calc > ')
   except EOFError:
       break
   if not s: continue
   result = parser.parse(s)
   print result
```

在这个例子中，每个语法规则被定义成一个 Python 的方法，方法的文档字符串描述了相应的上下文无关文法，方法的语句实现了对应规则的语义行为。每个方法接受一个单独的 p 参数，p 是一个包含有当前匹配语法的符号的序列，p[i] 与语法符号的对应关系如下：

```
def p_expression_plus(p):
    'expression : expression PLUS term'
    #   ^            ^        ^    ^
    #  p[0]         p[1]     p[2] p[3]

    p[0] = p[1] + p[3]
```

其中，p[i] 的值相当于词法分析模块中对 p.value 属性赋的值，对于非终结符的值，将在归约时由 p[0] 的赋值决定，这里的值可以是任何类型，当然，大多数情况下只是 Python 的简单类型、元组或者类的实例。在这个例子中，我们依赖这样一个事实：NUMBER 标记的值保存的是整型值，所有规则的行为都是得到这些整型值的算术运算结果，并传递结果。

> 注意：在这里负数的下标有特殊意义--这里的 p[-1] 不等同于 p[3]。详见下面的嵌入式动作部分

在 yacc 中定义的第一个语法规则被默认为起始规则（这个例子中的第一个出现的 expression 规则）。一旦起始规则被分析器归约，而且再无其他输入，分析器终止，最后的值将返回（这个值将是起始规则的p[0]）。注意：也可以通过在 yacc() 中使用 start 关键字参数来指定起始规则

p_error(p) 规则用于捕获语法错误。详见处理语法错误部分

为了构建分析器，需要调用 yacc.yacc() 方法。这个方法查看整个当前模块，然后试图根据你提供的文法构建 LR 分析表。第一次执行 yacc.yacc()，你会得到如下输出：

```
$ python calcparse.py
Generating LALR tables
calc >
```

由于分析表的得出相对开销较大（尤其包含大量的语法的情况下），分析表被写入当前目录的一个叫 parsetab.py 的文件中。除此之外，会生成一个调试文件 parser.out。在接下来的执行中，yacc 直到发现文法发生变化，才会重新生成分析表和 parsetab.py 文件，否则 yacc 会从 parsetab.py 中加载分析表。注：如果有必要的话这里输出的文件名是可以改的。

如果在你的文法中有任何错误的话，yacc.py 会产生调试信息，而且可能抛出异常。一些可以被检测到的错误如下：

- 方法重复定义（在语法文件中具有相同名字的方法）
- 二义文法产生的移进-归约和归约-归约冲突
- 指定了错误的文法
- 不可终止的递归（规则永远无法终结）
- 未使用的规则或标记
- 未定义的规则或标记

下面几个部分将更详细的讨论语法规则

这个例子的最后部分展示了如何执行由 yacc() 方法创建的分析器。你只需要简单的调用 parse()，并将输入字符串作为参数就能运行分析器。它将运行所有的语法规则，并返回整个分析的结果，这个结果就是在起始规则中赋给 p[0] 的值。

 
 ### 6.2 将语法规则合并

如果语法规则类似的话，可以合并到一个方法中。例如，考虑前面例子中的两个规则：

```
def p_expression_plus(p):
    'expression : expression PLUS term'
    p[0] = p[1] + p[3]

def p_expression_minus(t):
    'expression : expression MINUS term'
    p[0] = p[1] - p[3]
```


比起写两个方法，你可以像下面这样写在一个方法里面：

```
def p_expression(p):
    '''expression : expression PLUS term
                  | expression MINUS term'''
    if p[2] == '+':
        p[0] = p[1] + p[3]
    elif p[2] == '-':
        p[0] = p[1] - p[3]
```

总之，方法的文档字符串可以包含多个语法规则。所以，像这样写也是合法的（尽管可能会引起困惑）：

```
def p_binary_operators(p):
    '''expression : expression PLUS term
                  | expression MINUS term
       term       : term TIMES factor
                  | term DIVIDE factor'''
    if p[2] == '+':
        p[0] = p[1] + p[3]
    elif p[2] == '-':
        p[0] = p[1] - p[3]
    elif p[2] == '*':
        p[0] = p[1] * p[3]
    elif p[2] == '/':
        p[0] = p[1] / p[3]
```

如果所有的规则都有相似的结构，那么将语法规则合并才是个不错的注意（比如，产生式的项数相同）。不然，语义动作可能会变得复杂。不过，简单情况下，可以使用`len()`方法区分，比如：

```
def p_expressions(p):
    '''expression : expression MINUS expression
                  | MINUS expression'''
    if (len(p) == 4):
        p[0] = p[1] - p[3]
    elif (len(p) == 3):
        p[0] = -p[2]
```

如果考虑解析的性能，你应该避免像这些例子一样在一个语法规则里面用很多条件来处理。因为，每次检查当前究竟匹配的是哪个语法规则的时候，实际上重复做了分析器已经做过的事（分析器已经准确的知道哪个规则被匹配了）。为每个规则定义单独的方法，可以消除这点开销。

 
### 6.3 字面字符

如果愿意，可以在语法规则里面使用单个的字面字符，例如：

```
def p_binary_operators(p):
    '''expression : expression '+' term
                  | expression '-' term
       term       : term '*' factor
                  | term '/' factor'''
    if p[2] == '+':
        p[0] = p[1] + p[3]
    elif p[2] == '-':
        p[0] = p[1] - p[3]
    elif p[2] == '*':
        p[0] = p[1] * p[3]
    elif p[2] == '/':
        p[0] = p[1] / p[3]
```

字符必须像'+'那样使用单引号。除此之外，需要将用到的字符定义单独定义在 lex 文件的`literals`列表里：

```
# Literals.  Should be placed in module given to lex()
literals = ['+','-','*','/' ]
```

字面的字符只能是单个字符。因此，像'<='或者'=='都是不合法的，只能使用一般的词法规则（例如 t_EQ = r'==')。


### 6.4 空产生式

yacc.py 可以处理空产生式，像下面这样做：

```
def p_empty(p):
    'empty :'
    pass
```

现在可以使用空匹配，只要将'empty'当成一个符号使用：

```
def p_optitem(p):
    'optitem : item'
    '        | empty'
    ...
```

注意：你可以将产生式保持'空'，来表示空匹配。然而，我发现用一个'empty'规则并用其来替代'空'，更容易表达意图，并有较好的可读性。

 
### 6.5 改变起始符号

默认情况下，在 yacc 中的第一条规则是起始语法规则（顶层规则）。可以用 start 标识来改变这种行为：

```
start = 'foo'

def p_bar(p):
    'bar : A B'

# This is the starting rule due to the start specifier above
def p_foo(p):
    'foo : bar X'
...
```

用 start 标识有助于在调试的时候将大型的语法规则分成小部分来分析。也可把 start 符号作为yacc的参数：

```
yacc.yacc(start='foo')
```


### 6.6 处理二义文法

上面例子中，对表达式的文法描述用一种特别的形式规避了二义文法。然而，在很多情况下，这样的特殊文法很难写，或者很别扭。一个更为自然和舒服的语法表达应该是这样的：

```
expression : expression PLUS expression
           | expression MINUS expression
           | expression TIMES expression
           | expression DIVIDE expression
           | LPAREN expression RPAREN
           | NUMBER
```

不幸的是，这样的文法是存在二义性的。举个例子，如果你要解析字符串"3 * 4 + 5"，操作符如何分组并没有指明，究竟是表示"(3 * 4) + 5"还是"3 * (4 + 5)"呢？

如果在 yacc.py 中存在二义文法，会输出"移进归约冲突"或者"归约归约冲突"。在分析器无法确定是将下一个符号移进栈还是将当前栈中的符号归约时会产生移进归约冲突。例如，对于"3 * 4 + 5"，分析器内部栈是这样工作的：

```
Step Symbol Stack           Input Tokens            Action
---- ---------------------  ---------------------   -------------------------------
1    $                                3 * 4 + 5$    Shift 3
2    $ 3                                * 4 + 5$    Reduce : expression : NUMBER
3    $ expr                             * 4 + 5$    Shift *
4    $ expr *                             4 + 5$    Shift 4
5    $ expr * 4                             + 5$    Reduce: expression : NUMBER
6    $ expr * expr                          + 5$    SHIFT/REDUCE CONFLICT ????
```

在这个例子中，当分析器来到第 6 步的时候，有两种选择：一是按照 expr : expr * expr 归约，一是将标记'+'继续移进栈。两种选择对于上面的上下文无关文法而言都是合法的。

默认情况下，所有的移进归约冲突会倾向于使用移进来处理。因此，对于上面的例子，分析器总是会将'+'进栈，而不是做归约。虽然在很多情况下，这个策略是合适的（像"if-then"和"if-then-else"），但这对于算术表达式是不够的。事实上，对于上面的例子，将'+'进栈是完全错误的，应当先将expr * expr归约，因为乘法的优先级要高于加法。

为了解决二义文法，尤其是对表达式文法，yacc.py 允许为标记单独指定优先级和结合性。需要像下面这样增加一个 precedence 变量：

```
precedence = (
    ('left', 'PLUS', 'MINUS'),
    ('left', 'TIMES', 'DIVIDE'),
)
```

这样的定义说明 PLUS/MINUS 标记具有相同的优先级和左结合性，TIMES/DIVIDE 具有相同的优先级和左结合性。在 precedence 声明中，标记的优先级从低到高。因此，这个声明表明 TIMES/DIVIDE（他们较晚加入 precedence）的优先级高于 PLUS/MINUS。

由于为标记添加了数字表示的优先级和结合性的属性，所以，对于上面的例子，将会得到：

```
PLUS      : level = 1,  assoc = 'left'
MINUS     : level = 1,  assoc = 'left'
TIMES     : level = 2,  assoc = 'left'
DIVIDE    : level = 2,  assoc = 'left'
```

随后这些值被附加到语法规则的优先级和结合性属性上，这些值由最右边的终结符的优先级和结合性决定：

```
expression : expression PLUS expression                 # level = 1, left
           | expression MINUS expression                # level = 1, left
           | expression TIMES expression                # level = 2, left
           | expression DIVIDE expression               # level = 2, left
           | LPAREN expression RPAREN                   # level = None (not specified)
           | NUMBER                                     # level = None (not specified)
```

当出现移进归约冲突时，分析器生成器根据下面的规则解决二义文法：

1. 如果当前的标记的优先级高于栈顶规则的优先级，移进当前标记
2. 如果栈顶规则的优先级更高，进行归约
3. 如果当前的标记与栈顶规则的优先级相同，如果标记是左结合的，则归约，否则，如果是右结合的则移进
4. 如果没有优先级可以参考，默认对于移进归约冲突执行移进

比如，当解析到"expression PLUS expression"这个语法时，下一个标记是 TIMES，此时将执行移进，因为 TIMES 具有比 PLUS 更高的优先级；当解析到"expression TIMES expression"，下一个标记是 PLUS，此时将执行归约，因为 PLUS 的优先级低于 TIMES。

如果在使用前三种技术解决已经归约冲突后，yacc.py 将不会报告语法中的冲突或者错误（不过，会在 parser.out 这个调试文件中输出一些信息）

使用 precedence 指定优先级的技术会带来一个问题，有时运算符的优先级需要基于上下文。例如，考虑"3 + 4 * -5"中的一元的'-'。数学上讲，一元运算符应当拥有较高的优先级。然而，在我们的 precedence 定义中，MINUS 的优先级却低于 TIMES。为了解决这个问题，precedene 规则中可以包含"虚拟标记"：

```
precedence = (
    ('left', 'PLUS', 'MINUS'),
    ('left', 'TIMES', 'DIVIDE'),
    ('right', 'UMINUS'),            # Unary minus operator
)
```

在语法文件中，我们可以这么表示一元算符：

```
def p_expr_uminus(p):
    'expression : MINUS expression %prec UMINUS'
    p[0] = -p[2]
```

在这个例子中，%prec UMINUS 覆盖了默认的优先级（MINUS 的优先级），将 UMINUS 指代的优先级应用在该语法规则上。

起初，UMINUS 标记的例子会让人感到困惑。UMINUS 既不是输入的标记也不是语法规则，你应当将其看成 precedence 表中的特殊的占位符。当你使用 %prec 宏时，你是在告诉 yacc，你希望表达式使用这个占位符所表示的优先级，而不是正常的优先级。

还可以在 precedence 表中指定"非关联"。这表明你不希望链式运算符。比如，假如你希望支持比较运算符'<'和'>'，但是你不希望支持 a < b < c，只要简单指定规则如下：

```
precedence = (
    ('nonassoc', 'LESSTHAN', 'GREATERTHAN'),  # Nonassociative operators
    ('left', 'PLUS', 'MINUS'),
    ('left', 'TIMES', 'DIVIDE'),
    ('right', 'UMINUS'),            # Unary minus operator
)
```

此时，当输入形如  a < b < c 时，将产生语法错误，却不影响形如 a < b 的表达式。

 
对于给定的符号集，存在多种语法规则可以匹配时会产生归约/归约冲突。这样的冲突往往很严重，而且总是通过匹配最早出现的语法规则来解决。归约/归约冲突几乎总是相同的符号集合具有不同的规则可以匹配，而在这一点上无法抉择，比如：

```
assignment :  ID EQUALS NUMBER
           |  ID EQUALS expression
           
expression : expression PLUS expression
           | expression MINUS expression
           | expression TIMES expression
           | expression DIVIDE expression
           | LPAREN expression RPAREN
           | NUMBER
```

这个例子中，对于下面这两条规则将产生归约/归约冲突：

```
assignment  : ID EQUALS NUMBER
expression  : NUMBER
```

比如，对于"a = 5"，分析器不知道应当按照 assignment  : ID EQUALS NUMBER 归约，还是先将 5 归约成 expression，再归约成 assignment : ID EQUALS expression。

应当指出的是，只是简单的查看语法规则是很难减少归约/归约冲突。如果出现归约/归约冲突，yacc()会帮助打印出警告信息：

```
WARNING: 1 reduce/reduce conflict
WARNING: reduce/reduce conflict in state 15 resolved using rule (assignment -> ID EQUALS NUMBER)
WARNING: rejected rule (expression -> NUMBER)
```

上面的信息标识出了冲突的两条规则，但是，并无法指出究竟在什么情况下会出现这样的状态。想要发现问题，你可能需要结合语法规则和`parser.out`调试文件的内容。

 
### 6.7 parser.out调试文件

使用 LR 分析算法跟踪移进/归约冲突和归约/归约冲突是件乐在其中的事。为了辅助调试，yacc.py 在生成分析表时会创建出一个调试文件叫 parser.out：

```
Unused terminals:


Grammar

Rule 1     expression -> expression PLUS expression
Rule 2     expression -> expression MINUS expression
Rule 3     expression -> expression TIMES expression
Rule 4     expression -> expression DIVIDE expression
Rule 5     expression -> NUMBER
Rule 6     expression -> LPAREN expression RPAREN

Terminals, with rules where they appear

TIMES                : 3
error                : 
MINUS                : 2
RPAREN               : 6
LPAREN               : 6
DIVIDE               : 4
PLUS                 : 1
NUMBER               : 5

Nonterminals, with rules where they appear

expression           : 1 1 2 2 3 3 4 4 6 0


Parsing method: LALR


state 0

    S' -> . expression
    expression -> . expression PLUS expression
    expression -> . expression MINUS expression
    expression -> . expression TIMES expression
    expression -> . expression DIVIDE expression
    expression -> . NUMBER
    expression -> . LPAREN expression RPAREN

    NUMBER          shift and go to state 3
    LPAREN          shift and go to state 2


state 1

    S' -> expression .
    expression -> expression . PLUS expression
    expression -> expression . MINUS expression
    expression -> expression . TIMES expression
    expression -> expression . DIVIDE expression

    PLUS            shift and go to state 6
    MINUS           shift and go to state 5
    TIMES           shift and go to state 4
    DIVIDE          shift and go to state 7


state 2

    expression -> LPAREN . expression RPAREN
    expression -> . expression PLUS expression
    expression -> . expression MINUS expression
    expression -> . expression TIMES expression
    expression -> . expression DIVIDE expression
    expression -> . NUMBER
    expression -> . LPAREN expression RPAREN

    NUMBER          shift and go to state 3
    LPAREN          shift and go to state 2


state 3

    expression -> NUMBER .

    $               reduce using rule 5
    PLUS            reduce using rule 5
    MINUS           reduce using rule 5
    TIMES           reduce using rule 5
    DIVIDE          reduce using rule 5
    RPAREN          reduce using rule 5


state 4

    expression -> expression TIMES . expression
    expression -> . expression PLUS expression
    expression -> . expression MINUS expression
    expression -> . expression TIMES expression
    expression -> . expression DIVIDE expression
    expression -> . NUMBER
    expression -> . LPAREN expression RPAREN

    NUMBER          shift and go to state 3
    LPAREN          shift and go to state 2


state 5

    expression -> expression MINUS . expression
    expression -> . expression PLUS expression
    expression -> . expression MINUS expression
    expression -> . expression TIMES expression
    expression -> . expression DIVIDE expression
    expression -> . NUMBER
    expression -> . LPAREN expression RPAREN

    NUMBER          shift and go to state 3
    LPAREN          shift and go to state 2


state 6

    expression -> expression PLUS . expression
    expression -> . expression PLUS expression
    expression -> . expression MINUS expression
    expression -> . expression TIMES expression
    expression -> . expression DIVIDE expression
    expression -> . NUMBER
    expression -> . LPAREN expression RPAREN

    NUMBER          shift and go to state 3
    LPAREN          shift and go to state 2


state 7

    expression -> expression DIVIDE . expression
    expression -> . expression PLUS expression
    expression -> . expression MINUS expression
    expression -> . expression TIMES expression
    expression -> . expression DIVIDE expression
    expression -> . NUMBER
    expression -> . LPAREN expression RPAREN

    NUMBER          shift and go to state 3
    LPAREN          shift and go to state 2


state 8

    expression -> LPAREN expression . RPAREN
    expression -> expression . PLUS expression
    expression -> expression . MINUS expression
    expression -> expression . TIMES expression
    expression -> expression . DIVIDE expression

    RPAREN          shift and go to state 13
    PLUS            shift and go to state 6
    MINUS           shift and go to state 5
    TIMES           shift and go to state 4
    DIVIDE          shift and go to state 7


state 9

    expression -> expression TIMES expression .
    expression -> expression . PLUS expression
    expression -> expression . MINUS expression
    expression -> expression . TIMES expression
    expression -> expression . DIVIDE expression

    $               reduce using rule 3
    PLUS            reduce using rule 3
    MINUS           reduce using rule 3
    TIMES           reduce using rule 3
    DIVIDE          reduce using rule 3
    RPAREN          reduce using rule 3

  ! PLUS            [ shift and go to state 6 ]
  ! MINUS           [ shift and go to state 5 ]
  ! TIMES           [ shift and go to state 4 ]
  ! DIVIDE          [ shift and go to state 7 ]

state 10

    expression -> expression MINUS expression .
    expression -> expression . PLUS expression
    expression -> expression . MINUS expression
    expression -> expression . TIMES expression
    expression -> expression . DIVIDE expression

    $               reduce using rule 2
    PLUS            reduce using rule 2
    MINUS           reduce using rule 2
    RPAREN          reduce using rule 2
    TIMES           shift and go to state 4
    DIVIDE          shift and go to state 7

  ! TIMES           [ reduce using rule 2 ]
  ! DIVIDE          [ reduce using rule 2 ]
  ! PLUS            [ shift and go to state 6 ]
  ! MINUS           [ shift and go to state 5 ]

state 11

    expression -> expression PLUS expression .
    expression -> expression . PLUS expression
    expression -> expression . MINUS expression
    expression -> expression . TIMES expression
    expression -> expression . DIVIDE expression

    $               reduce using rule 1
    PLUS            reduce using rule 1
    MINUS           reduce using rule 1
    RPAREN          reduce using rule 1
    TIMES           shift and go to state 4
    DIVIDE          shift and go to state 7

  ! TIMES           [ reduce using rule 1 ]
  ! DIVIDE          [ reduce using rule 1 ]
  ! PLUS            [ shift and go to state 6 ]
  ! MINUS           [ shift and go to state 5 ]

state 12

    expression -> expression DIVIDE expression .
    expression -> expression . PLUS expression
    expression -> expression . MINUS expression
    expression -> expression . TIMES expression
    expression -> expression . DIVIDE expression

    $               reduce using rule 4
    PLUS            reduce using rule 4
    MINUS           reduce using rule 4
    TIMES           reduce using rule 4
    DIVIDE          reduce using rule 4
    RPAREN          reduce using rule 4

  ! PLUS            [ shift and go to state 6 ]
  ! MINUS           [ shift and go to state 5 ]
  ! TIMES           [ shift and go to state 4 ]
  ! DIVIDE          [ shift and go to state 7 ]

state 13

    expression -> LPAREN expression RPAREN .

    $               reduce using rule 6
    PLUS            reduce using rule 6
    MINUS           reduce using rule 6
    TIMES           reduce using rule 6
    DIVIDE          reduce using rule 6
    RPAREN          reduce using rule 6
```

文件中出现的不同状态，代表了有效输入标记的所有可能的组合，这是依据文法规则得到的。当得到输入标记时，分析器将构造一个栈，并找到匹配的规则。每个状态跟踪了当前输入进行到语法规则中的哪个位置，在每个规则中，'.'表示当前分析到规则的哪个位置，而且，对于在当前状态下，输入的每个有效标记导致的动作也被罗列出来。当出现移进/归约或归约/归约冲突时，被忽略的规则前面会添加!，就像这样：

```
! TIMES           [ reduce using rule 2 ]
  ! DIVIDE          [ reduce using rule 2 ]
  ! PLUS            [ shift and go to state 6 ]
  ! MINUS           [ shift and go to state 5 ]
```

通过查看这些规则并结合一些实例，通常能够找到大部分冲突的根源。应该强调的是，不是所有的移进归约冲突都是不好的，想要确定解决方法是否正确，唯一的办法就是查看 parser.out。


### 6.8 处理语法错误

如果你创建的分析器用于产品，处理语法错误是很重要的。一般而言，你不希望分析器在遇到错误的时候就抛出异常并终止，相反，你需要它报告错误，尽可能的恢复并继续分析，一次性的将输入中所有的错误报告给用户。这是一些已知语言编译器的标准行为，例如 C,C++,Java。在 PLY 中，在语法分析过程中出现错误，错误会被立即检测到（分析器不会继续读取源文件中错误点后面的标记）。然而，这时，分析器会进入恢复模式，这个模式能够用来尝试继续向下分析。LR 分析器的错误恢复是个理论与技巧兼备的问题，yacc.py 提供的错误机制与 Unix 下的 yacc 类似，所以你可以从诸如 O'Reilly 出版的《Lex and yacc》的书中找到更多的细节。

当错误发生时，yacc.py 按照如下步骤进行：

1. 第一次错误产生时，用户定义的 p_error()方法会被调用，出错的标记会作为参数传入；如果错误是因为到达文件结尾造成的，传入的参数将为 None。随后，分析器进入到“错误恢复”模式，该模式下不会在产生`p_error()`调用，直到它成功的移进 3 个标记，然后回归到正常模式。
2. 如果在 p_error() 中没有指定恢复动作的话，这个导致错误的标记会被替换成一个特殊的 error 标记。
3. 如果导致错误的标记已经是 error 的话，原先的栈顶的标记将被移除。
4. 如果整个分析栈被放弃，分析器会进入重置状态，并从他的初始状态开始分析。
5. 如果此时的语法规则接受 error 标记，error 标记会移进栈。
6. 如果当前栈顶是 error 标记，之后的标记将被忽略，直到有标记能够导致 error 的归约。
 

#### 6.8.1 根据 error 规则恢复和再同步

最佳的处理语法错误的做法是在语法规则中包含 error 标记。例如，假设你的语言有一个关于 print 的语句的语法规则：

```
def p_statement_print(p):
     'statement : PRINT expr SEMI'
     ...
```

为了处理可能的错误表达式，你可以添加一条额外的语法规则：

```
def p_statement_print_error(p):
     'statement : PRINT error SEMI'
     print "Syntax error in print statement. Bad expression"
```

这样（expr 错误时），error 标记会匹配任意多个分号之前的标记（分号是`SEMI`指代的字符）。一旦找到分号，规则将被匹配，这样 error 标记就被归约了。

这种类型的恢复有时称为"分析器再同步"。error 标记扮演了表示所有错误标记的通配符的角色，而紧随其后的标记扮演了同步标记的角色。

重要的一个说明是，通常 error 不会作为语法规则的最后一个标记，像这样：

```
def p_statement_print_error(p):
    'statement : PRINT error'
    print "Syntax error in print statement. Bad expression"
```

这是因为，第一个导致错误的标记会使得该规则立刻归约，进而使得在后面还有错误标记的情况下，恢复变得困难。


#### 6.8.2 悲观恢复模式

另一个错误恢复方法是采用“悲观模式”：该模式下，开始放弃剩余的标记，直到能够达到一个合适的恢复机会。

悲观恢复模式都是在 p_error() 方法中做到的。例如，这个方法在开始丢弃标记后，直到找到闭合的'}'，才重置分析器到初始化状态：

```
def p_error(p):
    print "Whoa. You are seriously hosed."
    # Read ahead looking for a closing '}'
    while 1:
        tok = yacc.token()             # Get the next token
        if not tok or tok.type == 'RBRACE': break
    yacc.restart()
```

下面这个方法简单的抛弃错误的标记，并告知分析器错误被接受了：

```
def p_error(p):
    print "Syntax error at token", p.type
    # Just discard the token and tell the parser it's okay.
    yacc.errok()
```

在`p_error()`方法中，有三个可用的方法来控制分析器的行为：

- `yacc.errok()` 这个方法将分析器从恢复模式切换回正常模式。这会使得不会产生 error 标记，并重置内部的 error 计数器，而且下一个语法错误会再次产生 p_error()  调用
- `yacc.token()` 这个方法用于得到下一个标记
- `yacc.restart()` 这个方法抛弃当前整个分析栈，并重置分析器为起始状态

注意：这三个方法只能在`p_error()`中使用，不能用在其他任何地方。

p_error()方法也可以返回标记，这样能够控制将哪个标记作为下一个标记返回给分析器。这对于需要同步一些特殊标记的时候有用，比如：

```
def p_error(p):
    # Read ahead looking for a terminating ";"
    while 1:
        tok = yacc.token()             # Get the next token
        if not tok or tok.type == 'SEMI': break
    yacc.errok()

    # Return SEMI to the parser as the next lookahead token
    return tok
```


#### 6.8.3 从产生式中抛出错误

如果有需要的话，产生式规则可以主动的使分析器进入恢复模式。这是通过抛出`SyntaxError`异常做到的：

```
def p_production(p):
    'production : some production ...'
    raise SyntaxError
```

raise SyntaxError 错误的效果就如同当前的标记是错误标记一样。因此，当你这么做的话，最后一个标记将被弹出栈，当前的下一个标记将是 error 标记，分析器进入恢复模式，试图归约满足 error 标记的规则。此后的步骤与检测到语法错误的情况是完全一样的，p_error() 也会被调用。

手动设置错误有个重要的方面，就是 p_error() 方法在这种情况下不会调用。如果你希望记录错误，确保在抛出 SyntaxError 错误的产生式中实现。

注：这个功能是为了模仿 yacc 中的`YYERROR`宏的行为

 
#### 6.8.4 错误恢复总结

对于通常的语言，使用 error 规则和再同步标记可能是最合理的手段。这是因为你可以将语法设计成在一个相对容易恢复和继续分析的点捕获错误。悲观恢复模式只在一些十分特殊的应用中有用，这些应用往往需要丢弃掉大量输入，再寻找合理的同步点。

 
#### 6.9 行号和位置的跟踪

位置跟踪通常是个设计编译器时的技巧性玩意儿。默认情况下，PLY 跟踪所有标记的行号和位置，这些信息可以这样得到：

- p.lineno(num) 返回第 num 个符号的行号
- p.lexpos(num) 返回第 num 个符号的词法位置偏移

例如：

```
def p_expression(p):
    'expression : expression PLUS expression'
    p.lineno(1)        # Line number of the left expression
    p.lineno(2)        # line number of the PLUS operator
    p.lineno(3)        # line number of the right expression
    ...
    start,end = p.linespan(3)    # Start,end lines of the right expression
    starti,endi = p.lexspan(3)   # Start,end positions of right expression
```

注意：lexspan() 方法只会返回的结束位置是最后一个符号的起始位置。

虽然，PLY 对所有符号的行号和位置的跟踪很管用，但经常是不必要的。例如，你仅仅是在错误信息中使用行号，你通常可以仅仅使用关键标记的信息，比如：

```
def p_bad_func(p):
    'funccall : fname LPAREN error RPAREN'
    # Line number reported from LPAREN token
    print "Bad function call at line", p.lineno(2)
```

类似的，为了改善性能，你可以有选择性的将行号信息在必要的时候进行传递，这是通过 p.set_lineno() 实现的，例如：

```
def p_fname(p):
    'fname : ID'
    p[0] = p[1]
    p.set_lineno(0,p.lineno(1))
```

对于已经完成分析的规则，PLY 不会保留行号信息，如果你是在构建抽象语法树而且需要行号，你应该确保行号保留在树上。

 
### 6.10 构造抽象语法树

yacc.py 没有构造抽像语法树的特殊方法。不过，你可以自己很简单的构造出来。

一个最为简单的构造方法是为每个语法规则创建元组或者字典，并传递它们。有很多中可行的方案，下面是一个例子：

```
def p_expression_binop(p):
    '''expression : expression PLUS expression
                  | expression MINUS expression
                  | expression TIMES expression
                  | expression DIVIDE expression'''

    p[0] = ('binary-expression',p[2],p[1],p[3])

def p_expression_group(p):
    'expression : LPAREN expression RPAREN'
    p[0] = ('group-expression',p[2])

def p_expression_number(p):
    'expression : NUMBER'
    p[0] = ('number-expression',p[1])
```

另一种方法可以是为不同的抽象树节点创建一系列的数据结构，并赋值给 p[0]：

```
class Expr: pass

class BinOp(Expr):
    def __init__(self,left,op,right):
        self.type = "binop"
        self.left = left
        self.right = right
        self.op = op

class Number(Expr):
    def __init__(self,value):
        self.type = "number"
        self.value = value

def p_expression_binop(p):
    '''expression : expression PLUS expression
                  | expression MINUS expression
                  | expression TIMES expression
                  | expression DIVIDE expression'''

    p[0] = BinOp(p[1],p[2],p[3])

def p_expression_group(p):
    'expression : LPAREN expression RPAREN'
    p[0] = p[2]

def p_expression_number(p):
    'expression : NUMBER'
    p[0] = Number(p[1])
```

这种方式的好处是在处理复杂语义时比较简单：类型检查、代码生成、以及其他针对树节点的功能。

为了简化树的遍历，可以创建一个通用的树节点结构，例如：

```
class Node:
    def __init__(self,type,children=None,leaf=None):
         self.type = type
         if children:
              self.children = children
         else:
              self.children = [ ]
         self.leaf = leaf
         
def p_expression_binop(p):
    '''expression : expression PLUS expression
                  | expression MINUS expression
                  | expression TIMES expression
                  | expression DIVIDE expression'''

    p[0] = Node("binop", [p[1],p[3]], p[2])
```


### 6.11 嵌入式动作

yacc 使用的分析技术只允许在规则规约后执行动作。假设有如下规则：

```
def p_foo(p):
    "foo : A B C D"
    print "Parsed a foo", p[1],p[2],p[3],p[4]
```

方法只会在符号 A,B,C和D 都完成后才能执行。可是有的时候，在中间阶段执行一小段代码是有用的。假如，你想在 A 完成后立即执行一些动作，像下面这样用空规则：

```
def p_foo(p):
    "foo : A seen_A B C D"
    print "Parsed a foo", p[1],p[3],p[4],p[5]
    print "seen_A returned", p[2]

def p_seen_A(p):
    "seen_A :"
    print "Saw an A = ", p[-1]   # Access grammar symbol to left
    p[0] = some_value            # Assign value to seen_A
```

在这个例子中，空规则 seen_A 将在 A 移进分析栈后立即执行。p[-1] 指代的是在分析栈上紧跟在 seen_A 左侧的符号。在这个例子中，是 A 符号。像其他普通的规则一样，在嵌入式行为中也可以通过为 p[0] 赋值来返回某些值。

使用嵌入式动作可能会导致移进归约冲突，比如，下面的语法是没有冲突的：

```
def p_foo(p):
    """foo : abcd
           | abcx"""

def p_abcd(p):
    "abcd : A B C D"

def p_abcx(p):
    "abcx : A B C X"
```

可是，如果像这样插入一个嵌入式动作：

```
def p_foo(p):
    """foo : abcd
           | abcx"""

def p_abcd(p):
    "abcd : A B C D"

def p_abcx(p):
    "abcx : A B seen_AB C X"

def p_seen_AB(p):
    "seen_AB :"
```


会产生移进归约冲，只是由于对于两个规则 abcd 和 abcx 中的 C，分析器既可以根据 abcd 规则移进，也可以根据 abcx 规则先将空的 seen_AB 归约。

嵌入动作的一般用于分析以外的控制，比如为本地变量定义作用于。对于 C 语言：

```
def p_statements_block(p):
    "statements: LBRACE new_scope statements RBRACE"""
    # Action code
    ...
    pop_scope()        # Return to previous scope

def p_new_scope(p):
    "new_scope :"
    # Create a new scope for local variables
    s = new_scope()
    push_scope(s)
    ...
```

在这个例子中，new_scope 作为嵌入式行为，在左大括号{之后立即执行。可以是调正内部符号表或者其他方面。statements_block 一完成，代码可能会撤销在嵌入动作时的操作（比如，pop_scope())
 

### 6.12 Yacc 的其他

- 默认的分析方法是 LALR，使用 SLR 请像这样运行 yacc()：yacc.yacc(method="SLR") 注意：LRLR 生成的分析表大约要比 SLR 的大两倍。解析的性能没有本质的区别，因为代码是一样的。由于 LALR 能力稍强，所以更多的用于复杂的语法。
- 默认情况下，yacc.py 依赖 lex.py 产生的标记。不过，可以用一个等价的词法标记生成器代替： yacc.parse(lexer=x) 这个例子中，x 必须是一个 Lexer 对象，至少拥有 x.token() 方法用来获取标记。如果将输入字串提供给 yacc.parse()，lexer 还必须具有 x.input() 方法。
- 默认情况下，yacc 在调试模式下生成分析表（会生成 parser.out 文件和其他东西），使用 yacc.yacc(debug=0) 禁用调试模式。
- 改变 parsetab.py 的文件名：yacc.yacc(tabmodule="foo")
- 改变 parsetab.py 的生成目录：yacc.yacc(tabmodule="foo",outputdir="somedirectory")
- 不生成分析表：yacc.yacc(write_tables=0)。注意：如果禁用分析表生成，yacc()将在每次运行的时候重新构建分析表（这里耗费的时候取决于语法文件的规模）
- 想在分析过程中输出丰富的调试信息，使用：yacc.parse(debug=1)
- yacc.yacc()方法会返回分析器对象，如果你想在一个程序中支持多个分析器：

```
p = yacc.yacc()
...
p.parse()
```

注意：yacc.parse() 方法只绑定到最新创建的分析器对象上。

- 由于生成生成 LALR 分析表相对开销较大，先前生成的分析表会被缓存和重用。判断是否重新生成的依据是对所有的语法规则和优先级规则进行 MD5 校验，只有不匹配时才会重新生成。生成分析表是合理有效的办法，即使是面对上百个规则和状态的语法。对于复杂的编程语言，像 C 语言，在一些慢的机器上生成分析表可能要花费 30-60 秒，请耐心。
- 由于 LR 分析过程是基于分析表的，分析器的性能很大程度上取决于语法的规模。最大的瓶颈可能是词法分析器和语法规则的复杂度。
