---
published: false
layout: post
categories: cheatsheet
author: Moevis
---
最近又想用 Typescript 写一个小型服务器，但是想到又要重新折腾一下环境，感觉不太开心，所以打算记在这里以后就可以直接抄了。

第一步也是最重要一步当然是：

```shell
$ npm init
```

接着是几个关键的库：

```shell
$ npm install --save typescript ts-node @types/node
```

然后接着去修改 `package.json` 中的内容，主要是 scripts 那部分，加上：

```json
{
	"scripts": {
    	"start": "ts-node src/index.ts"
    }
}
```

要注意是 `src/index.ts` 就是你的入口程序，我们直接用 ts-node 来运行，简单方便。

然后记得运行：

```shell
$ tsc --init
```

这样你就生成了一个 tsconfig.json 的配置，我会在里面继续改一点，比如：

```shell
{
	"compilerOptions": {
    	"target": "es6",
        "lib": ["dom", "es6"],
        "types": ["node"]
    }
}
```

以上应该是最简的可运行配置了。