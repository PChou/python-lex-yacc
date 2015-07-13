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
正则表达式是描述标记规则的典型方法，下一节展示如何用 lex.py 实现。


### 4.1 Lex 的例子

下面的例子展示了如何使用 lex.py 对输入进行标记

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

为了使 lexer 工作，你需要给定一个输入，并传递给`input()`方法。然后，重复调用`token()`方法来获取标记序列，下面的代码展示了这种用法：

```
# Test it out
data = '''
3 + 4 * 10
  + -20 *2


# Give the lexer some input
lexer.input(data)

# Tokenize
while True:
    tok = lexer.token()
    if not tok: break      # No more input
    print tok
```

程序执行，将给出如下输出：

```
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

```
t_PLUS = r'\+'
```
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


### 4.4 标记的值

标记被lex返回后，它们的值被保存在`value`属性中。正常情况下，value是匹配的实际文本。事实上，value可以被赋为任何Python支持的类型。例如，当扫描到标识符的时候，你可能不仅需要返回标识符的名字，还需要返回其在符号表中的位置，可以像下面这样写：

```
def t_ID(t):
    ...
    # Look up symbol table information and return a tuple
    t.value = (t.value, symbol_lookup(t.value))
    ...
    return t
```

需要注意的是，不推荐用其他属性来保存值，因为yacc.py模块只会暴露出标记的value属性，访问其他属性会变得不自然。如果想保存多种属性，可以将元组、字典、或者对象实例赋给value。

 
### 4.5 丢弃标记

想丢弃像注释之类的标记，只要不返回value就行了，像这样：

```
def t_COMMENT(t):
    r'\#.*'
    pass
    # No return value. Token discarded
```

为标记声明添加"ignore_"前缀同样可以达到目的：

```
t_ignore_COMMENT = r'\#.*'
```

如果有多种文本需要丢弃，建议使用方法来定义规则，因为方法能够提供更精确的匹配优先级控制（方法根据出现的顺序，而字符串的正则表达式依据正则表达式的长度）


### 4.6 行号和位置信息

默认情况下，lex.py对行号一无所知。因为lex.py根本不知道何为"行"的概念（换行符本身也作为文本的一部分）。不过，可以通过写一个特殊的规则来记录行号：

```
# Define a rule so we can track line numbers
def t_newline(t):
    r'\n+'
    t.lexer.lineno += len(t.value)
```

在这个规则中，当前lexer对象t.lexer的lineno属性被修改了，而且空行被简单的丢弃了，因为没有任何的返回。

lex.py也不自动做列跟踪。但是，位置信息被记录在了每个标记对象的`lexpos`属性中，这样，就有可能来计算列信息了。例如：每当遇到新行的时候就重置列值：

```
# Compute column. 
#     input is the input text string
#     token is a token instance
def find_column(input,token):
    last_cr = input.rfind('\n',0,token.lexpos)
    if last_cr < 0:
        last_cr = 0
    column = (token.lexpos - last_cr) + 1
    return column
```

通常，计算列的信息是为了指示上下文的错误位置，所以只在必要时有用。


### 4.7 忽略字符

`t_ignore`规则比较特殊，是lex.py所保留用来忽略字符的，通常用来跳过空白或者不需要的字符。虽然可以通过定义像`t_newline()`这样的规则来完成相同的事情，不过使用t_ignore能够提供较好的词法分析性能，因为相比普通的正则式，它被特殊化处理了。

 
### 4.8 字面字符

字面字符可以通过在词法模块中定义一个`literals`变量做到，例如：

```
literals = [ '+','-','*','/' ]
```

或者

```
literals = "+-*/"
```

字面字符是指单个字符，表示把字符本身作为标记，标记的`type`和`value`都是字符本身。不过，字面字符是在其他正则式之后被检查的，因此如果有规则是以这些字符开头的，那么这些规则的优先级较高。

 
 ### 4.9 错误处理

最后，在词法分析中遇到非法字符时，`t_error()`用来处理这类错误。这种情况下，`t.value`包含了余下还未被处理的输入字串，在之前的例子中，错误处理方法是这样的：

```
# Error handling rule
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)
```

这个例子中，我们只是简单的输出不合法的字符，并且通过调用`t.lexer.skip(1)`跳过一个字符。

 
### 4.10 构建和使用lexer

函数`lex.lex()`使用Python的反射机制读取调用上下文中的正则表达式，来创建lexer。lexer一旦创建好，有两个方法可以用来控制lexer对象：

- `lexer.input(data)` 重置lexer和输入字串
- `lexer.token()` 返回下一个LexToken类型的标记实例，如果进行到输入字串的尾部时将返回`None`

推荐直接在lex()函数返回的lexer对象上调用上述接口，尽管也可以向下面这样用模块级别的lex.input()和lex.token()：

```
lex.lex()
lex.input(sometext)
while 1:
    tok = lex.token()
    if not tok: break
    print tok
