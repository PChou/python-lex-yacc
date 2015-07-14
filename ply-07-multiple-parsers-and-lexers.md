# 7 多个语法和词法分析器

在高级的分析器程序中，你可能同时需要多个语法和词法分析器。

依照规则行事不会有问题。不过，你需要小心确定所有东西都正确的绑定(hooked up)了。首先，保证将 lex() 和 yacc() 返回的对象保存起来：

```
lexer  = lex.lex()       # Return lexer object
parser = yacc.yacc()     # Return parser object
```

接着，在解析时，确保给 parse() 方法一个正确的 lexer 引用：

```
parser.parse(text,lexer=lexer)
```

如果遗漏这一步，分析器会使用最新创建的 lexer 对象，这可能不是你希望的。

词法器和语法器的方法中也可以访问这些对象。在词法器中，标记的 lexer 属性指代的是当前触发规则的词法器对象：

```
def t_NUMBER(t):
   r'\d+'
   ...
   print t.lexer           # Show lexer object
```

在语法器中，lexer 和 parser 属性指代的是对应的词法器对象和语法器对象

```
def p_expr_plus(p):
   'expr : expr PLUS expr'
   ...
   print p.parser          # Show parser object
   print p.lexer           # Show lexer object
```

如果有必要，lexe r对象和 parser 对象都可以附加其他属性。例如，你想要有不同的解析器状态，可以为 parser 对象附加更多的属性，并在后面用到它们。
