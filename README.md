# ec-devtools

ec-devtools是支持canvas库 ( easycanvas , https://github.com/chenzhuo1992/easycanvas ) 的chrome调试工具，能对canvas的元素的样式、物理属性等进行修改，达到所见即所得的效果，提高调试效率，如下：

![demo](./screenshot/index.gif?raw=true)

具体使用，参考 canvas库 [easycanvas](https://github.com/chenzhuo1992/easycanvas)，以下聊聊 **ec-devtools** 的实现

## chrome devtools 通信

在我们平时的开发工作中，chrome开发者工具是我们必不可少的工具，除了Chrome原生的工具之外，还有比如：
* 使用vue开发时使用的 [vue-devtools](https://github.com/vuejs/vue-devtools)
* 使用react开发时使用的 [react-devtools](https://github.com/facebook/react-devtools)

本文主要分享的就是这些开发者工具怎么与页面进行消息通信的。

首先，先看一个例子：

![index](https://user-gold-cdn.xitu.io/2018/8/5/1650a963045a6471?w=1182&h=609&f=gif&s=1003873)

这个开发者工具源码：[https://github.com/HuJiaoHJ/ec-devtools](https://github.com/HuJiaoHJ/ec-devtools)，这个工具是一个支持canvas库 ( easycanvas , [https://github.com/chenzhuo1992/easycanvas](https://github.com/chenzhuo1992/easycanvas) ) 的chrome调试工具，能对canvas的元素的样式、物理属性等进行修改，达到所见即所得的效果，提高调试效率。感兴趣的小伙伴可以自行了解下~

本文主要是通过这个工具，分享一下 chrome devtools 通信相关的知识（chrome devtools的基础开发在这里就不介绍了，不熟悉的小伙伴可以去官网看看~）当然，没有接触过chrome devtools开发的小伙伴，也可以通过这篇文章了解到chrome devtools的基本组成，了解其基本的通信方式，对平时的工作也能有一些借鉴和帮助哒~

### chrome devtools 简单介绍

chrome devtools 主要分为三部分：

* DevTools Page：开发者工具，就是我们平时使用时，接触到的面板
* Background Page：后台页面，虽然叫页面，其实是在后台的js脚本
* Content Script：内容脚本，是在网页的上下文中允许的js文件

下面会详细介绍各部分，下面这张图是这三部分之间的通信全景图：

<img width="509" alt="ec-devtools" src="https://user-gold-cdn.xitu.io/2018/8/5/1650a962d2828ff7?w=1017&h=603&f=png&s=75361">

我们根据这张图，来详细的看看每个部分的具体实现吧~

* 网页与内容脚本通信
* 内容脚本与后台页面通信
* 后台页面与开发者工具通信
* 网页的消息怎么传递到开发者工具？
* 开发者工具的消息怎么传递到网页？

### 网页与内容脚本通信

内容脚本（Content Script）是在网页的上下文中运行的js文件，可以通过此js文件获取DOM和捕获DOM事件。

网页不能直接与开发者工具（DevTools Page）进行通信，需要通过在内容脚本中监听网页事件，通过chrome.runtime API将消息传递到后台页面中，从而传递到开发者工具中。

内容脚本可以监听网页的DOM事件或者window.postMessage事件，如下：

#### web page

``` js
window.postMessage({
    name: 'hello wolrd'
}, '*');
```

#### content-script.js

``` js
window.addEventListener('message', e => {
    if (e.source === window) {
        chrome.runtime.sendMessage(e.data);
    }
});
```

### 内容脚本与后台页面通信

后台页面，虽然叫页面，其实是在后台的js脚本。

内容脚本监听的事件触发之后，通过chrome.runtime.sendMessage()方法将消息传递到后台页面（Background Page）中。

后台脚本通过chrome.runtime.onMessage.addListener()方法监听消息，如下：

#### background.js

``` js
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    if (sender.tab) {
        const tabId = sender.tab.id;
        ......
    } else {
        console.log("sender.tab not defined.");
    }
    return true;
});
```

### 后台页面与开发者工具通信

后台页面与开发者工具通过长连接进行通信。（chrome.runtime API）如下：

#### devtool.js

``` js
// 与后台页面消息通信-长连接
const port = chrome.runtime.connect({name: 'devtools'});
// 监听后台页面消息
port.onMessage.addListener((message) => {
    ......
});
// 往后台页面发送消息
port.postMessage({
    name: 'original',
    tabId: chrome.devtools.inspectedWindow.tabId
});
```

#### background.js

``` js
chrome.runtime.onConnect.addListener(function (port) {
 
    const extensionListener = function (message, sender, sendResponse) {
        if (message.name == 'original') {
            ......
        }
    };
    port.onMessage.addListener(extensionListener);
 
    port.onDisconnect.addListener(function(port) {
        port.onMessage.removeListener(extensionListener);
    });
});
```

以上，就介绍了网页与内容脚本、内容脚本与后台页面、后台页面与开发者工具之间的通信，所以可以发现，网页的消息是通过内容脚本、后台页面，最后到达开发者工具，那么达到内容脚本的消息怎么传递到开发工具的呢？

### 网页的消息怎么传递到开发者工具？

显而易见，其实就是通过后台页面作为桥，将内容脚本的消息传递到开发者工具中。具体代码如下：

#### background.js

``` js
// 作为content script 与 devtool 通信的桥
const connections = {};
 
chrome.runtime.onConnect.addListener(function (port) {
 
    const extensionListener = function (message, sender, sendResponse) {
        if (message.name == 'original') {
            connections[message.tabId] = port;
        }
    };
    port.onMessage.addListener(extensionListener);
 
    port.onDisconnect.addListener(function(port) {
        port.onMessage.removeListener(extensionListener);
 
        const tabs = Object.keys(connections);
        for (let i = 0, len = tabs.length; i < len; i++) {
            if (connections[tabs[i]] == port) {
                delete connections[tabs[i]];
                break;
            }
        }
    });
});
 
// 接收内容脚本的消息，并发送到devtool的消息
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    if (sender.tab) {
        const tabId = sender.tab.id;
        if (tabId in connections) {
            connections[tabId].postMessage(message);
        } else {
            console.log("Tab not found in connection list.");
        }
    } else {
        console.log("sender.tab not defined.");
    }
    return true;
});
```

以上，就完成了网页的消息传递到开发者工具的过程，那么开发者工具的消息怎么传递到网页？

### 开发者工具的消息怎么传递到网页？

开发者工具的消息传递到网页主要有两种方法：

1、直接使用chrome.devtools.inspectedWindow.eval()方法，在网页的上下文中执行js代码，如下：

#### devtool.js

``` js
chrome.devtools.inspectedWindow.eval('console.log(window)');
```

2、开发者工具通过长连接将消息传递到后台页面，在后台页面中，通过调用chrome.tab.excuteScript()方法，在网页中执行js代码，如下：

#### background.js

``` js
chrome.tab.excuteScript(tabId, {
    code: 'console.log(window)'
});
```

推荐使用第一种方法~

以上，就介绍了网页与开发者工具之间的通信全过程啦~~~

以上通信方式在文章一开始提到的工具都用到了，仓库：[https://github.com/HuJiaoHJ/ec-devtools](https://github.com/HuJiaoHJ/ec-devtools)，其实也基本涵盖了 chrome 开发者工具的所有通信方式~

![index](https://user-gold-cdn.xitu.io/2018/8/5/1650a963045a6471?w=1182&h=609&f=gif&s=1003873)

### 写在最后

学习了 chrome devtools 的通信方式之后，就能愉快的开发自己的开发者工具啦，希望能对有需要的小伙伴有帮助~~~