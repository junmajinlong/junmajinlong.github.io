# hash类型

在不同语言中，hash类型有时候也被称为关联数组、映射、字典、散列等，这些概念描述的是同一种结构：存储键值对(key/value)的数据结构。

key和value之间一一映射，每一个key和对应的value组成一个键值对，每一个键值对也被称为hash的一个元素或一个entry。

例如，下面的变量person是一个hash变量，存储了三对key/value，name对应junmajinlong，age对应23，gender对应male。

```perl
%person = (
  name   => "junmajinlong",
  age    => 23,
  gender => "male",
);
```

