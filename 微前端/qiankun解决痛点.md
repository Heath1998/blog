##  qiankun解决痛点



###   问题

1. js entry导入的形式无法并发，因为是操作dom元素，只能全部打包成一块，css不能分割开。

最主要的还是js文件要带个chunk编号，那导入的时候也要改变。

且需要在spa的时候一个个导入文件。



2. 样式隔离问题，所有样式都挂载head的style里面，会冲突
3. js隔离完全没做
4. 预加载，single-spa没做。
5. 通信，也没有



以上是单单用singespa可能产生的问题。但是也可以自己在single-spa之上封装解决，qiankun就是这么做的。



###  了解乾坤 快速上手

```javascript
import { registerMicroApps, start } from 'qiankun';

registerMicroApps([
  {
    name: 'react app', // app name registered
    entry: '//localhost:7100',
    container: '#yourContainer',
    activeRule: '/yourActiveRule',
  },
  {
    name: 'vue app',
    entry: { scripts: ['//localhost:7100/main.js'] },
    container: '#yourContainer2',
    activeRule: '/yourActiveRule2',
  },
]);

start(opts?);
```

- name - `string` - 必选，微应用的名称，微应用之间必须确保唯一。
- entry - `string | { scripts?: string[]; styles?: string[]; html?: string }` - 必选，微应用的入口。
  - 配置为字符串时，表示微应用的访问地址，例如 `https://qiankun.umijs.org/guide/`。
  - 配置为对象时，`html` 的值是微应用的 html 内容字符串，而不是微应用的访问地址。微应用的 `publicPath` 将会被设置为 `/`。
- container - `string | HTMLElement` - 必选，微应用的容器节点的选择器或者 Element 实例。如`container: '#root'` 或 `container: document.querySelector('#root')`。
- activeRule - `string | (location: Location) => boolean | Array<string | (location: Location) => boolean> `- 必选，微应用的激活规则。





Start()

```
Options
```

- prefetch - `boolean | 'all' | string[] | (( apps: RegistrableApp[] ) => { criticalAppNames: string[]; minorAppsName: string[] })` - 可选，是否开启预加载，默认为 `true`。

  配置为 `true` 则会在第一个微应用 mount 完成后开始预加载其他微应用的静态资源

  配置为 `'all'` 则主应用 `start` 后即开始预加载所有微应用静态资源

  配置为 `string[]` 则会在第一个微应用 mounted 后开始加载数组内的微应用资源

  配置为 `function` 则可完全自定义应用的资源加载时机 (首屏应用及次屏应用)

- sandbox - `boolean` | `{ strictStyleIsolation?: boolean, experimentalStyleIsolation?: boolean }` - 可选，是否开启沙箱，默认为 `true`。





有点像single-spa的方法。



###   1.js entry and html entry

qiankun通过`importEntry`解析html文件，再去加载对应这个html上面的script and style

可以看看源码

```
 /**
   * 获取微应用的入口 html 内容和脚本执行器
   * template 是 link 替换为 style 后的 template
   * execScript 是 让 JS 代码(scripts)在指定 上下文 中运行
   * assetPublicPath 是静态资源地址
   */
  const { template, execScripts, assetPublicPath } = await importEntry(entry, importEntryOpts);

......
  // get the lifecycle hooks from module exports
  // execScripts(andbox?: object | undefined, strictGlobal?: boolean | undefined)
  const scriptExports: any = await execScripts(global, sandbox && !useLooseSandbox);

  const { bootstrap, mount, unmount, update } = getLifecyclesFromExports(
    scriptExports,
    appName,
    global,
    sandboxContainer?.instance?.latestSetProp,
  );

```



`importEntry`返回的template，会将javascript的script外链注释掉，css的外链请求生成style。这样能保证css的范围只在当前子应用。

`execScripts`可以执行从html解析到的script内联语句或者script的脚本链接。并且以最后一个解析成包含生命周期的函数。

execScripts执行时传入沙箱环境，会promise.all请求所有script脚本外链，获取到script文本在做处理。



这差不多就是html entry了。



###   2.样式隔离

主应用，微应用之间都有样式冲突的可能

有两种方式一种shadow dom，一种scope方法



####   shadow dom方式

- DOM 隔离：组件的 DOM 是独立的（例如，`document.querySelector()` 不会返回组件 shadow DOM 中的节点）。这意味着在主文档里，通过 `querySelectorAll`、`getElementsByTagName` 等方法，无法获取到 shadow DOM 内的任何元素。
- 样式隔离：shadow DOM 内部定义的 CSS 在其作用域内。样式规则不会泄漏至组件外，页面样式也不会渗入。



