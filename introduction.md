# 序言

如果你从事编译器或解析器的开发工作，你可能对lex和yacc不会陌生，[PLY](http://www.dabeaz.com/ply/)是David Beazley实现的基于Python的lex和yacc。作者最著名的成就可能是其撰写的Python Cookbook, 3rd Edition。我因为偶然的原因接触了PLY，觉得是个好东西，但是似乎国内没有相关的资料。于是萌生了翻译的想法，虽然内容不算多，但是由于能力有限，很多概念不了解，还专门补习了编译原理，这对我有很大帮助。为了完成翻译，经过初译，复审，排版等，花费我很多时间，最终还是坚持下来了，希望对需要的人有所帮助。另外，第一次大规模翻译英文，由于水平有限，如果错误或者不妥的地方还请指正，非常感谢。

## 一些翻译约定

| 英        | 译           |
|:--------- |:-------------|
| token | 标记 |
| context free grammar | 上下文无关文法 |
| syntax directed translation | 语法制导的翻译 |
| ambiguity | 二义 |
| terminals | 终结符 |
| non-terminals | 非终结符 |
| documentation string | 文档字符串（python中的`_docstring_`） |
| shift-reduce | 移进-归约 |
| Empty Productions | 空产生式 |
| Panic mode recovery | 悲观恢复模式 |