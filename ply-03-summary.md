# 3 PLY 概要

PLY 包含两个独立的模块：lex.py 和 yacc.py，都定义在 ply 包下。lex.py 模块用来将输入字符通过一系列的正则表达式分解成标记序列，yacc.py 通过一些上下文无关的文法来识别编程语言语法。yacc.py 使用 LR 解析法，并使用 LALR(1)算法（默认）或者 SLR 算法生成分析表。

这两个工具是为了一起工作的。lex.py 通过向外部提供`token()`方法作为接口，方法每次会从输入中返回下一个有效的标记。yacc.py 将会不断的调用这个方法来获取标记并匹配语法规则。yacc.py 的功能通常是生成抽象语法树(`AST`)，不过，这完全取决于用户，如果需要，yacc.py 可以直接用来完成简单的翻译。

就像相应的 unix 工具，yacc.py 提供了大多数你期望的特性，其中包括：丰富的错误检查、语法验证、支持空产生式、错误的标记、通过优先级规则解决二义性。事实上，传统 yacc 能够做到的 PLY 都应该支持。

yacc.py 与 Unix 下的 yacc 的主要不同之处在于，yacc.py 没有包含一个独立的代码生成器，而是在 PLY 中依赖反射来构建词法分析器和语法解析器。不像传统的 lex/yacc 工具需要一个独立的输入文件，并将之转化成一个源文件，Python 程序必须是一个可直接可用的程序，这意味着不能有额外的源文件和特殊的创建步骤（像是那种执行 yacc 命令来生成 Python 代码）。又由于生成分析表开销较大，PLY 会缓存生成的分析表，并将它们保存在独立的文件中，除非源文件有变化，会重新生成分析表，否则将从缓存中直接读取。