这样就能做到样式隔离了，



```javascript
 // strictStyleIsolation为true将包一个shadow dom
 let initialAppWrapperElement: HTMLElement | null = createElement(
    appContent,
    strictStyleIsolation,
    scopedCSS,
    appInstanceId,
  );
  
  ....
  
  mount:[
  ......
  //  appWrapperGetter()的结果是一个shadow 对象
  //  子应用就能拿到并挂载
  	  async (props) => mount({ ...props, container: appWrapperGetter(), setGlobalState, onGlobalStateChange }),
  ]
```



```javascript
function createElement(
  appContent: string,
  strictStyleIsolation: boolean,
  scopedCSS: boolean,
  appInstanceId: string,
): HTMLElement {
  const containerElement = document.createElement('div');
  containerElement.innerHTML = appContent;
  // appContent always wrapped with a singular div
  const appElement = containerElement.firstChild as HTMLElement;
  if (strictStyleIsolation) {
    if (!supportShadowDOM) {
      console.warn(
        '[qiankun]: As current browser not support shadow dom, your strictStyleIsolation configuration will be ignored!',
      );
    } else {
      const { innerHTML } = appElement;
      appElement.innerHTML = '';
      let shadow: ShadowRoot;

      if (appElement.attachShadow) {
        shadow = appElement.attachShadow({ mode: 'open' });
      } else {
        // createShadowRoot was proposed in initial spec, which has then been deprecated
        shadow = (appElement as any).createShadowRoot();
      }
      shadow.innerHTML = innerHTML;
    }
  }

  if (scopedCSS) {
    const attr = appElement.getAttribute(css.QiankunCSSRewriteAttr);
    if (!attr) {
      appElement.setAttribute(css.QiankunCSSRewriteAttr, appInstanceId);
    }

    const styleNodes = appElement.querySelectorAll('style') || [];
    forEach(styleNodes, (stylesheetElement: HTMLStyleElement) => {
      css.process(appElement!, stylesheetElement, appInstanceId);
    });
  }
 
  return appElement;
}
```

简单判断`attachShadow`建立个shadow 实例





####  通过scoped的方式

除此以外，qiankun 还提供了一个实验性的样式隔离特性，当 experimentalStyleIsolation 被设置为 true 时，qiankun 会改写子应用所添加的样式为所有样式规则增加一个特殊的选择器规则来限定其影响范围，因此改写后的代码会表达类似为如下结构：

```javascript
// 假设应用名是 vue
.app-main {
  font-size: 14px;
}

body .test{
  font-size: 14px;
}


div[data-qiankun="vue"] .app-main {
  font-size: 14px;
}
div[data-qiankun="vue"] .test {
  font-size: 14px;
}
```



能这样做是因为importEntry，把所有外链的css都转成文本了。

这里我们只需要把上面的css选择器加前缀就可以了，同时div容器加上`data-qiankun`属性，值为vue





```javascript
  if (scopedCSS) {
    const attr = appElement.getAttribute(css.QiankunCSSRewriteAttr);
    if (!attr) {
      appElement.setAttribute(css.QiankunCSSRewriteAttr, appInstanceId);
    }

    const styleNodes = appElement.querySelectorAll('style') || [];
    forEach(styleNodes, (stylesheetElement: HTMLStyleElement) => {
      css.process(appElement!, stylesheetElement, appInstanceId);
    });
  }
 
```

这里就把所有style都拿出来处理了下。







#### 总结样式隔离