```

在这个例子中，lex.input()和lex.token()是模块级别的方法，在lex模块中，input()和token()方法绑定到最新创建的lexer对象的对应方法上。最好不要这样用，因为这种接口可能不知道在什么时候就失效（译者注：垃圾回收？）


### 4.11 @TOKEN装饰器

在一些应用中，你可能需要定义一系列辅助的记号来构建复杂的正则表达式，例如：

```
digit            = r'([0-9])'
nondigit         = r'([_A-Za-z])'
identifier       = r'(' + nondigit + r'(' + digit + r'|' + nondigit + r')*)'        

def t_ID(t):
    # want docstring to be identifier above. ?????
    ...
```

在这个例子中，我们希望ID的规则引用上面的已有的变量。然而，使用文档字符串无法做到，为了解决这个问题，你可以使用`@TOKEN`装饰器：

```
from ply.lex import TOKEN

@TOKEN(identifier)
def t_ID(t):
    ...
```

装饰器可以将identifier关联到t_ID()的文档字符串上以使lex.py正常工作，一种等价的做法是直接给文档字符串赋值：

```
def t_ID(t):
    ...

t_ID.__doc__ = identifier
```

注意：@TOKEN装饰器需要Python-2.4以上的版本。如果你在意老版本Python的兼容性问题，使用上面的等价办法。

 
### 4.12 优化模式

为了提高性能，你可能希望使用Python的优化模式（比如，使用-o选项执行Python）。然而，这样的话，Python会忽略文档字串，这是lex.py的特殊问题，可以通过在创建lexer的时候使用optimize选项：

```
lexer = lex.lex(optimize=1)
```

接着，用Python常规的模式运行，这样，lex.py会在当前目录下创建一个lextab.py文件，这个文件会包含所有的正则表达式规则和词法分析阶段的分析表。然后，lextab.py可以被导入用来构建lexer。这种方法大大改善了词法分析程序的启动时间，而且可以在Python的优化模式下工作。

想要更改生成的文件名，使用如下参数：

```
lexer = lex.lex(optimize=1,lextab="footab")
```

在优化模式下执行，需要注意的是lex会被禁用大多数的错误检查。因此，建议只在确保万事俱备准备发布最终代码时使用。


### 4.13 调试

如果想要调试，可以使lex()运行在调试模式：

```
lexer = lex.lex(debug=1)
```

这将打出一些调试信息，包括添加的规则、最终的正则表达式和词法分析过程中得到的标记。

除此之外，lex.py有一个简单的主函数，不但支持对命令行参数输入的字串进行扫描，还支持命令行参数指定的文件名：

```
if __name__ == '__main__':
     lex.runmain()
```

想要了解高级调试的详情，请移步至最后的高级调试部分。

 
### 4.14 其他方式定义词法规则

上面的例子，词法分析器都是在单个的Python模块中指定的。如果你想将标记的规则放到不同的模块，使用module关键字参数。例如，你可能有一个专有的模块，包含了标记的规则：

```
# module: tokrules.py
# This module just contains the lexing rules

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
```

现在，如果你想要从不同的模块中构建分析器，应该这样（在交互模式下）：

```
>>> import tokrules
>>> lexer = lex.lex(module=tokrules)
>>> lexer.input("3 + 4")
>>> lexer.token()
LexToken(NUMBER,3,1,1,0)
>>> lexer.token()
LexToken(PLUS,'+',1,2)
>>> lexer.token()
LexToken(NUMBER,4,1,4)
>>> lexer.token()
None
```

`module`选项也可以指定类型的实例，例如：

```
import ply.lex as lex

