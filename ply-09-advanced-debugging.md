## 9 高级调试

调试一个编译器不是件容易的事情。PLY提供了一些高级的调试能力，这是通过Python的logging模块实现的，下面两节介绍这一主题：

 
### 9.1 调试lex()和yacc()命令

lex()和yacc()命令都有调试模式，可以通过debug标识实现：

```
lex.lex(debug=True)
yacc.yacc(debug=True)
```

正常情况下，调试不仅输出标准错误，对于yacc()，还会给出parser.out文件。这些输出可以通过提供logging对象来精细的控制。下面这个例子增加了对调试信息来源的输出：

```
# Set up a logging object
import logging
logging.basicConfig(
    level = logging.DEBUG,
    filename = "parselog.txt",
    filemode = "w",
    format = "%(filename)10s:%(lineno)4d:%(message)s"
)
log = logging.getLogger()

lex.lex(debug=True,debuglog=log)
yacc.yacc(debug=True,debuglog=log)
```

如果你提供一个自定义的logger，大量的调试信息可以通过分级来控制。典型的是将调试信息分为DEBUG,INFO,或者WARNING三个级别。

PLY的错误和警告信息通过日志接口提供，可以从errorlog参数中传入日志对象

```
lex.lex(errorlog=log)
yacc.yacc(errorlog=log)
```

如果想完全过滤掉警告信息，你除了可以使用带级别过滤功能的日志对象，也可以使用lex和yacc模块都内建的Nulllogger对象。例如：

```
yacc.yacc(errorlog=yacc.NullLogger())
```


### 9.2 运行时调试

为分析器指定debug选项，可以激活语法分析器的运行时调试功能。这个选项可以是整数（表示对调试功能是开还是关），也可以是logger对象。例如：

```
log = logging.getLogger()
parser.parse(input,debug=log)
```

如果传入日志对象的话，你可以使用其级别过滤功能来控制内容的输出。INFO级别用来产生归约信息；DEBUG级别会显示分析栈的信息、移进的标记和其他详细信息。ERROR级别显示分析过程中的错误相关信息。

对于每个复杂的问题，你应该用日志对象，以便输出重定向到文件中，进而方便在执行结束后检查。
