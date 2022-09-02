##  微前端沙盒实现新思路

> 如果用`with`的话性能变差

1). 主要在js语法自身使用`with`的问题





##   mirco-app的优化

```js
`;(function(proxyWindow){with(proxyWindow){(function(${globalKeyToBeCached}){;${code}\n}).call(proxyWindow,${globalKeyToBeCached})}})(proxyWindow);`

const globalKeyToBeCached = 'window,self,globalThis,Array,Object,String,Boolean,Math,Number,Symbol,Date,Promise,Function,Proxy,WeakMap,WeakSet,Set,Map,Reflect,Element,Node,Document,RegExp,Error,TypeError,JSON,isNaN,parseFloat,parseInt,performance,console,decodeURI,encodeURI,decodeURIComponent,encodeURIComponent,location,navigator,undefined'
```

使用with编译阶段无法进行性能优化，所以手动优化，通过一个函数变量保存这些变量，这样拿的时候就不用到`with`作用域上找。



这样在大量的dom操作情况下，不用每次从with作用域去读，省去了大量的查找时间，性能接近原生了。





##   沙箱解决思路



我们可以将js放入iframe中执行js，用一个原生的沙箱去执行代码。同时操作的dom将是ShadowRoot。

```js
  const { template, getExternalScripts, getExternalStyleSheets } = await importHTML(url, {
    fetch: fetch || window.fetch,
    plugins: newSandbox.plugins,
    loadError: newSandbox.lifecycles.loadError,
  });
```

获取到外链js文本后，插入到iframe的`header`头部，进行执行

```js
`(function(window, self, global, location) {
      ${code}
}).bind(window.__WUJIE.proxy)(
  window.__WUJIE.proxy,
  window.__WUJIE.proxy,
  window.__WUJIE.proxy,
  window.__WUJIE.proxyLocation,
);`
```

这里用到window的代理，和location的代理传入，因为有些值还是要拿外部window的



由于iframe的历史记录栈会记录到主应用外部，所以我们不用做太多操作。



##  Css隔离解决思路

qiankun，这里用shadow 当然可以隔离样式之间冲突，但是如果用户插入`body`的弹窗，qiankun就会样式丢失。



子应用的实例`instance`在`iframe`内运行，`dom`在主应用容器下的`webcomponent`内，通过代理 `iframe`的`document`到`webcomponent`，可以实现两者的互联。

将`document`的查询类接口：`getElementsByTagName、getElementsByClassName、getElementsByName、getElementById、querySelector、querySelectorAll、head、body`全部代理到`webcomponent`，这样`instance`和`webcomponent`就精准的链接起来。

这样`append`添加其实是添加到shadowRoot的内部





##  wujie的源码流程



主要是主应用执行`startApp`





###   初次流程

```ts
  const newSandbox = new WuJie({ name, url, attrs, fiber, degrade, plugins, lifecycles });
  newSandbox.lifecycles?.beforeLoad?.(newSandbox.iframe.contentWindow);
  const { template, getExternalScripts, getExternalStyleSheets } = await importHTML(url, {
    fetch: fetch || window.fetch,
    plugins: newSandbox.plugins,
    loadError: newSandbox.lifecycles.loadError,
  });

  const processedHtml = await processCssLoader(newSandbox, template, getExternalStyleSheets);
  await newSandbox.active({ url, sync, prefix, template: processedHtml, el, props, alive, fetch, replace });
  await newSandbox.start(getExternalScripts);
  return newSandbox.destroy;
```



这里主要`new`了个Wujie的实例，返回一个沙箱环境



### 子应用iframe初始化

```
    const iframe = iframeGenerator(this, attrs, mainHostPath, appHostPath, appRoutePath);
    this.iframe = iframe
```



```
  patchIframeVariable(iframeWindow, appHostPath);
  
  // 处理iframe的一些重写
  stopIframeLoading(iframeWindow, url);
  
  // 劫持pushState和replaceState，保证子应用的iframe路径改变的时候同步到主应用
  patchIframeHistory(iframeWindow, appHostPath, mainHostPath);
  
  // iframeWindow上addEventListener重写，绑定到外部window上。如绑定一个click事件，绑在iframe里面是没用的。
  patchIframeEvents(iframeWindow);
  
  // 降级的
  if (iframeWindow.__WUJIE.degrade) recordEventListeners(iframeWindow);
  
  // 重写子iframewindow上的popState, hashchange, 如前进后退会触发
  syncIframeUrlToWindow(iframeWindow);
```



