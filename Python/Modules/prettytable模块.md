[TOC]

---

PrettyTable 是用于生成简单 ASCII 表的 Python 库，例如用于美化数据库查询结果集。

## 生成PrettyTable

### 自定义PrettyTable
**初始化表格**
```python
import prettytable as pt

tab = pt.PrettyTable()
tab.field_names = ["id", "name", "age"]
```

**添加行**
```
tab.add_row([1,'lu','25'])
tab.add_row([1,'heng','28'])
```

**删除行**
```
tab.del_row(1)
```
==注：下标由0开始==

**清除所有行**

```
tab.clear_rows()
```

### 从CSV生成PrettyTable
```
from prettytable import from_csv

with open("data.csv", "r") as fp: 
    tab = from_csv(fp)

print(tab)
```

### 从数据库游标生成PrettyTable
```
import sqlite3 as lite
from prettytable import from_db_cursor

con = lite.connect('data.db')

with con:

    cur = con.cursor()    
    cur.execute('SELECT * FROM Sales')   
    tab = from_db_cursor(cur) 

print(tab)
```

### 从HTML生成PrettyTable

from_html()从一串 HTML 代码生成一个 PrettyTables 列表。 HTML 中的每个\<table>\</table>都成为一个 PrettyTable 对象。 from_html_one()从仅包含单个\<table>\</table>的 HTML 代码字符串中生成 PrettyTable
```
from prettytable import from_html_one

with open("data.html", "r") as fp: 
    html = fp.read()

x = from_html_one(html)
print(x)
```

## 表格输出

### HTML输出
get_html_string()从 PrettyTable 生成 HTML 输出
```
import prettytable as pt

tab = pt.PrettyTable(["City name", "Area", "Population", "Annual Rainfall"])

tab.add_row(["Adelaide",1295, 1158259, 600.5])
tab.add_row(["Brisbane",5905, 1857594, 1146.4])
tab.add_row(["Darwin", 112, 120900, 1714.7])
tab.add_row(["Hobart", 1357, 205556, 619.5])
tab.add_row(["Sydney", 2058, 4336374, 1214.8])
tab.add_row(["Melbourne", 1566, 3806092, 646.9])
tab.add_row(["Perth", 5386, 1554769, 869.4])

print(tab.get_html_string())
```

### 格式化输出

**数据排序**

使用sortby属性，我们将指定要排序的列。reversesort属性控制排序的方向
```
from prettytable as pt

tab = pt.PrettyTable()
tab.field_names = ["City name", "Area", "Population", "Annual Rainfall"]

tab.add_row(["Adelaide", 1295, 1158259, 600.5])
tab.add_row(["Brisbane", 5905, 1857594, 1146.4])
tab.add_row(["Darwin", 112, 120900, 1714.7])
tab.add_row(["Hobart", 1357, 205556, 619.5])
tab.add_row(["Sydney", 2058, 4336374, 1214.8])
tab.add_row(["Melbourne", 1566, 3806092, 646.9])
tab.add_row(["Perth", 5386, 1554769, 869.4])

print("Table sorted by population:")
tab.sortby = "Population"
tab.reversesort = True
print(tab)
```

**表格边框**
vrules为表格内部边框竖线，hrules为表格内部边框横线，可选值为FRAME,ALL,NONE
```
tab.vrules=ALL
tab.hrules=ALL
```

**列长度**
```
tab.max_width=30
tab.min_width=120
```

**对齐方式**
align属性控制字段的对齐方式，可选值为l(left)，c(center)和r(right)
```
tab.align["City name"] = "l"
tab.align["Area"] = "r"
tab.align["Annual Rainfall"] = "r"
```
**get_string方法**

get_string()方法返回当前状态下的表的字符串表示形式，它可以实现下列操作

显示标题
```
print(tab.get_string(title="Australian cities"))
```

选择列
```
print(tab.get_string(fields=["City name", "Population"]))
```

选择行
```
print(tab.get_string(start=1, end=4))
```