基于 ShadowDOM 的严格样式隔离并不是一个可以无脑使用的方案，大部分情况下都需要接入应用做一些适配后才能正常在 ShadowDOM 中运行起来（比如 react 场景下需要解决这些 [问题](https://github.com/facebook/react/issues/10422)，使用者需要清楚开启了 `strictStyleIsolation` 意味着什么。后续 qiankun 会提供更多官方实践文档帮助用户能快速的将应用改造成可以运行在 ShadowDOM 环境的微应用。



比如塞到body的弹出窗口，在上面两种模式下都会丢失样式。所以样式隔离一般不开

有取就有舍





###  3.js隔离

jS 全局对象污染是一个很常见的现象，比如：微应用 A 在全局对象上添加了一个自己特有的属性，`window.A`，这时候切换到微应用 B，这时候如何保证 `window` 对象是干净的呢？



qiankun的单例沙箱和多例沙箱来解决。

单例沙箱就是完全以window做为基础，会对window造成污染。

- 相比较而言，ProxySandbox 是最完备的沙箱模式，完全隔离了对 window 对象的操作，也解决了快照模式中子应用运行期间仍然会对 window 造成污染的问题。



####  快照沙箱

通过一个快照window保存，即`windowSnapshot`

子应用激活的时候遍历保存`window`每一个属性到`windowSnapshot`，并且`modifyPropsMap`进行还原到`window`上.

卸载的时候，遍历将变更保存到`modifyPropMap` 同时快照`windowSnapshot`恢复到`window`上





优点：不用proxy

缺点：遍历慢。



####  单例沙箱

就是进入子应用的时候声明一个proxy对象，get方法直接返回window下面的属性，

set方法，由三个Map来记录当前做了那些操作。

```js
    // 沙箱期间新增的全局变量
    this.addedPropsMapInSandbox = {};
    // 沙箱期间更新的全局变量
    this.modifiedPropsOriginalValueMapInSandbox = {};
    // 持续记录更新的(新增和修改的)全局变量的 map，用于在任意时刻做 snapshot
    this.currentUpdatedPropsValueMap = {};

```



```js
    const setTrap = (p: PropertyKey, value: any, originalValue: any, sync2Window = true) => {
      if (this.sandboxRunning) {
        if (!rawWindow.hasOwnProperty(p)) {
          addedPropsMapInSandbox.set(p, value);
        } else if (!modifiedPropsOriginalValueMapInSandbox.has(p)) {
          // 如果当前 window 对象存在该属性，且 record map 中未记录过，则记录该属性初始值
          modifiedPropsOriginalValueMapInSandbox.set(p, originalValue);
        }

        currentUpdatedPropsValueMap.set(p, value);

        if (sync2Window) {
          // 必须重新设置 window 对象保证下次 get 时能拿到已更新的数据
          (rawWindow as any)[p] = value;
        }

        this.latestSetProp = p;

        return true;
      }

      if (process.env.NODE_ENV === 'development') {
        console.warn(`[qiankun] Set window.${p.toString()} while sandbox destroyed or inactive in ${name}!`);
      }

      // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，在沙箱卸载的情况下应该忽略错误
      return true;
    };

    const proxy = new Proxy(fakeWindow, {
      set: (_: Window, p: PropertyKey, value: any): boolean => {
        const originalValue = (rawWindow as any)[p];
        return setTrap(p, value, originalValue, true);
      },

      get(_: Window, p: PropertyKey): any {
        // avoid who using window.window or window.self to escape the sandbox environment to touch the really window
        // or use window.top to check if an iframe context
        // see https://github.com/eligrey/FileSaver.js/blob/master/src/FileSaver.js#L13
        if (p === 'top' || p === 'parent' || p === 'window' || p === 'self') {
          return proxy;
        }

        const value = (rawWindow as any)[p];
        return getTargetValue(rawWindow, value);
      },

      // trap in operator
      // see https://github.com/styled-components/styled-components/blob/master/packages/styled-components/src/constants.js#L12
      has(_: Window, p: string | number | symbol): boolean {
        return p in rawWindow;
      },

      getOwnPropertyDescriptor(_: Window, p: PropertyKey): PropertyDescriptor | undefined {
        const descriptor = Object.getOwnPropertyDescriptor(rawWindow, p);
        // A property cannot be reported as non-configurable, if it does not exists as an own property of the target object
        if (descriptor && !descriptor.configurable) {
          descriptor.configurable = true;
        }
        return descriptor;
      },

      defineProperty(_: Window, p: string | symbol, attributes: PropertyDescriptor): boolean {
        const originalValue = (rawWindow as any)[p];
        const done = Reflect.defineProperty(rawWindow, p, attributes);
        const value = (rawWindow as any)[p];
        setTrap(p, value, originalValue, false);

        return done;
      },
    });

    this.proxy = proxy;
```

可以看到proxy劫持`set`主要是在设置值的时候，更新沙箱新增map， 沙箱修改map，以及对沙箱做的所有操作记录map。然后往window上挂值

`get`方法直接从window获取值。

当调用 get 从子应用 proxy/window 对象取值时，会直接从 window 对象中取值。对于非构造函数的取值将会对 this 指针绑定到 window 对象后，再返回函数。



```js
  active() {
    if (!this.sandboxRunning) {
      this.currentUpdatedPropsValueMap.forEach((v, p) => this.setWindowProp(p, v));
    }

    this.sandboxRunning = true;
  }

  inactive() {
    // renderSandboxSnapshot = snapshot(currentUpdatedPropsValueMapForSnapshot);
    // restore global props to initial snapshot
    this.modifiedPropsOriginalValueMapInSandbox.forEach((v, p) => this.setWindowProp(p, v));
    this.addedPropsMapInSandbox.forEach((_, p) => this.setWindowProp(p, undefined, true));

    this.sandboxRunning = false;
  }

```

激活和关闭沙箱如上，激活时拿到持续记录更新的(新增和修改的)全局变量的 map，进行还原沙箱所做的修改。



关闭是拿到新增的记录，和修改的记录，的两个map进行还原，和删除





####  多实例沙箱



![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71ae2f46a7544b0facafb29f657a3af0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3326:0:0:0.awebp)



激活沙箱后，每次对`window`取值的时候，先从自己沙箱环境的`fakeWindow`里面找，如果不存在，就从`rawWindow`(外部的`window`)里去找；当对沙箱内部的`window`对象赋值的时候，会直接操作`fakeWindow`，而不会影响到`rawWindow`。



```js
    const { fakeWindow, propertiesWithGetter } = createFakeWindow(globalContext);

    const descriptorTargetMap = new Map<PropertyKey, SymbolTarget>();
    const hasOwnProperty = (key: PropertyKey) => fakeWindow.hasOwnProperty(key) || globalContext.hasOwnProperty(key);

    const proxy = new Proxy(fakeWindow, {
      set: (target: FakeWindow, p: PropertyKey, value: any): boolean => {
        if (this.sandboxRunning) {
          this.registerRunningApp(name, proxy);
          // We must kept its description while the property existed in globalContext before
          if (!target.hasOwnProperty(p) && globalContext.hasOwnProperty(p)) {
            const descriptor = Object.getOwnPropertyDescriptor(globalContext, p);
            const { writable, configurable, enumerable } = descriptor!;
            if (writable) {
              Object.defineProperty(target, p, {
                configurable,
                enumerable,
                writable,
                value,
              });
            }
          } else {
            // @ts-ignore
            target[p] = value;
          }

          if (variableWhiteList.indexOf(p) !== -1) {
            // @ts-ignore
            globalContext[p] = value;
          }

          updatedValueSet.add(p);

          this.latestSetProp = p;

          return true;
        }

        if (process.env.NODE_ENV === 'development') {
          console.warn(`[qiankun] Set window.${p.toString()} while sandbox destroyed or inactive in ${name}!`);
        }

        // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，在沙箱卸载的情况下应该忽略错误
        return true;
      },

      get: (target: FakeWindow, p: PropertyKey): any => {
        this.registerRunningApp(name, proxy);

        if (p === Symbol.unscopables) return unscopables;
        // avoid who using window.window or window.self to escape the sandbox environment to touch the really window
        // see https://github.com/eligrey/FileSaver.js/blob/master/src/FileSaver.js#L13
        if (p === 'window' || p === 'self') {
          return proxy;
        }

        // hijack globalWindow accessing with globalThis keyword
        if (p === 'globalThis') {
          return proxy;
        }

        if (
          p === 'top' ||
          p === 'parent' ||
          (process.env.NODE_ENV === 'test' && (p === 'mockTop' || p === 'mockSafariTop'))
        ) {
          // if your master app in an iframe context, allow these props escape the sandbox
          if (globalContext === globalContext.parent) {
            return proxy;
          }
          return (globalContext as any)[p];
        }

        // proxy.hasOwnProperty would invoke getter firstly, then its value represented as globalContext.hasOwnProperty
        if (p === 'hasOwnProperty') {
          return hasOwnProperty;
        }

        if (p === 'document') {
          return document;
        }

        if (p === 'eval') {
          return eval;
        }

        const value = propertiesWithGetter.has(p)
          ? (globalContext as any)[p]
          : p in target
          ? (target as any)[p]
          : (globalContext as any)[p];
        /* Some dom api must be bound to native window, otherwise it would cause exception like 'TypeError: Failed to execute 'fetch' on 'Window': Illegal invocation'
           See this code:
             const proxy = new Proxy(window, {});
             const proxyFetch = fetch.bind(proxy);
             proxyFetch('https://qiankun.com');
        */
        const boundTarget = useNativeWindowForBindingsProps.get(p) ? nativeGlobal : globalContext;
        return getTargetValue(boundTarget, value);
      },
      
      
  }
```

首先`createFakeWindow`创建一个假的window，这里会遍历复制window所有的configurable属性为false的内容。创建一个沙箱环境。



get方法的时候就直接获取假沙箱的内容，获取不到就去获取window的



set也是直接操作fakewindow





```
  active() {
    if (!this.sandboxRunning) activeSandboxCount++;
    this.sandboxRunning = true;
  }

  inactive() {
    if (process.env.NODE_ENV === 'development') {
      console.info(`[qiankun:sandbox] ${this.name} modified global properties restore...`, [
        ...this.updatedValueSet.keys(),
      ]);
    }

    if (--activeSandboxCount === 0) {
      variableWhiteList.forEach((p) => {
        if (this.proxy.hasOwnProperty(p)) {
          // @ts-ignore
          delete this.globalContext[p];
        }
      });
    }

    this.sandboxRunning = false;
  }
```

激活如上，

关闭的话会把System这些对象删除。因为这些对象只有在真实window上才有效





###  4.预加载

通过默认fetch保存字符

```
start({
prefetch:true,
})
```

通过上面启动



```
export function prefetchImmediately(apps: AppMetadata[], opts?: ImportEntryOpts): void {
  if (process.env.NODE_ENV === 'development') {
    console.log('[qiankun] prefetch starting for apps...', apps);
  }

  apps.forEach(({ entry }) => prefetch(entry, opts));
}


function prefetch(entry: Entry, opts?: ImportEntryOpts): void {
  if (!navigator.onLine || isSlowNetwork) {
    // Don't prefetch if in a slow network or offline
    return;
  }

  requestIdleCallback(async () => {
    const { getExternalScripts, getExternalStyleSheets } = await importEntry(entry, opts);
    requestIdleCallback(getExternalStyleSheets);
    requestIdleCallback(getExternalScripts);
  });
}
```

提前获取了一次外链



因为importEntry中的脚本是通过fetch获取到的文本，先执行就是保存了一遍，不是通过浏览器强制缓存，存的。



###  5. 通信

主应用

```ts
import { initGlobalState, MicroAppStateActions } from 'qiankun';

// 初始化 state
const actions: MicroAppStateActions = initGlobalState(state);

actions.onGlobalStateChange((state, prev) => {
  // state: 变更后的状态; prev 变更前的状态
  console.log(state, prev);
});
actions.setGlobalState(state);
actions.offGlobalStateChange();
```

微应用

```ts
// 从生命周期 mount 中获取通信方法，使用方式和 master 一致
export function mount(props) {
  props.onGlobalStateChange((state, prev) => {
    // state: 变更后的状态; prev 变更前的状态
    console.log(state, prev);
  });


  props.setGlobalState(state);
}
```

1. qiankun 通过发布订阅模式来实现应用间通信，状态由框架来统一维护，每个应用在初始化时由框架生成一套通信方法，应用通过这些方法来更改全局状态和注册回调函数，全局状态发生改变时触发各个应用注册的回调函数执行，将新旧状态传递到所有应用







###  总结

总之qiankun是一个较完善的微前端解决方案，但是在使用的过程中也有可能出现一些问题，body挂载。

而且用了proxy沙箱后会很慢



# 几个示例表示展示他们的慢（with和proxy）

如下代码，没有proxy  耗时： 2ms

```
const fakeWindow = { window: { testVar: 't'} } 
console.time(); for (let i=0; i<120000;i++ ) 
var a = fakeWindow.window; 
console.timeEnd();
```

有with

```js
const fakeWindow = { window: { testVar: 't'} } 
console.time(); for (let i=0; i<120000;i++ ) 
with(fakeWindow){  var a = fakeWindow.window;} 
console.timeEnd();
```

40ms秒

如下代码，**有proxy** 耗时：4ms

```
 const fakeWindow = { window: { testVar: 't'} } 
 const pwindow = new Proxy(fakeWindow, { get(target, p) { return target[p] } }) 
 console.time() 
 for (let i=0; i<120000;i++ ) var a = pwindow.window 
 console.timeEnd()
```

如下代码，**有proxy且有with** 耗时：80ms！

```js
  const fakeWindow = { window: { testVar: 't'} }
  const pwindow = new Proxy(fakeWindow, {
    get(target, p) { 
      if (p === 'window') return pwindow
      return target.window[p]
    }
  })
  console.time()
  with(pwindow) {
    for (let i=0; i<120000;i++ ) var a = window.testVar
  }
  console.timeEnd()
```







```js
function func() {
    console.time("func");
    var obj = {
        a: [1, 2, 3]
    };
    for (var i = 0; i < 100000; i++) {
        var v = obj.a[0];
    }
    console.timeEnd("func");//0.847ms
}
func();

function funcWith() {
    console.time("funcWith");
    var obj = {
        a: [1, 2, 3]
    };
    with (obj) {
        for (var i = 0; i < 100000; i++) {
            var v = a[0];
        }
    }
    console.timeEnd("funcWith");//88.260ms
}
funcWith();
```



这里**JavaScript 引擎会在编译阶段进行性能优化**。其中有些优化依赖于能够根据代码的词法进行静态分析，并预先确定所有变量和函数的定义位置，才能在执行过程中快速找到标识符。