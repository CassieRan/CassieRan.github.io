---
layout: post
title: Web移动端适配你应该了解得再多一些
tags: [移动端, 适配]
comments: true
---



之前在开发个人项目的时候，想要实现响应式地适配移动端和PC端，遇到了诸如不同设备像素比，字体大小等问题，当时只是盲目地参考网上的解决方案，但是一些问题一直萦绕心头，所以花了一天时间去探了个究竟。



### 第一章 重要概念

**1、Retina高分辨率显示屏**

> Retina显示屏（**英文：**Retina Display）是一种具备足够高像素密度而使得人体肉眼无法分辨其中单独像素点的液晶屏，最初采用该种屏幕的产品iPhone 4，其屏幕分辨率为960×640（每英寸像素数326ppi）。这种分辨率在正常观看距离下足以使人肉眼无法分辨其中的单独像素。

以iPhone 4为例，屏幕尺寸为480×320，工作时渲染出960×640个像素点，其中每四个像素一组，输出原来屏幕的一个像素显示的大小区域内的图像。这样一来，用户所看到的图标与文字的大小与原来的480×320分辨率显示屏相同，但精细度是原来的4倍，但对于特殊元素，如视频与图像，则以一个位图像素对应一个设备像素的方式显示。通过下图对比就可以较明显地观察到这种关系。