class MyLexer:
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
    # Note addition of self parameter since we're in a class
    def t_NUMBER(self,t):
        r'\d+'
        t.value = int(t.value)    
        return t

    # Define a rule so we can track line numbers
    def t_newline(self,t):
        r'\n+'
        t.lexer.lineno += len(t.value)

    # A string containing ignored characters (spaces and tabs)
    t_ignore  = ' \t'

    # Error handling rule
    def t_error(self,t):
        print "Illegal character '%s'" % t.value[0]
        t.lexer.skip(1)

    # Build the lexer
    def build(self,**kwargs):
        self.lexer = lex.lex(module=self, **kwargs)
    
    # Test it output
    def test(self,data):
        self.lexer.input(data)
        while True:
             tok = lexer.token()
             if not tok: break
             print tok

# Build the lexer and try it out
m = MyLexer()
m.build()           # Build the lexer
m.test("3 + 4")     # Test it
```

当从类中定义lexer，你需要创建类的实例，而不是类本身。这是因为，lexer的方法只有被绑定（bound-methods）对象后才能使PLY正常工作。

当给lex()方法使用module选项时，PLY使用`dir()`方法，从对象中获取符号信息，因为不能直接访问对象的`__dict__`属性。（译者注：可能是因为兼容性原因，__dict__这个方法可能不存在）

最后，如果你希望保持较好的封装性，但不希望什么东西都写在类里面，lexers可以在闭包中定义，例如：

```
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

def MyLexer():
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

    # Build the lexer from my environment and return it    
    return lex.lex()
```


### 4.15 额外状态维护

在你的词法分析器中，你可能想要维护一些状态。这可能包括模式设置，符号表和其他细节。例如，假设你想要跟踪`NUMBER`标记的出现个数。

一种方法是维护一个全局变量：

```
num_count = 0
def t_NUMBER(t):
    r'\d+'
    global num_count
    num_count += 1
    t.value = int(t.value)    
    return t
```

如果你不喜欢全局变量，另一个记录信息的地方是lexer对象内部。可以通过当前标记的lexer属性访问：

```
def t_NUMBER(t):
    r'\d+'
    t.lexer.num_count += 1     # Note use of lexer attribute
    t.value = int(t.value)    
    return t

lexer = lex.lex()
lexer.num_count = 0            # Set the initial count
```

上面这样做的优点是当同时存在多个lexer实例的情况下，简单易行。不过这看上去似乎是严重违反了面向对象的封装原则。lexer的内部属性（除了lineno）都是以lex开头命名的（lexdata、lexpos）。因此，只要不以lex开头来命名属性就很安全的。

如果你不喜欢给lexer对象赋值，你可以自定义你的lexer类型，就像前面看到的那样：

```
class MyLexer:
    ...
    def t_NUMBER(self,t):
        r'\d+'
        self.num_count += 1
        t.value = int(t.value)    
        return t

    def build(self, **kwargs):
        self.lexer = lex.lex(object=self,**kwargs)

    def __init__(self):
        self.num_count = 0
```

如果你的应用会创建很多lexer的实例，并且需要维护很多状态，上面的类可能是最容易管理的。

状态也可以用闭包来管理，比如，在Python3中：

```
def MyLexer():
    num_count = 0
    ...
    def t_NUMBER(t):
        r'\d+'
        nonlocal num_count
        num_count += 1
        t.value = int(t.value)    
        return t
    ...
```


### 4.16 Lexer克隆

如果有必要的话，lexer对象可以通过`clone()`方法来复制：

```
lexer = lex.lex()
...
newlexer = lexer.clone()
```

当lexer被克隆后，复制品能够精确的保留输入串和内部状态，不过，新的lexer可以接受一个不同的输出字串，并独立运作起来。这在几种情况下也许有用：当你在编写的解析器或编译器涉及到递归或者回退处理时，你需要扫描先前的部分，你可以clone并使用复制品，或者你在实现某种预编译处理，可以clone一些lexer来处理不同的输入文件。

创建克隆跟重新调用lex.lex()的不同点在于，PLY不会重新构建任何的内部分析表或者正则式。当lexer是用类或者闭包创建的，需要注意类或闭包本身的的状态。换句话说你要注意新创建的lexer会共享原始lexer的这些状态，比如：

```
m = MyLexer()
a = lex.lex(object=m)      # Create a lexer

