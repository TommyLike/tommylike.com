# reStructuredText入门


# reStructuredText入门

## 什么是reStructuredText?
reStructuredText通常简称为RST,ReST,reST,是Python里面最为常见的文档格式，他跟Markdown一样，语法在很多地方也很类似，如果有过Markdown的写作经验，入门reStructuredText是一件非常容易的事情。

## RST规范
如果你本地没有即时浏览的编辑器，强烈推荐通过[Online Editor](http://rst.ninjs.org/)来快速上手，我们的介绍几乎都是从[官方文档](http://docutils.sourceforge.net/docs/user/rst/quickref.html)移植过来的。目的是便于个人整理和回顾，个人学习还是推荐去官方文档查看。

### 文本

#### 加粗和斜体  
RST中的加粗和斜体跟Markdown中用法一致都是通过\*和\*\*来指定的: 

```rst
*this is italics*
**this is bold**
```

#### 替代文本(interpreted text)   
一般配合rst中的描述标记使用，用来做链接，引用，解释等,例如站外链接:  

```rst
please contact email `<http://username.organisze.org/>`_
```

#### Embedded hyperlink   
内嵌的外部链接，引用和定义放在一起  

```rst
`display_text link_address`_ :在引用符中间就是具体的显示字符和链接地址，后面紧跟_,  
#以下面实际的例子对比Markdown:  

External hyperlinks, like `Python <http://www.python.org/>`_.  RST  
External hyperlinks, like [Python](http://www.python.org/).  Markdown  
```  

#### Hyperlink target   
引用和定义分开，可以多次引用，使用_建立对应关系，可用于纸质输出文档，他的效果和我们上面最终的效果一致:  

```rst
External hyperlinks, like Python_.  

.. _Python: http://www.python.org/  
```   

#### Inline hyperlink  
引用文章内部的区域,就类似于url里面的\#,语法跟外部target引用一样，只是定义的时候需要使用如下的格式:    

```rst
# 其实就是在定义的时候生成一个带id的div，到时候再通过#指向该区域而已
External hyperlinks, like Python2_.  

.. _Python2:  

this is python2 content  

#RST生成的title默认就带有id，所以可以直接引用title  

Title link :`This is title`_.  

This is title  
=============    
```     

#### 图片引用     
简单的图片插入可以使用**::image**,具体参照如下:   

```rst
     For instance:
     .. image:: images/ball1.gif (这里支持相对和绝对路径)  
``` 

如果需要在inline中使用png图标，可以配合使用\|\|:   

```rst
     The |biohazard| symbol must be used on containers used to dispose of medical waste.

     .. |biohazard| image:: biohazard.png
```  

#### 页脚注释引用      
可以使用[index]\_:     

```rst
    Footnote references, like [5]_.

    .. [5] A numerical footnote. Note
```

多个引用支持配置自增长,\#是基于已有的footage列表做累加，遇到已有的定义会自动跳过  

```rst
    Footnote references, like footage_1 [#]_.
    Footnote references, like footage_2 [#]_.

    .. [1] this is footage 1
    .. [#] This is the first one.    actual 2
    .. [#] This is the second one.   actual 3

```

#### 引证   
引证的用法跟页脚注释引用一致:   

```rst
  Citation references, like [CIT2002]_.
  Note that citations may get
  rearranged, e.g., to the bottom of
  the "page".

  .. [CIT2002] A citation
     (as often used in journals).
```

### 段落和其他

#### 标题    
标题是通过在文字的下面或者上下面增加一行特殊字符表示，字符的长度不能短于文字的长度，可供选择的字符有:`= -  : ' " ~ ^ _ * + # < >`, 比如:   

```rst
  =====
  Title
  =====
  Subtitle
  --------
```

虽然RST本身没有规定层级标题的用法，但是python对不同级别的标题还是有建议规范:   

    1. \# with overline, for parts
    2. \* with overline, for chapters
    3. =, for sections
    4. -, for subsections
    5. ^, for subsubsections
    6. ", for paragraphs

#### 段落  
段落不需要缩进，但是新的段落需要空行，如一下面所示:    

```rst
  This is a paragraph.

  This is another paragraph
```

#### 无序列表
RST中的无序列表跟markdown很像，支持多个不同的符号`- * +`,符号和内容之前需要有空格:  

```rst
  This is a list

  - item1
  - item2
  - item3
```

### 有序列表
RST中的有序列表支持使用**#**作为自增长符号:    

```rst
  This is sequenced list:

  1. item1
  2. item2
  3. item3
  #. item4
```

#### 定义列表  
定义列表在python的documentation中很常见，需要注意的是在定义的内容中需要缩进，不同的内容之间需要有空行:  

```rst
  Definition lists:

  what
    Definition lists associate a term with
    a definition.

  how
    The term is a one-line phrase, and the
    definition is one or more paragraphs or
    body elements, indented relative to the
    term. Blank lines are not allowed
    between term and definition.
```

#### 字段列表  
字段列表在form和brief中比较常见，格式跟定义列表很像，区别在于需要以**:**开头:   

```rst
  :Authors:
      Tony J. (Tibs) Ibbs,
      David Goodger
      (and sundry other good-natured folks)

  :Version: 1.0 of 2001/08/08
  :Dedication: To my father.
```

#### 命令选项列表
这个能帮我们快速构建命令行参数文档，需要注意的是在描述和参数中间至少需要两个空格:   

```rst
  -a            command-line option "a"
  -b file       options can have arguments
                and long descriptions
  --long        options can be long also
  --input=file  long options can also have
                arguments
  /V            DOS/VMS-style options too
```

### Literal Blocks(Quota Blocks)
这个跟我们的引用区域很像，通过查看生成的html源代码可以看到literal blocks会生成**\<pre\>**标记，
而我们的引用区域是直接生成**\<blockquota\>**标签
这个也能方便我们理解他的作用,一般通过**::**来标记开始，内容区域需要使用缩进:

```rst
  content below is literal Blocks::

    this is content

    this is also content

  this is another paragraph
```

或者通过**>**来表示:  

```rst
  content below is literal Blocks::

  > this is content

```

#### 表格
表格的格式如下,跟markdown一样都比较清晰，自己写文档的时候可以直接借助工具生成，不需要手动编辑,在这里推荐[Table Generator](http://www.tablesgenerator.com/text_tables),不过他目前只是简单支持:

```rst
  +------------+------------+-----------+
  | Header 1   | Header 2   | Header 3  |
  +============+============+===========+ 
  | body row 1 | column 2   | column 3  |
  +------------+------------+-----------+
  | body row 2 | Cells may span columns.|
  +------------+------------+-----------+
  | body row 3 | Cells may  | - Cells   |
  +------------+ span rows. | - contain |
  | body row 4 |            | - blocks. |
  +------------+------------+-----------+
```

#### 过渡  
即我们html常见的\<hr\>,一般用的很少，在视觉上隔开段落:

```rst
  A transition marker is a horizontal line
  of 4 or more repeated punctuation
  characters.

  ------------

  A transition should not begin or end a
  section or document, nor should two
  transitions be immediately adjacent.
```

## RST文件预览(windows环境)  

跟markdown一样，在编写文档的时候，我们也需要能够本地即时预览，在这一点上Sublimetext和他强大的plugin再合适不过了，这里推荐[**OmniMarkupPreviewer**](https://github.com/timonwong/OmniMarkupPreviewer)，大部分的markup语言都支持:    

1. Markdown
2. reStruecturedText
3. Textile
4. Pod
5. RDoc
6. Org Mode
7. MediaWiki
8. AsciiDoc
9. Literate Haskell
10. WikiCreole

插件支持实时预览,浏览器预览快捷键(<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>O</kbd>)  

