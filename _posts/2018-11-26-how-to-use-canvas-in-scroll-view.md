---
layout: post
title: canvas 无法在 scroll-view 中使用的解决方案
tags: [微信小程序, canvas, scroll-view]
comments: true
excerpt: 真机上滚动盒子内的 canvas 是不会随着盒子内容滚动的，看起来像是固定在某个位置，类似于 fixed 定位。
cover: 
---



​	最近正在做的微信小程序遇到一个需求，需要在一个滚动盒子里面显示 canvas，可是实践发现在真机上滚动盒子内的 canvas 是不会随着盒子内容滚动的，看起来像是固定在某个位置，类似于 fixed 定位。出现这种现象是小程序的原声组件限制所致的，官方文档如下说：

> 原生组件还无法在 `scroll-view`、`swiper`、`picker-view`、`movable-view` 中使用。

​	官方只是说了无法使用，但并没有说使用了会出现什么样的现象，只有亲自踩雷了才知道。而且实践中发现 canvas 不仅仅不能在 scroll-view 中使用，在 `overflow` 为 `scroll` 的普通 view 中使用也会出现令人不适的 bug 。

![](/images/canvas.gif)

​	不得不说这种限制真的超鸡肋。工作中类似的需求很常见，如果因此改需求改设计也说不过去吧，所以只能想办法改代码来实现。



**实现思路**

1. 实现思路 

   用 image 组件替代 canvas 

   - 在 canvas 上绘制图形/图像
   - 将 canvas 转换成媒体图片
   - 隐藏 canvas ，显示 image

2. API

   这里的关键点在于第二步，如何将 canvas 转换成媒体图片？去官方文档一查，果然有现成的 API ：

   - wx.canvasGetImageData 获取 canvas 区域隐含的像素数据
   - wx.canvasToTempFilePath 把当前画布指定区域的内容导出生成指定大小的图片



**具体实现**

1. 在 canvas 上绘制图形/图像

   ```html
   <!-- wxml -->
   <view class="page-body">
     <scroll-view scroll-y class="page-body-wrapper">
       <view class='number'>1</view>
       <view class='number'>2</view>
       <view class='number'>3</view>
       <view class='number'>4</view>
       <view class='number'>5</view>
       <canvas canvas-id="canvas" class="canvas"></canvas>
       <image src='{{canvasUrl}}'></image>
       <view class='number'>6</view>
       <view class='number'>7</view>
       <view class='number'>8</view>
       <view class='number'>9</view>
       <view class='number'>10</view>
     </scroll-view>
   </view>
   ```

   ```javascript
   // js
   const ctx = wx.createCanvasContext("canvas");
   // draw something
   ctx.draw()
   ```

2. 将 canvas 转换成媒体图片

   - 生成临时图片路径

   ```javascript
   ctx.draw(false, function() {
       wx.canvasToTempFilePath({
           x: 0, // 画布区域的左上角横坐标
           y: 0, // 画布区域的左上角纵坐标
           width: 150, // 画布区域的宽度
           height: 130, // 画布区域的高度
           destWidth: 150, // 输出的图片的宽度
           destHeight: 130, // 输出的图片的高度
           canvasId: 'canvas', // 画布标识，传入 <canvas> 组件的 canvas-id
           fileType: 'png', // 目标文件的类型
           success(res) {
               this.setData({
                   canvasUrl: res.tempFilePath
               })
           }
       })
   })
   ```

   这种方法比较简单，不需要处理结果，直接显示生成的图片就好了。

   坑1: 你可能会发现生成的图片很模糊，这是因为以上是 canvas 画布多大就生成多大的图片，而现代手机的设备像素比一般都大于1，所以我们需要导出原画布大小乘设备像素比尺寸的图片才能高清显示，即先用 wx.getSystemInfo 获取当前设备的分辨率 pixelRatio 。然而眼尖的我又发现该 API 默认输出的是高清图片，所以为了省事可以不写这个选项。

   坑2: `filetype` 建议最好选择 png ，选择 jpg 在安卓系统中图片底色为黑色，不明原因，官方也未给出解释。

   - 生成像素数据

   ```javascript
   ctx.draw(false, function() {
   	wx.canvasGetImageData({
           x: 0,
           y: 0,
           width: 150,
           height: 130,
           canvasId: "canvas",
           success(res) {
             	const pngData = upng.encode(res.data.buffer,res.width,res.height)
             	const base64 = wx.arrayBufferToBase64(pngData)
             	this.setData({
             		canvasUrl: `data:image/png;base64,${base64}`
             	})
           }
       });
   })
   ```

   这种方法需要将像素数据处理成 base64 格式的图片，这里使用了 upng.js 。

3. 隐藏 canvas ，显示 image

   说到隐藏，可能你第一时间想到的是 `display: none` `opacity: 0` 之类的方法，可是在小程序中加上这些属性后 canvas 将会变得不可绘制，这里采用将其定位到屏幕之外的方法：

   ```css
   canvas {
       position: absolute;
       left: -999px;
   }
   ```



最后，来看看效果：

![](/images/canvas_normal.gif)

以上，希望对你有些许帮助。