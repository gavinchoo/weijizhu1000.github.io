---
title: React + Webpack多入口打包配置（一）基础配置快速打包，优化打包速度
categories:
  - React
tags:
  - React
  - Webpack
---

#### 1. 安装编译插件
编译 jsx、es6、scss 等资源
  ● 使用 bael 和 babel-loader 编译 jsx、es6
  ● 安装插件: babel-preset-es2015 用于解析 es6
  ● 安装插件：babel-preset-react 用于解析 jsx

安装插件
```
$ npm i --save-dev babel-core babel-loader babel-preset-es2015 babel-preset-stage-0 babel-preset-react

// 安装loader
$ npm i --save-dev css less css-loader less-loader style-loader

```
.babelrc配置文件
```
{ "presets": ["es2015","react","stage-0"] }
```
在工程根目录下创建webpack文件夹，所有的webpack相关配置都写在该文件夹下。
#### 2. 配置打包参数：Loader、输出路径、打包插件
新建文件夹config
#####2.1 新建module.config.js 配置打包loader
这里使用Happypack打包插件，多线程打包，可以加快打包速度。
```js
var path = require('path')
var nodeModulesPath = path.join(__dirname, '/node_modules');
module.exports = {
    noParse: [
        path.join(nodeModulesPath, '/react/dist/react.min'),
        path.join(nodeModulesPath, '/react-dom/dist/react-dom.min'),
    ],
    loaders: [
        {test: /\.(css|less)$/, loader: 'happypack/loader?id=happybabelstyles'},
        {
            test: /\.scss$/,
            loader: 'style-loader!css-loader?modules&importLoaders=2&sourceMap&localIdentName=[local]___[hash:base64:5]!autoprefixer?browsers=last 2 version!sass-loader?outputStyle=expanded&sourceMap'
        },
        {test: /\.(gif|jpg|png)$/, loader: 'url-loader?limit=8192&name=images/[name].[hash].[ext]'},
        {test: /\.js$/, exclude: /node_modules/, loader: 'happypack/loader?id=happybabeljs'},
        {test: /\.json$/, loader: 'json-loader'},
        {test: /\.(woff|woff2|svg|eot|ttf)$/, loader: 'url-loader?limit=50000&name=fonts/[name].[hash].[ext]'}
    ]
}
```
##### 2.2  新建output.config.js 配置包输出路径
这里 配置打包输出的文件放在public/build目录下， 因为在NodeJS服务配置该目录下为static文件，可以访问。
```javascript
var path = require('path')

module.exports = {
    path: path.join(__dirname, '../public/build'),
    publicPath: '/build/',
    filename: '[name].bundle.js'
}
```
#### 2.3 新建plugins.config.js 配置打包插件
```js
var os = require('os')
var webpack = require('webpack')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const CompressionPlugin = require("compression-webpack-plugin")
var HappyPack = require('happypack');
var HappyThreadPool = HappyPack.ThreadPool({size: os.cpus().length});
const bundleConfig = require("./json/bundle-config.json");
const {entryConfig} = require("./entry.config")

var htmlPlugins = []

for (var key in entryConfig.web){
    htmlPlugins.push(new HtmlWebpackPlugin({
        template: './webpack/template/index.ejs',
        filename: `${key.replace('/', '/assets/')}.ejs`,
        chunks: ['vendor', key],
        bundleName: bundleConfig.vendor.js,
    }))
}

for (var key in entryConfig.mobile){
    htmlPlugins.push(new HtmlWebpackPlugin({
        template: './webpack/template/mobile.ejs',
        filename: `${key.replace('/', '/assets/')}.ejs`,
        chunks: ['vendor', key],
        bundleName: bundleConfig.vendor.js,
    }))
}

htmlPlugins.push(new HtmlWebpackPlugin({
    template: './webpack/template/404.ejs',
    filename: 'common/view/404.ejs',
    chunks: [],
}))
htmlPlugins.push(new HtmlWebpackPlugin({
    template: './webpack/template/error.ejs',
    filename: 'common/view/error.ejs',
    chunks: [],
}))

module.exports = [
    new webpack.DllReferencePlugin({
        context: __dirname,
        manifest: require('./json/vendor-manifest.json')
    }),
    new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),
    new webpack.DefinePlugin({
        'process.env.NODE_ENV': JSON.stringify('production')
    }),
    new HappyPack({
        id: 'happybabeljs',
        loaders: ['babel-loader'],
        threadPool: HappyThreadPool,
    }),
    new HappyPack({
        id: 'happybabelstyles',
        threadPool: HappyThreadPool,
        loaders: ['style-loader', 'css-loader', 'less-loader']
    }),
    new CompressionPlugin({
        asset: "[path].gz[query]",
        algorithm: "gzip",
        test: /\.js$|\.css$|\.html$/,
        threshold: 10240,
        minRatio: 0
    }),
].concat(htmlPlugins)
```
#### 3. 配置打包脚本，预编译公共代码、资源
创建build文件夹。
#### 3.1 公共代码打包脚本webpack.dll.config.js
使用webpack.DllPlugin和webpack.DllReferencePlugin静态资源预编译

