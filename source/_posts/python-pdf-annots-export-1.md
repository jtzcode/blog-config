---
title: 一键导出PDF文档中的高亮文字以及笔记（Python实现）
date: 2021-01-12 11:04:29
categories:
    - 技术
tags:
    - Tech
    - Python
    - 工具
    - PDF
---
![python](Python-Tools.png)
## 需求
最近在阅读一些PDF格式的资料，经常会进行划线并做笔记，我希望这些内容在阅读结束之后能够方便地整理出来并回顾，于是探索了一下到处划线文字和笔记的方法。

首先，我去确认PDF阅读器是否提供了需要的功能。以Foxit Reader为例，它的确提供了评论导出的功能，但是有局限性：
- 只能导出高亮的文字到文本文件
- 笔记内容只能导出到`.fdf`格式，这就需要后续处理
<!--more-->
而我希望的功能是将**高亮文字以及其对应的笔记**导出到**文本文件**，或者上传到某个**App的云端**。因此，我决定自己去实现，并在网上找了一些已有的Python模块看是否能满足功能。

## 工具
### PyPDF2
这是很常见的操作PDF文档的Python库，可以提供文档读写、编辑、剪裁、加密解密等功能。如果利用一些比较底层的API，还可以操作文档中的评论，比如：
```python
from PyPDF2 import PdfFileReader
pypdf_doc = PdfFileReader(open('test.pdf', "rb"))
pypdf_page = pypdf_doc.getPage(0)
if '/Annots' in pypdf_page:
	print("Page comments num: %d" % len(pypdf_page["/Annots"]))
	for annot in pypdf_page['/Annots'] :
	    subtype = annot.getObject()['/Subtype']
	    if subtype == "/Highlight":
	        print(annot.getObject()['/Contents'])
```
上面这段代码可以读取一个PDF文档中某一页的所有评论（弹出框中的笔记），但是不包含被高亮的文字。事实上，PDF文档并没有特意保存高亮文字的信息，而只是**在文档的特定位置画出特定样式的矩形框**而已。对于PyPDF2，我也没有找到提取高亮文字的方案。

### PyMuPDF
于是我又找到并尝试了PyMuPDF库。这个库正好补充了PyPDF2的不足：它可以巧妙地分析出高亮矩形框的位置，并读取其中的文字：
```python
import fitz
mupdf_doc = fitz.open('test.pdf')
mupdf_page = mupdf_doc.loadPage(0)
wordlist = mupdf_page.getText("words")  # list of words on page
wordlist.sort(key=lambda w: (w[3], w[0]))  # ascending y, then x
for annot in mupdf_page.annots():
    # underline / highlight / strikeout / squiggly : 8 / 9 / 10 / 11
    if annot.type[0] == 8:
    	print(_parse_highlight(annot, wordlist))

def _parse_highlight(annot, wordlist)
    points = annot.vertices
    quad_count = int(len(points) / 4)
    sentences = ['' for i in range(quad_count)]
    for i in range(quad_count):
        r = fitz.Quad(points[i * 4: i * 4 + 4]).rect
        words = [w for w in wordlist if fitz.Rect(w[:4]).intersects(r)]
        sentences[i] = ' '.join(w[4] for w in words)
    sentence = ' '.join(sentences)
    return sentence
```
该工具集成于`fitz`库中，需要先安装`fitz`。从上面代码可以看出，本质上，提取高亮的文字还是需要获取高亮矩形框的位置和大小，并从中提取单词，最后拼接成句。遗憾的是，该库反而没有提供比较好的获取弹窗中笔记评论的方法（可能是我没找到😂）。不过这已经与PyPDF2工具形成互补了。

两种工具结合，我就写出了一个初步的自定义工具。该工具读取一个PDF文档，并提取出每一页中的高亮文字和笔记评论。详细代码见[这里](https://github.com/jtzcode/python-useful-tools/blob/master/pdf-comments-extractor/app.py)。

后续的工作可以包括格式化输出笔记内容，将结果保存至云端（如印象笔记）等。
### 其他工具
我还探索了其他工具比如[Python Poppler](https://github.com/cbrunet/python-poppler)。从代码看，似乎提取注释的功能更优雅简洁，但是这个库的安装过程比较麻烦，并对系统有一定要求。感兴趣的朋友可以尝试一下。另外，如果有人发现了更简洁的方式满足上面的需求，欢迎留言讨论！
## 参考
- https://github.com/mstamy2/PyPDF2
- https://github.com/pymupdf/PyMuPDF
- https://github.com/pymupdf/PyMuPDF/issues/318
- https://stackoverflow.com/questions/1106098/parse-annotations-from-a-pdf
- https://github.com/cbrunet/python-poppler