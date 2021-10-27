# 尤雨溪把 vue3 源码切换成 pnpm ，发现了神器 ni，用了它之后我想放弃使用

## 前言

>大家好，我是[若川](https://lxchuan12.gitee.io)。欢迎关注我的[公众号若川视野](https://lxchuan12.gitee.io)，最近组织了[**源码共读活动**](https://www.yuque.com/ruochuan12/notice)，感兴趣的可以加我微信 [ruochuan12](https://juejin.cn/pin/7005372623400435725) 参与，已进行两个多月，大家一起交流学习，共同进步，很多人都表示收获颇丰。

想学源码，极力推荐之前我写的[《学习源码整体架构系列》](https://juejin.cn/column/6960551178908205093) 包含`jQuery`、`underscore`、`lodash`、`vuex`、`sentry`、`axios`、`redux`、`koa`、`vue-devtools`、`vuex4`、`koa-compose`、`vue-next-release`、`vue-this`、`create-vue`等10余篇源码文章。

最近组织了[源码共读活动](https://www.yuque.com/ruochuan12/notice)，大家一起学习源码。于是各种搜寻值得我们学习，且代码行数不多的源码。

写了[]()，[]()，两篇文章。

参加源码的小伙伴按照我的文章，却拉取的最新仓库代码，发现 `yarn install` 安装不了依赖，向我反馈报错。于是我去 `github仓库` 一看，发现尤雨溪把 `vue3仓库` 从 `yarn` 换成了 [`pnpm`](https://github.com/vuejs/vue-next/pull/4766/files)。贡献文档中有一句话。

```md
We also recommend installing [ni](https://github.com/antfu/ni) to help switching between repos using different package managers. `ni` also provides the handy `nr` command which running npm scripts easier.
```

这个 `ni` 项目虽然是 `ts` 项目，没用过 `ts` 小伙伴也是很好理解的，而且主文件其实不到100行，非常适合我们学习。

```md
~~npm i in a yarn project, again? F**k!~~

ni - use the right package manager
```

阅读本文，你将学到：

```sh
1. 学会 ni 使用和其原理
2. 
```

## 使用

举几个常用例子，更多可以去 [ni github文档](https://github.com/antfu/ni) 去看。

全局安装。

```bash
npm i -g @antfu/ni
```

如果全局安装遭遇冲突，我们可以加上 `--force` 参数强制安装。

**我看源码发现：以下命令，都可以在末尾追加`\?`，表示只打印，不是真正执行**。

所以全局安装后，可以尽情测试，比如 `ni \?`，`nr dev --port=3000 \?`，`nx jest \?`，因为打印，所以可以在各种目录下执行，有助于理解测试`ni`。

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
根据 `dev` 命令，我们找到主入口文件 `src/ni.ts`。

### 源码主入口

```ts
// ni/src/ni.ts
import { parseNi } from './commands'
import { runCli } from './runner'

runCli(parseNi)
```

```ts
import execa from 'execa'

const DEBUG_SIGN = '?'

export async function runCli(fn: Runner, options: DetectOptions = {}) {
  const args = process.argv.slice(2).filter(Boolean)
  try {
    await run(fn, args, options)
  }
  catch (error) {
    process.exit(1)
  }
}
// 源码有删减
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

```js
主要做三件事：
根据锁文件猜测用哪个 npm yarn pnpm
抹平命令之间的差异
执行脚本
```

```ts
const DEBUG_SIGN = '?'
export async function run(fn: Runner, args: string[], options: DetectOptions = {}) {
  // 命令参数包含 问号? 则是调试模式，不执行脚本
  const debug = args.includes(DEBUG_SIGN)
  if (debug)
    remove(args, DEBUG_SIGN)

  let cwd = process.cwd()
  let command

  // 支持指定 文件目录 nr -C path/to/xxx 
  if (args[0] === '-C') {
    cwd = resolve(cwd, args[1])
    args.splice(0, 2)
  }

  const isGlobal = args.includes('-g')
  if (isGlobal) {
    command = await fn(getGlobalAgent(), args)
  }
  else {
    let agent = await detect({ ...options, cwd }) || getDefaultAgent()
    if (agent === 'prompt') {
      agent = (await prompts({
        name: 'agent',
        type: 'select',
        message: 'Choose the agent',
        choices: agents.map(value => ({ title: value, value })),
      })).agent
      if (!agent)
        return
    }
    command = await fn(agent as Agent, args, {
      hasLock: Boolean(agent),
      cwd,
    })
  }

  if (!command)
    return

  if (debug) {
    // eslint-disable-next-line no-console
    console.log(command)
    return
  }

  await execa.command(command, { stdio: 'inherit', encoding: 'utf-8', cwd })
}
```

### 根据锁文件猜测用哪个 npm yarn pnpm

代码相对不多，我就全部放出来了。

```ts
export async function detect({ autoInstall, cwd }: DetectOptions) {
  const result = await findUp(Object.keys(LOCKS), { cwd })
  const agent = (result ? LOCKS[path.basename(result)] : null)

  if (agent && !cmdExists(agent)) {
    if (!autoInstall) {
      console.warn(`Detected ${agent} but it doesn't seem to be installed.\n`)

      if (process.env.CI)
        process.exit(1)

      const link = terminalLink(agent, INSTALL_PAGE[agent])
      const { tryInstall } = await prompts({
        name: 'tryInstall',
        type: 'confirm',
        message: `Would you like to globally install ${link}?`,
      })
      if (!tryInstall)
        process.exit(1)
    }

    await execa.command(`npm i -g ${agent}`, { stdio: 'inherit', cwd })
  }

  return agent
}
```

### 抹平包安装工具的命令差异  parseNi

```ts

export const parseNi = <Runner>((agent, args, ctx) => {
  if (args.length === 1 && args[0] === '-v') {
    // eslint-disable-next-line no-console
    console.log(`@antfu/ni v${version}`)
    process.exit(0)
  }

  if (args.length === 0)
    return getCommand(agent, 'install')
  // 省略一些代码
})
```

```ts
// 有删减
export const AGENTS = {
  npm: {
    'install': 'npm i'
  },
  yarn: {
    'install': 'yarn install'
  },
  pnpm: {
    'install': 'pnpm i'
  },
}
```

```ts
export function getCommand(
  agent: Agent,
  command: Command,
  args: string[] = [],
) {
  if (!(agent in AGENTS))
    throw new Error(`Unsupported agent "${agent}"`)

  const c = AGENTS[agent][command]

  if (typeof c === 'function')
    return c(args)

  if (!c)
    throw new Error(`Command "${command}" is not support by agent "${agent}"`)
  return c.replace('{0}', args.join(' ')).trim()
}
```


### 最终执行脚本

```ts
await execa.command(command, { stdio: 'inherit', encoding: 'utf-8', cwd })
```

## 总结