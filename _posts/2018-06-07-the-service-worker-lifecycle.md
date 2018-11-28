---
layout: post
title: PWA初探之Service Worker生命周期（译）
tags: [PWA, service worker]
comments: true
excerpt: Service Worker 的生命周期可以分为6个阶段：解析(parsed)、安装(installing)、安装完成(installed)、激活(activating)、激活完成(activated)、闲置(redundant)
cover: https://bitsofco.de/content/images/2016/07/Lifecycle-3.png
---



翻译加工来自 Ire Aderinokun 文章 [The Service Worker Lifecycle](https://bitsofco.de/the-service-worker-lifecycle/)，感谢原作者。

Service Worker 的生命周期可以分为6个阶段：解析(parsed)、安装(installing)、安装完成(installed)、激活(activating)、激活完成(activated)、闲置(redundant)

![Service Worker States](https://bitsofco.de/content/images/2016/07/Lifecycle-3.png)



#### **Paersed**

当您第一次尝试注册 Service Worker 时，用户浏览器会解析 Service Worker 脚本并获得入口，如果解析成功，您将接收到一个 Service Worker registration 对象，它包含了 Service Worker 的状态及作用域信息。

```javascript
/* In main.js */
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('./sw.js')
    .then(function(registration) {
        console.log("Service Worker Registered", registration);
    })
    .catch(function(err) {
        console.log("Service Worker Failed to Register", err);
    })
}
```

成功注册 Service Worker 并不代表 Service Worker 安装或激活成功，只能说此时代码被成功解析。

当检查service-worker.js与受控网页同源，且该源是https安全源，在成功注册之后 Service Worker 会变为下一个状态——installing。



#### **Installing**

您可以在 registration 对象的 installing 属性中检测到此状态。

```javascript
/* In main.js */
navigator.serviceWorker.register('./sw.js').then(function(registration) {
    if (registration.installing) {
        // Service Worker is Installing
    }
})
```

在 Installing 阶段， install 事件会被触发。在 install 事件回调中，您通常需要缓存某些静态资产。如果所有文件均已成功缓存，那么服务工作线程就安装完毕。如果任何文件下载失败或缓存失败，那么安装步骤将会失败，服务工作线程就无法激活（也就是说，不会安装）。 如果发生这种情况，不必担心，它下次会再试一次。 但这意味着，如果安装完成，您可以知道您已在缓存中获得那些静态资产。

```javascript
/* In sw.js */
self.addEventListener('install', function(event) {
    event.waitUntil(
        caches.open(currentCacheName).then(function(cache) {
            return cache.addAll(arrayOfFilesToCache);
        })
    );
});
```

如果在回调中调用 install 事件的 waitUntil 方法，则 install event 在 Promise 对象 resolved 后成功安装；如果 Promise 对象 rejected ，则 install event 会失败，Service Worker 的状态会变成 redundant 。

```javascript
/* In sw.js */
self.addEventListener('install', function(event) {
    event.waitUntil(
        return Promise.reject(); // Failure
    );
});
```



#### **Installed / Waiting**

如果安装成功，Service Worker 的状态变为 installed （也叫 waiting ）。处于这个状态时， Service Worker 是有效的但是是未激活的 worker，还不能控制页面，而是等待从当前 worker 获得控制权，您可以在 registration 对象的 waiting 属性中检测到此状态。 

```javascript
/* In main.js */
navigator.serviceWorker.register('./sw.js').then(function(registration) {
    if (registration.waiting) {
        // Service Worker is Waiting
    }
})
```

这是更新新版本或自动更新缓存的绝佳时机。



#### **Activating**

在以下情况之一时，处于 Waiting 状态的 worker 的 activating 状态会被触发：

- 当前没有激活的 worker
-  `self.skipWaiting()` 在 sw.js 中有调用
- 用户导航离开当前页面，从而释放了前一个 active worker
- 经过了指定时间段，从而释放了前一个 active worker

在 activating 状态，activate 事件会被触发，在 install 事件回调中，您通常需要清除旧缓存

```javascript
/* In sw.js */
self.addEventListener('activate', function(event) {
    event.waitUntil(
        // Get all the cache names
        caches.keys().then(function(cacheNames) {
            return Promise.all(
                // Get all the items that are stored under a different cache name than the current one
                cacheNames.filter(function(cacheName) {
                    return cacheName != currentCacheName;
                }).map(function(cacheName) {
                    // Delete the items
                    return caches.delete(cacheName);
                })
            ); // end Promise.all()
        }) // end caches.keys()
    ); // end event.waitUntil()
});
```

同上，如果 Promise 对象 rejected ，则 activate event 会失败，Service Worker 的状态会变成 redundant 。



#### **Activated**

如果激活成功，Service Worker 状态会变成 active ，在这个状态下，Service Worker 是一个可以完全控制网页的激活 worker，您可以在 registration 对象的 active 属性中检测到此状态。 

```javascript
/* In main.js */
navigator.serviceWorker.register('./sw.js').then(function(registration) {
    if (registration.active) {
        // Service Worker is Active
    }
})
```

当 Service Worker 被成功激活后，即可处理绑定的 fetch 和 message 事件

```javascript
/* In sw.js */
self.addEventListener('fetch', function(event) {
    // Do stuff with fetch events
});

self.addEventListener('message', function(event) {
    // Do stuff with postMessages received from document
});
```



#### **Redundant**

以下任一情况，Service Worker 都会变成 redundant：

- 如果安装失败
- 如果激活失败
- 如果有新的 Service Worker 将其替代成为现有的激活 worker
