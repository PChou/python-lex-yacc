## 8 使用Python的优化模式

由于 PLY 从文档字串中获取信息，语法解析和词法分析信息必须通过正常模式下的 Python 解释器得到（不带 有-O 或者 -OO 选项）。不过，如果你像这样指定 optimize 模式：

```
lex.lex(optimize=1)
yacc.yacc(optimize=1)
```

PLY 可以在下次执行，在 Python 的优化模式下执行。但你必须确保第一次执行是在 Python 的正常模式下进行，一旦词法分析表和语法分析表生成一次后，在 Python 优化模式下执行，PLY 会使用生成好的分析表而不再需要文档字串。
