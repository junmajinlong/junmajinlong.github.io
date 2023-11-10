---
title: json文件格式说明
p: coding/about_json.md
date: 2019-07-06 18:20:41
tags: Coding
categories: Coding
---

## json文件格式说明

1. json文件由对象(集合)、数组、key/value元素组成，可以相互嵌套。  
2. 使用大括号包围的是对象，使用中括号包围的是数组，冒号分隔的是元素。  
3. 元素的key只能是字符串。  
4. 元素的value数据类型可以是：  
    - number：整数和浮点数都属于number类型，可以是正负数  
    - string：字符串  
    - bool：true/false  
    - array：使用中括号包围的部分是array  
    - object：使用大括号包围的是对象  
    - null：空。一般是这个值本来应该是某个object的，但是object不存在，于是为Null  
5. 对象、数组容器中每个元素之间使用逗号隔开，容器的最后一个元素不加逗号  
6. 顶级对象都是匿名的，也就是没有key  

![](/img/referer.jpg)

下面是一个json格式数据的示例：

```
{
	"id":1,
	"content":"hello world",
	"author":{
		"id":2,
		"name":"userA"
	},
	"published":true,
	"label":[],
	"nextPost":null,
	"comments":[
		{
			"id":3,
			"content":"good post1",
			"author":"userB"
		},
		{
			"id":4,
			"content":"good post2",
			"author":"userC"
		}
	]
}
```

用注释分析这个json：
```
{ # 对象容器，下面全是这个对象中的属性。注意key全都是字符串
	"id":1,   # 文章ID号，元素，value类型为number
	"content":"hello world",  # 文章内容
	"author":{   # 子对象，文章作者
		"id":2,   # 作者ID
		"name":"userA"    # 作者名称，注意子容器结束，没有逗号
	},
	"published":true,   # 文章是否发布，布尔类型
	"label":[],         # 文章标签，没有给标签，所以空数组
	"nextPost":null,    # 下一篇文章，是对象，因为没有，所以为null
	"comments":[     # 文章评论，因为可能有多条评论，每条评论都是一个对象结构
		{     # 对象容器，表示评论对象
			"id":3,        # 评论的ID号
			"content":"good post1",    # 评论的内容
			"author":"userB"         # 评论者
		},
		{
			"id":4,
			"content":"good post2",
			"author":"userC"
		}
	]
}
```

一般来说，json格式转换成语言中的数据结构时，有以下几个比较通用的规则(只是比较普通的方式，并非一定)：  
- **json对象映射成语言中的hash/struct，有时候没有合适的结构，将映射成类。其实class、hash、struct在数据组织方式上都是一样的，都是key/value的容器**  
- **json数组映射成语言中的列表/数组/切片**  

例如，上面的示例，转换成Go中的数据结构时，得到的结果如下：
```
// 使用名称A代替顶层的匿名对象
type A struct {
	ID        int64         `json:"id"`       
	Content   string        `json:"content"`  
	Author    Author        `json:"author"`   
	Published bool          `json:"published"`
	Label     []interface{} `json:"label"`    
	NextPost  interface{}   `json:"nextPost"` 
	Comments  []Comment     `json:"comments"` 
}

type Author struct {
	ID   int64  `json:"id"`  
	Name string `json:"name"`
}

type Comment struct {
	ID      int64  `json:"id"`     
	Content string `json:"content"`
	Author  string `json:"author"` 
}
```

![](/img/referer.jpg)

比如转换成python中的数据时，得到的结果如下：

```
from typing import List, Any

class Author:
    id: int
    name: str

    def __init__(self, id: int, name: str) -> None:
        self.id = id
        self.name = name

class Comment:
    id: int
    content: str
    author: str

    def __init__(self, id: int, content: str, author: str) -> None:
        self.id = id
        self.content = content
        self.author = author

# 使用了名称A代替顶层的匿名对象
class A:
    id: int
    content: str
    author: Author
    published: bool
    label: List[Any]
    next_post: None
    comments: List[Comment]

    def __init__(self, id: int, content: str, author: Author, published: bool, label: List[Any], next_post: None, comments: List[Comment]) -> None:
        self.id = id
        self.content = content
        self.author = author
        self.published = published
        self.label = label
        self.next_post = next_post
        self.comments = comments
```


## json转代码数据结构推荐工具

quicktype工具，可以轻松地将json文件转换成各种语言对应的数据结构。

地址：https://quicktype.io

在vscode中有相关插件

1. 先在命令面板中输入"set quicktype target language"选择要将json转换成什么语言的数据结构
2. 再输入"open quicktype for json"就可以将当前json文件转换对应的数据结构。