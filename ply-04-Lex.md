4 Lex

`lex.py`是用来将输入字符串标记化。例如，假设你正在设计一个编程语言，用户的输入字符串如下：

```{% highlight python %}

x = 3 + 42 * (s - t)

{% endhighlight %}```

标记器将字符串分割成独立的标记：

```{% highlight text %}
'x','=', '3', '+', '42', '*', '(', 's', '-', 't', ')'
{% endhighlight %}```


标记通常用一组名字来命名和表示：

```{% highlight text %}
'ID','EQUALS','NUMBER','PLUS','NUMBER','TIMES','LPAREN','ID','MINUS','ID','RPAREN'
{% endhighlight %}```

将标记名和标记值本身组合起来：

```{% highlight python %}
('ID','x'), ('EQUALS','='), ('NUMBER','3'),('PLUS','+'), ('NUMBER','42), ('TIMES','*'),('LPAREN','('), ('ID','s'),('MINUS','-'),('ID','t'), ('RPAREN',')
{% endhighlight %}```
正则表达式是描述标记规则的典型方法，下一节展示如何用lex.py实现。
