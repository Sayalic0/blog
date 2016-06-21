# 漫谈PhantomJS

PhantomJS涉及的东西比较多，我简单整理一下相关资料，鸡零狗碎，不成系统，故名漫谈。

由于该教程的重要内部听众是Java程序员(没有太多JavaScript经验)，所以会使用一些Java的类比和介绍一些JavaScript。

## 入门篇

PhantomJS相对于同类产品：

![image](http://i.stack.imgur.com/Scv5g.jpg)

这里提一下`Emulated User Agent`，我的理解就是实现了HTML到DOM，因为有DOM，所以可以支持JS对DOM的调用。

### Hello World

```js
var webPage = require('webpage');
var page = webPage.create();

page.open('http://www.google.com/', function(status) {
  console.log('Status: ' + status);
  // Do other things here...
});
```
简单的Open加回调函数。

### 登录实战
引用[某前端社区的一段代码](http://imweb.io/topic/556c287879878a3b386dd026)（实际上这人也是抄袭[SO上的一段代码](http://stackoverflow.com/questions/9246438/how-to-submit-a-form-using-phantomjs)）:

```js
var page = require('webpage').create(),
    testindex = 0,
    loadInProgress = false;

page.onLoadStarted = function() {
    loadInProgress = true;
    console.log("load started");
};

page.onLoadFinished = function() {
    loadInProgress = false;
    console.log("load finished");
};

var steps = [
    function() {
      page.open("https://account.xiaomi.com/pass/serviceLogin");
    },
    function() {
        page.evaluate(function(obj) {
            var form = document.getElementById("miniLogin");
            form.elements["miniLogin_username"].value = '用户名';
            form.elements["miniLogin_pwd"].value = '密码';
            form.elements['message_LOGIN_IMMEDIATELY'].click();
            return document.title;
        });
        loadInProgress = true;
    },
    function() {
        page.render('login-succ.png');
    }
];

var interval = setInterval(function() {
    if (!loadInProgress && typeof steps[testindex] == "function") {
        steps[testindex]();
        testindex++;
    }
    if (typeof steps[testindex] != "function") {
        phantom.exit();
    }
}, 10);
```

由于接口是异步的，所以需要等待页面加载完成之后再进行操作：

```js
page.onLoadStarted = function() {
    loadInProgress = true;
    console.log("load started");
};

page.onLoadFinished = function() {
    loadInProgress = false;
    console.log("load finished");
};
```

### PhantomJS的问题

控制流程比较复杂，手动处理页面是否加载完成等，比较麻烦，可以自己写一套代码把这些事情搞定。或者使用CasperJS来处理这些事情。

### CasperJS

为了针对PhantomJS的流程控制问题，CasperJS官网上Quick Start：

```js
var casper = require('casper').create();
casper.start('http://casperjs.org/');

casper.then(function() {
    this.echo('First Page: ' + this.getTitle());
});

casper.thenOpen('http://phantomjs.org', function() {
    this.echo('Second Page: ' + this.getTitle());
});

casper.run();
```

总结：CasperJS四部曲：

1. `var casper = require('casper').create()`
2. `start(url)`
3. `then(function(){...})`
4. `casper.run()`

通过then来控制步骤，比PhantomJS手动控制更方便。

### CasperJS的登录

```js
var casper = require('casper').create();
var username = casper.cli.get('username').toString();
var password = casper.cli.get('password').toString();

casper.start('http://mail.163.com/')

casper.then(function () {
    casper.evaluate(function (username, password) {
        document.querySelector('#idInput').value = username
        document.querySelector('#pwdInput').value = password
    }, username, password)
}).then(function () {
    casper.click('#loginBtn')
}).thenOpen('https://reg.163.com/account/accountInfo.jsp', function () {
    //check login success
})

casper.run()
```
是否感觉很简单？

### JavaScript

考虑到大部分读者可能不是很懂JavaScript，这里我简单说几句。

##### 断章取义的艺术

JavaScript是一门很糟糕的语言：

> Python凑合可以用在不重要的地方，Ruby是垃圾，JavaScript是垃圾中的垃圾。—— 王垠, [给Java说句公道话](http://www.yinwang.org/blog-cn/2016/01/18/java)

充满了糟粕：

![image](http://blog.wix.engineering/wp-content/uploads/2015/04/mIuuwgx-1024x576.jpg)

创始人也不甚满意：

> The part that is good is not original, and the part that is original is not good. —— Brendan Eich, [Popularity](https://brendaneich.com/2008/04/popularity/)

##### 学习路径

虽然JavaScript很糟糕，但：

* 快速浏览一遍JavaScript权威指南
* 通读PhantomJS和CasperJS的API文档

可满足日常CasperJS开发。

##### JavaScript生态

* 浏览器唯一支持的操作DOM的语言(唯一前端语言)
* 移动前端
* NodeJS提供了后端开发的解决方案
* Chromium桌面软件，在一个Chrome里面跑HTML/CSS/JavaScript
* Transpiling,因为JavaScript实在太烂，所以发明新语言，转译成JavaScript
* 还有我们用的PhantomJS

### CasperJS若干基本问题

##### Then步骤的原理

每执行一次then，往Step栈中压入一个step，执行`casper.run()`以后，不断从栈顶取出step来执行。值得注意的是，可以在then中包含then，由于栈先进先出的机制，里面包含的then会立即执行。

```js
var casper = require('casper').create();

var url_list = [
    'http://phantomjs.org/',
    'https://github.com/',
    'https://nodejs.org/'
]

casper.start()

casper.then(function () {
        for (var i = 0; i < url_list.length; i++) {
            casper.echo('assign a then step for ' + url_list[i])
            casper.thenOpen(url_list[i], function () {
                casper.echo("current url is " + casper.getCurrentUrl());
            })
        }
    }
)

casper.run()
```
输出:

```
assign a then step for http://phantomjs.org/
assign a then step for https://github.com/
assign a then step for https://nodejs.org/
current url is http://phantomjs.org/
current url is https://github.com/
current url is https://nodejs.org/en/
```

更多参考[StackOverflow上的讨论](http://stackoverflow.com/a/11957919/1291716)，此回答为CasperJS创始人亲力亲为，有心者可以看一下此人回答的所有的关于CasperJS的问题。

##### Evalutate的原理

你可以想象成传入一段JavaScript在浏览器的Console中执行。

```js
casper.evaluate(function (a, b) {
    document.querySelector('#idInput').value = a
    document.querySelector('#pwdInput').value = b
}, username, password)
```
在浏览器中执行
```js
function (a, b) {
    document.querySelector('#idInput').value = a
    document.querySelector('#pwdInput').value = b
}
```  
其中a参数的值为`username`,b参数的值为`password`。

需要注意的是，只能传入simple primitive object(能被正确JSON序列化的对象，不能是函数等……)。

更多参考[StackOverflow上的讨论](http://stackoverflow.com/a/24219029/1291716)，此回答为SO上首席CasperJS专家所写，此人也回答了我很多CasperJS的问题，不啻再造之恩，在此隆重致谢。

另外，登录时候推荐使用Evaluate而不是sendKeys()，前者直接对input元素操作，不会错误，后者依赖于WebKit对连续键盘事件的正确处理，容易出错。

##### 调试Log

```js
var casper = require('casper').create({
    verbose: true,
    logLevel: 'debug'
})
```
如果不加这两个参数，会少很多调试的log，推荐加上。
##### Log优化

因为CasperJS自带的log缺少时间戳，让我这个习惯了logback的Java程序员很不习惯，所以改写了一下他的函数，添加Log时间戳：

```js
casper.logCopy = casper.log.bind(casper);

casper.log = function(message, level, space)
{
    var currentDate = '[' + new Date().toJSON() + '] ';
    this.logCopy(currentDate + message, level, space);
};
```

##### PhantomJS错误事件

PhantomJS的执行报错是通过捕获事件来处理的:

```js
casper.on('error', function (msg, backtrace) {
    var msgStack = ['PHANTOM ERROR: ' + msg];
    if (backtrace && backtrace.length) {
        msgStack.push('TRACE:');
        backtrace.forEach(function(t) {
            msgStack.push(' -> ' + (t.file || t.sourceURL) + ': ' + t.line + (t.function ? ' (in function ' + t.function +')' : ''));
        });
    }
    this.log(msgStack.join('\n'), "error");
});

```

效果:

```
PHANTOM ERROR: TypeError: undefined is not an object (evaluating 'casper.cli.get('cookie').toString')
TRACE:
 -> phantomjs://code/taobao_order.js: 45 (in function global code)
 -> undefined: 0 (in function injectJs)
 -> phantomjs://code/bootstrap.js: 435
```

更多事件参考CasperJS文档。

##### CasperJS同步问题

CasperJS的同步是通过step来完成的——当前step尚未结束，就不会开始新的step。

```js
casper.start()

casper.then(function () {
    casper.log("1")
    casper.wait(1000)
    casper.log("2")
}).then(function () {
    casper.log("3");
})
casper.run()
```

输出：

```
[info] [phantom] [2016-06-06T07:47:32.151Z] Starting...
[info] [phantom] [2016-06-06T07:47:32.156Z] Running suite: 2 steps
[debug] [phantom] [2016-06-06T07:47:32.185Z] Successfully injected Casper client-side utilities
[debug] [phantom] [2016-06-06T07:47:32.186Z] 1
[debug] [phantom] [2016-06-06T07:47:32.187Z] 2
[info] [phantom] [2016-06-06T07:47:32.187Z] Step anonymous 1/2: done in 35ms.
[info] [phantom] [2016-06-06T07:47:32.204Z] Step _step 2/3: done in 52ms.
[info] [phantom] [2016-06-06T07:47:33.207Z] wait() finished waiting for 1000ms.
[debug] [phantom] [2016-06-06T07:47:33.208Z] 3
[info] [phantom] [2016-06-06T07:47:33.208Z] Step anonymous 3/3: done in 1056ms.
[info] [phantom] [2016-06-06T07:47:33.231Z] Done 3 steps in 1079ms
[debug] [phantom] [2016-06-06T07:47:33.233Z] Navigation requested: url=about:blank, type=Other, willNavigate=true, isMainFrame=true
[debug] [phantom] [2016-06-06T07:47:33.234Z] url changed to "about:blank"
```

可以看到1和2是同时输出的，而下一个step却等待了1秒才执行。所以wait()在step里面是异步执行的，然而整个step会等待wait执行的结束，从而达到了等待的效果。所以想要做同步的话，就多开几个then step就好了。

另外，这里的log，就是打开调试log并且输出时间戳的效果。

## 进阶篇

### PhantomJS源码剖析

这里以PhantomJS 2.1的源码来简单分析一下PhantomJS的架构。

PhantomJS的代码分为三大块：

1. [C++代码](https://github.com/ariya/phantomjs/tree/2.1/src/)，调用QTWebKit来实现相关功能。
2. [JavaScript代码](https://github.com/ariya/phantomjs/tree/2.1/src/modules)，提供JavaScript相关API，包括webpage对象等。
3. [QT库代码](https://github.com/ariya/phantomjs/tree/2.1/src/qt)

##### WebKit Port

首先我们来看一下PhantomJSC++代码方面的架构，比较简单：

![image](http://i.stack.imgur.com/avdDC.png)

QTWebkit包装了WebKit，PhantomJS调用QTWebKit来使用WebKit。在PhantomJS的源码里面，可以看到PhantomJS深度使用了QTWebKit的代码。关于QTWebKit和WebKit的关系，可以阅读PhantomJS创始人的博客进一步了解[WebKit Port技术](http://ariya.ofilabs.com/2011/06/your-webkit-port-is-special-just-like-every-other-port.html)。

##### QTWebKit Bridge

那么PhantomJS的JavaScript代码是如何调用PhantomJS的C++代码呢？核心就在[phantom.cpp:477](https://github.com/ariya/phantomjs/blob/2.1/src/phantom.cpp#L477)：

```cpp
// Add 'phantom' object to the global scope
m_page->mainFrame()->addToJavaScriptWindowObject("phantom", this);
```
这个`addToJavaScriptWindowObject`就是把一个C++对象转化一个JavaScript对象并放入JavaScript环境中，使得对于其属性的调用等于调用其对应的C++对象的方法，这里是官方文档：

> Make object available under name from within the frame's JavaScript context. The object will be inserted as a child of the frame's window object.
> 
> Qt properties will be exposed as JavaScript properties and slots as JavaScript methods. The interaction between C++ and JavaScript is explained in the documentation of the QtWebKit bridge.

这里用到的技术就是QtWebKit bridge，可以参阅官方文档了解[详细](http://doc.qt.io/qt-4.8/qtwebkit-bridge.html)。

##### Phantom对象的内涵

通过bridge技术，我们可以在JavaScript环境中调用C++方法，那么phantom对象里面到底有哪些东西呢？代码[phantom.h:227](https://github.com/ariya/phantomjs/blob/2.1/src/phantom.h#L227-L239):

```cpp
Encoding m_scriptFileEnc;
WebPage* m_page;
bool m_terminated;
int m_returnValue;
QString m_script;
QVariantMap m_defaultPageSettings;
FileSystem* m_filesystem;
System* m_system;
ChildProcess* m_childprocess;
QList<QPointer<WebPage> > m_pages;
QList<QPointer<WebServer> > m_servers;
Config m_config;
CookieJar* m_defaultCookieJar;
```
  
这里可以看到phantom对象持有了三个API对象的指针：

* FileSystem - 提供JavaScript fs模块的API
* ChildProcess - 提供JavaScript child_process模块的API
* System - 提供JavaScript system模块的API

除此之外，还提供了其他的一些C++代码供JavaScript调用。总而言之，phantom对象飞架JavaScript和C++，天堑变通途。

##### Phantom对象的创建

构造器：
[phantom.cpp:57](https://github.com/ariya/phantomjs/blob/2.1/src/phantom.cpp#L57-L72)

```cpp
Phantom::Phantom(QObject* parent)
    : QObject(parent)
    , m_terminated(false)
    , m_returnValue(0)
    , m_filesystem(0)
    , m_system(0)
    , m_childprocess(0)
{
    QStringList args = QApplication::arguments();

    // Prepare the configuration object based on the command line arguments.
    // Because this object will be used by other classes, it needs to be ready ASAP.
    m_config.init(&args);
    // Apply debug configuration as early as possible
    Utils::printDebugMessages = m_config.printDebugMessages();
}
```
这里用到了[initializer_list](http://en.cppreference.com/w/cpp/language/initializer_list)，就是把括号里面的值赋给对应的成员变量。可以看到Phantom类继承自QObject，然后API类的指针最开始都赋值为0(等于Java的null)，后续实际调用的时候，才会初始化。

当然，由于Phantom类的构造器是私有的，通过调用instance()方法来获取，可以看到这里使用单例模式。
[phantom.cpp:148](https://github.com/ariya/phantomjs/blob/2.1/src/phantom.cpp#L148-L155):

```cpp
Phantom* Phantom::instance()
{
    if (NULL == phantomInstance) {
        phantomInstance = new Phantom();
        phantomInstance->init();
    }
    return phantomInstance;
}
```

可以看到，每次创建时候会调用init()方法，而init()方法会把Phantom对象桥接到JavaScript环境中。那么在哪里调用了`Phantom::instance()`呢？请自行使用代码检索找到答案。

##### bootstrap.js
光有个能调用C++原生方法的phantom对象还是不够，还需要配置好一套JavaScript环境供开发者调用(就是执行你输入的JS代码的那套环境)，那么这套环境是怎么准备的呢？代码在[phantom.cpp:480](https://github.com/ariya/phantomjs/blob/2.1/src/phantom.cpp#L480-L483):

```cpp
m_page->mainFrame()->evaluateJavaScript(
    Utils::readResourceFileUtf8(":/bootstrap.js"),
    QString(JAVASCRIPT_SOURCE_PLATFORM_URL).arg("bootstrap.js")
);
```

可以看到，就是简单的执行了一下`boostrap.js`代码，那么`bootstrap.js`又做了什么事情呢？也没有什么别的，大概三件事：

1. 定义了[onCallBack](http://phantomjs.org/api/webpage/handler/on-callback.html)并绑定到phantom对象
2. 定义了[onError](http://phantomjs.org/api/webpage/handler/on-error.html)并绑定到phantom对象
3. 实现了模块加载机制

如果说还有一点什么成绩，就是支持了创建了一个WebPage的全局变量，这对遗留代码的兼容有很大的关系。

##### JavaScript的模块机制

JavaScript一开始设计的时候就没有考虑模块的事情，所以大家各显神通，都搞一套类似的模块系统。这种各自为战使得沟通和交流颇为不便，于是CommonJS横空出世了。这个项目希望定义一组通用的JavaScript server端接口，让各自的应用实现这套接口，这样不同的项目就可以隔离开发了。更多参考[CommonJS官方Wiki](http://wiki.commonjs.org/wiki/Introduction)。

所以PhantomJS和NodeJS都实现了[CommonJS里面规定的模块机制](http://wiki.commonjs.org/wiki/Modules/1.1.1)，也就是大家熟知的require和exports关键字。

我们参考这一段代码：

**math.js**:
```js
exports.add = function() {
    var sum = 0, i = 0, args = arguments, l = args.length;
    while (i < l) {
        sum += args[i++];
    }
    return sum;
};
```
**increment.js**:

```js
var add = require('math').add;
exports.increment = function(val) {
    return add(val, 1);
};
```

**program.js**:

```js
var inc = require('increment').increment;
var a = 1;
inc(a); // 2

module.id == "program"
```

简而言之，exports是一个对象，只有这个对象会被导出，而其他函数和变量的定义的都对调用方保持透明。require是一个函数，参数是module id(具体可以参考CommonJS文档，不过一般就是文件名)。通过require来引用一个模块。

##### PhantomJS模块机制的实现

那么PhantomJS是如何实现这套模块的机制的呢？JavaScript代码就在之前提到的`bootstrap.js`里面，实现了一套CommonJS的模块。具体的实现可以自己去看，简单地讲，流程就是：

1. 先拿到要引入的模块的源代码
2. 如果是JSON文件，直接导出对应JSON所表示的对象。如果是JS文件，调用phantom对象的loadModule方法将该moudule引入JavaScript环境，然后返回module对应的exports对象。

我们来看一下[loadModule方法](https://github.com/ariya/phantomjs/blob/2.1/src/phantom.cpp#L370-L385):

```cpp
void Phantom::loadModule(const QString& moduleSource, const QString& filename)
{
    if (m_terminated) {
        return;
    }

    QString scriptSource =
        "(function(require, exports, module) {\n" +
        moduleSource +
        "\n}.call({}," +
        "require.cache['" + filename + "']._getRequire()," +
        "require.cache['" + filename + "'].exports," +
        "require.cache['" + filename + "']" +
        "));";
    m_page->mainFrame()->evaluateJavaScript(scriptSource, QString(JAVASCRIPT_SOURCE_PLATFORM_URL).arg(QFileInfo(filename).fileName()));
}
```

这就很明白了，把你整个Module的源代码放入到函数`function(require, exports, module) {}`中，然后调用该函数，从而使得引用的模块。

##### PhantomJS evalute的实现

现在我们搞清楚了，PhantomJS就是通过JavaScript代码封装phantom对象，最底层还是调用phantom对象的对应的C++方法。那么以此为指导思想，我们来探索一下PhantomJS evaluate的实现。

首先找到[evaluate的JavaScript源码](https://github.com/ariya/phantomjs/blob/2.1/src/modules/webpage.js#L362-L390):

```js
page.evaluate = function (func, args) {
    var str, arg, argType, i, l;
    if (!(func instanceof Function || typeof func === 'string' || func instanceof String)) {
        throw "Wrong use of WebPage#evaluate";
    }
    str = 'function() { return (' + func.toString() + ')(';
    for (i = 1, l = arguments.length; i < l; i++) {
        arg = arguments[i];
        argType = detectType(arg);

        switch (argType) {
        case "object":      //< for type "object"
        case "array":       //< for type "array"
            str += JSON.stringify(arg) + ","
            break;
        case "date":        //< for type "date"
            str += "new Date(" + JSON.stringify(arg) + "),"
            break;
        case "string":      //< for type "string"
            str += quoteString(arg) + ',';
            break;
        default:            // for types: "null", "number", "function", "regexp", "undefined"
            str += arg + ',';
            break;
        }
    }
    str = str.replace(/,$/, '') + '); }';
    return this.evaluateJavaScript(str);
};
```

可以看到就是把函数toString()（JavaScript里面，函数toString等于其源码），然后替换参数，把函数转化为一个字符串，该字符表示一个无参函数，返回一个[IIFE](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression)执行的结果，然后传入`this.evaluateJavaScript`执行。

因为evaluateJavaScript调用的是桥接的C++方法，所以我们再来看一下[evaluateJavaScript的源码](https://github.com/ariya/phantomjs/blob/2.1/src/webpage.cpp#L748-L762)：

```cpp
QVariant WebPage::evaluateJavaScript(const QString& code)
{
    QVariant evalResult;
    QString function = "(" + code + ")()";

    qDebug() << "WebPage - evaluateJavaScript" << function;

    evalResult = m_currentFrame->evaluateJavaScript(
                     function,                                   //< function evaluated
                     QString("phantomjs://webpage.evaluate()")); //< reference source file

    qDebug() << "WebPage - evaluateJavaScript result" << evalResult;

    return evalResult;
}
```

就是简单地通过[IIFE](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression)的形式执行，然后返回执行的结果。

这里举个简单的evaluate例子：

1. 输入函数`function(a){return a}`, 和参数`1`，得到字符串`function() { return (function (a){return a})(1); }`，传入`this.evaluateJavaScript`。
2. 把字符串以IIFE的形式执行。

为什么要以IIFE来执行呢？我的理解，主要的目的是为了避免作用域污染，具体可以查看IIFE的维基文档。

##### PhantomJS架构总结

JavaScript调用phantom对象，phantom对象桥接到C++方法，以此为指导思想，看源码就可以按图索骥了。

### CapserJS源码剖析

CasperJS的架构比较简单，这里以1.1.1版本简单分析一下。

##### CasperJS对PhantomJS的调用

CasperJS是通过命令行调用PhantomJS的，对PhantomJS的调用做了一个简单的Python封装。参见[casperjs](https://github.com/casperjs/casperjs/blob/1.1.1/bin/casperjs)。

##### CasperJS的初始化

在[调用PhantomJS的时候](https://github.com/casperjs/casperjs/blob/1.1.1/bin/casperjs#L132-L136)，传入了一份`bootstrap.js`，PhantomJS会执行该文件，并完成了对CasperJS环境的初始化。

```py
CASPER_COMMAND.extend([
    os.path.join(CASPER_PATH, 'bin', 'bootstrap.js'),
    '--casper-path=%s' % CASPER_PATH,
    '--cli'
])
```
##### bootstrap.js

CasperJS的`bootstrap.js`也没有什么别的，大概三件事：

1. 重定义fs对象
2. 重定义require对象，使得CasperJS的模块能够被require到
3. 在phantom对象的基础上添加了关于CasperJS的信息

既然都可以直接require CasperJS的模块了，那么我们就可以愉快地使用`var casper = require('casper').create();`来调用CasperJS的源码了。

##### CasperJS总结

通过传入一份`bootstrap.js`给PhantomJS执行，让CasperJS的模块能够正确地在PhantomJS的环境中引入和执行，就达到了扩展PhantomJS API的目的。
