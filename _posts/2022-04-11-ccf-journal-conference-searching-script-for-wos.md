---
layout: post
title: CCF Journals & Conferences Searching Script for WOS
description: This is post by Yuxuan Yang
date: 2022-04-11 20:23:00
hero_image: https://images.unsplash.com/photo-1498050108023-c5249f4df085?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=1066&q=80
hero_height: is-large
hero_darken: true
image: https://images.unsplash.com/photo-1498050108023-c5249f4df085?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=1066&q=80
tags: resource jekyll docs
---

分享一个用于生成CCF期刊/会议检索式的Python脚本。

## Python脚本

```python
from bs4 import BeautifulSoup
import pandas as pd

def parse_df():
    soup = BeautifulSoup(open("中国计算机学会推荐国际学术会议和期刊目录（第五版）.html", encoding='UTF-8'), 'html.parser')
    cf_table, j_table = soup.find_all('div', id='ccf')  # 找到会议和期刊表格
    cf_lines = cf_table.tbody.findAll('tr')
    j_lines = j_table.tbody.findAll('tr')
    lines = cf_lines + j_lines  # 合并会议表格和期刊表格

    # 提取单元格值
    table = []
    for line in lines:
        cells = line.findAll('td')
        short, full, rank, type_, field = list(map(lambda cell: cell.get_text(), cells))
        table.append([short, full, rank, type_, field])

    # 放入dataframe中
    df = pd.DataFrame(table, columns=['short', 'full', 'rank', 'type', 'field'])

    return df

def filter_table(df, target, ranks, types, fields):
    df = df.copy()

    def filter_col(df, col, values, fuzzy=False):
        if not isinstance(values, list):
            assert isinstance(values, str)
            if fuzzy:
                return df[df[col].str.contains(values)]
            else:
                return df[df[col] == values]
        else:
            results = []
            for value in values:
                assert isinstance(value, str)
                if fuzzy:
                    results.append(df[df[col].str.contains(value)])
                else:
                    results.append(df[df[col] == value])
            return pd.concat(results, axis=0)

    df = filter_col(df, 'rank', ranks)
    df = filter_col(df, 'type', types)
    df = filter_col(df, 'field', fields, fuzzy=True)

    return list(df[target])

def opt4wos(filter_list):
    new_list = []
    for name in filter_list:
        name = name.upper()
        # 跳过空字符串（部分期刊无简称的情况）
        if name == '':
            continue
        # 替换Trans缩写为全称
        name = name.replace(' TRANS ', ' TRANSACTIONS ')
        # 给AND OR NOT加引号
        name = name.replace(' AND ', ' "AND" ')
        name = name.replace(' OR ', ' "OR" ')
        name = name.replace(' NOT ', ' "NOT" ')
        new_list.append(name)
    return new_list

if __name__ == '__main__':
    # 为wos生成检索式
    df = parse_df()

    """
    领域列表：
    '计算机体系结构/并行与分布计算/存储系统',
    '计算机网络',
    '网络与信息安全',
    '软件工程/系统软件/程序设计语言',
    '数据库/数据挖掘/内容检索',
    '计算机科学理论',
    '计算机图形学与多媒体',
    '人工智能',
    '人机交互与普适计算',
    '交叉/综合/新兴'
    """

    # 1. 会议
    # target: short（缩写）, full（全称），部分期刊没有缩写
    # ranks: A, B, C
    # types: 会议, 期刊
    # fields: 在领域列表中选择，支持部分匹配
    cf = filter_table(df, target='short', ranks='A', types='会议', fields=['数据库', '人工智能', '多媒体', '交叉'])  # 更改这一行
    cf = opt4wos(cf)
    print('会议检索式：')
    print("CF=(" + ' OR '.join(cf) + ")")

    # 2. 期刊（部分期刊没有缩写）
    j = filter_table(df, target='full', ranks=['A', 'B'], types='期刊', fields=['人工智能', '交叉']) # 更改这一行
    j = opt4wos(j)
    print('期刊检索式：')
    print("SO=(" + ' OR '.join(j) + ")")
```

Python环境需要装有Pandas和BeautifulSoup库，可以用以下命令安装：  
```bash
pip install pandas
pip install beautifulsoup4
```

## 用法

根据自己需要修改`filter_table(df, target=..., ranks=..., types=..., fields=...)`这一行代码。

`target`可以是`'full'`或`'short'`，指输出的期刊/会议使用全称（full）还是简称（short），注意部分期刊没有简称，因此对于期刊建议使用全称（full）。

`ranks`可以是`'A'`, `'B'`, `'C'`或包含其中多个的列表，代表CCF-A, B, C三个类别。

`types`可以是`'期刊'`或`'会议'`。

`fields`指的是期刊/会议的分类，应在领域列表中选择，支持部分匹配。

> 领域列表：  
> '计算机体系结构/并行与分布计算/存储系统', '计算机网络', '网络与信息安全', '软件工程/系统软件/程序设计语言', '数据库/数据挖掘/内容检索', '计算机科学理论', '计算机图形学与多媒体', '人工智能', '人机交互与普适计算', '交叉/综合/新兴'

## 例子

例如想要搜索CCF-A和CCF-B类，在数据库/数据挖掘/内容检索、计算机图形学与多媒体、人工智能、交叉/综合/新兴领域的会议论文，则可以将filter_table函数写为：

```python
filter_table(df, target='short', ranks='A', types='会议', fields=['数据库', '人工智能', '多媒体', '交叉'])
```

输出结果为：

```text
会议检索式：
CF=(SIGMOD OR SIGKDD OR ICDE OR SIGIR OR VLDB OR AAAI OR NEURIPS OR ACL OR CVPR OR ICCV OR ICML OR IJCAI OR ACM MM OR SIGGRAPH OR VR OR IEEE VIS OR WWW OR RTSS)
......
```

可以将该检索式与其他字段的检索式结合，在Web of Science中使用高级检索进行查询。