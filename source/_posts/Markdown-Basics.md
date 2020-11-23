---
title: Markdown Basics
date: 2020-11-21 13:01:34
categories:
- [Markdown, Basics]
tags:
- Markdown
---

## 属性Markdown语法要点

本节展示基本的Markdown用法。在[Syntax](https://daringfireball.net/projects/markdown/syntax)中提供了更为详细的语法介绍。下面通过同时显示了转换前和转换后的例子，展示Markdown语法与HTML格式的对照关系。
同时我们可以使用[Dingus](https://daringfireball.net/projects/markdown/dingus)在线语法验证工具来更好的理解Markdown语法。

## 段落、标题、引用

段落由一行或多行连续的文本构成，段落之间通过一行或多行空格分隔。正常的段落不应该使用space或tabs进行缩进。

Markdown提供两种风格的标题格式：Setext和atx：
- Setext风格是采用下滑线的形式来对标题进行标记。例如一级标题和二级标题，分别使用 = 和 - 来代表；
- atx风格的标题，采用1-6个置于行首的#号来表示。#号的个数代表标题的等级；

> 块引用使用的是邮件风格的>号表示。
>
Markdown:
``` markdown
	一级标题
	========
	二级标题
	--------
	然后这里是贝拉巴啦啦吧一堆描述的、正儿八经的段落。
	再来一段。
	
	### 三级标题
	> 这是块引用
	> 
	> 这是块引用的第二段
	> 这是块引用的第三段
	>
	> ## 这是块引用中的二级标题
```
HTML:
``` html
	<h1>一级标题</h1>
	<h2>二级标题</h2>
	<p>然后这里是贝拉巴啦啦吧一堆描述的、正儿八经的段落。</p>
	<p>再来一段<p>
	<h3>三级标题</h3>
	<blockquote>
		<p>这是块引用</p>
		<p>这是块引用的第二段</p>
		<p>这是块引用的第三段</p>
		<h2>这是块引用中的二级标题</h2>
	</blockquote>
```

## 短语强调

Markdown使用 \* 或 \_ 来表示强调范围
Markdown：
``` markdown
	有些*人*就是特殊
	有些_事_就是漂亮
	
	还有些**人**特殊到无以复加
	还有些__事__漂亮到难以形容
```
HTML：
``` html
	<p>有些<em>人</em>就是特殊</p>
	<p>有些<em>事</em>就是漂亮</p>
	
	<p>还有些<strong>人</strong>特殊到无以复加</p>
	<p>还有些<strong>事</strong>漂亮到难以形容</p>
```

## 列表

无序号的列表使用 \* 和 \+ 以及 \- 号来表示。这三个符号表达的是一个意思;
例如这个：
``` markdown
* Candy
* Gum
* Booze
```
这个：
``` markdown
+ Candy
+ Gum
+ Booze
```
还有这个：
``` markdown
- Candy
- Gum
- Booze
```
这三个输出的结果都是同样的HTML格式：
``` html
<ul>
<li>Candy</li>
<li>Gum</li>
<li>Booze</li>
</ul>
```

有需要的列表使用的是数字加 . 的方式：
``` markdown
1. Red
2. Green
3. Blue
```
HTML：
``` html
<ol>
<li>Red</li>
<li>Green</li>
<li>Blue</li>
</ol>
```
如果列表之间有空行，那么这些列表元素就会被认为是段落。如果单个元素由多个段落构成，那么需要将后续的段落进行缩进处理：
``` markdown
1. Red

2. Green
green
3. Blue
   blue

4. yellow

5. Orange
```
HTML:
```html
<ol>
<li><p>Red</p></li>
<li><p>Green
green</p></li>
<li><p>Blue
blue</p></li>
<li><p>yellow</p></li>
<li><p>Orange</p></li>
</ol>
```

## 链接
Markdown提供两种链接方式：
- inline，内联方式；
- reference，引用方式；

这两种方式都需要使用方括号来分隔需要显示的文本。
内联方式直接在方括号后面使用圆括号来表示链接，例如：
``` markdown
这是[链接名称](http://example.com/)。
```
HTML：
``` html
<p>这是<a href="http://example.com/">链接名称</a>。</p>
```
另外，我们也可以给网址命个名：
``` markdown
这是[链接名称](http://example.com/ "黑网")。
```
HTML：
``` html
<p>这是<a href="http://example.com/" title="黑网">链接名称</a>。</p>
```

引用方式通过给链接命名的方式进行链接：
``` markdown
我从[黑网][1]跳到[白网][2]再跳到[红网][3]

[1]: http://black-net.com/ "黑网"
[2]: http://white-net.com/ "白网 1"
[3]: http://red-net.com/ "红网 2"
```
HTML：
``` html
<p>我从<a href="http://black-net.com/" title="黑网">黑网</a>跳到<a href="http://white-net.com/" title="白网 1">白网</a>再跳到<a href="http://red-net.com/" title="红网 2">红网</a></p>
```
上面的title属性可有可无。链接名由字母、数字、空格构成，不区分大小写：
``` markdown
我在[红网a][red 1]看到你的照片
[REd 1]: http://red-a-net.com/
```
HTML:
``` html
<p>我在<a href="http://red-a-net.com/">红网a</a>看到你的照片</p>
```

## 图片

图片的语法和链接非常相似，也是有Inline和Reference两种方式，如下所示，其中Title属性是可有可无的：
- Inline，内联方式：
  ``` markdown
  ![alt text](/path/to/img.jpg "Title")
  ```
- Reference,引用方式：
  ``` markdown
  ![alt tex][id]
  [id]: /path/to/img.jpg "Title"
  ```

无论哪种方式得到的结果都一样：
``` html
<img src="/path/to/img.jpg" alt="alt tex" title="Title" /></li>
```

## 代码块

在普通的段落中，我们可以给使用反引号 \` ，也就是ECS键下面的那个 \` 键， 从而将代码插入到普通段落中。这样可以使得普通段落和代码块的之间界限分明。还有一点就是 \& 和 \< 、 \> 这几个符合是HTML的命令实体，需要转义后才能以普通字符的形式显示。但是在反引号 \` 包裹下的所有字符都只被当作普通字符，而不需要而外的转义操作。这就很方便在Markdown文档中编写HTML的示例代码：
``` markdown
I strongly recommend against using any `<blink>` tags.

I wish SmartyPants used named entities like `&mdash;`
instead of decimal-encoded entites like `&#8212;`.
```
HTML：
``` html 
<p>I strongly recommend against using any
<code>&lt;blink&gt;</code> tags.</p>

<p>I wish SmartyPants used named entities like
<code>&amp;mdash;</code> instead of decimal-encoded
entites like <code>&amp;#8212;</code>.</p>
```
最终在网页上显示的效果：
I strongly recommend against using any `<blink>` tags.

I wish SmartyPants used named entities like `&mdash;`
instead of decimal-encoded entites like `&#8212;`.
如果希望将整个代码块进行格式化，那么将代码块中的每一行缩进4个spaces或1个tab。其所产生的效果和使用反引号 \` 标记的代码块一样，其中的 \& 、\< 、\> 这些符号都被自动转义了：
Markdown:
``` markdown
If you want your page to validate under XHTML 1.0 Strict,
you've got to put paragraph tags in your blockquotes:

    <blockquote>
        <p>For example.</p>
    </blockquote>
```
HTML:
``` html
<p>If you want your page to validate under XHTML 1.0 Strict,
you've got to put paragraph tags in your blockquotes:</p>

<pre><code>&lt;blockquote&gt;
    &lt;p&gt;For example.&lt;/p&gt;
&lt;/blockquote&gt;
</code></pre>
```
最终在网页上显示的效果：
If you want your page to validate under XHTML 1.0 Strict,
you've got to put paragraph tags in your blockquotes:

	<blockquote>
		<p>For example.</p>
	</blockquote>