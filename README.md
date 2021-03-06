EventProxy [![Build Status](https://secure.travis-ci.org/JacksonTian/eventproxy.png)](http://travis-ci.org/JacksonTian/eventproxy)
======

> 这个世界上不存在所谓回调函数深度嵌套的问题。 —— [Jackson Tian](http://weibo.com/shyvo)

> 世界上本没有嵌套回调，写得人多了，也便有了`}}}}}}}}}}}}`。 —— [fengmk2](http://fengmk2.github.com)

* API文档: [API Documentation](http://html5ify.com/eventproxy/api.html)
* jscoverage: [95%](http://fengmk2.github.com/coverage/eventproxy.html)

EventProxy 仅仅是一个很轻量的工具，但是能够带来一种事件式编程的思维变化。有几个特点：

1. 利用事件机制解耦复杂业务逻辑
2. 移除被广为诟病的深度callback嵌套问题
3. 将串行等待变成并行等待，提升多异步协作场景下的执行效率
4. 友好的Error handling
5. 无平台依赖，适合前后端，能用于浏览器和Node.js
6. 兼容CMD，AMD以及CommonJS模块环境

现在的，无深度嵌套的，并行的

```
var ep = EventProxy.create("template", "data", "l10n", function (template, data, l10n) {
  _.template(template, data, l10n);
});

$.get("template", function (template) {
  // something
  ep.emit("template", template);
});
$.get("data", function (data) {
  // something
  ep.emit("data", data);
});
$.get("l10n", function (l10n) {
  // something
  ep.emit("l10n", l10n);
});
```

过去的，深度嵌套的，串行的。

```
var render = function (template, data) {
  _.template(template, data);
};
$.get("template", function (template) {
  // something
  $.get("data", function (data) {
    // something
    $.get("l10n", function (l10n) {
      // something
      render(template, data, l10n);
    });
  });
});
```
## 安装
### Node用户
通过NPM安装即可使用：

```bash
$ npm install eventproxy
```

调用:

```
var EventProxy = require('eventproxy');
```
### 前端用户
以下示例均指向Github的源文件地址，您也可以[下载源文件](https://raw.github.com/JacksonTian/eventproxy/master/lib/eventproxy.js)到你自己的项目中。整个文件注释全面，带注释和空行，一共400多行。为保证EventProxy的易嵌入，项目暂不提供压缩版。用户可以自行采用Uglify、YUI Compressor或Google Closure Complier进行压缩。

#### 普通环境
在页面中嵌入脚本即可使用：

```
<script src="https://raw.github.com/JacksonTian/eventproxy/master/lib/eventproxy.js"></script>
```
使用：

```
// EventProxy此时是一个全局变量
var ep = new EventProxy();
```

#### SeaJS用户
SeaJS下只需配置别名，然后`require`引用即可使用。

```
// 配置
seajs.config({
  alias : {
    'eventproxy' : 'https://raw.github.com/JacksonTian/eventproxy/master/lib/eventproxy.js'
  }
});
// 使用
seajs.use(['eventproxy'], function (EventProxy) {
  // TODO
});
// 或者
define('test', function(require, exports, modules) {
  var EventProxy = require('eventproxy');
});
```
#### RequireJS用户
RequireJS实现的是AMD规范。

```
// 配置路径
require.config({
  paths: {
    "eventproxy": "https://raw.github.com/JacksonTian/eventproxy/master/lib/eventproxy.js"
  }
});
// 使用
require(["eventproxy"], function(EventProxy) {
  // TODO
});
```
## 异步协作
### 多类型异步协作
此处以页面渲染为场景，渲染页面需要模板、数据。假设都需要异步读取。

```
var ep = new EventProxy();
ep.all('tpl', 'data', function (tpl, data) {
  // 在所有指定的事件触发后，将会被调用执行
  // 参数对应各自的事件名
});
fs.readFile('template.tpl', 'utf-8', function (err, content) {
  ep.emit('tpl', content);
});
db.get('some sql', function (err, result) {
  ep.emit('data', result);
});
```
`all`方法将handler注册到事件组合上。当注册的多个事件都触发后，将会调用handler执行，每个事件传递的数据，将会依照事件名顺序，传入handler作为参数。
#### 快速创建
EventProxy提供了`create`静态方法，可以快速完成注册`all`事件。

```
var ep = EventProxy.create('tpl', 'data', function (tpl, data) {
  // TODO
});
```
以上方法等效于

```
var ep = new EventProxy();
ep.all('tpl', 'data', function (tpl, data) {
  // TODO
});
```
### 重复异步协作
此处以读取目录下的所有文件为例，在异步操作中，我们需要在所有异步调用结束后，执行某些操作。

```
var ep = new EventProxy();
ep.after('got_file', files.length, function (list) {
  // 在所有文件的异步执行结束后将被执行
  // 所有文件的内容都存在list数组中
});
for (var i = 0; i < files.length; i++) {
  fs.readFile(files[i], 'utf-8', function (err, content) {
    // 触发结果事件
    ep.emit('got_file', content);
  });
}
```
`after`方法适合重复的操作，比如读取10个文件，调用5次数据库等。将handler注册到N次相同事件的触发上。达到指定的触发数，handler将会被调用执行，每次触发的数据，将会按触发顺序，存为数组作为参数传入。

### 持续型异步协作
此处以股票为例，数据和模板都是异步获取，但是数据会是刷新，视图会重新刷新。

```
var ep = new EventProxy();
ep.tail('tpl', 'data', function (tpl, data) {
  // 在所有指定的事件触发后，将会被调用执行
  // 参数对应各自的事件名的最新数据
});
fs.readFile('template.tpl', 'utf-8', function (err, content) {
  ep.emit('tpl', content);
});
setInterval(function () {
  db.get('some sql', function (err, result) {
    ep.emit('data', result);
  });
}, 2000);
```

`tail`与`all`方法比较类似，都是注册到事件组合上。不同在于，指定事件都触发之后，如果事件依旧持续触发，将会在每次触发时调用handler，极像一条尾巴。


## 基本事件
通过事件实现异步协作是EventProxy的主要亮点。除此之外，它还是一个基本的事件库。携带如下基本API

- `on`/`addListener`，绑定事件监听器
- `emit`，触发事件
- `once`，绑定只执行一次的事件监听器
- `removeListener`，移除事件的监听器
- `removeAllListeners`，移除单个事件或者所有事件的监听器

为了照顾各个环境的开发者，上面的方法多具有别名。

- YUI3使用者，`subscribe`和`fire`你应该知道分别对应的是`on`/`addlistener`和`emit`。
- jQuery使用者，`trigger`对应的方法是`emit`，`bind`对应的就是`on`/`addlistener`。
- `removeListener`和`removeAllListeners`其实都可以通过别名`unbind`完成。

所以在你的环境下，选用你喜欢的API即可。

更多API的描述请访问[API Docs](http://html5ify.com/eventproxy/api.html)。
## 异常处理
在异步方法中，实际上，异常处理需要占用一定比例的精力。在过去一段时间内，我们都是通过额外添加`error`事件来进行处理的，代码大致如下：

```
exports.getContent = function (callback) {
 var ep = new EventProxy();
  ep.all('tpl', 'data', function (tpl, data) {
    // 成功回调
    callback(null, {
      template: tpl,
      data: data
    });
  });
  // 侦听error事件
  ep.bind('error', function (err) {
    // 卸载掉所有handler
    ep.unbind();
    // 异常回调
    callback(err);
  });
  fs.readFile('template.tpl', 'utf-8', function (err, content) {
    if (err) {
      // 一旦发生异常，一律交给error事件的handler处理
      return ep.emit('error', err);
    }
    ep.emit('tpl', content);
  });
  db.get('some sql', function (err, result) {
    if (err) {
      // 一旦发生异常，一律交给error事件的handler处理
      return ep.emit('error', err);
    }
    ep.emit('data', result);
  });
};
```
代码量因为异常的处理，一下子上去了很多。在这里EventProxy经过很多实践后，我们根据我们的最佳实践提供了优化的错误处理方案。

```
exports.getContent = function (callback) {
 var ep = new EventProxy();
  ep.all('tpl', 'data', function (tpl, data) {
    // 成功回调
    callback(null, {
      template: tpl,
      data: data
    });
  });
  // 添加error handler
  ep.fail(callback);

  fs.readFile('template.tpl', 'utf-8', ep.done('tpl'));
  db.get('some sql', ep.done('data'));
};
```

上述代码优化之后，业务开发者几乎不用关心异常处理了。代码量降低效果明显。  
这里代码的转换，也许有开发者并不放心。其实秘诀在`fail`方法和`done`方法中。

### 神奇的fail

```
ep.fail(callback);
// 由于参数位相同，它实际是
ep.fail(function (err) {
  callback(err);
});
// 等价于
ep.bind('error', function (err) {
  // 卸载掉所有handler
  ep.unbind();
  // 异常回调
  callback(err);
});
```

`fail`方法侦听了`error`事件，默认处理卸载掉所有handler，并调用回调函数。

### 神奇的done
```
ep.done('tpl');
// 等价于
function (err, content) {
  if (err) {
    // 一旦发生异常，一律交给error事件的handler处理
    return ep.emit('error', err);
  }
  ep.emit('tpl', content);
}
```
在Node的最佳实践中，回调函数第一个参数一定会是一个`error`对象。检测到异常后，将会触发`error`事件。剩下的参数，将触发事件，传递给对应handler处理。

#### done也接受回调函数
done方法除了接受事件名外，还接受回调函数。如果是函数时，它将剔除第一个`error`对象(此时为`null`)后剩余的参数，传递给该回调函数作为参数。该回调函数无需考虑异常处理。

```
ep.done(function (content) {
  // 这里无需考虑异常
});
```
## 注意事项

- 请勿使用`all`作为业务中的事件名。该事件名为保留事件。
- 异常处理部分，请遵循Node的最佳实践。

## 贡献者们
谢谢EventProxy的使用者们，享受EventProxy的过程，也给EventProxy回馈良多。

```

 project  : eventproxy
 repo age : 1 year, 9 months
 active   : 55 days
 commits  : 132
 files    : 18
 authors  : 
   120  Jackson Tian            90.9%
     5  fengmk2                 3.8%
     4  dead-horse              3.0%
     1  haoxin                  0.8%
     1  redky                   0.8%
     1  yaoazhen                0.8%

```

## License 

[The MIT License](https://github.com/JacksonTian/eventproxy/blob/master/MIT-License)。请自由享受开源。
