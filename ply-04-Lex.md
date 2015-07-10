#4 Lex

`lex.py`是用来将输入字符串标记化。例如，假设你正在设计一个编程语言，用户的输入字符串如下：

```
x = 3 + 42 * (s - t)
```

标记器将字符串分割成独立的标记：

```
'x','=', '3', '+', '42', '*', '(', 's', '-', 't', ')'
```


标记通常用一组名字来命名和表示：

```
'ID','EQUALS','NUMBER','PLUS','NUMBER','TIMES','LPAREN','ID','MINUS','ID','RPAREN'
```

将标记名和标记值本身组合起来：

```
('ID','x'), ('EQUALS','='), ('NUMBER','3'),('PLUS','+'), ('NUMBER','42), ('TIMES','*'),('LPAREN','('), ('ID','s'),('MINUS','-'),('ID','t'), ('RPAREN',')
```
正则表达式是描述标记规则的典型方法，下一节展示如何用lex.py实现。

### 4.1 Lex的例子

下面的例子展示了如何使用lex.py对输入进行标记
```
# ------------------------------------------------------------
# calclex.py
#
# tokenizer for a simple expression evaluator for
# numbers and +,-,*,/
# ------------------------------------------------------------
import ply.lex as lex

# List of token names.   This is always required
tokens = (
   'NUMBER',
   'PLUS',
   'MINUS',
   'TIMES',
   'DIVIDE',
   'LPAREN',
   'RPAREN',
)

# Regular expression rules for simple tokens
t_PLUS    = r'\+'
t_MINUS   = r'-'
t_TIMES   = r'\*'
t_DIVIDE  = r'/'
t_LPAREN  = r'\('
t_RPAREN  = r'\)'

# A regular expression rule with some action code
def t_NUMBER(t):
    r'\d+'
    t.value = int(t.value)    
    return t

# Define a rule so we can track line numbers
def t_newline(t):
    r'\n+'
    t.lexer.lineno += len(t.value)

# A string containing ignored characters (spaces and tabs)
t_ignore  = ' \t'

# Error handling rule
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)

# Build the lexer
lexer = lex.lex()
```

为了使lexer工作，你需要给定一个输入，并传递给`input()`方法。然后，重复调用`token()`方法来获取标记序列，下面的代码展示了这种用法：

```
# Test it out
data = '''
3 + 4 * 10
  + -20 *2
'''

# Give the lexer some input
lexer.input(data)

# Tokenize
while True:
    tok = lexer.token()
    if not tok: break      # No more input
    print tok
{% endhighlight %}

程序执行，将给出如下输出：

{% highlight text %}
$ python example.py
LexToken(NUMBER,3,2,1)
LexToken(PLUS,'+',2,3)
LexToken(NUMBER,4,2,5)
LexToken(TIMES,'*',2,7)
LexToken(NUMBER,10,2,10)
LexToken(PLUS,'+',3,14)
LexToken(MINUS,'-',3,16)
LexToken(NUMBER,20,3,18)
LexToken(TIMES,'*',3,20)
LexToken(NUMBER,2,3,21)
```

Lexers也同时支持迭代，你可以把上面的循环写成这样：

```
for tok in lexer:
    print tok
```

由lexer.token()方法返回的标记是LexToken类型的实例，拥有`tok.type`,`tok.value`,`tok.lineno`和`tok.lexpos`属性，下面的代码展示了如何访问这些属性：

```
# Tokenize
while True:
    tok = lexer.token()
    if not tok: break      # No more input
    print tok.type, tok.value, tok.line, tok.lexpos
```

`tok.type`和`tok.value`属性表示标记本身的类型和值。`tok.line`和`tok.lexpos`属性包含了标记的位置信息，`tok.lexpos`表示标记相对于输入串起始位置的偏移。
