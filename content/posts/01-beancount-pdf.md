+++
title = "Beancount 记账：从 PDF 中提取表格"
date = 2022-04-25

[taxonomies]
tags = ["beancount", "pdf"]
[extra]
toc = true
+++

Beancount 复式记账的介绍可以参考[这篇文章](https://lug.ustc.edu.cn/planet/2020/08/keeping-account-with-beancount/)，Beancount 最吸引我的是它灵活的账单导入方式。

<!-- more -->

对于借记卡来说，招商银行和浦发银行虽然会在“网上银行”提供 csv 或 xls 格式的交易详情下载，但是不包含交易对方的账号，难以理清自己给自己的转账；中国银行则没有下载按钮。

而它们的手机 APP 都提供了“打印交易流水”功能，可以生成 PDF 文件，内容详细，包含交易对方以及其账号。

正好我最近读了 [*PDF Explained*](https://www.oreilly.com/library/view/pdf-explained/9781449321581/) 这本很棒的 PDF 入门书，便来折腾一下 PDF。

# 原理

这一节参考了 *PDF Explained* 第六章 Text and fonts。

首先我们需要知道:
- PDF 格式中存在字体的概念，并且字体通常会内嵌在文件中。
- PDF 格式只支持小规模的文本布局，比如不含换行且字符间距相等的字符串。
- PDF 内嵌的字体通常会有`/ToUnicode`属性提供将文本转换到 Unicode 的方法。

接下来以某个中文 PDF 为例，从中提取一段字符串。

- 首先使用`pdftk`解压缩:
```bash
pdftk input.pdf output output.pdf uncompress
```
- 使用`vim -b output.pdf`打开，搜索`Tj`找到一处文本，可以看出使用了`F1`字体:
```
BT
1 0 0 1 98.73 588.56 Tm
1 0 0 1 98.73 588.56 Tm
0 Tc
/F1 9.75 Tf
0 0 0 rg
(\\<85>-_)Tj
0 g
1 0 0 1 118.23 588.56 Tm
0 Tc
-19.5 0 Td
ET
```
注意括号中的`\`是转义符号，`<85>`为不可打印字符，所以括号中有 4 个字符`\x5c\x85\x2d\x5f`。
- 搜索`F1`发现字体定义在`6 0`对象:
```
<<
/F1 6 0 R
/F2 7 0 R
>>
```
- 在字体定义中找到`/ToUnicode`指向`10 0`对象:
```
6 0 obj

<<
/Subtype /Type0
/Type /Font
/BaseFont /TNUMLT+SimSun
/Encoding /Identity-H
/DescendantFonts [9 0 R]
/ToUnicode 10 0 R
>>
endobj
```
- 在`10 0`对象的`CMap`中找到`Unicode`映射
```
<5c85><5c85><8d27>
<2d5f><2d5f><5e01>
```
所以这一处文本是`\u8d27\u5e01`，即`货币`。

# Camelot

[Camelot](https://camelot-py.readthedocs.io/) 是一个从 PDF 中提取表格的 Python 库，简单易用，它的工作原理（[Lattice模式](https://camelot-py.readthedocs.io/en/master/user/how-it-works.html#lattice)）是先将 PDF 转换成图片，经过形态学处理得到表格的边框，再根据每个格子的边界提取文本。

中国银行和浦发银行生成的 PDF 边框完整，只需要调整一些参数 Camelot 就能正确提取出数据。

## 中国银行

可以通过`更多 -> 助手 -> 交易流水打印`获取，示例如下：

![中国银行交易流水示例](/images/01/boc.svg)

```python
import datetime
import camelot
from beancount.core.amount import Amount
from beancount.core.number import D

tables = camelot.read_pdf(file.name, pages='all', split_text=True)
for table in tables:
    for line in table.data[1:]:
        raw_entry = []
        for i in line:
            e = replace_newline(i.strip())
            if e.startswith('-' * 3):
                raw_entry.append('')
            else:
                raw_entry.append(e)
        date = datetime.date.fromisoformat(raw_entry[0])
        amount = Amount(D(raw_entry[3]), 'CNY')
        payee = raw_entry[9] + ' ' + raw_entry[10] if raw_entry[10] else raw_entry[9]
        narration = raw_entry[8]
```

其中`split_text=True`是为了防止靠近格子边缘的文本被合并到另一格子，若不指定，示例中的`对方账户名`会提取出`陕西闰能科技有限公司 -------------------`，`对方卡号/账号`则为空。

## 浦发银行

可以通过`我的账户 -> 卡片 -> 打印交易`获取，与中国银行类似，示例如下：

![中国银行交易流水示例](/images/01/spdb.svg)

```python
import datetime
import camelot
from beancount.core.amount import Amount
from beancount.core.number import D

tables = camelot.read_pdf(file.name, pages='all', split_text=True)
for table in tables:
    for line in table.data[1:]:
        lineno += 1
        raw_entry = [replace_newline(i.strip()) for i in line]
        date = raw_entry[0]
        date = datetime.date(int(date[0:4]), int(date[4:6]), int(date[6:8]))
        amount = Amount(D(raw_entry[3]), 'CNY')
        balance = Amount(D(raw_entry[4]), 'CNY')
        payee = raw_entry[5] + ' ' + raw_entry[6]
        narration = raw_entry[2]
```

# pdfminer

## 招商银行

可以通过`收支明细 -> ... -> 打印流水`获取，如下图所示，除了表头之外完全没有边框：

![招商银行交易流水示例](/images/01/cmb.svg)

如果用 Camelot 的 stream 模式处理，得到的数据比较杂乱，所以我转向了 Camelot 依赖的一个库——pdfminer。

[pdfminer](https://pdfminersix.readthedocs.io/) 相对于 Camelot 比较底层，它从 PDF 中提取文本元素（通常 1 行是 1 个元素），根据间距将文本元素组合成段落。我们需要调整间距（`char_margin`和`line_margin`），使得只有各个格子内部的文本被合并。

于是问题转换成找到每个格子的坐标，思路是首先找到表头的两条分割线，然后找到表头各项，得到横坐标，接着找到每一笔交易的日期，得到纵坐标，最后找到所有格子，代码如下：

```python
import re
from pdfminer.high_level import extract_pages
from pdfminer.layout import LAParams, LTLine, LTTextBoxHorizontal

titles = {
    'date':     {'name': '记账日期'},
    'currency': {'name': '货币'},
    'amount':   {'name': '交易金额'},
    'balance':  {'name': '联机余额'},
    'summary':  {'name': '交易摘要'},
    'payee':    {'name': '对手信息'}
}

laparams = LAParams(char_margin=1.2, line_margin=0.3)
pages = extract_pages(file.name, laparams=laparams)
raw_entries = []
for page in pages:
    elements = list(page)
    # Find 2 upper most lines
    line1_y = 0
    line2_y = 0
    for e in elements:
        if isinstance(e, LTLine) and e.y0 == e.y1:
            if e.y0 > line1_y:
                line2_y = line1_y
                line1_y = e.y0
            elif e.y0 != line1_y and e.y0 > line2_y:
                line2_y = e.y0
    # Find x coordinate of columns
    for e in elements:
        y_avg = (e.y0 + e.y1) / 2
        if isinstance(e, LTTextBoxHorizontal) and y_avg < line1_y and y_avg > line2_y:
            for i in titles.values():
                if e.get_text().startswith(i['name']):
                    i['x'] = e.x0
                    continue
    # Find date elements
    date_e = []
    for e in elements:
        if isinstance(e, LTTextBoxHorizontal) and abs(e.x0 - titles['date']['x']) < 0.5
        and re.fullmatch(r'[0-9]{4}-[0-9]{2}-[0-9]{2}', e.get_text().strip()):
            date_e.append(e)
    date_e.sort(key=lambda item: (item.y0 + item.y1) / 2, reverse=True)
    # Find all elements
    raw_entries_page = [{'date': i.get_text().strip()} for i in date_e]
    for e in elements:
        for i, raw_entry in enumerate(raw_entries_page):
            for key, value in titles.items():
                target_y_sum = date_e[i].y0 + date_e[i].y1
                if key != 'date' and isinstance(e, LTTextBoxHorizontal)
                and abs(e.y0 + e.y1 - target_y_sum) < 0.5 and abs(e.x0 - value['x']) < 0.5:
                    raw_entry[key] = replace_newline(e.get_text().strip())
    raw_entries += raw_entries_page
```

# 招商银行理财账单

招商银行的招招宝是类似余额宝的一个产品，不过招招宝的分红只显示在理财账单中。

招商银行理财账单可以通过`财富 -> 理财产品 -> 持仓 -> 其它 -> 理财月度账单 -> x月 -> ... -> 分享到微信`，从链接中获取，同样是 PDF 格式，示例如下：

![招商银行理财账单示例](/images/01/cmb-investment.svg)

看到表格有边框，首先尝试用`Camelot`提取表格，但是`Camelot`却被斜着的“招商银行理财月度账单”水印迷惑了，提取出了很多错误信息。

接下来当然要编辑 PDF 去除水印，先用`pdftk`解压，然后搜索`招`和`单`的 Unicode 码`62db`和`5355`，发现有两个字体包含这两个字:
```
<0525><0525><5355>
<0725><0725><62db>
...
<0524><0524><5355>
<0706><0706><62db>
```

于是在 vim 中分别搜索`(\%x07\%x25.*\%x05\%x25)`和`(\%x07\%x06.*\%x05\%x24)`，搜索到两组文本，其中一组文本的开头含有`cm`操作符，能将文本倾斜，是我们要找的水印:
```
0.75294 0.75294 0.75294 rg
/GS4 gs
q
0.96593 0.25882 -0.25882 0.96593 280 680 cm
BT
1 0 0 1 -290 0 Tm
/F3 58 Tf
0.75294 0.75294 0.75294 rg
(^G^F^E<8b>^K2\n6\b<c2>\n<8f>^G<a4>^F<85>\n<92>^E$)Tj
0 g
ET
```

接下来将括号中的内容替换为空，然后使用`ghostscript`修复 PDF 文件，`Camelot`就能正确提取表格了。

使用`bash`完整的处理过程如下（注意`fish`语法不同，`\\`要用`\\\`替换）:
```bash
pdftk input.pdf output - uncompress \
  | sed -e 's/\x07\x06\x05\x8b\x0b2\\n6\\b\xc2\\n\x8f\x07\xa4\x06\x85\\n\x92\x05\$//g' \
  | gs -o output.pdf -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress -
```