stopIframeLoading做了很多事情主要有

```
// 停止同个页面的脚本
iframe.stop()

function initIframeDom(iframeWindow: Window): void {
  const iframeDocument = iframeWindow.document;
  clearChild(iframeDocument);
  const html = iframeDocument.createElement("html");
  html.innerHTML = "<head></head><body></body>";
  iframeDocument.appendChild(html);

  initBase(iframeWindow, iframeWindow.__WUJIE.url);
  
  // 处理window上特殊字段如height，width这些还有getComputedStyle这些函数需要拿window上面的key值 
  patchWindowEffect(iframeWindow);
  
  
 // 重写iframeWindow.Document.prototype的addEventerListener, 当拿iframeWindow.document.addEventListener让它绑定到shadow Root上去。
 // 重写iframeWindow.Document.prototype的on监听事件，当在document绑定事件都往shadowRoot的第一个元素上绑定
 
 // 需要Object.DefinePerporty,对querySelector等查找方法在shadowRoot上查找或者append这些往shadowRoot上挂
  patchDocumentEffect(iframeWindow);
 
 // 兼容getRootNode返回shadowRoot
 patchNodeEffect(iframeWindow);
 
 //	script等setAttribute的时候链接要拼上子应用的域名
  patchRelativeUrlEffect(iframeWindow);
}
```



```ts
      const { proxyWindow, proxyDocument, proxyLocation } = proxyGenerator(
        iframe,
        urlElement,
        mainHostPath,
        appHostPath
      );
      this.proxy = proxyWindow;
      this.proxyDocument = proxyDocument;
      this.proxyLocation = proxyLocation;
```

在iframe初始化后还要做下proxy代理window和location让代码在对应的环境执行。



```ts
proxywindow代理一些self或者location到proxylocation去
proxyDocument就是重要的querySelector,append全部都操作到shadowRoot上
proxyLocation，比如set，href时要设置主应用才有效果
```





有了这些代理对象是如何使用的呢?

```ts
  const { template, getExternalScripts, getExternalStyleSheets } = await importHTML(url, {
    fetch: fetch || window.fetch,
    plugins: newSandbox.plugins,
    loadError: newSandbox.lifecycles.loadError,
  });

  const processedHtml = await processCssLoader(newSandbox, template, getExternalStyleSheets);
  await newSandbox.active({ url, sync, prefix, template: processedHtml, el, props, alive, fetch, replace });
  await newSandbox.start(getExternalScripts);
```

`getExternalScripts`和`getExternalStyleSheets`是js或css外链

```
processCssLoader
```

此处是css插入template的html中

```ts
await newSandbox.start(getExternalScripts);
```

此处是需要请求外链js了

```ts
  // 内联脚本
  if (content) {
    // patch location
    if (!iframeWindow.__WUJIE.degrade && !module) {
      code = `(function(window, self, global, location) {
      ${code}
}).bind(window.__WUJIE.proxy)(
  window.__WUJIE.proxy,
  window.__WUJIE.proxy,
  window.__WUJIE.proxy,
  window.__WUJIE.proxyLocation,
);`;
    }
    // 解决 webpack publicPath 为 auto 无法加载资源的问题
    Object.defineProperty(scriptElement, "src", { get: () => src });
    // 非内联脚本
  } else {
    src && scriptElement.setAttribute("src", src);
    crossorigin && scriptElement.setAttribute("crossorigin", crossoriginType);
  }
  module && scriptElement.setAttribute("type", "module");
  scriptElement.textContent = code || "";
  nextScriptElement.textContent =
    "if(window.__WUJIE.execQueue && window.__WUJIE.execQueue.length){ window.__WUJIE.execQueue.shift()()}";

```

会以一个立即执行函数放入代理内容，

并执行下一个js，最后插入head中



