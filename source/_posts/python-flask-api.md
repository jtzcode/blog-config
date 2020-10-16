---
title: Python Web API 入门 —— Flask实现
date: 2020-09-28 20:51:24
categories:
    - 技术
tags:
    - Tech
    - Python
    - Flask
---
今天分享一下用Flask框架实现一套简易的Python API Service。Flask是一个轻量级的Web框架，它有一个较小的核心库，并具备良好的[扩展性](https://flask.palletsprojects.com/en/1.1.x/design/)。相比于[Django](https://www.djangoproject.com/)，这个框架更适合初学者上手，也更像python的编程风格。

本文通过实现一个获取书签的简单功能，学习如何使用Flask实现API的设计。如果想要深入学习FLASK框架，建议参考官方文档以及其他网络教程。

### 启动服务器
我们先来搭建一个简易的HTTP Server，用于host这个API服务：

```python
import flask
app = flask.Flask(__name__)
app.config["DEBUG"] = True
app.run()
```
启动一个Server的过程很简单，只需导入flask库，然后实例化一个app对象，并调用`run`方法。运行后，该Server会默认监听`5000`端口，可以通过localhost来访问和测试。
<!--more-->
### 简单请求
在没有连接数据库前，我们先用测试数据构建一个简单的GET请求：
```python
from flask import request, jsonify
@app.route('/', methods=['GET'])
def home():
    return "<h1>Hello Flask</h1><p>This is an API prototype in Python.</p>"

@app.route('/api/v1/bookmarks/all', methods=['GET'])
def getAllBookmarks():
    return jsonify(bookmarks)
```
其中，bookmarks数据来自定义：
```python
bookmarks = [
    {
        'id': '0',
        'name': 'python',
        'children': [
            {
                'id': '00',
                'name': 'python offical',
                'link': 'https://www.python.org/'
            },
            {
                'id': '01',
                'name': 'python webapi',
                'link': 'https://programminghistorian.org/en/lessons/creating-apis-with-python-and-flask'
            }
        ]
    },
    {
        'id': '1',
        'name': 'python org',
        'link': 'https://www.python.org/'
    }
]
```
可以看到，我们通过**装饰器方法**（decorator）`route`来定义API的路由，根路由由home方法处理，获取所有书签的路由则由`getAllBookmarks`方法处理。我们还通过导入`jsonify`函数来将python列表转为标准的JSON格式。

### 参数化请求
一个常见的需求是，我们要通过一个参数来返回相应的数据，比如通过id来查找bookmark：
```python
import util
@app.route('/api/v1/bookmarks', methods=['GET'])
def getBookmarkById():
    if 'id' in request.args:
        id = request.args['id']
    else:
        return "Error: No ID field is specified."

    results = []
    linkResults = []
    utils.flattenBookmarks(bookmarks, linkResults)
    print(f"Flatten results: {len(linkResults)}")
    for link in linkResults:
        if link['id'] == id:
            results.append(link)
    
    return jsonify(results)
```
这里，我们通过request.args字段来获取URL中的查询字符串，并从中解析出参数。`args`是个字典，解析的方式很直接。这段程序还引用了`util`模块，目的是调用`flattenBookmarks`方法将书签数据结构摊平，因为书签栏中书签可能是由层次结构的。该模块方法与API实现无关，可参考[这里](https://github.com/jtzcode/flask-in-action/blob/master/utils.py)。

### 连接数据库
下面我们以`sqlite`数据库为例，学习FLASK API如何通过连接数据库返回数据:
```python
import sqlite3
@app.route('/api/v1/bookmarks/all', methods=['GET'])
def getAllBooks(): 
    conn = sqlite3.connect('flask-in-action/books.db')
    conn.row_factory = dict_facotry
    cur = conn.cursor()
    allBookmarks = cur.execute('SELECT * FROM books;').fetchall()
    return jsonify(allBookmarks)
```
首先要导入`sqlite3`模块，然后通过`connect`方法连接数据库，并在参数中指定数据库文件的路径。如果是在本地调试，要注意.db文件的路径，虽然它与api定义的的python文件在同一级，但在**server上却要通过完整的路径**找到，即父目录`flask-in-action/`不能省略。

为了获取查询结果，需要给`connection`对象指定一个`row_factory`方法，该方法被`connection`内部调用，决定了如何读取每一行的数据：

```python
def dict_facotry(cursor, row):
    d = {}
    for idx, col in enumerate(cursor.description):
        d[col[0]] = row[idx]
    return d
```
可以看到，最终的查询是通过`cursor`对象完成的。关于sqlite3模块，可以参考其[官网](https://docs.python.org/2/library/sqlite3.html)获取详细的用法。

### 条件查询
类似地，如果需要根据参数查询结果，可以Filter：

```python
@app.route('/api/v1/books', methods=['GET'])
def getBookByFilter():
    queryStrings = request.args
    id = queryStrings.get('id')
    published = queryStrings.get('published')
    author = queryStrings.get('author')

    query = 'SELECT * FROM books WHERE'
    toFilter = []

    if id:
        query += ' id=? AND'
        toFilter.append(id)
    if published:
        query += ' published=? AND'
        toFilter.append(published)
    if author:
        query += ' author=? AND'
        toFilter.append(author)

    query = query[:-4] + ';'

    conn = sqlite3.connect('flask-in-action/books.db')
    conn.row_factory = dict_facotry
    cur = conn.cursor()

    results = cur.execute(query, toFilter).fetchall()
    return jsonify(results)
```
这次的查询语句使用`WHERE`关键字添加了条件，并把可能的条件放到一个列表`toFilter`中，并一起传递给`execute`方法。

### POST请求
以上的例子都是GET请求，如果是POST请求，可以参考官网的一个用户登录的例子：

```python
@app.route('/login', methods=['POST', 'GET'])
def login():
    error = None
    if request.method == 'POST':
        if valid_login(request.form['username'],
                       request.form['password']):
            return log_the_user_in(request.form['username'])
        else:
            error = 'Invalid username/password'
    # the code below is executed if the request method
    # was GET or the credentials were invalid
    return render_template('login.html', error=error)
```
POST请求的表单数据可以通过`request.form`对象获得，对表单数据处理后可以渲染相应的页面。

### 参考资料
 - https://www.fullstackpython.com/flask.html
 - https://flask.palletsprojects.com/en/1.1.x/
 - https://programminghistorian.org/en/lessons/creating-apis-with-python-and-flask
 - https://docs.python.org/2/library/sqlite3.html
 - https://flask.palletsprojects.com/en/1.1.x/quickstart/
