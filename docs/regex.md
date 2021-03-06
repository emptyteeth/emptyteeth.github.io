# regex正则表达式基础

><https://www.runoob.com/regexp>

## 修饰符

修饰符是一个表达式的全局选项,写在表达式最后
>/exp/igms

- i 表示不区分大小写
- g 表示匹配所有项,反之则表示只匹配第一项
- m 多行模式,使定位符也匹配newline的相应位置
- s 默认情况下.匹配除\n换行符之外的所有字符,使用s选项使.也匹配\n换行

## 范围匹配

匹配范围内的单个字符

```re
[0-9] #0到9的所有数字
[a-z] #a-z的所有小写字母
[a-zA-Z]  #a-z的所有大小写字母
[0-9a-zA-Z] #0-9所有的数字和a-z的所有大小写字母
[abc123]  #匹配abc三个字母和123三个数字
[^0-9]  #范围取反
\w  #所有数字大小写字母和下划线,等于 [A-Za-z0-9_]
\W  #\w的取反,等于 [^A-Za-z0-9_]
\d  #等于 [0-9]
\D  #\d的取反,等于 [^0-9]
\s  #所有空格 tab 换行
\S  #\s的取反
.   #匹配除\n \r之外的任意字符
```

## 限定符

限定前一个字符或表达式出现次数

```re
abc+ #必须出现1次或多次
abc* #可以出现1次或多次,也可以不出现
abc? #最多出现1次,也可以不出现
abc{n} #必须出现n次
abc{n,} #必须出现n或n+次
  abc{1,} 等于 abc+
  abc{0,} 等于 abc*
abc{n,m} #必须至少出现n次,最多出现m次
  abc{0,1} 等于 abc?
```

## 定位符

只匹配位置,不匹配任何字符,所以是zero-width,需要搭配字符或表达式使用

```re
^定位字符串开始的位置 ^abc匹配以abc开头的字符串,在multiline模式下也匹配newline的开头,就是\n或\r之后的位置
$定位字符串结束的位置 abc$匹配以abc结束的字符串,在multiline模式下也匹配\n或\r之前的位置
\b定位字母和数字与符号/空格/tab/换行之间的边界 \bword\b表示word前后都是符号/空格/tab/换行
\B与\b相反 \Bword\B表示word前后都是字母或数字
```

## 其它/多义字符

- \ 转义字符
- | 表示表达式间的或关系
- ? 除了限定符的功能外,出现在任意限定符后表示关闭前限定符的贪婪模式
- ^ 除了定位符的功能外,出现在[]内的开头表示范围取反

## 捕获组

向前引用,引用的是捕获组中的内容,而不是模式

```re
\b([a-z]+)(?: \1)+ 匹配连续出现的单词
(\d)(?:\1)+ 连续出现的数字
(\w+):\/\/((?:(?:[a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9])\.)+[a-zA-Z0-9]+)(?::(\d+))?(\/.*)?  解析uri
```

## 非捕获组

```re
(?:a|b) 用于纯分组
exp1(?=exp2) 查找 exp2 前面的 exp1
(?<=exp2)exp1：查找 exp2 后面的 exp1
exp1(?!exp2)：查找后面不是 exp2 的 exp1
(?<!exp2)exp1：查找前面不是 exp2 的 exp1
```

## test

```re
^(?:[0-1][0-9]|2[0-3]):(?:[0-5][1-9]|60)$ 匹配12:34时间格式

^(?:(?:[1-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])\.){3}(?:[1-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])$ 匹配ipv4地址
```
