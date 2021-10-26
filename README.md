# 尤雨溪把 vue3 源码切换成 pnpm ，我发现了这个神器 ni

## 前言

>大家好，我是[若川](https://lxchuan12.gitee.io)。欢迎关注我的[公众号若川视野](https://mp.weixin.qq.com/s?__biz=MzA5MjQwMzQyNw==&mid=2650756550&idx=1&sn=9acc5e30325963e455f53ec2f64c1fdd&chksm=8866564abf11df5c41307dba3eb84e8e14de900e1b3500aaebe802aff05b0ba2c24e4690516b&token=917686367&lang=zh_CN#rd)，最近组织了[**源码共读活动**](https://mp.weixin.qq.com/s?__biz=MzA5MjQwMzQyNw==&mid=2650756550&idx=1&sn=9acc5e30325963e455f53ec2f64c1fdd&chksm=8866564abf11df5c41307dba3eb84e8e14de900e1b3500aaebe802aff05b0ba2c24e4690516b&token=917686367&lang=zh_CN#rd)，感兴趣的可以加我微信 [ruochuan12](https://mp.weixin.qq.com/s?__biz=MzA5MjQwMzQyNw==&mid=2650756550&idx=1&sn=9acc5e30325963e455f53ec2f64c1fdd&chksm=8866564abf11df5c41307dba3eb84e8e14de900e1b3500aaebe802aff05b0ba2c24e4690516b&token=917686367&lang=zh_CN#rd) 参与，已进行两个多月，大家一起交流学习，共同进步。

想学源码，极力推荐之前我写的[《学习源码整体架构系列》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA5MjQwMzQyNw==&action=getalbum&album_id=1342211915371675650&scene=173&from_msgid=2650746362&from_itemidx=1&count=3&nolastread=1#wechat_redirect) 包含`jQuery`、`underscore`、`lodash`、`vuex`、`sentry`、`axios`、`redux`、`koa`、`vue-devtools`、`vuex4`、`koa-compose`、`vue-next-release`、`vue-this`、`create-vue`等10余篇源码文章。

最近组织了[源码共读活动](https://mp.weixin.qq.com/s?__biz=MzA5MjQwMzQyNw==&mid=2650756550&idx=1&sn=9acc5e30325963e455f53ec2f64c1fdd&chksm=8866564abf11df5c41307dba3eb84e8e14de900e1b3500aaebe802aff05b0ba2c24e4690516b&token=917686367&lang=zh_CN#rd)，大家一起学习源码。于是各种搜寻值得我们学习，且代码行数不多的源码。

在 [vuejs组织](https://github.com/vuejs) 下，找到了尤雨溪几年前写的“玩具 vite”
[vue-dev-server](https://github.com/vuejs/vue-dev-server)，发现100来行代码，很值得学习。于是有了这篇文章。

阅读本文，你将学到：

```sh
1. 学会 vite 简单原理
2. 学会使用 VSCode 调试源码
3. 学会如何编译 Vue 单文件组件
4. 学会如何使用 recast 生成 ast 转换文件
5. 如何加载包文件
6. 等等
```

## 使用

举几个常用例子，更多可以去[github文档](https://github.com/antfu/ni)去看。

全局安装。

```bash
npm i -g @antfu/ni
```

如果全局安装遭遇冲突，我们可以加上 --force 参数强制安装。

### `ni` - install

```bash
ni

# npm install
# yarn install
# pnpm install
```

```bash
ni axios

# npm i axios
# yarn add axios
# pnpm i axios
```

<br>

### `nr` - run

```bash
nr dev --port=3000

# npm run dev -- --port=3000
# yarn run dev --port=3000
# pnpm run dev -- --port=3000
```

```bash
nr
# 交互式选择命令去执行
# interactively select the script to run
# supports https://www.npmjs.com/package/npm-scripts-info convention
```

```bash
nr -

# 重新执行最后一次执行的命令
# rerun the last command
```

<br>

### `nx` - execute

```bash
nx jest

# npx jest
# yarn dlx jest
# pnpm dlx jest
```

## 原理

[ni#how](https://github.com/antfu/ni#how)

**ni** 假设您使用锁文件（并且您应该）

在它运行之前，它会检测你的 `yarn.lock` / `pnpm-lock.yaml` / `package-lock.json` 以了解当前的包管理器，并运行相应的命令。

## 准备工作

### 克隆
[本文仓库 ni-analysis，求个star^_^](https://github.com/lxchuan12/ni-analysis.git)

```sh
# 推荐克隆我的仓库(我的保证对应文章版本)
git clone https://github.com/lxchuan12/ni-analysis.git
cd ni-analysis/ni
# npm i -g pnpm
# 安装依赖
pnpm i

# 或者克隆官方仓库
git clone https://github.com/vuejs/ni.git
cd ni
# npm i -g pnpm
# 安装依赖
pnpm i
```

### package.json

```js
{
    "name": "@antfu/ni",
    "version": "0.10.0",
    "description": "Use the right package manager",
    // 暴露了六个命令
    "bin": {
        "ni": "bin/ni.js",
        "nci": "bin/nci.js",
        "nr": "bin/nr.js",
        "nu": "bin/nu.js",
        "nx": "bin/nx.js",
        "nrm": "bin/nrm.js"
    },
    "scripts": {
        // 省略了其他的命令 用 esno 执行 ts 文件
        "dev": "esno src/ni.ts"
    },
}
```

### 源码主入口

```ts
// ni/src/ni.ts
import { parseNi } from './commands'
import { runCli } from './runner'

runCli(parseNi)
```

```ts
import execa from 'execa'

const DEBUG_SIGN = '?';
export async function runCli(fn: Runner, options: DetectOptions = {}) {
  const args = process.argv.slice(2).filter(Boolean)
  try {
    await run(fn, args, options)
  }
  catch (error) {
    process.exit(1)
  }
}
export async function run(fn: Runner, args: string[], options: DetectOptions = {}) {
    const debug = args.includes(DEBUG_SIGN);
    if (debug) {
        // eslint-disable-next-line no-console
        console.log(command)
        return
    }

    await execa.command(command, { stdio: 'inherit', encoding: 'utf-8', cwd })
}
```

## 调试全流程

```
```

## 总结