![](https://upload.wikimedia.org/wikipedia/commons/a/ad/Non-Retina_Display.jpg)

iPhone 3GS ( 屏幕尺寸：480×320  分辨率：480×320 )

![](https://upload.wikimedia.org/wikipedia/commons/thumb/6/63/Retina_Display.jpg/1280px-Retina_Display.jpg)

iPhone 4 ( 屏幕尺寸：480×320  分辨率：960×640 )



虽然iPhone4的分辨率提高了，但它不同于普通显示屏，提升分辨率不是为了显示更多内容，而是为了提升显示相同内容时的精细程度。故不会产生分辨率提升使屏幕文字与图像变小，造成阅读困难的问题。

刚开始看到上面的内容的时候我非常疑惑：

像素到底怎么定义的，难道不是显示器能够显示的最小单元吗？所谓屏幕像素、衡量分辨率的长边像素数和短边像素数所指的像素不一样吗？

于是去查了像素相关的资料。

**2、像素并非像素**

> 像素，为视频显示的基本单位。每个这样的消息元素不是一个点或者一个方块，而是一个抽象的取样。一个像素通常被视为视频的最小的完整取样。这个定义和上下文很相关。

看到这句话的我更加懵了，而且这颠覆了我对“像素”的认识，它是一个抽象的取样？那我们平常在CSS里面写的px到底是什么？

> 不同的设备，图像的基本采样单元是不同的。

比如电脑显示器如LCD或CRT是由一个个极小的点来描绘视频的，显示器的像素等于点距，而打印机的像素是打印机的墨点。

所以像素对于不同的上下文，它的定义是不同的。也就是说不同分辨率的显示屏的像素颗粒是不一样的，这样会导致相同像素数的内容在不同分辨率的显示屏上所显示的大小不一，用户的阅读体验不佳。

为了保持阅读体验一致，浏览器会对像素值进行缩放调节，使一定像素的长度在不同设备上看上去的大小差不多。于是有了**CSS像素**即**参考像素**的概念，CSS规范中定义，1参考像素即为从一臂之遥看解析度为96DPI的设备输出（即1英寸96点）时，1点（即1/96英寸）的视角。 CSS样式代码中使用的px就是参考像素。

这样一来，1CSS像素可能包含多个设备像素（这里也叫物理像素）。

![](https://upload-images.jianshu.io/upload_images/8133-5669b7902cf35255.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

**3、设备像素比（device pixel ratio）**

**设备像素比 = 设备像素/设备独立像素**

此处的设备独立像素（device independent pixel）也叫逻辑像素，不难推出设备独立像素 = CSS像素。

对与iPhone 4来说，它的设备像素比为2。

**4、像素密度（pixels per inch）**

衡量单位面积内拥有的物理像素的数量，单位为像素每英寸，计算方式：

![](/images/formula.jpg)
其中，

$d_p$为屏幕对角线的物理分辨率；

$w_p$为屏幕横向的物理分辨率；

$h_p$为屏幕纵向的物理分辨率；

$d_i$为屏幕对角线的长度（*inch*）；

由此可见，ppi越高，每英寸的像素数越多。

**备注**：设备像素比一般是ppi/160的整数倍。



### 第二章 开发中的问题

**1、高清图片问题**

从上面的内容我们知道了：

在普通屏幕下，1个css像素 对应 1个物理像素（`1:1`）。

在Retina屏幕下，1个css像素对应 4个物理像素（`1:4`）。

前面有提到，对于图像来说，一个位图像素对应一个物理像素显示的效果是最高清的。在普通屏幕下确实是这样，但是同样尺寸的图片在Retina屏幕下就会出现图片模糊的情况，原因是Retina屏下，物理像素是图片位图像素的4倍，而单个位图像素是不可以再分割的，4个物理像素显示的是这个位图像素的近取色，所以导致了图片模糊，用一张图表示：

![](http://divio.qiniudn.com/Fuex59zSiV9pbaJG-s9wg_UpCERP)

**解决办法**：高倍图，即dpr为2的使用2倍图，dpr为3的使用3倍图。

**注意**：普通屏幕下最好不要使用高倍图，如果1个物理像素对应4个位图像素会出现什么情况呢？它的颜色也会通过一定算法得到近取色，这个过程称作downsampling，这样的图片虽然不会模糊，但是会少一些锐利度，有些色差。而且从前端性能角度来看，增加了不必要的资源大小。

其实这里的原理跟放大或缩小图像类似，都会因为颜色计算而使图像质量降低。

所以最好的做法是不同的dpr下，加载不同尺寸的图片。具体的方法这里就不介绍了。

**2、border: 1px问题**

实际上CSS中border-width: 1px在普通屏和高清屏上并无太大差别，但是一般设计师要求一条直线尽可能的细，即屏幕能显示的最小宽度，所以此处1px实际指的是1物理像素，倘若在dpr为2的屏幕上，CSS像素应该是0.5px，但是不是所有手机浏览器都支持border-width: 0.5px的写法，有可能被当成0px或1px处理。那如何实现呢，这里提供两种常用的解决办法：

**目标元素缩放法**

```css
// 下边框
div  {
    position: relative;
}
div::after {
    content: "";
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    border-bottom: 1px solid #000;
    transform: scale(.5);
}
```

这种方法简单粗暴，但是用起来还是比较麻烦的，只要需要实现0.5px的地方都需要写一遍以上的代码。

**全局缩放法**

在head标签中添加如下meta标签，设置scale为0.5:

```html
<meta name="viewport" content="width=device-width, initial-scale=0.5, maximum-scale=0.5, minimun=0.5, user-scalable=0">
```

这样页面所有的border-width: 1px都将被缩小至0.5px，但是同样会带来其他问题，页面字体和容器大小会被缩放。这个问题我们先放一放，到下一章节解决。

**3、引申**

这里说到了name为viewport的标签，相信做过移动开发的程序员都用过，但是相信大多数对它都是一种熟悉又陌生的感觉。

[A tale of two viewports](https://www.quirksmode.org/mobile/viewports.html)的作者ppk认为，移动设备上有三个viewport：layout viewport、visual viewport、ideal viewport。

layout viewport：手机浏览器默认的区域，默认好像是980px、1024px等等，宽度通过document.documentElement.clientWidth来获取；

visual viewport：手机浏览器可视区域，宽度通过window.innerWidth来获取；

ideal viewport：网站设计与移动设备完美贴合，不需要用户缩放和滚动就能正常查看网站内容，ideal viewport的宽度等于移动设备的屏幕宽度。

![](https://www.quirksmode.org/mobile/pix/viewport/mobile_layoutviewport.jpg)

![](https://www.quirksmode.org/mobile/pix/viewport/mobile_visualviewport.jpg)

![](https://www.quirksmode.org/mobile/pix/viewport/mobile_viewportzoomedout.jpg)



meta标签可以对viewport进行控制，其中：

width属性用来设置layout viewport的宽度，通常我们通过书写width=device-width来得到ideal viewport；

initial-scale属性控制页面最初加载时的缩放等级；

maximum-scale、minimum-scale及user-scalable属性控制允许用户以怎样的方式放大或缩小页面。

实践得知以下两种方式都能实现ideal viewport：

```html
<meta name="viewport" content="width=device-width">
```

```html
<meta name="viewport" content="initial-scale=1">
```

可以推断缩放是相对于ideal viewport进行的。

备注：设置为ideal viewport的最佳方式是：

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```



**设置initial-scale后对viewport的影响**

前面提到过，缩放是相对于 ideal viewport 缩放的，缩放值越大，当前viewport的宽度就会越小，反之亦然。例如在iphone 4中，ideal viewport的宽度是320px，如果我们设置 initial-scale=2，此时visiual viewport的宽度会变为只有160px了，这就是说在屏幕宽度没有变的情况下，1px（CSS像素）变得跟原来的2px（CSS像素）的长度一样了，所以放大2倍后原来需要320px才能填满的宽度现在只需要160px就做到了。因此，可以得出下面的公式：

**visual viewport宽度=ideal viewport宽度/当前缩放值**

**当前缩放值=ideal viewport宽度/visual viewport宽度**



**媒体查询**

在使用媒体查询时，width指的是layout viewport，device-width指的是移动设备屏幕的宽度，两者都是用CSS像素来衡量的。



### 第三章 多屏幕适配

1、上一章讲到页面缩放后字体、容器也跟着缩放，为了让页面看起来正常，必须将字体大小、容器宽高设置为原来的1/initial-scale倍。

2、市面上的移动设备种类繁多，为了更好的视觉体验，我们希望相同的内容在不同尺寸的移动设备上表现出相同的比例。对比未适配与适配后的差异：


未适配：
![](/images/320*568.png)
![](/images/375*667.png)
![](/images/414*736.png)

适配后：
![](/images/320*568_rem.png)
![](/images/375*667_rem.png)
![](/images/414*736_rem.png)


很明显，设计的初衷是三个元素并列一排并横向占据整个屏幕，如果不适配的话，就会因为屏幕宽了多出空白区域或因为屏幕窄了把内容给挤下去。

适配的方式多样，可以采用相对长度单位，em、rem、%、vm等来实现，这里只介绍rem布局。

> CSS以rem为单位的元素，它的大小是相对于html根节点的font-size值的。

我们先来进行一次换算，如果想要相同的内容在不同的设备上显示出相同的比例，则：

设计稿元素尺寸/设计稿宽度=元素CSS像素/视口宽度

上面已知 visual viewport宽度=ideal viewport宽度/当前缩放值=deal viewport宽度×dpr，

综合得到 **元素CSS像素=设计稿元素尺寸/设计稿宽度×viewport宽度×dpr**，

我们令 rem=ideal viewport宽度×dpr，

则在书写CSS代码时， **{property name}: {设计稿元素尺寸/设计稿宽度}rem**。



综上所述，html根元素的font-size应该设置成ideal viewport宽度×dpr，由于不同的手机ideal viewport宽度以及dpr不同，所以最好动态设置html根元素的font-size：

```javascript
var dpr, rem, scale;
var docEl = document.documentElement;
var styleEl = document.createElement( 'style' );
var metaEl = document.querySelector( 'meta[name="viewport"]' );

dpr = window.devicePixelRatio || 1;
rem = docEl.screen.width * dpr / 10; // 这里除以10是为了让写CSS像素值时数值不至于太小
scale = 1 / dpr;

// 设置viewport，进行缩放，达到高清效果
metaEl.setAttribute( 'content', 'initial-scale=' + scale + ',maximum-scale=' + scale + ', minimum-scale=' + scale + ',user-scalable=no' );

// 设置data-dpr属性，留作的css hack之用
docEl.setAttribute( 'data-dpr', dpr );

// 动态写入样式，设置根节点字体大小
docEl.firstElementChild.appendChild( styleEl );
styleEl.innerHTML = 'html{font-size:' + rem + 'px!important;}';
```

这种方式，可以精确地算出不同屏幕所应有的rem基准值，缺点就是要加载这么一段js代码，并且编写CSS代码时要经过较复杂的换算（**{property name}: {设计稿元素尺寸／设计稿宽度 * 10}rem**）。



那么问题又来了，**字体需要用rem吗**？

最好不要，一是通过rem表示的字体不精确，二是大多数设计要求在一定屏幕大小范围内，显示的字体一样大。

所以字体通常采用媒体查询的方式设置，这里需要注意两点：

（1）CSS字体大小应该为：

**设计稿字体大小/设计稿倍数×dpr**

举例说明如果设计稿基于iPhone 6设计的尺寸为750×1334的二倍图，某字体标记28px，则字体设置应该为28／2 × 2 = 28px。

（2）媒体查询时应该查询device-width而不是width，原因是页面经缩放width会大于或小于屏幕宽度。

```css
@media (min-device-width : 375px) {
   body{font-size: 12px;}
}

@media (min-device-width : 414px) {
   body{font-size: 14px;}
}
```



这里又引申出一个需求——相同内容在不同移动设备上显示尺寸相同，其实这跟设置固定的font-size类似，为了方便，可以用less写一个通用的mixin:

```scss
@ratio: 2; //预先定义设计图倍率
.px2px(@name, @px){
    @{name}: round(@px / @ratio) * 1px;
    [data-dpr="2"] & {
        @{name}: round(@px / @ratio * 2) * 1px;
    }
    // for mx3
    [data-dpr="2.5"] & {
        @{name}: round(@px / @ratio * 2.5) * 1px;
    }
    // for px2
    [data-dpr="2.625"] & {
        @{name}: round(@px / @ratio * 2.625) * 1px;
    }
    // for 小米note
    [data-dpr="2.75"] & {
        @{name}: round(@px / @ratio * 2.75) * 1px;
    }
    [data-dpr="3"] & {
        @{name}: round(@px / @ratio * 3) * 1px
    }
    // for px2 XL
    [data-dpr="3.5"] & {
        @{name}: round(@px / @ratio * 3.5) * 1px;
    }
    // for 三星note4
    [data-dpr="4"] & {
        @{name}: round(@px / @ratio * 4) * 1px;
    }
}
```

用的时候，就像这样：

```scss
.px2px(font-size, 32);
.px2px(padding, 20);
.px2px(right, 8);
```





参考：

「1」[在移动浏览器中使用viewport元标签控制布局](https://developer.mozilla.org/zh-CN/docs/Mobile/Viewport_meta_tag)

「2」[A tale of two viewports](https://www.quirksmode.org/mobile/viewports.html)

「3」[移动端高清、多屏适配方案](https://div.io/topic/1092)

