[TOC]

# Linux文本处理

## 1. grep命令

grep可以进行文本检索过滤，常用选项如下：

选项 | 描述
-- | --
-o | 只输出匹配的文本行
-v | 只输出没有匹配的文本行
-c | 统计文件包含文本的次数
-n | 打印匹配的行号
-i | 忽略大小写
-l | 只打印文件名

遍历层级目录
```
[root@t-luhx01-v-szzb ~]# grep "data" . -R -n
```
匹配多个patten时可以采用`-e`选项
```
[root@t-luhx01-v-szzb ~]# grep -e "data" -e "log" test.txt
```

## 2. xargs命令

xargs能够将输入数据转化为特定命令的参数，配合其它命令进行操作。常用选项如下：

选项 | 描述
-- | --
-n | 将单行转化输出多行，-n为每行的字段数
-d | 定义分割符，默认为空格，多行的定义为n
-l {} | 指定替换字符串
-0 | 指定0为输入分割符

单行转为多行
```
[root@t-luhx01-v-szzb ~]# cat test.txt 
11d ds d 1
[root@t-luhx01-v-szzb ~]# cat test.txt | xargs -n 3
11d ds d
1
```
批量删除文件
```
[root@t-luhx01-v-szzb ~]# find ./ -name *.bak | xargs rm -rf
```



## 3. sort命令

sort命令可以用于文本排序，常用选项如下：

选项 | 描述
-- | --
-n | 按数字进行排序
-d | 按字典进行排序
-r | 逆序
-k N | 指定第N列排序

```
[root@t-luhx01-v-szzb ~]# cat test.txt 
1 lu
2 heng
3 xing
[root@t-luhx01-v-szzb ~]# sort -nrk 1 test.txt 
3 xing
2 heng
1 lu
[root@t-luhx01-v-szzb ~]# sort -drk 2 test.txt 
3 xing
1 lu
2 heng
```
此外，sort排序后可以通过`uniq`命令消除重复行
```
[root@t-luhx01-v-szzb ~]# cat test.txt 
1 lu
2 heng
3 xing
3 xing
[root@t-luhx01-v-szzb ~]# sort test.txt | uniq -d
3 xing
[root@t-luhx01-v-szzb ~]# sort test.txt | uniq -c
      1 1 lu
      1 2 heng
      2 3 xing   /*出现了2次*/
[root@t-luhx01-v-szzb ~]# sort test.txt | uniq
1 lu
2 heng
3 xing
```



## 4. tr命令

tr命令用于转换或删除文件中的字符，常用参数如下：

选项 | 描述
-- | --
-c | 条件取反
-d | 删除指定字符
-s | 将连续重复的字符转化为单个字符
-t | 缩减SET的长度与SET2相等

tr支持的字符类包含以下内容：
- [:alnum:]：字母和数字
- [:alpha:]：字母
- [:blank:]：水平空格
- [:cntrl:]：控制字符
- [:digit:]：数字
- [:graph:]：可打印字符(不含空格)
- [:lower:]：小写字母
- [:print:]：可打印字符(含空格)
- [:punct:]：标点符号
- [:space:]：水平空格符和垂直空格符
- [:upper:]：大写字母
- [:xdigit:]：16进制的数字
- [=CHAR=]：指定的字符

**示例**

将文件的中的小写全部转换为大写输出
```
[root@t-luhx01-v-szzb ~]# cat test.txt | tr a-z A-Z
1 LU
2 HENG
3 XING
3 XING
```
也可以通过[:lower:] [:upper:]的方式实现
```
[root@t-luhx01-v-szzb ~]# cat test.txt | tr [:lower:] [:upper:]
1 LU
2 HENG
3 XING
3 XING
```
删除数字
```
[root@t-luhx01-v-szzb ~]# cat test.txt | tr -d [:digit:]
 lu
 heng
 xing
 xing
```



## 5. cut命令

cut可以用于切割文本，常用选项如下：d

选项 | 描述
-- | --
-b | 以字节为单位进行分割
-c | 以字符为单位进行分割
-d | 自定义分割符，默认为制表符
-f | 指定显示区域
-n | 取消分割多字节字符，仅与-b选项配合使用

cut取值范围：
- N-：第N个字段到结尾
- -M：第一个字段到M
- N-M：N到M字段


**示例**

查看第二列数据
```
[root@t-luhx01-v-szzb ~]# cut -f2 -d" " test.txt 
lu
heng
xing
xing
```
查看第一个到第三个字符
```
[root@t-luhx01-v-szzb ~]# cut -c1-3 test.txt 
1 l
2 h
3 x
3 x
```



## 6. paste命令

paste可以用于将两个文件的内容合并到一起
```
paste file1 file2
```
输出结果默认的分割符为制表符，可以用-d指定



## 7. sed命令

