---
title: Peer Dependencies
ctime: 2015-05-05 18:34
---

# Peer Dependencies

2013-02-08

[本文](https://blog.domenic.me/peer-dependencies/) 由 [Ivan Yan](http://yanxyz.net/) 翻译。译文版权["署名-非商用-相同方式共享"](http://creativecommons.org/licenses/by-nc-sa/4.0/)，意见[反馈](https://github.com/hongfanqie/peer-dependencies/issues)。

npm 是非常棒的包管理器。特别是对子依赖处理得非常好。如果我的包依赖于 `request` v2 及 `some-other-library`，但是 `some-other-library` 依赖于 `request` v1，安装后的依赖树是这样的：

```
├── request@2.12.0
└─┬ some-other-library@1.2.3
  └── request@1.9.9
```

通常这样不错：`some-other-library` 用自己的 `request` v1，不会影响到我的包的 `request` v2。

## 问题：插件

不过有一种情况这样会失败：插件。一个插件（plugin package ）跟另一个宿主（host plugin）一起用，即使它不直接使用宿主。在 Node.js 里已有有不少的例子：

- Grunt [插件](http://gruntjs.com/#plugins-all)
- Chai [插件](http://chaijs.com/plugins)
- Levelup [插件](https://npmjs.org/package/level-hooks)
- Express [中间件](http://expressjs.com/api.html#middleware)
- Winston [transports](https://github.com/flatiron/winston/blob/master/docs/transports.md)

如果你是客户端开发者，即使你不熟悉上面这些包，你可以想一下 “jQuery plugins”：向页面插入 `<script>`，它们向 `jQuery.prototype` 添加属性，方便后面使用。

大体上，插件是要与宿主一起用。但是更重要的是，它们与特定版本的宿主一起用。例如，我的插件 `chai-as-promised` 1.x 与 2.x 与 `Chai` v0.5 一起用，3.x 与 `Chai` v1.x 一起用。又比如，对于不遵守 semver 的 Grunt 插件，`grunt-contrib-stylus` v0.3.1 与 `Grunt` 0.4.0rc4 一起用，但是不能与 `Grunt` 0.4.0rc5 一起用，因为移除了相关 API。

作为包管理器，npm 在安装依赖包的时候一大块的工作是处理版本。但是它的常用模式—— `package.json` 的 `dependencies` 表，对于插件毫无疑问地失败。多数插件从不依赖于它们的宿主，例如 grunt 插件从不 `require("grunt")`，所以即使写下依赖的宿主，也不会使用下载的宿主。于是我们回到了起点，所用的插件不兼容于它的宿主。

即使是直接依赖宿主的插件——可能用到宿主的 API，在插件的 `package.json` 指定依赖，这将下载多个版本的宿主，却不是你想要的。举个例子，假如 `winston-mail` 在它的 `dependencies` 表中指定了 `"winston": "0.5.x"`，这是它测试的最新版本。作为一个应用开发者，你想要最新最棒的，所以你查找最新版本的 `winston` 和 `winston-mail`，在你的 `package.json` 这样写道：

```json
{
  "dependencies": {
    "winston": "0.6.2",
    "winston-mail": "0.2.3"
  }
}
```

但是运行 `npm install` 得到的依赖树不是预期的：

```json
├── winston@0.6.2
└─┬ winston-mail@0.2.3
  └── winston@0.5.11
```

插件使用不同的 Winston API，由此导致应用出现种种错误，我就留给你自个想像了。

## 解决办法：同伴依赖

我们需要一个办法表达插件与它们的宿主之间的依赖。好比说 “我是宿主 1.2.x 的扩展，如果你安装我，请确定安装兼容的宿主。” 我们称这种关系为同伴依赖（peer dependency）。

同伴依赖的思路提出来已经有"好些年"了（[930](https://github.com/isaacs/npm/issues/930) - [1400](https://github.com/isaacs/npm/issues/1400)）。[九个月前](https://github.com/isaacs/npm/issues/1400#issuecomment-5932027)我说用“整个周末”实现它，终于我得到一个空闲的周末，现在 npm 有同伴依赖了！

特别是它作为基础部分引入到 npm 1.2.0，在后面几个版本中又继续完善，我很高兴看到这点。今天 Isaac 将 npm 1.2.10 打包进 [Node.js 0.8.19](http://blog.nodejs.org/2013/02/06/node-v0-8-19-stable/)，所以如果你安装最新的 Node, 你能马上使用同伴依赖。

为证明我说的，看我用 npm 1.2.10 安装 `jistu` 0.11.6:

```
npm ERR! peerinvalid The package flatiron does not satisfy its siblings' peerDependencies requirements!
npm ERR! peerinvalid Peer flatiron-cli-config@0.1.3 wants flatiron@~0.1.9
npm ERR! peerinvalid Peer flatiron-cli-users@0.1.4 wants flatiron@~0.3.0
```

正如所见，`jistu` 依赖于两个 Flatiron 插件，它们同伴依赖于版本相冲突的 Flatiron。npm 在帮我们判断这个冲突，`jistu` 0.11.7 将修复这个问题。

## 使用同伴依赖

同伴依赖用法很简单。写插件时，确定插件使用哪个版本的宿主，将它添加到你的 `package.json`：

```json
{
  "name": "chai-as-promised",
  "peerDependencies": {
    "chai": "1.x"
  }
}
```

当安装 `chai-as-promised` 时，也会安装 `chai`。如果后面你打算安装另一个适用于 Chai 0.x 的 插件，将会得到一个错误。好！

一个建议：不像一般的依赖，同伴依赖需要放宽版本。你不应该将你的同伴依赖锁定到特定的补丁版本号。这将会很烦人，一个 Chai 同伴依赖于 Chai 1.4.1，另一个同伴依赖于 Chai 1.5.0，只是因为作者比较懒，不花时间去确定最低可以使用的 Chai 版本。

确定同伴依赖的最好办法是确实地遵守 [semver](http://semver.org/)。假定只有宿主的主版本号的变动才会损坏你的插件。因此，如果是 1.x 宿主，使用 "~1.0" 或 "1.x"；如果依赖于 1.5.2 引入的功能，使用 ">= 1.5.2 < 2"。

## 译注

我将 “peer dependency” 翻译为“同伴依赖”。你有好的建议的话，欢迎[提出来](https://github.com/hongfanqie/peer-dependencies/issues)。本文评论里面有人问： "Also, as a side question: how did you (or whoever) come up with the term "peerDependencies"? Hearing "peer" does not make me think "host". "，作者回答：“The idea is that your program depends on the plugin being installed next to (i.e. as a peer of) the host, since the program uses both, but needs to ensure they are compatible.”

[npm 文档](https://docs.npmjs.com/files/package.json#peerdependencies) 提到，npm 1 与 npm2 会自动安装同伴依赖，npm 3 不再自动安装，会产生一条警告，比如：

```
npm WARN peerDependencies The peer dependency mocha@>=1.x.x included from grunt-mocha-istanbul will no
npm WARN peerDependencies longer be automatically installed to fulfill the peerDependency
npm WARN peerDependencies in npm 3+. Your application will need to depend on it explicitly.
```

意思是，npm 3+ 不再自动安装同伴依赖，应用需要明确的依赖它（添加到 `package.json` 的 `dependencies` 等地方）。[官方博客](http://blog.npmjs.org/post/110924823920/npm-weekly-5) 提到，开发者需要手动处理同伴依赖冲突的问题。
