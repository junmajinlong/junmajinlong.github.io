---
title: Form的enctype属性
p: coding/html_form_enctype.md
date: 2019-07-06 18:20:41
tags: Coding
categories: Coding
---

## Form的enctype属性

一般都使用html的Form表单通过HTTP POST方法发送Request body。下面是一个form：

```
<form action="/process" method="post">
  <input type="text" name="first_name"/>
  <input type="text" name="last_name"/>
  <input type="submit"/>
</form>
```

如果使用GET方法，input中的key/vaule会编码后放进URL的query部分发送出去。如果使用POST方法，input中的key/value会编码后放进request的body中发送出去。

![](/img/referer.jpg)

例如：
```
first_name=xxx&last_name=yyy
```

这是默认的情况：key=value，并使用"&"符号连接多个key/value。

但并非一定会如此组织，它由form的enctype属性决定的：`<form action="/process" method="post" enctype=xxxxxx>`。这个属性有三种值：
```
application/x-www-form-urlencoded
multipart/form-data
text/plain     // HTML5支持该值
```

默认的值为`application/x-www-form-urlencoded`，它的组织形式上面已经解释了。

如果设置为`multipart/form-data`，则会将编码后的key/value将组织成MIME数据的形式。例如:
```
------WebKitFormBoundaryMPNjKpeO9cLiocMw
 Content-Disposition: form-data; name="first_name"

sau sheong
------WebKitFormBoundaryMPNjKpeO9cLiocMw
 Content-Disposition: form-data; name="last_name"

chang
------WebKitFormBoundaryMPNjKpeO9cLiocMw--
```

那么选择哪个enctype组织形式？如果是简单的文本类型，使用默认的`application/x-www-form-urlencoded`会更简单、更高效。但如果发送大量数据(比如上传文件)，应该使用`multipart/form-data`，甚至将数据进行Base64编码后，也可以使用`multipart/form-data`的形式。