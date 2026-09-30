# Keil的那些奇怪配置

## 编码

古老的keil编程会选择GB2312编码风格，在其他编译器选择UTF-8编码时，中文变成了乱码

## 缩进

keil里面一个tab的大小是4，其他编译器如果要对齐keil格式，也要将tab的大小设成4

##串口重定向

使用了串口重定向要勾选魔术棒target里面的MircoLib，否则编译会报错

## 项目文件树不小心叉掉消失

点击工具栏最右边倒数第二个，也就是扳手前边的project windows重新打开

##ctrl+f 查询词要一直点Find next

ctrl+f 出来后选择find in files 选择current documents就可以将查询结果全部显示在下端日志台，一条一条筛选即可

##代码风格报错，不允许大括号带个代码之类的

在魔术棒里面的C/C++ 选择C语言为C99mode

##将keil编译出的axf文件转成bin文件

在魔术棒里面的User的After Build/Rebuild的Run1加上

```
fromelf --bin --output ./Objects/CharingCar.bin ./Objects/CharingCar.axf
```

