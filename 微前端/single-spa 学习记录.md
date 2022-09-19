##  single-spa 学习记录



> Single-spa 是一个将多个单页面应用聚合为一个整体应用的 JavaScript 微前端框架
>
> 这句话是官方文档对这个库的详细介绍。但我认为，singleSpa只是劫持路由，激活app并执行一列的生命周期的一个底座。

[single-spa demo](https://github.com/czkm/Single-spa-demo/blob/master/vue-child/vue.config.js)

###  如何将多个应用聚合使用

理下思路，

```javascript
// name   app名字
// app  最终返回一个对象，包括bootstrap刚激活，mount挂载， unmount卸载的生命周期函数
// activewhen 匹配激活的路径
singleSpa.registerApplication({ name, app, activeWhen });

// 启动
singleSpa.start();
```

这两个函数基本就是single-spa的全部了。



所以对于应用的挂载和卸载都要我们自己写代码去处理。（vue 和react都可）



###  如何处理多个业务子应用

> 我们如果独立开发每一个子应用，我们必定会有自己的基础配置或特定包版本，和专属的后端接口开发。



这里有两种方法获取到生命周期对象app。



####  打包挂载到window

```javascript
       output: {
            library: "singleVue", // 导出名称
            libraryTarget: "window", //挂载目标
        },
```

直接挂载到window.singleVue使用

```javascript
singleSpa.registerApplication(
  //注册微前端服务
  'singleVue',
  async () => {
    	// 用一个script标签
      await runScript('http://127.0.0.1:3000/js/chunk-vendors.js');
      await runScript('http://127.0.0.1:3000/js/app.js');
      return window.singleVue;
  },
  (location) => location.pathname.startsWith('/vue-antd') // 配置微前端模块前缀
);;
```

这里拿到后就正常执行生命周期



####  通过请求拿到对应包

这里要引入一个知识[Import maps](https://github.com/WICG/import-maps)

这个功能是Chrome 89才支持的。它就是个映射

主要是利用浏览器支持的

```
<script type="module"></script>
```

来实现导入文件的。



但真真写的时候我们用SystemJS这个polyfill工具库。

```
    <script src="https://cdn.jsdelivr.net/npm/systemjs/dist/system.js"></script>
```

引入后

```js
singleSpa.registerApplication(
  'elementVue',
  async () => {
      await runScript('http://127.0.0.1:3001/js/chunk-vendors.js');
      // 获取某个包返回一个对象包含各个生命周期
      const abc = await window.System.import('http://127.0.0.1:3001/js/app.js');
      return abc;
  },
  (location) => location.pathname.startsWith('/vue-element')
);
```



```js
打包时记得以umd格式打包
    output: {
      library: 'elementVue', // 导出名称
      // libraryTarget: "window", //挂载目标
      libraryTarget: 'umd',
    },
```







###   出现的问题

1. js entry导入的形式无法并发，因为是操作dom元素，只能全部打包成一块，css不能分割开。

最主要的还是js文件要带个chunk编号，那导入的时候也要改变。

且需要在spa的时候一个个导入文件。

子应用所有资源打包到一个文件中，会失去 css 提取、静态资源并行加载。

如果不打包成一个资源就会获取不到。因为相对路径的缘故，在主应用会访问不到资源。





2. 样式隔离问题，所有样式都挂载head的style里面，会冲突
3. js隔离完全没做
4. 预加载，single-spa没做。
5. 通信，也没有







上面的问题并不是不能解决，在single-spa的基础上可以自定义代码模组来解决，single-spa只是一个基座。











###  相关资料

[SYstem](https://www.cnblogs.com/vvjiang/p/15240799.html)

[demo](https://github.com/czkm/Single-spa-demo/blob/master/vue-child/vue.config.js)







