b = a.clone()              # Clone the lexer
```


### 4.17 Lexer的内部状态

lexer有一些内部属性在特定情况下有用：

- `lexer.lexpos`。这是一个表示当前分析点的位置的整型值。如果你修改这个值的话，这会改变下一个token()的调用行为。在标记的规则方法里面，这个值表示紧跟匹配字串后面的第一个字符的位置，如果这个值在规则中修改，下一个返回的标记将从新的位置开始匹配
- `lexer.lineno`。表示当前行号。PLY只是声明这个属性的存在，却永远不更新这个值。如果你想要跟踪行号的话，你需要自己添加代码（ 4.6 行号和位置信息）
- `lexer.lexdata`。当前lexer的输入字串，这个字符串就是input()方法的输入字串，更改它可能是个糟糕的做法，除非你知道自己在干什么。
- `lexer.lexmatch`。PLY内部调用Python的re.match()方法得到的当前标记的原始的Match对象，该对象被保存在这个属性中。如果你的正则式中包含分组的话，你可以通过这个对象获得这些分组的值。注意：这个属性只在有标记规则定义的方法中才有效。
 

### 4.18 基于条件的扫描和启动条件

在高级的分析器应用程序中，使用状态化的词法扫描是很有用的。比如，你想在出现特定标记或句子结构的时候触发开始一个不同的词法分析逻辑。PLY允许lexer在不同的状态之间转换。每个状态可以包含一些自己独特的标记和规则等。这是基于GNU flex的“启动条件”来实现的，关于flex详见<http://flex.sourceforge.net/manual/Start-Conditions.html#Start-Conditions>

要使用lex的状态，你必须首先声明。通过在lex模块中声明"states"来做到：

```
states = (
   ('foo','exclusive'),
   ('bar','inclusive'),
)
```

这个声明中包含有两个状态：'foo'和'bar'。状态可以有两种类型：'排他型'和'包容型'。排他型的状态会使得lexer的行为发生完全的改变：只有能够匹配在这个状态下定义的规则的标记才会返回；包容型状态会将定义在这个状态下的规则添加到默认的规则集中，进而，只要能匹配这个规则集的标记都会返回。

一旦声明好之后，标记规则的命名需要包含状态名：

```
t_foo_NUMBER = r'\d+'                      # Token 'NUMBER' in state 'foo'        
t_bar_ID     = r'[a-zA-Z_][a-zA-Z0-9_]*'   # Token 'ID' in state 'bar'

def t_foo_newline(t):
    r'\n'
    t.lexer.lineno += 1
```

一个标记可以用在多个状态中，只要将多个状态名包含在声明中：

```
t_foo_bar_NUMBER = r'\d+'         # Defines token 'NUMBER' in both state 'foo' and 'bar'
```

同样的，在任何状态下都生效的声明可以在命名中使用`ANY`：

```
t_ANY_NUMBER = r'\d+'         # Defines a token 'NUMBER' in all states
```

不包含状态名的情况下，标记被关联到一个特殊的状态`INITIAL`，比如，下面两个声明是等价的：

```
t_NUMBER = r'\d+'
t_INITIAL_NUMBER = r'\d+'
```

特殊的`t_ignore()`和`t_error()`也可以用状态关联：

```
t_foo_ignore = " \t\n"       # Ignored characters for state 'foo'

def t_bar_error(t):          # Special error handler for state 'bar'
    pass 
```

词法分析默认在`INITIAL`状态下工作，这个状态下包含了所有默认的标记规则定义。对于不希望使用“状态”的用户来说，这是完全透明的。在分析过程中，如果你想要改变词法分析器的这种的状态，使用`begin()`方法：

```
def t_begin_foo(t):
    r'start_foo'
    t.lexer.begin('foo')             # Starts 'foo' state
