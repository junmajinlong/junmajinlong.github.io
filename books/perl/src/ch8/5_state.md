## state声明变量

Perl子程序内部还可以使用`state`关键字来声明一个持久化的私有变量：这个变量只在第一次调用子程序时被初始化，以后调用该子程序时将忽略state声明变量的代码，且子程序外部不能看见这个变量。

state是v5.10添加的功能，因此需加上`use 5.010;`才能使用state。

例如，定义一个记录子程序被调用总次数的变量：

```perl
use 5.010;
sub roll{
  state $cnt=0;  # 只初始化一次的变量，初始值为0
  $cnt++;  # 每次调用子程序都自增一次
  return 1 + int( rand(6) );
}
```

state的效果类似于在一个独立的作用域内定义子程序(即类似于闭包)：

```perl
{
  my $cnt = 0;
  sub roll{
    $cnt++;
    say $cnt;
    return 1 + int(rand(6));
  }
}
roll();
roll();
```