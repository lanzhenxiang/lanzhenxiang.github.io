---
title: 搭建私有仓库verdaccio并使用
date: 2019-08-26 17:49:59
tags:
  - verdaccio
  - vue
  - node.js
---

# 一、搭建 verdaccio 私有仓库

## 1.搭建 docker 环境——Centos7.6

卸载已经安装的旧版本的 docker 环境，以及其依赖

```
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## 2.安装 docker-ce 源

安装 yum-config-manager 的依赖

```
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

下载安装稳定的 docker 源

```
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

## 3. 安装 docker engine - community

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

启动 docker

```
$ sudo systemctl start docker

```

测试 docker 安装是否成功

```
$ sudo docker run hello-world

```

设置 docker 开机启动

```
systemctl enable docker
```

# 二、安装 verdaccio

## 1.拉取镜像

```
docker pull verdaccio/verdaccio
```

## 2.拉取配置文件 demo

```
mkdir -p ~/docker
cd /data/demo
git clone https://github.com/verdaccio/docker-examples
cd docker-examples
mv docker-local-storage-volume ~/docker/verdaccio
```

## 3.修改配置

```
cd ~/docker/verdaccio/conf && vi conf.yaml
```

这里修改了 verdaccio 的上游源地址，如果在 npm install 时在 verdaccio 无法找到的安装包，回在上游源地址下载，并缓存到 storage 目录，之后安装都会使用缓存在服务器上的文件进行下载，大大的加快了 npm install 的速度。

```
uplinks:
  npmjs:
    url: https://registry.npm.taobao.org/
```

也可以添加备用的源地址，具体设置可以参考官网[https://verdaccio.org/](https://verdaccio.org/)

```
uplinks:
  npmjs:
   url: https://registry.npmjs.org/
  server2:
    url: https://registry.npm.taobao.org/
    timeout: 100ms
  server3:
    url: http://mirror2.local.net:9000/
  baduplink:
    url: http://localhost:55666/
```

设置目录权限

```
chown -R 100:101 ~/docker/verdaccio
```

启动镜像

```
docker run --name verdaccio --restart always -itd -v ~/docker/verdaccio:/verdaccio -p 4873:4873 verdaccio/verdaccio
```

打开 http://localhost:4873/#/ 就可以看到已经启动起来了

## 添加用户

设置 registry,添加用户

```
npm set registry http://localhost:4873
npm adduser --registry http://localhost:4873

```

登录,输入账号，密码

```
npm login
```

# 三、 自定义 Vue 组件并上传到 verdaccio

安装 vue-cli，如果已经安装请忽略这一步

```
npm install -g vue-cli
```

使用 vue-cli 创建一个简单项目

```
vue init webpack-simple peony-circle
```

![peony-circle](/assets/img/peony-circle.png)

- index.js 为插件入口文件
- circle.vue 为 vue 组件

vue 组件代码如下:

```
<template>
  <div class="peony-circle" :style="circleSize">
    <svg viewBox="0 0 100 100">
      <!-- 渐变 -->
      <defs v-if="isGradient">
        <linearGradient id="peony-circle__gradient" x1="10%" y1="45%" x2="50%" y2="0%">
          <stop offset="0%" :style="{'stop-color': strokeColor[0], 'stop-opacity': 1}"></stop>
          <stop offset="100%" :style="{'stop-color': strokeColor[1], 'stop-opacity': 1}"></stop>
        </linearGradient>
      </defs>
      <!-- 默认的底层线条 -->
      <path :d="pathString" :stroke="trailColor" :stroke-width="trailWidth" :fill-opacity="0"></path>
      <!-- 默认的底层背景圆圈 -->
      <circle cx="50" cy="50" r="36" stroke-width="0" :fill="circleBgColor"></circle>
      <!-- 进度线条 -->
      <path
        :d="pathString"
        :stroke="isGradient ? 'url(#peony-circle__gradient)' : strokeColor"
        :stroke-width="strokeWidth"
        :stroke-linecap="strokeLinecap"
        fill-opacity="0"
        :style="pathStyle"
      ></path>
    </svg>
    <div class="peony-circle__content">
      <slot></slot>
    </div>
  </div>
</template>

<script>
export default {
  name: 'PeonyCircle',
  props: {
    percent: { type: Number, default: 0 }, // 进度百分比
    size: Number, // 大小
    strokeLinecap: { type: String, default: 'butt' }, // svg path stroke-linecap 属性: 定义不同类型的开放路径的终结 butt round square
    strokeWidth: { type: Number, default: 1 }, // 线条宽度
    strokeColor: { type: [Array, String], default: '#3FC7FA' }, // 线条颜色，数组时表示渐变
    circleBgColor: { type: String, default: '#f1f4f8' },
    trailWidth: { type: Number, default: 1 }, // 背景线条宽度
    trailColor: { type: String, default: '#D9D9D9' }, // 背景线条颜色
    anticlockwise: { type: Boolean, default: false } // 是否按逆时针方向展示百分比
  },
  computed: {
    circleSize () { // 大小
      if (this.size) {
        return { width: `${this.size}px`, height: `${this.size}px` }
      } else {
        return {}
      }
    },
    radius () {
      return 50 - this.strokeWidth / 2
    },
    pathString () {
      return `M 50,50 m 0,-${this.radius}
      a ${this.radius},${this.radius} 0 1 1 0,${2 * this.radius}
      a ${this.radius},${this.radius} 0 1 1 0,-${2 * this.radius}`
    },
    len () {
      return Math.PI * 2 * this.radius
    },
    pathStyle () {
      return {
        'stroke-dasharray': `${this.len}px ${this.len}px`,
        'stroke-dashoffset': `${((this.anticlockwise ? this.percent - 100 : 100 - this.percent) / 100 * this.len)}px`,
        'transition': 'stroke-dashoffset 0.6s ease 0s, stroke 0.6s ease'
      }
    },
    isGradient () { // 是否渐变
      return typeof this.strokeColor !== 'string'
    }
  }
}
</script>
<style lang="scss" scoped>
.peony-circle {
  position: relative;
  width: 100%;
  height: 100%;
}