1. 在build新建一个webpack.dll.config.js ：
2. 使用assets-webpack-plugin将bundle的文件名输出，保存成json，在打包业务代码时配合html-webpack-plugin插件，将bundle添加到index.html中
```js
var os = require('os')
var path = require("path");
var webpack = require("webpack");
var CompressionPlugin = require("compression-webpack-plugin")
const UglifyJSPlugin = require('uglifyjs-webpack-plugin');
const AssetsPlugin = require('assets-webpack-plugin');
var HappyPack = require('happypack');
var HappyThreadPool = HappyPack.ThreadPool({ size: os.cpus().length });
const moduleConfig = require("../module.config")

module.exports = {
    entry: {
        vendor: ['react', 'react-dom', 'react-router-dom', 'moment'],
    },
    output: {
        path: path.join(__dirname, '../../public/build/dll'), // 打包后文件输出的位置
        publicPath: '/build/dll/',
        filename: '[name].bundle.js',
        library: '[name]_library'
        // vendor.dll.js中暴露出的全局变量名。
        // 主要是给DllPlugin中的name使用，
        // 故这里需要和webpack.DllPlugin中的`name: '[name]_library',`保持一致。
    },
    module: moduleConfig,
    plugins: [
        new UglifyJSPlugin(),
        new webpack.DllPlugin({
            path: path.join(__dirname, '../json', '[name]-manifest.json'),
            name: '[name]_library',
            context: __dirname
        }),
        new AssetsPlugin({
            filename: 'bundle-config.json',
            path: path.join(__dirname, '../json')
        }),
        new CompressionPlugin({
            asset: "[path].gz[query]",
            algorithm: "gzip",
            test: /\.js$|\.css$|\.html$/,
            threshold: 10240,
            minRatio: 0
        }),

        new HappyPack({
            id: 'happybabeljs',
            loaders: ['babel-loader'],
            threadPool: HappyThreadPool,
        }),
        new HappyPack({
            id: 'happybabelstyles',
            threadPool: HappyThreadPool,
            loaders: [ 'style-loader', 'css-loader', 'less-loader' ]
        }),
        new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),
    ]
};
```
##### 3.2 测试环境打包脚本webpack.dev.config.js
新建webpack.dev.config.js文件，在plugins字段中增加DllReferencePlugin插件，
```
var webpack = require('webpack')

var BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
const moduleConfig = require("../config/module.config")
const pluginConfig = require("../config/plugins.config")
const outputConfig = require('../config/output.config')
var { allEntry } = require("../../src/entries/index")
var allEntryConfig = allEntry()

module.exports = {
    devtool: 'eval', // 构建快：eval 调试使用， 构建慢：source-map 生产使用
    entry: allEntryConfig,
    output: outputConfig,
    module: moduleConfig,

    plugins: [
        new BundleAnalyzerPlugin(),
        new webpack.HotModuleReplacementPlugin(),
    ].concat(pluginConfig),
}
```
webpack.dev.config.js引用pluginconfig, 所有在plugins.config.js文件种添加下面代码
```
new webpack.DllReferencePlugin({
    context: __dirname,
    manifest: require('./public/build/vendor-manifest.json')
}),
```
##### 3.3 正式环境打包脚本webpack.config.js
```
var webpack = require('webpack')

const UglifyJSPlugin = require('uglifyjs-webpack-plugin');


const pluginConfig = require("../plugins.config")
const moduleConfig = require("../module.config")
const outputConfig = require('../output.config')
var { allEntry } = require("../entry.config")
const allEntryConfig = allEntry()

module.exports = {
    devtool: 'source-map', // 构建快：eval 调试使用， 构建慢：source-map 生产使用
    entry: allEntryConfig,
    output: outputConfig,
    module: moduleConfig,
    plugins: [
        new UglifyJSPlugin(),
        new webpack.HotModuleReplacementPlugin(),
    ].concat(pluginConfig),
}
```
##### 3.4 开发环境使用webpack-dev-server测试服务器
创建webpack.dev.server.config.js文件，配置webpack-dev-server 插件，该插件使开发调试代码运行在内存中， 支持热更新， 避免每次在本地打包生成很多hot-update文件。
```
var webpack = require('webpack')
const moduleConfig = require("../config/module.config")
const pluginConfig = require("../config/plugins.config")
const outputConfig = require('../config/output.config')
var { allEntry } = require("../../src/entries/index")
var allEntryConfig = allEntry()

module.exports = {
    devtool: 'eval', // 构建快：eval 调试使用， 构建慢：source-map 生产使用
    entry: allEntryConfig,
    output: outputConfig,
    module: moduleConfig,

    plugins: [
        new webpack.HotModuleReplacementPlugin(),
    ].concat(pluginConfig),
    devServer: {
        contentBase: 'dist', //默认webpack-dev-server会为根文件夹提供本地服务器，如果想为另外一个目录下的文件提供本地服务器，应该在这里设置其所在目录（本例设置到"build"目录）
        historyApiFallback: true, //在开发单页应用时非常有用，它依赖于HTML5 history API，如果设置为true，所有的跳转将指向index.html
        compress: true,   // 开启gzip压缩
        hot: true,
        host: '0.0.0.0',  // 同一局域网段下，可以通过IP (192.168.X.X:8000) 访问
        inline: true, //设置为true，当源文件改变时会自动刷新页面
        port: 8002, //设置默认监听端口，如果省略，默认为"8080"
        proxy: {    // 设置代理解决跨域问题
            '/': {
                target: 'http://localhost:8000/', // 目标服务器地址
                secure: false,
                withCredentials: true
            }
        }
    }
}
```
#### 3.5.  将执行脚本配置到package.json
```
"scripts": {
    "start": "nodemon ./bin/www",
    "build": "webpack --progress --config ./webpack/build/webpack.config.js",
    "build-dev": "webpack --progress --watch --config ./webpack/build/webpack.dev.config.js",
    "build-dev-server": "webpack-dev-server --progress --config ./webpack/build/webpack.dev.server.config.js",
    "build-dll": "webpack --progress --config ./webpack/build/webpack.dll.config.js",
  },
```
后续可以使用以下命令完成打包
```
npm run build-dll  // 打包公共代码
npm run build      // 生产环境打包
npm run build-dev  // 测试环境打包
npm run build-dev-server  // 开发环境打包
```
开发调试时设置devtool: 'eval', // eval source-map， 不压缩代码， 提高打包速度。

###未完，待续......