```

使用begin()切换回初始状态：

```
def t_foo_end(t):
    r'end_foo'
    t.lexer.begin('INITIAL')        # Back to the initial state
```

状态的切换可以使用栈：

```
def t_begin_foo(t):
    r'start_foo'
    t.lexer.push_state('foo')             # Starts 'foo' state

def t_foo_end(t):
    r'end_foo'
    t.lexer.pop_state()                   # Back to the previous state
```

当你在面临很多状态可以选择进入，而又仅仅想要回到之前的状态时，状态栈比较有用。

举个例子会更清晰。假设你在写一个分析器想要从一堆C代码中获取任意匹配的闭合的大括号里面的部分：这意味着，当遇到起始括号'{'，你需要读取与之匹配的'}'以上的所有部分。并返回字符串。使用通常的正则表达式几乎不可能，这是因为大括号可以嵌套，而且可以有注释，字符串等干扰。因此，试图简单的匹配第一个出现的'}'是不行的。这里你可以用lex的状态来做到：

```
# Declare the state
states = (
  ('ccode','exclusive'),
)

# Match the first {. Enter ccode state.
def t_ccode(t):
    r'\{'
    t.lexer.code_start = t.lexer.lexpos        # Record the starting position
    t.lexer.level = 1                          # Initial brace level
    t.lexer.begin('ccode')                     # Enter 'ccode' state

# Rules for the ccode state
def t_ccode_lbrace(t):     
    r'\{'
    t.lexer.level +=1                

def t_ccode_rbrace(t):
    r'\}'
    t.lexer.level -=1

    # If closing brace, return the code fragment
    if t.lexer.level == 0:
         t.value = t.lexer.lexdata[t.lexer.code_start:t.lexer.lexpos+1]
         t.type = "CCODE"
         t.lexer.lineno += t.value.count('\n')
         t.lexer.begin('INITIAL')           
         return t

# C or C++ comment (ignore)    
def t_ccode_comment(t):
    r'(/\*(.|\n)*?*/)|(//.*)'
    pass

# C string
def t_ccode_string(t):
   r'\"([^\\\n]|(\\.))*?\"'

# C character literal
def t_ccode_char(t):
   r'\'([^\\\n]|(\\.))*?\''

# Any sequence of non-whitespace characters (not braces, strings)
def t_ccode_nonspace(t):
   r'[^\s\{\}\'\"]+'

# Ignored characters (whitespace)
t_ccode_ignore = " \t\n"

# For bad characters, we just skip over it
def t_ccode_error(t):
    t.lexer.skip(1)
```


这个例子中，第一个'{'使得lexer记录了起始位置，并且进入新的状态'ccode'。一系列规则用来匹配接下来的输入，这些规则只是丢弃掉标记（不返回值），如果遇到闭合右括号，t_ccode_rbrace规则收集其中所有的代码（利用先前记录的开始位置），并保存，返回的标记类型为'CCODE'，与此同时，词法分析的状态退回到初始状态。


### 4.19 其他问题

- lexer需要输入的是一个字符串。好在大多数机器都有足够的内存，这很少导致性能的问题。这意味着，lexer现在还不能用来处理文件流或者socket流。这主要是受到re模块的限制。
- lexer支持用Unicode字符描述标记的匹配规则，也支持输入字串包含Unicode
- 如果你想要向`re.compile()`方法提供flag，使用reflags选项：lex.lex(reflags=re.UNICODE)
- 由于lexer是全部用Python写的，性能很大程度上取决于Python的re模块，即使已经尽可能的高效了。当接收极其大量的输入文件时表现并不尽人意。如果担忧性能，你可以升级到最新的Python，或者手工创建分析器，或者用C语言写lexer并做成扩展模块。

如果你要创建一个手写的词法分析器并计划用在yacc.py中，只需要满足下面的要求：

- 需要提供一个token()方法来返回下一个标记，如果没有可用的标记了，则返回None。
- token()方法必须返回一个tok对象，具有type和value属性。如果行号需要跟踪的话，标记还需要定义lineno属性。
