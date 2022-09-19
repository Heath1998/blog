##  微前端ppt总结



###  微前端发展

以前写个模板，前后端分离，后端微服务

微服务就是将一个大的单体应用，拆分成多个低耦合的小型服务，支持独立部署。



比如

一个正常的购物系统有，用户服务，订单服务，数据分析几个。

拆出各自模块独立部署，各个应用后台只需从这些服务获取所需的数据，从而删去了大量冗余的代码，就剩个轻薄的控制层和前端。这一阶段的架构如下



做后端的时候有微服务，每个微服务可以单独运行，通过[注册中心](https://cloud.tencent.com/product/tse?from=10680)拉起成为一个大项目。

做移动端的时候我们可以组件化，每个组件都可以是一个app单独运行，我们通过一个中间件将每个组件拉起，组合成想要的app。

到了前端难道我们只能通过npm打包的方式去集成吗（体感太差，要打包）？带着这个问题，首先找到了IFrame



### 微前端发展

2018年: 第一个微前端工具single-spa在github上开源。实现技术无关和路由导入

2019年: 基于single-spa的qiankun问世。



###  single-spa路由匹配

```js
activeWhen: () =＞ location.pathname.startsWith('/vue');


const appShouldBeActive = app.status !== SKIP_BECAUSE_BROKEN && shouldBeActive(app);
```

匹配到的话执行生命周期钩子函数





###  如何做到技术⽆关?

我们肯定需要与浏览器原生的概念进行交互，不能涉及到任何在js以上层面的库。

就是规定协议，然后操作dom。

给一个生命周期大家在mounted和unmounted的时候操作dom挂载和卸载子应用。