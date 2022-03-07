---
title: Web 键盘输入法应用开发指南——实战（一）
date: 2022-03-07 14:33:48
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 介绍
从这篇文章开始，我们通过一个小项目来实践键盘和输入法相关的开发要点。这是一个在线输入法（Online IME）工具，功能类似Google提供的一个在线输入工具[1]。有了这类工具，你可以在Web页面里面直接使用输入法输入，而不依赖本地设备是否安装输入法。完整代码可以访问[这里](https://github.com/jtzcode/online-ime)[2]。

![google-input](google-input.png)
<center><div style="font-size:16px;">Google Input Tools</div></center>

<!--more-->
## 功能与技术点
这个在线输入法工具有以下功能点：
- 支持简体中文的输入
- 支持中文输入法基本输入功能（组字、选择、提交、取消等）
- 支持数字和符号的输入
- 构造类似本地输入法的UI体验
- 屏蔽功能键防止干扰输入
- 支持中英文切换
- 其他功能

涉及技术点有：
- 键盘事件的处理
- 输入法事件的处理
- 组合键的处理
- UI控件的操作
- 输入法服务端的实现

这篇文章我们先实现最基本的功能，然后在下篇文章中丰富一些细节。这里提一下服务端的实现，因为我们相当于自己实现了一个输入法，因此需要一个数据库来提供**输入字符到候选字词的映射**。一般本地安装的输入法（比如搜狗、百度等）都会有自己的词库，而我们这个是在线的输入，因此需要通过API来获取相应的字词。这里我直接使用了**Google Input Tools提供的公开API**。

## 基本实现
我们的实现分前后两端。前端会展示在线输入法基本的UI界面，包括输入的文本框、组字框和候选框。然后给文本框绑定需要的事件（比如keydown），捕获并将用户**输入的拼音**发送到服务端。

服务端会调用Google的输入服务返回候选词并返回。从服务端拿到候选词后，前端解析结果并调整UI的样式（展示、移动候选框）。最后还要提交用户输入的结果到文本框完成输入。

### 前端实现
首先我们需要一个textarea来测试输入。为了模拟本地输入法的UI，我们还需要一个组字框（ID是`ime-buffer`，用于容纳拼音），和一个候选框（ID是`candidate-contaier`，用于容纳候选字词列表）：
```html
<div id ="app-container">
    <textarea id="input-area"></textarea>
    <div id="ime-container" class="ime-container">
        <span id="ime-buffer" contenteditable="true" class="ime-buffer" spellcheck="false" tabindex="0"></span>
    </div>
    <div id="candidate-contaier"></div>
</div>
```
这里我用了一个可编辑（`contenteditable=true`）的span，当然你也可以直接用input等控件。接着我们在textarea上绑定一些键盘事件，用于监听用户的输入：
```javascript
...
inputArea.addEventListener('keydown', evt => {
    console.log("keydown: ", evt);
    currentCursorPos = getCaretCoordinates(inputArea, inputArea.selectionEnd);
    console.log('Curret cursor at: ', currentCursorPos);      
});
...
```
这里我们还调用了一个`getCaretCoordinates`用于获取当前输入光标在屏幕上的位置，这可以帮助我们移动组字框和候选列表的位置，使其用于跟随光标，获得好的用户体验。不过获取光标位置的操作容易有兼容性问题，因此我采用了一个第三方的实现，可以参考这里[3]。每次用户有输入，我们就更新一下当前光标的位置。

随后就要把输入的拼音发送给服务端处理：
```javascript
const url = `/candidate?text=${text}`;
isIMEActive = true;
fetch(url).then(res => {
    if (res.status === 200) {
        res.json().then(data => {
            console.log("Candidate data: ", data);
        });
    }
});
```
我们通过`/candidate`路由处理发送过来的`text`，并将结果转换为JSON对象。注意这里还使用了一个`isIMEActive`变量，用于记录**当前IME是否启用**的状态。因为正在使用输入法与未使用输入法时，键的表现是不同的。比如，直接输入空格键就会得到一个空格，而在输入法启用时，它会提交当前第一个候选词，并不会产生空格。

当服务器返回结果时，我们要先用数据填充候选框，并将组字框和候选框调整到光标的位置：
```javascript
function setCandidates(data) {
    let dataArray = JSON.parse(data);
    currentCandidates = dataArray[1][0][1] || [];
    let resultStr = "";
    currentCandidates.forEach((candidate, index) => {
        resultStr += `${index + 1}. ${candidate} `;
    });
    candidateWindow.innerText = resultStr;
    moveCandidateWindow();
}

function moveCandidateWindow() {
    ...
    imeContainer.style.left = imeLeft + 'px';
    imeContainer.style.top = imeTop + 'px';
    candidateWindow.style.left = candidateLeft + 'px';
    candidateWindow.style.top = candidateTop + 'px';
}
```
最后前端还要处理用户提交输入的过程，包括按数字键选择目标字词、按空格键选择第一个词和按回车键取消选择。此时应该更新textarea的内容，并情况所有的控件：
```javascript
function endComposition(index) {
    let isEnter = index === undefined ? true : false
    inputArea.value += (!isEnter ? currentCandidates[index - 1] : textInput.innerText);
    currentCandidates = [];
    textInput.innerText = "";
    textBuffer = "";
    candidateWindow.innerText = "";
    isIMEActive = false;
    ...
}
```
以上就是前端的大致实现。

### 服务端实现
对于服务端来说，先利用express起一个http server，并监听一个端口：
```javascript
const express = require('express');
const https = require('https');

const app = express();
app.listen(2022);

...
```

然后发起对Google Input Tool的API请求即可：
```javascript
...
const options = {
    hostname: 'inputtools.google.com',
    port: 443,
    path: '/request?itc=zh-t-i0-pinyin&num=11&cp=0&cs=1&ie=utf-8&oe=utf-8&app=demopage',
    method: 'GET'
};
const requestIMECandidate = function(req, res, callback) {
    const text = req.query.text;
    options.path += `&text=${text}`;
    return https.request(options, res => {
        let body = '';
        res.on('data', chunk => {
            body = body + chunk;
        });
        res.on('end',function(){
            if (res.statusCode != 200) {
              callback("Api call failed with response code " + res.statusCode);
            } else {
              callback(body);
            }
        });
    });
};
```
## 总结
以上实现只是覆盖了基本的功能，搭建出了基本的UI和事件处理框架。下一篇文章我会关注其中的一些细节，比如输入法功能键、组合键的处理，以及其他注意事项。最后的实现结果预览如下：

![demo](demo.png)
<center><div style="font-size:16px;">在线输入法Demo</div></center>

## 参考阅读
[1] [Google Input Tools](https://www.google.com/inputtools/try/)
[2] [Online-IME Demo](https://github.com/jtzcode/online-ime)
[3] [Get Caret Position](https://github.com/component/textarea-caret-position)