---
title: JavaScript的Generator和自动执行
layout: post
categories: JavaScript
tags: JavaScript Generator 自动执行
author: John He
---

* content
{:toc}

## thunk函数

thunk函数的定义：能将执行结果传入回调函数，并将该回调函数返回的函数。(呃, 太抽象了...)

通常拿readFile来举例。下面是一个名为readFile的thunk函数:
```JavaScript
var readFile = function (filename){
    return function (callback){
        return fs.readFile(filename, callback)
    }
}
```

readFile如何调用和执行呢？如下:
```JavaScript
readFile('./package.json')((err, str) => {
    console.log(str.toString())
    })
```

可以看到, 在调用readFile的thunk函数后, 异步操作返回结果的获取权, 被交给了thunk函数的调用者。

这便是thunk函数的关键优势: 它将异步操作返回结果的获取权交给了thunk函数的返回值, 而不是把异步操作返回结果的获取权留在thunk函数本身的作用域内。

这一点优势很重要, 能结合Generator语法让Generator函数自动执行。

## Generator函数

Generator是从ECMAScript6里引入的新概念。

一个Generator函数示例:

```JavaScript
function* readFiles(){
    let r1 = yield readFile('./package.json')
    let r2 = yield readFile('./pom.xml')
}
```

如果要调用这个Generator函数使其执行完毕并获取结果, 可以用下面的代码:

```JavaScript
let g = readFiles()
let r1 = g.next()
r1.value(function (err, data){
    if (err){
        throw err
    }
    let r2 = g.next(data)
    r2.value(function (err, data){
        if(err){
            throw er
        }
        g.next(data)
        })
    })
```

可以看到, 上面的代码有回调函数嵌套, 而且代码有很多重复.
Generator自动执行应运而生。

## Generator自动执行

Generator自动执行器的核心代码如下:

```JavaScript
function run(fn){
    let gen = fn();
    function next(err, data){
        let result = gen.next(data)
        if (result.done){
            return
        }
        result.value(next)
    }
    next()
}
```

其核心逻辑是一个递归, 最终是将Generator函数中所有的yield都执行完毕.

调用就是简单的一行:

```JavaScript
run(readFiles)
```

可见, 比起直接回调函数嵌套调用readFiles简单直观。

### 使用Promise代替thunk函数

第一部分的readFile thunk函数可以改写为基于Promise的方式:

``` JavaScript
function readFile(fileName){
    return new Promise((resolve, reject) => {
        fs.readFile(fileName, (error, data) => {
            if(error){
                reject(error)
            } else {
                resolve(data)
            }
        })
    });
}
```

和第一部分的thunk函数相比大同小异, 只是把函数返回值的获取权以Promise的方式交出.

https://github.com/BriteSnow/node-async6 实现了Generator自动执行器.
