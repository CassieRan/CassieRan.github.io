---
layout: post
title: PWA初探之注册service-worker
tags: [PWA, service worker]
comments: true
---





#### Code

想要安装 service-worker ，需要先在浏览器中进行注册。

```javascript
	// 注册 service worker
	if ( 'serviceWorker' in navigator ) {
      navigator.serviceWorker.register( '/service-worker.js' ).then( function ( registration ) {
        // 注册成功
        console.log( 'ServiceWorker registration successful with scope: ', registration.scope );
      } ).catch( function ( err ) {
        // 注册失败 
        console.log( 'ServiceWorker registration failed: ', err );
      } );
    }
```

现代浏览器 Chrome、Safari、Firefox 、Edge 和 Opera 都已经支持 service-worker。详细的浏览器支持情况可以在 Jake Archibald 的 [is Serviceworker ready](https://jakearchibald.github.io/isserviceworkerready/) 网站上查看。

如果注册成功，我们会接收到 ServiceWorkerRegistration 对象，它包含了ServiceWorker 的状态以及其作用域。打印如下：

ServiceWorkerRegistration: {

​    active:
        ServiceWorker: { 
            scriptURL: "http://localhost/service-worker.js", 
            state: "activated", 
            onstatechange: ƒ, 
            onerror: null
        },

​    installing: null,

​    navigationPreload: NavigationPreloadManager: {},

​    onupdatefound: ƒ (),

​    pushManager: PushManager: {},

​    scope: "http://localhost/",

​    sync: SyncManager: {},

​    waiting: null

}，

这里需要注意的是 ServiceWorker 的作用域与 service-worker.js 的位置有关，若置于项目根目录，则其可作用于整个项目。



#### Issues

**issue1**: ServiceWorker registration failed:  DOMException: Only secure origins are allowed.

使用服务工作线程，您可以劫持连接、编撰以及过滤响应。 这是一个很强大的工具。您可能会善意地使用这些功能，但中间人可会将其用于不良目的。 为避免这种情况，可仅在通过 HTTPS 提供的页面上注册服务工作线程，如此我们便知道浏览器接收的服务工作线程在整个网络传输过程中都没有被篡改。

在开发过程中，可以通过 localhost 使用服务工作线程（我是基于 Vue 和 WebPack 在开发，所以把 dev-server 的 host 选项修改成 'localhost' 就可以了），但如果要在网站上部署服务工作线程，需要在服务器上设置 HTTPS。

**issue2**: ServiceWorker registration failed:  DOMException: Failed to register a ServiceWorker: The script has an unsupported MIME type ('text/html').

下面是 Google's Web Developer Relations team 的工程师 **jeffposnick** 对该问题的解答

> In a development environment, you traditionally won't want a service worker installed (since local caching can mess up your ability to iterate on code changes), so there isn't one generated. The local dev HTTP server responds to the request for `/servrice-worker.js` generated from the service worker registration with a HTML error document, leading to that message being logged in the console. This error can be safely ignored.
>
> `npm run build` will generate the production deployment, including a `service-worker.js` in the correct location.

大致意思是，在开发环境下，service-worker 是不会被安装的，因为本地缓存会影响代码改变时的迭代更新，所以这个报错信息可以安全地忽略。当使用 npm run build 编译部署时，会正确地包含一个 service-worker.js 文件。

**Issue3**: Service Worker termination by a timeout timer was canceled because DevTools is attached.

这个打印日志是为了让开发者知道，通常在此时服务工作线程应该被杀死了，但是由于 DevTools 的开启从而阻止了这项工作。



附：Service worker生命周期

![Service Worker States](https://bitsofco.de/content/images/2016/07/Lifecycle-3.png)​
