title: 不会写shell的程序员照样是好前端——用Node.JS实现git hooks
date: 2016-04-16 16:30:49
tags:
- git-hook
- node
- lint
categories:
- Node.js
thumbnail: /gallery/git-hooks-node/thumbnail.png
banner: /gallery/git-hooks-node/banner.png
---
git hooks想必很多攻城狮都不陌生，官方对于hooks有详细的[文档](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)，也有网友的文章[Git Hooks (1)：介绍](https://segmentfault.com/a/1190000000356485),[GIt Hooks (2)：脚本分类](https://segmentfault.com/a/1190000000356487)，说的非常详细了，这里就不多做介绍，这里主要介绍一下如何写一个hook。
<!-- more -->

## 一个基本的git hook长什么样？

对git-hooks有一个入门认识的朋友都知道，hooks存放在git仓库的`.git/hooks`目录下，其中包括很多hooks，这些是在git 仓库创建的时候自动生成的，后缀名统一都是`.sample`，表示这些hooks都是默认不启用的，当把后缀名去掉之后，就变成了可以使用的hook。

### 举个栗子

![][1]

pre-commit这个hook是在`git commit`的时候触发的hook，这个hook里面写了什么呢？代码我就不贴了，没啥劲，主要的几点就是：

1. 这是一个shell脚本
2. 这个脚本运行了一些东西然后退出了
3. 退出的时候退出的错误码不是确定的

这就是一个hook的最基本的组成：在命令行执行git操作的时候，自动执行hooks目录下相应的可执行脚本，然后根据脚本的退出状态决定此次操作是否成功。当退出的错误码不为0的时候，表示失败，操作终止，否则操作继续。

### 模拟场景

如果现在有这样一个场景，在你的git仓库里，要求不允许提交`dist`目录，并且通过`mocha`的测试，否则不允许提交，用git hook 怎么做呢？

首先，这是在提交的时候的一个限制，所以应该考虑使用`pre-commit`这个hook，代码就不写了（不会写shell... Orz），整个过程如下：

1. 检查是否有`dist`目录，如果没有的话下一步，否则退出，错误码置为1。
2. 执行mocha命令进行测试，如果测试全部通过的话，退出，错误码为0，否则错误码为1，同样退出。

这样，当上述任何一步没有通过的时候，这个hook就会被终止，git-commit就无法通过，也就达到了限制提交的目的。


## shell脚本的局限性——不会写

作为一名普通的前端，兼，一名不太合格的工程师，我对于shell脚本实在是不熟悉，连Linux命令都玩不转，别说写出666的shell脚本了，囧~ 所以要另辟巧径做这件事。

前端仔们对js应该是非常熟练的，所以如果能用js写hooks，那不就爽了？而Node.JS正好给了我们希望，感激涕零的话就不多说了，绝对感动到哭！

![][2]

Node.js写起脚本来也非常简单，比如一个最简单的脚本

```
#!/usr/bin/env node

console.log('Hello World!');
```

给脚本赋予可执行权限之后就完全可以当做shell脚本来跑了，麻麻再也不用担心我不会shell了。同样的，在hooks中我们也可以这样用。再举个栗子

![][3]

还是刚才的场景，不允许有`dist`目录，同时通过所有mocha测试，用Node就可以这样写（这次我能show出代码了）

``` javascript
#!/usr/bin/env node
var fs = require('fs'),
    spawnSync = require('child_process').spawnSync;
if(fs.existsSync('./dist')){
    console.log('Commit Abort!Please remove dist directory.');
    process.exit(1);
}
// 使用同步方法spawnSync执行mocha，测试的结果在result.status中，通过为0，不通过为1
var result = spawnSync('./node_modules/.bin/mocha',['test']); 
if(result.status){
    console.log('Commit Abort!Test failure.');
}
process.exit(result.status);
```

这就是一个用Node.JS实现的基本的git-hook。


## Node.JS的局限性——不能动

client-side hook的一个问题就是没法在随着仓库变动，如果项目成员多的话，每个人都需要在自己本地添加一次，hooks有变动了更新也比较麻烦。

#### 解决方案 ####
我个人对这个问题有一个简单解决方案，我做了一个仓库[git-hooks-node](https://github.com/y8n/git-hooks-node)，每次写好git hooks之后通过自己写的工具进行build，生成一个类似于安装器的文件，然后提交到远程仓库，如[pre-commit.js](https://github.com/y8n/git-hooks-node/blob/master/xgfe-ma/pre-commit.js)是hook具体的内容，[pre-commit.installer.js](https://github.com/y8n/git-hooks-node/blob/master/xgfe-ma/pre-commit.installer.js)是生成的安装文件，也是一个脚本，github上的每一个文件都有相应的raw地址，如这个安装文件的地址为[raw pre-commit.installer.js](https://raw.githubusercontent.com/y8n/git-hooks-node/master/xgfe-ma/pre-commit.installer.js)，然后mac OS下的用户就可以使用`curl`获取脚本并运行，如下：

```
curl https://raw.githubusercontent.com/y8n/git-hooks-node/master/xgfe-ma/pre-commit.installer.js | node
```

安装效果如下

![安装结果][4]

这样只要写好一个hook并发布，项目成员只要知道地址就可以一键安转（想想还有点小激动呢）。这样虽然没有解决hook不会随着仓库移动的问题，但也提供了一种在项目组里通用一套hook的方案。

#### 其他解决方法 ####

[husky](https://github.com/typicode/husky)是GitHub上一个开源项目，它的做法是在`npm install`这个模块的时候自动在`.git/hooks`目录下创建很多hooks，然后再在package.json中指定每一个hook的执行脚本，如下

```
"scripts": {
    "precommit": "npm test",
    "prepush": "npm test",
    "commit-msg": "./validate-commit-msg.js",
    "...": "..."
  }
```

这样就可以把hooks随着项目变动，真正做到项目成员共用一个git hook，但问题就是必须在项目中依赖husky，不过想想这样的方法也比上面我的方法高明许多 -.-! 




  [1]: /gallery/git-hooks-node/example1.jpeg
  [2]: /gallery/git-hooks-node/cry.jpeg
  [3]: /gallery/git-hooks-node/example2.jpeg
  [4]: /gallery/git-hooks-node/install.png