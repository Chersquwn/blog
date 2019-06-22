# 前端脚手架命令行工具

## 前言

程序的本质就是偷懒（误）。~~为了偷懒~~为了减少重复劳动，提高开发效率，现在许多的框架和库都会自带CLI工具，如`vue-cli`、`create-react-app`、`egg-init`等，可以快速地创建项目工程，而不需要从头开始搭建，或者从一个已有的项目中`copy`过来，然后去删减一堆的东西。

## 设计思路

如果有个脚手架模版工程，我们直接使用最熟练的`copy`就能快速创建一个工程功能了，但是，如果我们需要为这些工程动态配置一些东西，以及脚手架模版越来越多，这时候就需要有个工具来进行管理了（其实就是通过工具来进行`copy`）。

通过对`vue-cli`和`create-react-app`研究发现，它们在思路上基本都是相同：

1. 脚手架模版
2. 通过`CLI`去拉取模版
3. 进行依赖安装

当然，不同的命令行工具会有不同的实现，`vue-cli`会通过交互式命令来获取工程配置，通过`git clone`来拉取模版进行初始化，而`create-react-app`将模版拉取、初始化以及工具命令放到了`react-scripts`里面。

下面将会以自己写的 [easyapp](https://github.com/Chersquwn/easyapp) 为例进行分析。

## 核心库

- [chalk](https://github.com/chalk/chalk): 控制终端输出字符串的样式。
- [commander](https://github.com/tj/commander.js/): 命令行核心库，提供了用户命令行输入和参数解析的强大功能，可以简化命令行开发。
- [cross-spawn](https://github.com/moxystudio/node-cross-spawn): 跨平台处理子进程系统命令。
- [download-git-repo](https://github.com/flipxfx/download-git-repo): 通过`git`方式下载 repository 。
- [fs-extra](https://github.com/jprichardson/node-fs-extra): 增强`Node.js`的fs模块。
- [inquirer](https://github.com/SBoudrias/Inquirer.js): `Node.js` 命令行交互工具，提供通用的命令行用户界面集合，用于和用户进行交互。
- [npm-check-updates](https://github.com/tjunnone/npm-check-updates): 检查 packages 是否需要更新。
- [ora](https://github.com/sindresorhus/ora): 提供 loading 的样式。
- [request](https://github.com/request/request): `Node.js`的 http 请求库。
- [semver](https://github.com/npm/node-semver): `semver`版本规范，提供版本的判断。
- [validate-npm-package-name](https://github.com/npm/validate-npm-package-name): 校验是否符合 `npm package`的命名规范。

## CLI命令

```bash
Commands:
  create                  create <project-name>

Options:
  -v, --version           Show version number
```

目前只有核心命令 `create`， 接受一个参数作为工程名字。

`Node.js` 中，一个可执行的命令，是通过 `package.json` 中的 `bin` 字段来实现的。

```json
  "bin": {
    "easyapp": "bin/easyapp.js"
  },
```

在执行 `easyapp` 命令时，实际上执行的是 `bin` 目录下的 `easyapp.js` 文件。

## 解析获取命令行参数

首先`checkNodeVersion`函数对当前Node版本进行校验，然后是定义`create`的命令和参数解析，当命令行输入的命令为`create`时，执行`create`函数。

```ts
// src/index.ts
import program from 'commander'
import { version } from './package.json'
import create from './src/commands/create'
import list from './src/commands/list'
import { checkNodeVersion } from './src/utils/check-version'

// 校验 Node 版本
checkNodeVersion()

// 定义
program
  .version(version, '-v, --version')
  .command('create [name]')
  .description('create project')
  .action(async (name: string) => {
    await create(name)
  })

program.parse(process.argv)

if (program.args.length < 1) {
  program.help()
}

```

## create 函数

`create` 函数是核心方法，该方法实现了：

- `isValidPackageName`: 项目工程 name 的合法校验
- `createAppDir`: 创建以 name 命名的目录
- `isSafeDirectory`: 校验该目录是否为合法目录
- `getProjectInfo`: 从交互式命令行界面获取工程配置参数
- `download`: 根据工程配置参数去拉取对应的模版
- `generate`: 根据工程配置参数去初始化项目工程
- `install`: 安装依赖

```ts
import chalk from 'chalk'
import ora from 'ora'
import path from 'path'
import generate from '../utils/generate'
import download from '../utils/download'
import install from '../utils/install'
import {
  isSafeDirectory,
  isValidPackageName,
  createAppDir,
  getProjectInfo
} from '../utils'

const { red, green } = chalk

async function create(name: string): Promise<void> {
  // 校验create命令接收的参数 - name 是否合法
  if (!isValidPackageName(name)) process.exit(1)

  // 判断是否存在以 name 命名的目录，如果无则创建
  createAppDir(name)

  // 判断该目录是否为合法的目录
  if (!isSafeDirectory(name)) process.exit(1)

  const root = path.resolve(name)
  // 从交互式命令行界面获取工程配置参数
  const { template, ...projectInfo } = await getProjectInfo(name)

  console.log()
  console.log()

  const spinner = ora('Downloading please wait...')

  spinner.start()

  try {
    // 根据交互式命令行选择的模版名称下载拉取模版
    await download(`${template.path}#${template.version}`, `./${name}`)
  } catch (error) {
    console.log()
    console.log(
      red(`Failed to download template ${template}: ${error.message}.`)
    )
    process.exit(1)
  }

  // 初始化和处理下载的模版
  generate(name, projectInfo)

  spinner.succeed(`${green('Template download successfully!')}`)

  spinner.start('Installing packages. This might take a couple of minutes.')

  // 安装依赖
  await install(name)

  spinner.succeed(`${green('All packages installed successfully!')}`)

  console.log()
  console.log(green(`Success! Created ${name} at ${root}`))
  console.log()
}

export default create
```

## 交互式命令行获取配置参数

通过 `inquirer` 来实现交互式的命令行，主要获取`name`、`version`、`description`、`repository`、`author`、`license`和`template`，用于之后选择下载的模版和初始化。

```ts
export async function getProjectInfo(name: string): Promise<inquirer.Answers> {
  const question = getQuestion(name)
  const answers = await inquirer.prompt(question)

  return answers
}

function getQuestion(name: string): inquirer.Questions {
  const author = getGitAuthor()
  const choices = Object.keys(TEMPLATE).map(
    (name: string): inquirer.ChoiceType => ({
      name,
      value: TEMPLATE[name]
    })
  )

  return [
    {
      type: 'input',
      name: 'name',
      message: 'Project name',
      default: name,
      filter(value: string): string {
        return value.trim()
      }
    },
    {
      type: 'input',
      name: 'version',
      message: 'Project version',
      default: '0.1.0',
      filter(value: string): string {
        return value.trim()
      }
    },
    {
      type: 'input',
      name: 'description',
      message: 'Project description',
      filter(value: string): string {
        return value.trim()
      }
    },
    {
      type: 'input',
      name: 'repository',
      message: 'Repository',
      filter(value: string): string {
        return value.trim()
      }
    },
    {
      type: 'input',
      name: 'author',
      message: 'Author',
      default: `${author.name} <${author.email}>`,
      filter(value: string): string {
        return value.trim()
      }
    },
    {
      type: 'input',
      name: 'license',
      message: 'License',
      default: 'MIT',
      filter(value: string): string {
        return value.trim()
      }
    },
    {
      type: 'list',
      name: 'template',
      message: 'Please select a template for the project',
      choices,
      default: choices[0]
    },
    {
      type: 'confirm',
      name: 'confirm',
      message: 'Is this ok?',
      default: true
    }
  ]
}
```

## 下载脚手架模版

使用`download-git-repo`进行下载。

```ts
import downloadRepo from 'download-git-repo'

export default async function download<T>(
  repository: string,
  destination: string
): Promise<T> {
  return new Promise(
    (resolve, reject): void => {
      downloadRepo(
        repository,
        destination,
        { clone: true },
        (error: Error, data: any): void => {
          if (error) {
            reject(error)
          } else {
            resolve(data)
          }
        }
      )
    }
  )
}
```

## 初始化模版

```ts
export default function generate(name: string, packageInfo: PackageInfo): void {
  const packageFile = path.resolve(name, 'package.json')
  const readmeFile = path.resolve(name, 'README.md')

  try {
    // 读取模版的 package.json
    const data = fs.readFileSync(packageFile, 'utf-8')
    // 将 package.json 解析成 json 对象
    const pkg = JSON.parse(data)

    // 将获取到的工程配置重新赋值给package.json
    pkg.name = packageInfo.name
    pkg.version = packageInfo.version
    pkg.description = packageInfo.description
    pkg.author = packageInfo.author
    pkg.license = packageInfo.license
    pkg.repository = { type: 'git', url: packageInfo.repository }
    pkg.bugs = { url: `${packageInfo.repository}/issues` }
    pkg.homepage = `${packageInfo.repository}#readme`

    if (pkg.module) pkg.module = `dist/${packageInfo.name}.mjs`
    if (pkg['umd:main']) pkg['umd:main'] = `dist/${packageInfo.name}.js`
    if (pkg.main) pkg.main = `dist/${packageInfo.name}.js`

    // 将解析过的 package.json 重新写到 package.json 文件中
    fs.writeFileSync(packageFile, JSON.stringify(pkg, null, 2), 'utf-8')
    // 同时生成 README.md 文件
    fs.writeFileSync(
      readmeFile,
      `# ${packageInfo.name}${os.EOL}${packageInfo.description}`,
      'utf-8'
    )
  } catch (error) {
    console.log(red(`Fail to generate: ${error.message}`))
    process.exit(1)
  }
}
```

## 安装依赖

```ts
export default async function install(name: string): Promise<void> {
  const command = getPackageManager()
  const root = path.resolve(name)
  const args = []

  // 根据 package.json 中的依赖包，判断是否需要进行版本更新升级
  await ncu.run({
    jsonUpgraded: true,
    upgrade: true,
    packageManager: 'npm',
    silent: true,
    packageFile: `./${name}/package.json`
  })

  // 是否使用 yarn
  if (command === 'yarn') {
    args.push('--cwd', root)
  }

  args.push('--silent')

  try {
    // 子进程中执行 yarn / npm install
    spawn.sync(command, args, { stdio: 'ignore', cwd: root })
  } catch (error) {
    console.log(`  ${cyan(command)} has failed.`)
  }
}
```

## 结语

以上基本上就是一个CLI基本的实现过程，总结下，其实就是获取配置、下载模版、初始化模版和安装依赖。后面可以自己再进行扩展，比如命令行界面的优化、添加新的命令、列举模版、缓存模版，等等。

至此，脚手架命令行工具的原理和`easyapp`的实现已经介绍完毕。

项目github地址: [https://github.com/Chersquwn/easyapp](https://github.com/Chersquwn/easyapp)

欢迎大家 star。

## 参考

- [vue-cli](https://github.com/vuejs/vue-cli)
- [create-react-app](https://github.com/facebook/create-react-app)