sed可以高效处理指定文本操作
```
[root@t-luhx01-v-szzb ~]# sed 's/xing/luhengxing/g' test.txt 
1 lu
2 heng
3 luhengxing
3 luhengxing
```
默认只是替换输出结果，如果需要替换原文件，可以使用`-i`选项
```
[root@t-luhx01-v-szzb ~]# sed -i 's/xing/luhengxing/' test.txt 
[root@t-luhx01-v-szzb ~]# cat test.txt 
1 lu
2 heng
3 luhengxing
3 luhengxing
```
移除空白行
```
[root@t-luhx01-v-szzb ~]# sed '/^$/d' test.txt
```
已匹配的字符串可以通过`&`来引用
```
[root@t-luhx01-v-szzb ~]# echo 'this is example' | sed 's/\w\+/[&]/g'
[this] [is] [example]
```
> sed支持正则表达式，上面的\w\\+就表示每个匹配到的字符串

sed除了替换的操作，还包含其它动作：
- a：新增，可以用于追加数据(下一行)
- c：取代
- d：删除
- i：插入(上一行)
- p：打印
- s：替换



## 8. awk

awk是一个更为强大的文本处理工具，常用的选项如下：

选项 | 描述
-- | --
-F | 指定文件分割符，可以是字符串或正则表达式
-v | 定义一个用户变量
-f | 从脚本中执行

**awk的脚本结构**
```
awk 'BEGIN{ text1 } text2 END{ text3 }'
```
执行过程如下：
- 执行BEGIN中的语句块
- 从文件或stdin中读入一行，执行text2，重复该过程直到最后一行
- 执行END语句块

**内部变量**

变量 | 描述
-- | --
$n | 当前行的第n个字段，字段由FS分割
$0 | 完整的输入记录
ARGC | 命令行参数数量
ARGIND | 命令行的所出的位置
ARGV | 包含命令行参数的数组
CONVFMT | 数字转换格式
ERRNO | 最后的系统错误描述
FIELDWIDTHS | 字符宽度列表
FILENAME | 文件名
FNR | 文件计数的行号
FS | 字段分割符，默认为空格
IGNORECASE | 为true时忽略大小写
NF | 行记录的字段数目
NR | 利用行号获取已读的记录数
OFMT | 数字的输出格式，默认为%.6g
OFS | 输出字段的分割符
ORS | 输出记录分割符，默认为换行符
RLENGTH | 由match函数匹配的字符串长度
RS | 记录分割符，默认为换行符
RSTART | 由match函数匹配的字符串第一个位置
SUBSEP | 数组下标分割符，默认为/034

**内部函数**

函数 | 描述
-- | --
index(string,search_string) | 返回search_string在string中出现的位置
sub(regex,replacement_str,string) | 将正则表达式匹配到的第一处内容替换为replacement_str
match(regex,string) | 检查正则表达式是否能够匹配字符串
length(string) | 返回字符串长度

**示例**

查看文件第二列数据
```
[root@t-luhx01-v-szzb ~]# awk '{print $2}' test.txt 
lu
heng
luhengxing
luhengxing
```
统计文件的行数
```
[root@t-luhx01-v-szzb ~]# awk ' END {print NR}' test.txt 
4
```
累加第一个字段的数字
```
[root@t-luhx01-v-szzb ~]# cat  test.txt | awk 'BEGIN{sum=0;}{sum+=$1;}END{print sum}'
9
```
指定输出分割符
```
[root@t-luhx01-v-szzb ~]# awk '{print $1,$2 }' OFS="|" test.txt 
1|lu
2|heng
3|luhengxing
3|luhengxing
```

通过正则匹配字符串的行
```
[root@t-luhx01-v-szzb ~]# awk '/lu/' test.txt 
1 lu
3 luhengxing
3 luhengxing
```
打印9*9乘法表
```
[root@t-luhx01-v-szzb ~]# seq 9 | sed 'H;g' | awk -v RS='' '{for(i=1;i<=NF;i++)printf("%dx%d=%d%s", i, NR, i*NR, i==NR?"\n":"\t")}'
1x1=1
1x2=2	2x2=4
1x3=3	2x3=6	3x3=9
1x4=4	2x4=8	3x4=12	4x4=16
1x5=5	2x5=10	3x5=15	4x5=20	5x5=25
1x6=6	2x6=12	3x6=18	4x6=24	5x6=30	6x6=36
1x7=7	2x7=14	3x7=21	4x7=28	5x7=35	6x7=42	7x7=49
1x8=8	2x8=16	3x8=24	4x8=32	5x8=40	6x8=48	7x8=56	8x8=64
1x9=9	2x9=18	3x9=27	4x9=36	5x9=45	6x9=54	7x9=63	8x9=72	9x9=81
```

使用getline将命令结果读入变量中
```
[root@t-luhx01-v-szzb ~]# echo | awk '{"grep root /etc/passwd" | getline cmdout; print cmdout }'
root:x:0:0:root:/root:/bin/bash
```



## 9. 循环处理

迭代文件每一行数据
```
[root@t-luhx01-v-szzb ~]# while read line;
> do
> echo $line;
> done < test.txt 
1 lu
2 heng
3 luhengxing
3 luhengxing
```
迭代每一个字段
```
[root@t-luhx01-v-szzb ~]# export line='Abcd 123#'
[root@t-luhx01-v-szzb ~]# for word in $line; do echo $word; done
Abcd
123#
```
迭代每一个字符
```
[root@t-luhx01-v-szzb ~]# export word='abc'
[root@t-luhx01-v-szzb ~]# for((i=0;i<${#word};i++)); 
> do
> echo ${word:i:1};
> done
a
b
c
```
