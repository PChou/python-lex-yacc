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


### 4.2 标记列表

词法分析器必须提供一个标记的列表，这个列表将所有可能的标记告诉分析器，用来执行各种验证，同时也提供给yacc.py作为终结符。

在上面的例子中，是这样给定标记列表的：

```
tokens = (
   'NUMBER',
   'PLUS',
   'MINUS',
   'TIMES',
   'DIVIDE',
   'LPAREN',
   'RPAREN',
)
```


### 4.3 标记的规则

每种标记用一个正则表达式规则来表示，每个规则需要以"t_"开头声明，表示该声明是对标记的规则定义。对于简单的标记，可以定义成这样（在Python中使用raw string能比较方便的书写正则表达式）：

{% highlight python %}
t_PLUS = r'\+'
{% endhighlight %}
这里，紧跟在t_后面的单词，必须跟标记列表中的某个标记名称对应。如果需要执行动作的话，规则可以写成一个方法。例如，下面的规则匹配数字字串，并且将匹配的字符串转化成Python的整型：

```
def t_NUMBER(t):
    r'\d+'
    t.value = int(t.value)
    return t
```

如果使用方法的话，正则表达式写成方法的文档字符串。方法总是需要接受一个LexToken实例的参数，该实例有一个t.type的属性（字符串表示）来表示标记的类型名称，t.value是标记值（匹配的实际的字符串），t.lineno表示当前在源输入串中的作业行，t.lexpos表示标记相对于输入串起始位置的偏移。默认情况下，t.type是以t_开头的变量或方法的后面部分。方法可以在方法体里面修改这些属性。但是，如果这样做，应该返回结果token，否则，标记将被丢弃。 

在lex内部，lex.py用`re`模块处理模式匹配，在构造最终的完整的正则式的时候，用户提供的规则按照下面的顺序加入：

1. 所有由方法定义的标记规则，按照他们的出现顺序依次加入
2. 由字符串变量定义的标记规则按照其正则式长度倒序后，依次加入（长的先入）
3. 顺序的约定对于精确匹配是必要的。比如，如果你想区分‘=’和‘==’，你需要确保‘==’优先检查。如果用字符串来定义这样的表达式的话，通过将较长的正则式先加入，可以帮助解决这个问题。用方法定义标记，可以显示地控制哪个规则优先检查。 

为了处理保留字，你应该写一个单一的规则来匹配这些标识，并在方法里面作特殊的查询：

```
reserved = {
   'if' : 'IF',
   'then' : 'THEN',
   'else' : 'ELSE',
   'while' : 'WHILE',
   ...
}

tokens = ['LPAREN','RPAREN',...,'ID'] + list(reserved.values())

def t_ID(t):
    r'[a-zA-Z_][a-zA-Z_0-9]*'
    t.type = reserved.get(t.value,'ID')    # Check for reserved words
    return t
```

这样做可以大大减少正则式的个数，并稍稍加快处理速度。注意：你应该避免为保留字编写单独的规则，例如，如果你像下面这样写：

```
t_FOR   = r'for'
t_PRINT = r'print'
```

但是，这些规则照样也能够匹配以这些字符开头的单词，比如'forget'或者'printed'，这通常不是你想要的。

