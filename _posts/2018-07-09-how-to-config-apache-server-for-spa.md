---
layout: post
title: apache针对SPA应用的相关配置
tags: [apache]
comments: true
---





### 前端路由模式



- `hash` 模式——使用 URL 的 hash 来模拟一个完整的 URL，于是当 URL 改变时，页面不会重新加载。支持所有浏览器，包括不支持 HTML5 History Api 的浏览器。


- `history` 模式——利用 `history.pushState` API 来完成 URL 跳转而无须重新加载页面。依赖 HTML5 History API 和服务器配置。

为了美观，开发者通常选择 `history` 模式。





### 页面 404



若使用 Vue + Vue Router 开发，`vue-router` 默认 `hash` 模式，将 `VueRouter` 的实例绑定一个 `mode: 'history'` 属性则可切换到 `history` 模式。

但当你将生产代码部署到未正确配置的 apache 服务器上时，打开将发现除了首页，其他子页面全是 404，这是因为 HTTP 服务器默认情况下访问的是对应目录下的 `index.html`，如需正常访问子页面，需要将路由映射到 `index.html`，意思是如果 URL 匹配不到任何静态资源，则应该返回同一个 `index.html` 页面，这个页面就是你 app 依赖的页面。





### 服务器( apache )配置



#### **基本配置：**



**1、**打开 `httpd.conf`，此处以 mac 为例

```bash
vi /etc/apache2/httpd.conf
```



**2、**开启 apache 重定向模块，将  `#LoadModule rewrite_module libexec/apache2/mod_rewrite.so`

改为 `LoadModule rewrite_module libexec/apache2/mod_rewrite.so`



**3、**将

```
<Directory />
     AllowOverride None
     Require all denied
</Directory>
```

改为

```
<Directory />
	AllowOverride All
	Require all denied
</Directory>
```



#### **应用部署在服务器根目录：**



**4、**在文档尾部加上

```
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

或



**4、**若使用 .htaccess 文件，在服务器您的 app 的根目录新建 `.htaccess` 文件，内容为

```
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```



**5、**将 `httpd.conf` 中的

```
<Directory "/Library/WebServer/Documents">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options FollowSymLinks Multiviews
    MultiviewsMatch Any

    # 
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    # AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride None

    #
    # Controls who can get stuff from this server.
    #
    Require all granted
</Directory>


```

改为

```
<Directory "/Library/WebServer/Documents">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options FollowSymLinks Multiviews
    MultiviewsMatch Any

    # 
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    # AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride All

    #
    # Controls who can get stuff from this server.
    #
    Require all granted
</Directory>
```



#### **应用部署在服务器子目录：**



以上是 app 部署到服务器根目录下的情况，如果将 app 部署到服务器子目录下呢？

步骤4的

```
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

`RewriteRule . /index.html [L]` 须改成 `RewriteRule . /directoryname/index.html [L]`，即

```
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /directoryname/index.html [L]
</IfModule>
```



此时，所有页面均不会报 404 了，但是页面没有内容，控制台也没有报错，审查元素可以看到路由的内容没有被渲染出来，原因是 `VueRouter` 的 `base` 属性默认为 `'/'` ，如果您的 app 部署在服务器的 directoryname 的子目录下，须将 `VueRouter` 的 `base` 属性改为 `'/directoryname/'` ，再重新部署就没问题了。