.peony-circle__content {
  position: absolute;
  left: 0;
  top: 50%;
  width: 100%;
  text-align: center;
  transform: translateY(-50%);
}
</style>

```

## 编写组件入口文件

- index.js

```
import PeonyCircle from './circle.vue'
PeonyCircle.install = function (Vue) {
  Vue.component('PeonyCircle', PeonyCircle)
}

if (typeof window != 'undefined' && window.Vue) {
  window.Vue.use(PeonyCircle);
}

export default PeonyCircle

```

## 配置 webpack.config.js

```
var path = require('path')
var webpack = require('webpack')

module.exports = {
  # 修改入口文件
  entry: './src/assets/index.js',
  output: {
    path: path.resolve(__dirname, './dist'),
    publicPath: '/dist/',
    # 配置输出文件
    filename: 'peony-circle.js',
    library: 'peony-circle',
    libraryTarget: 'umd',
    umdNamedDefine: true
  },
  module: {
    rules: [{
        test: /\.css$/,
        use: [
          'vue-style-loader',
          'css-loader'
        ],
      },
      {
        test: /\.scss$/,
        use: [
          'vue-style-loader',
          'css-loader',
          'sass-loader'
        ],
      },
      {
        test: /\.sass$/,
        use: [
          'vue-style-loader',
          'css-loader',
          'sass-loader?indentedSyntax'
        ],
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          loaders: {
            // Since sass-loader (weirdly) has SCSS as its default parse mode, we map
            // the "scss" and "sass" values for the lang attribute to the right configs here.
            // other preprocessors should work out of the box, no loader config like this necessary.
            'scss': [
              'vue-style-loader',
              'css-loader',
              'sass-loader'
            ],
            'sass': [
              'vue-style-loader',
              'css-loader',
              'sass-loader?indentedSyntax'
            ]
          }
          // other vue-loader options go here
        }
      },
      {
        test: /\.js$/,
        loader: 'babel-loader',
        exclude: /node_modules/
      },
      {
        test: /\.(png|jpg|gif|svg)$/,
        loader: 'file-loader',
        options: {
          name: '[name].[ext]?[hash]'
        }
      }
    ]
  },
  resolve: {
    alias: {
      'vue$': 'vue/dist/vue.esm.js'
    },
    extensions: ['*', '.js', '.vue', '.json']
  },
  devServer: {
    historyApiFallback: true,
    noInfo: true,
    overlay: true
  },
  performance: {
    hints: false
  },
  devtool: '#eval-source-map'
}

if (process.env.NODE_ENV === 'production') {
  module.exports.devtool = '#source-map'
  // http://vue-loader.vuejs.org/en/workflow/production.html
  module.exports.plugins = (module.exports.plugins || []).concat([
    new webpack.DefinePlugin({
      'process.env': {
        NODE_ENV: '"production"'
      }
    }),
    new webpack.optimize.UglifyJsPlugin({
      sourceMap: true,
      compress: {
        warnings: false
      }
    }),
    new webpack.LoaderOptionsPlugin({
      minimize: true
    })
  ])
}

```

## 编译文件

```
npm run build
```

## 配置 package.json

```
{
  "name": "peony-circle",
  "description": "A Vue.js project",
  "version": "1.1.1",
  "author": "lanzhenxiang <490229045@qq.com>",
  "keywords": [
    "Vue2",
    "PeonyCircle"
  ],
  "main": "dist/peony-circle.js",
  "license": "MIT",
  "private": false,
  "repository": {
    "type": "git",
    "url": ""
  },
  "scripts": {
    "dev": "cross-env NODE_ENV=development webpack-dev-server --open --hot",
    "build": "cross-env NODE_ENV=production webpack --progress --hide-modules"
  },
  "dependencies": {
    "vue": "^2.5.11"
  },
  "browserslist": [
    "> 1%",
    "last 2 versions",
    "not ie <= 8"
  ],
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.2",
    "babel-preset-env": "^1.6.0",
    "babel-preset-stage-3": "^6.24.1",
    "cross-env": "^5.0.5",
    "css-loader": "^0.28.7",
    "file-loader": "^1.1.4",
    "node-sass": "^4.5.3",
    "sass-loader": "^6.0.6",
    "vue-loader": "^13.0.5",
    "vue-template-compiler": "^2.4.4",
    "webpack": "^3.6.0",
    "webpack-dev-server": "^2.9.1"
  }
}

```

## OK！！你可以发布上去了

```
npm login
# 输入你设置的账号密码
npm publish
```

![verdaccio](/assets/img/verdaccio.png)
