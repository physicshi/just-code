之前用`npm`非常随意，甚至于`package.json`中的各种属性值也没有仔细研究过，特地整理这份文档，方便自己回看。

## 简介

包管理工具`npm`，全称为：`Node Package Manager`，官网地址：https://www.npmjs.com

当我们安装一个包时，其实是从registry上下载（node包对应的服务器地址）：

```
npm get registry     # 显示当前的镜像网址 https://registry.npmjs.org/
npm config set registry http://registry.npm.taobao.org   # 使用淘宝的镜像网址
```

> 发布一个包也是发布到registry上

## 版本号

`npm`采用了`semver`规范作为依赖版本管理方案。

`semver`版本规范是`X.Y.Z`：

+ `X`是主版本号（`major`）：一般主版本的改变是颠覆性的改动，意味着增加了与旧版本不兼容的API；

+ `Y`是次版本号（`minor`）：特点是兼容同一个大版本内的API，一般是做了向下兼容的更新；
+ `Z`是修订号（`patch`）：一般是修复bug，当然也需要保证向下兼容；

> 有的时候会碰到大版本是0的情况，这是指软件还处于开发阶段，可能每个小版本之间也不兼容。

### 两个常见的版本格式

+ `^x.y.z` ：表示`x`是不变的，可以兼容`y`与`z`更新的版本号；
+ `~x.y.z`：表示`x`与`y`是不变的，只兼容补丁更新的版本号。

## 项目配置文件

每一个项目都会有配置文件，记录着项目的名称、版本号、项目描述、**项目依赖的其他库以及版本号**。

这个配置文件在node环境下就是`package.json`。

手动创建`package.json`文件：

```
npm init # 创建时填写信息
npm init -y # 创建时使用默认信息
```

当执行了`npm init -y`后：

```
{
  "name": "learn-npm",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

接下来就说一下这些属性。

### 必填属性

> name和version是必填的两个属性；

+ name是项目的名称；
+ version是当前项目的版本号；

+ description是项目的基本描述
+ author是作者相关的信息（发布项目时用到）
+ license是开源协议

### private属性

+ private属性记录当前的项目是否私有，即是否可以被发布：当值为true，npm是不可以发布他的（防止私有项目被发布）

### main属性

+ 设置程序的入口

### script属性

`package.json`中的`scripts`字段用来定义自定义脚本命令（以键值对形式存在），他的每一个属性，对应一段脚本：

```json
{
 "scripts":{
 	"statr":"next"
 }
}
```

最典型的就是`npm start`，其他还有`npm install` 、`npm test `；

对于默认的脚本，我们只需要加`npm`前缀即可，对于第三方脚本（自己编写的），我们需要加`npm run`前缀：

```json
{
 "scripts":{
 	"statr":"next",
 	"pc-customer": "cd packages/pc-customer && npm run start",
 }
}
```

### dependencies属性

+ dependencies属性是指运行时依赖的包，指的是生产环境。

```
npm install --save [module]，会在dependencies里面添加依赖
--save可以简写成-S

```

`npm install [module]`，默认写到dependencies里面。

### devDependencies属性

+ 一般指的是开发时依赖的包

+ 有些包在生产环境是不需要的，比如webpack、babel等；

```
npm install --save-dev [module]，会在devDependencies里面添加依赖

--save-dev可以简写成-D
```

## npx

npx是npm 5.2之后自带的一个命令，一般是用来调用项目中的某个模块的指令、或者是调用本地模块、或者执行一次性命令避免全局安装。

```
npx webpack --version

npx create-react-app my-app  # 安装临时的create-react-app命令并且调用不会污染全局安装
```

**npx的原理很简单，他会去当前目录的`node_moule/.bin`目录下去查找对应的命令。**

