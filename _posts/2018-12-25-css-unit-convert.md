---
layout: post
title: px转换成rem/vw的解决方案
tags: [移动端适配, loader, postcss plugin, rem, vw]
comments: true
excerpt: 近期在做一个移动端的web网页，当中选用了vm自适应适配方案。然而从设计图标注的px转换到vm是个麻烦事，作为程序员的我很抗拒人工计算，因为那样做CSS代码的可读性会变低，而且编码效率也很低。
cover: 
---


**TL;DR**

近期在做一个移动端的web网页，当中选用了vm自适应适配方案。然而从设计图标注的px转换到vm是个麻烦事，作为程序员的我很抗拒人工计算，因为那样做CSS代码的可读性会变低，而且编码效率也很低。经过实践我找出了以下几种解决方案，这里列举出来希望对同样患懒癌的你有些许帮助：

##### 0. Sass/Less

Sass/Less是通过mixin或者function进行计算，把计算工作交给mixin或者function。这个方法我曾在[Web移动端适配你应该了解得更多一些](https://cassieran.github.io/you-should-know-more-if-you-being-a-webapp-developer/)一文中提到过。

```scss
$ui-width: 750px;

@function px2vw($px) {
  @return $px / $ui-width * 100vw;
}

#sidebar { width: px2vw(50px); }
```

它虽然实现了我们的需求，可读性还行，但是我并不建议采用它，因为：

- 通常我们会把函数写到公用的scss文件中，那么如果您采用的是模块化开发，则需要在每个模块文件中导入这个公用的scss文件；
- 原来的`*px`必须由`px2vw(*px)`代替，代码量增加；
- 如果有一天我不想采纳这种方法了，修改起来很麻烦，扩展性低，不易维护；
- 不能转换内联样式单位。

##### 1. Postcss plugin

这是当下比较流行的解决方案，npm上有很多转换CSS单位的postcss插件，如postcss-px-to-viewport, postcss-plugin-px2rem等。它们是通过遍历CSS属性，匹配到`*px`时就进行换算和替换。最初我选的是postcss-px-to-viewport插件，我还对它进行了扩展支持转换rem，这里简单介绍一下：

```javascript
// convert.js
module.exports = postcss.plugin('postcss-unit-convert', function (options) {

    const opts = Object.assign({}, options)
    const pxReplace = createPxReplace(opts.UIWidth, opts.minPixelValue, opts.unitPrecision, opts.targetUnit, opts.rem)

    return function (root) {
        // 如果目标转换单位是rem，则设置html跟节点的字体大小为options.rem
        if (opts.targetUnit === 'rem' && opts.rem) {
            css.append(`html{ font-size: ${opts.rem}px}`) 
        }
		// 遍历css属性
        root.walkDecls(function (decl) { 
            // 如果当前属性不包含px，直接跳过
            if (decl.value.indexOf('px') === -1) return 
            // 如果options.fontUnit为px则font-size属性直接跳过，这是为了不转换font-size的单位
            if (opts.fontUnit === 'px' && decl.prop === 'font-size') return 
            // 如果当前容器包含黑名单的容器名称，则直接跳过
            if (blacklistedSelector(opts.selectorBlackList, decl.parent.selector)) return 
			// 转换及替换 关键！
            decl.value = decl.value.replace(pxRegex, pxReplace) 
        })
    }
})

/*
 * createPxReplace 根据目标单位返回替换规则
 * 如果px前没有数值，不替换
 * 如果数值小于等于option.minPixelValue，不替换
 */
function createPxReplace (UIWidth, minPixelValue, unitPrecision, targetUnit, rem) {
    return function (m, $1) {
        if (!$1) return m
        const pixels = parseFloat($1)
        if (pixels <= minPixelValue) return m
        if (targetUnit === 'vw') return px2vw(pixels, UIWidth).toFixed(unitPrecision) + targetUnit
        if (targetUnit === 'rem') return px2rem(pixels, rem).toFixed(unitPrecision) + targetUnit
    }
}

function blacklistedSelector (blacklist, selector) {
    if (typeof selector !== 'string') return
    return blacklist.some(function (regex) {
        return selector.match(regex) && selector.match(regex).length
    })
}

function px2vw (px, UIWidth) {
    return px / UIWidth * 100
}

function px2rem (px, rem) {
    return px / rem
}
```

使用的时候在.postcssrc.js中引入自定义插件就好了。

```javascript
module.exports = {
    "plugins": [
        require('autoprefixer')({})
        require('./convert.js')({
            UIWidth: 750, // 设计稿宽度
            unitPrecision: 3, // 指定'px'转换为目标单位数值的小数位数
            targetUnit: 'rem', // 指定需要转换成的目标单位，vw or rem
            fontUnit: 'px', // 指定字体单位，如果是px则不转换
            selectorBlackList: ['.ignore'], // 指定不转换的容器名
            minPixelValue: 1, // 小于或等于'1px'转换
            rem: 100 // rem基数，即根元素字体大小，目标单位为rem式需要
        })
    ]
}
```

![convert_postcss](/images/convert_postcss.png)

这种方式可以很细粒度地控制选择器，属性，属性值等，也很方便地给根元素设置字体大小，但最后我还是没有采用它，因为它存在一个弊端：

- 因为是postcss插件，所以只能处理css，不能处理内联样式单位（在编写Vue的过程中，有时会在template中为元素指定style属性）。

网上有工程师指出可以利用postHTML处理，但是我个人还是觉得不够优雅，那样需要针对HTML和CSS处理两次。那么到底有没有办法可以一次性处理呢？答案就是下面的Webpack loader。

##### 2. Webpack loader

如果您使用过webpack，应该已经了解过一些loader的知识，这里我是通过自己编写一个loader来实现我的需求，如果您也想这样做，可以参考[编写一个loader](https://webpack.docschina.org/contribute/writing-a-loader/)。

实现思路跟postcss插件类似，区别在于这里更多是依赖正则匹配来完成。因为它不能像postcss那样容易获取属性名，所以我通过加指定注释的方式来忽略当前属性的单位转换。

```js
// unit-convert-loader.js
const loaderUtils = require('loader-utils');
const pxRegex = /url\([^\)]+\)|(\d*\.?\d+)px((\;?)(\s*)(\/\*(\s*)([\s\S]*)(\s*)\*\/))?/ig

exports.default function (source) {
    const options = loaderUtils.getOptions(this)
    const opts = Object.assign({}, options)
    const pxReplace = createPxReplace(opts.UIWidth, opts.minPixelValue, opts.unitPrecision, opts.targetUnit, opts.rem)
    source = source.replace(pxRegex, pxReplace)

    return `export default ${source}`
}

function createPxReplace(UIWidth, minPixelValue, unitPrecision, targetUnit, rem) {
    return function (m, $1, $2, $3, $4, $5, $6, $7, $8) {
        if (!$1 || $7 === 'not convert') return m
        const pixels = parseFloat($1)
        if (pixels <= minPixelValue) return m
        if (targetUnit === 'vw') return px2vw(pixels, UIWidth).toFixed(unitPrecision) + targetUnit
        if (targetUnit === 'rem' && rem) return px2rem(pixels, rem).toFixed(unitPrecision) + targetUnit
    }
}
```

使用的时候在webpack配置中加上自定义的loader：

```javascript
config.module
    .rule('vue')
    .test(/\.vue$/)
    .use('unit-convert-loader')
    .loader(path.resolve('src/assets/js/unit-convert-loader.js'))
    .options({
        UIWidth: 750,
        unitPrecision: 3,
        targetUnit: 'rem',
        minPixelValue: 1,
        rem: 100
    })
```

![convert_loader](/images/convert_loader.png)

成功啦！在选择targetUnit为rem时，我还没找到设置html字体大小的更好的办法，如果您有idea，请您跟我分享吧！在没有更好的办法前，我建议您使用vw单位，它不会让您有这个困扰。

##### 3. 总结

Postcss plugin已经可以满足大多数项目的需求，如果您无需转换html中和js中的单位，您可以放心地使用它。

Webpack loader可以更全面地转换CSS单位，但是不是细粒度的。您可以通过在属性后注释not convert来阻止这个属性的转换。就像这样：

```css
.container {
    width: 100px; /* not convert */
    height: 100px;
    margin: auto;
    background-color: darkmagenta;
}
```

最后附上unit-convert-loader的仓库地址，如果您发现漏洞或有更好的实现欢迎提PR，感谢您的阅读！
