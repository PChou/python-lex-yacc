## 8 使用Python的优化模式

由于PLY从文档字串中获取信息，语法解析和词法分析信息必须通过正常模式下的Python解释器得到（不带有-O或者-OO选项）。不过，如果你像这样指定optimize模式：

```
lex.lex(optimize=1)
yacc.yacc(optimize=1)
```

PLY可以在下次执行，在Python的优化模式下执行。但你必须确保第一次执行是在Python的正常模式下进行，一旦词法分析表和语法分析表生成一次后，在Python优化模式下执行，PLY会使用生成好的分析表而不再需要文档字串。
