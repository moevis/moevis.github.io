---
published: true
layout: post
categories: cheatsheet
author: Moevis
---
是的，是从这个项目： https://github.com/Gaweph/p5-typescript-starter/ 里面，学到的搭建一个快速 demo 的东西。之前要是写 ts 的话，要么 angular-cli 来搞一个前端，要么像之前一篇文章 [最快最小 typescript 项目创建步骤](https://moevis.github.io/cheatsheet/2018/09/07/%E6%9C%80%E5%BF%AB%E6%9C%80%E5%B0%8F-Typescript-%E9%A1%B9%E7%9B%AE%E5%88%9B%E5%BB%BA%E6%AD%A5%E9%AA%A4.html) 里面提到，最快地创建一个 ts 的后台项目。

我在学 p5 的时候，发现这个项目在 ts 支持的同时，也支持浏览器自动刷新，而且几乎不用写什么东西，所以来记录一下。

首先肯定是：

```
npm init
tsc --init
mkdir build src
```

打开 `.tsconfig.json` 修改一些参数：

```json
{
	"module": "amd",
    "outFile": "./build/build.js",
    "rootDirs": ["src"]
}
```

接着依赖库必须的有三个：

```
npm install typescript concurrently browser-sync --save
```

其中 concurrently 用来并行运行多个任务，browser-sync 用来同步刷新浏览器。

在 package.json 中的 script 项添加：
```json
{
	"scripts": {
    	"start": "concurrently \"browser-sync start --server -w\" \" tsc --watch\""
    }
}
```

最后在项目根目录添加 index.html，内容如下：

```html
<html>
  <head>
      <script src="build/build.js"></script>
  </head>
  <body>
  </body>
</html>
```

最后运行就可以了
```
npm run start
```
