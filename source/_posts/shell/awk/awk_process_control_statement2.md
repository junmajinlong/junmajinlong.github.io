---
title: 精通awk系列(19)：awk流程控制：break、continue、next、nextfile、exit
p: shell/awk/awk_process_control_statement2.md
date: 2020-02-27 10:39:35
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# break和continue

break可退出for、while、do...while、switch语句。

continue可让for、while、do...while进入下一轮循环。

```
awk '
BEGIN{
  for(i=0;i<10;i++){
    if(i==5){
      break
    }
    print(i)
  }

  # continue
  for(i=0;i<10;i++){
    if(i==5)continue
    print(i)
  }
}'
```


# next和nextfile

next会在当前语句处立即停止后续操作，并读取下一行，进入循环顶部。

例如，输出除第3行外的所有行。

```
awk 'NR==3{next}{print}' a.txt
awk 'NR==3{getline}{print}' a.txt
```

nextfile会在当前语句处立即停止后续操作，并直接读取下一个文件，并进入循环顶部。

例如，每个文件只输出前2行：

```
awk 'FNR==3{nextfile}{print}' a.txt a.txt
```

# exit

```
exit [exit_code]
```

直接退出awk程序。

注意，END语句块也是exit操作的一部分，所以在BEGIN或main段中执行exit操作，也会执行END语句块。

如果exit在END语句块中执行，则立即退出。

所以，如果真的想直接退出整个awk，则可以先设置一个flag变量，然后在END语句块的开头检查这个变量再exit。

```
BEGIN{
    ...code...
    if(cond){
        flag=1
        exit
    }
}
{}
END{
    if(flag){
        exit
    }
    ...code...
}

awk '
    BEGIN{print "begin";flag=1;exit}
    {}
    END{if(flag){exit};print "end2"}
' 
```

exit可以指定退出状态码，如果触发了两次exit操作，即BEGIN或main中的exit触发了END中的exit，且END中的exit没有指定退出状态码时，则采取前一个退出状态码。

```
$ awk 'BEGIN{flag=1;exit 2}{}END{if(flag){exit 1}}' 
$ echo $?
1

$ awk 'BEGIN{flag=1;exit 2}{}END{if(flag){exit}}'   
$ echo $?
2
```


