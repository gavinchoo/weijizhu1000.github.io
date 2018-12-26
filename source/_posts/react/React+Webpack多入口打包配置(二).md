---
title: React + Webpack多入口打包配置（二）PC端、手机端多入口模块化打包配置
categories:
  - React
tags:
  - React
  - Webpack
---

多入口打包好处：
1. 可以同时开发PC端、手机端功能。全栈工程师必备。
2. 可以将代码分离成多个独立的bundle,  实现功能插件化，类似支付宝，首页可以显示每个功能的入口，使用时加载对应的模块。

#### 1. 新建页面模板index.ejs, mobile.ejs, 404.ejs, error.ejs
index.ejsÂ
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=yes">
</head>
<body>
<div id="root"></div>
<script type="text/javascript" src="<%= htmlWebpackPlugin.options.bundleName %>"></script>
</body>
</html>
```
mobile.ejs， 使用蚂蚁金服antd-mobile UI框架。
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no"/>
    <script src="https://as.alipayobjects.com/g/component/fastclick/1.0.6/fastclick.js"></script>
    <script>
        if ('addEventListener' in document) {
            document.addEventListener('DOMContentLoaded', function () {
                FastClick.attach(document.body);
            }, false);
        }
        if (!window.Promise) {
            document.writeln('<script src="https://as.alipayobjects.com/g/component/intl/1.0.1/??Intl.js,locale-data/jsonp/en.js,locale-data/jsonp/zh.js">' + '<' + '/script>');
        }
    </script>
</head>
<body>
<div id="root"></div>
<script type="text/javascript" src="<%= htmlWebpackPlugin.options.bundleName %>"></script>

<!-- history@3.2.1 -->
<script src="https://os.alipayobjects.com/rmsportal/VnttsLkEQmyLDBBluBQq.js"></script>
<!-- babel-polyfill@6.20.0 -->
<script src="https://os.alipayobjects.com/rmsportal/wzWaWInUcXErDyTwvySY.js"></script>
<script src="https://gw.alipayobjects.com/os/site-cdn/edba8a7f-b2f1-43a9-b054-dc11741642a4/site-cdn/2.0.3/common.js"></script>

</body>
</html>
```
#### 2. 新建entry.config.js 配置页面入口文件
以后新增页面入口只需要在这个文件中添加即可。
```
const entryConfig = {
    web: {
        'web/admin': './src/entries/web/admin',
    },
    mobile: {
        'mobile/portal': './src/entries/mobile/portal',
        'mobile/personal': './src/entries/mobile/personal',
    },
}

function allEntry() {
    var entry = {}
    for (var key in entryConfig) {
        entry = Object.assign(entry, entryConfig[key])
    }
    return entry
}

module.exports = {entryConfig, allEntry}
```
然后就可以使用以下命令完成打包了，先执行build-dll打包公共代码。
```
npm run build-dll
npm run build
npm run build-dev
npm run build-dev-server
```

##完

## 参考Demo https://github.com/weijizhu1000/personal_manage
