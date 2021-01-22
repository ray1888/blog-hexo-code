---
title: gomodule
date: 2020-01-22 11:33:08
tags:
---
# 在线一对一迁移GoModule
 
## 为什么需要迁移到GoModule
   主要是因为三个原因
   -  从 Golang 1.13 以后， GoModule 已经从各种可用的方法由官方建议统一成 GoModule（并且事实上已经在比较成熟的 Golang 的开源项目中都已经开始进行采用）,也是为了便于升级 Golang Runtime 的版本来获取一些底层方面的性能提升
   -  可以使用 GoProxy 的中国实现解决实际生产上 Go 下载依赖包困难的问题(Golang 默认对于公开的开源库是会选择回源进行下载，但由于不可抗力，实际生产中下载依赖有可能触发一下404的域名，在不使用代理的情况下，无法下载依赖)
   -  目的是可以抛开 GOPATH 来更加自由的组织自己的项目
 
## GoModule 原理相关
### GoModule管理版本方式
   GoModule采用语义化版本管理方式来管理库名的版本。
   ```
   此处在比较正式的软件工程的定义中，关于版本的定义如下：
   V $Major.$Minor.$Patch (eg: v2.10.1)
   其中定义：
      - Major 为大版本，即默认期待为可能会有破坏性的变动，如Api的整体大变更
      - Minor 为小版本，为引入新功能或者特性修改
      - Patch 为补丁，一般来说是修复对应Minor版本中的部分问题
   ```
   Golang 也引用了这个定义来进行 GoModule的构建。
   即如果包发生破坏性的变化，ModuleName 需要同时加一，以一个我们使用到的excel解析库为例子, 即 Major Version 升级也应该放入  ModuleName 中。
 
   ```
   // 原模块名
   github.com/360EntSecGroup-Skylar/excelize
   // 发生变化后 
   github.com/360EntSecGroup-Skylar/excelize/v2
   ```
 
   但是上面例子为比较符合规范的类库，但实际上开发者不一定完全根据这个执行(大部分开发者会跟着规范去执行)，没有强制去进行规范化和提供规范化的工具。因此当使用或者升级第三方类库的情况下，最好还是自己去检查一下升级版本前后是否会有影响。
 
 
### GONOPROXY/GONOSUMDB/GOPRIVATE 概念解析（私用仓库管理相关）
这三个环境变量都是用在当前项目依赖了私有模块，也就是依赖了由 GOPROXY 指定的 Go module proxy 或由 GOSUMDB 指定 Go checksum database 无法访问到的模块时的场景，他们具有如下特性：
```
它们三个的值都是一个以英文逗号 “,” 分割的模块路径前缀，匹配规则同 path.Match。其中 GOPRIVATE 较为特殊，它的值将作为 GONOPROXY 和 GONOSUMDB 的默认值，所以建议的最佳姿势是只是用 GOPRIVATE。
在使用上来讲，比如 GOPRIVATE=*.corp.example.com 表示所有模块路径以 corp.example.com 的下一级域名 (如 team1.corp.example.com) 为前缀的模块版本都将不经过 Go module proxy 和 Go checksum database，需要注意的是不包括 corp.example.com 本身。
```
 
#### GOPROXY
 
这个环境变量主要是用于设置 Go 模块代理，主要如它的值是一个以英文逗号 “,” 分割的 Go module proxy 列表（稍后讲解）
```
作用：用于使 Go 在后续拉取模块版本时能够脱离传统的 VCS 方式从镜像站点快速拉取。它拥有一个默认值，但很可惜 proxy.golang.org 在中国无法访问，故而建议使用 goproxy.cn 作为替代。
设置为 “off” ：禁止 Go 在后续操作中使用任 何 Go module proxy。
刚刚在上面，我们可以发现值列表中有 “direct” ，它又有什么作用呢？
其实值列表中的 “direct” 为特殊指示符，用于指示 Go 回源到模块版本的源地址去抓取 (比如 GitHub 等)，当值列表中上一个 Go module proxy 返回 404 或 410 错误时，Go 自动尝试列表中的下一个，遇见 “direct” 时回源，遇见 EOF 时终止并抛出类似 “invalid version: unknown revision...” 的错误。
```
 
在安装go1.13之后，go会在系统默认使用goproxy.io来进行代理获取。但是由于不可抗力，我们在国内是无法使用这个goproxy的地址的，因此我们只能选用中国区的goproxy的实现。
目前主要有两个实现
-  [七牛云 GOPROXY 地址](https://goproxy.cn)
-  [阿里云 GOPROXY 地址](https://https://mirrors.aliyun.com/goproxy)
 
但是基于七牛云上面的 goproxy.cn 是目前国内较多GO开发者使用的 GOPROXY ，并且七牛在Go语言方面的布道和投入明显多于阿里的实现。目前我们使用七牛云的实现。（后续文章的配置均以goproxy.cn为GOPROXY变量的值）
 
```
// go.mop go.mod 文件示例
module gitlab.xinghuolive.com/birds-backend/phoenix
 
go 1.13
 
require (
        github.com/360EntSecGroup-Skylar/excelize v1.4.1
        gitlab.xinghuolive.com/birds-backend/migrations v0.0.0-20191202065617-b123ba4a7cdc
        gopkg.in/go-playground/validator.v8 v8.18.2
        gopkg.in/mgo.v2 v2.0.0-20190816093944-a6b53ec6cb22
        mellium.im/sasl v0.2.1 // indirect
        ...
)
```
 
对于上面的模块每个字段的代表的是
  1. module 表示的是本项目的项目名称
  2. go 1.13 代表使用的是go 1.13进行运行（这里只是用于展示，非强制性，这个是跟实际控制go mod 的go运行版本相关）
  3. require 代表这个项目所依赖的第三方类库（包括公共库和私有库）require 的单个条目的组成为项目名称 version(-commit )
 
```
//示例：
github.com/robfig/cron v1.2.0
gitlab.xinghuolive.com/birds-backend/swan v0.0.0-20191112105628-9b584ccec9ef
```
为什么会有上面的两种样式，原因是因为依赖的库的方式不同。<br />使用Go module 后依然可以用go get 来新加库（会自动把新依赖添加到go.mod文件中）。<br />但是可以支持以下方法来更新
```
go get $project@latest 根据tag来进行拉取
go get $project@master 根据branch来进行拉取
go get $project@V1.0   根据版本号来获取
go get $project@d4a36278507 根据commit号来获取
```
因此会有着两种形式同时存在。
   - 如何管理私有库
      即使上面配置好了GOPROXY,但是实际上私有库也是不能直接获取到的，所以我们需要配置GOPRIVATE变量来告诉go module去构建的时候，这个直接走本地的gitlab去git clone，而且不是去外边去go get 外面的库。  
      主要原因：
       -  GOPROXY 是无权访问到任何人的私有模块的，所以你放心，安全性没问题。
       -  GOPROXY 除了设置模块代理的地址以外，还需要增加 “direct” 特殊标识才可以成功拉取私有库。
   - 同一类库的不同版本的处理 
    在GoModule中的预设，假设两个包的大版本如 v1,v2 。它理解为这里是发生了BreakingChange，它会默认把这种情况的两个版本当时两个包来引入。但是对于同一个版本上面的如v1.2和v1.3，它默认是没有破坏性的改动的，因此它会默认筛选更新的版本来进行使用。（可以通过Replace来指定本项目用到的这个包的版本）
 
### Replace的功能介绍
Replace 是 GoModule 中 允许用户进行依赖模块替换的功能
有大概如下三种用法：
- 把已经消失的类库或者 Fork 出来的类库，使用这个来替换掉原来计算的代码
- 把不同层级引用的依赖进行归一化直接重定向到统一个目录下的一个库版本
- 把频繁改动的依赖库隔离其他人影响
 
### GoModule的依赖保留方式
   1. 在目前使用了 GoModule 后，实际上所有的项目依赖都会保存到GOPATH[0]/pkg目录下【go1.13是这样的保留方式，1.14会放到 $GOMODCACHE 的变量路径中】，它会保留多个版本。
   以penguin代码为例子，它会保留多个commit在本地来继续全局的统一保存。
      {% asset_img dependies.png 依赖位置图 %}

 
## 迁移过程
 
### 1.梳理私有项目间的依赖
   {% asset_img Untitled Diagram (2) 依赖图 %}
 
   梳理私有项目间的依赖，以最底的依赖来开始迁移至 GoModule，然后逐层上升来构建GoModule。
 
   我们的处理逻辑是从Turkey和Swan开始入手改造项目变成使用 GoModule进行管理，再到Penguin，如此类推，知道所有应用层代码都变成了使用 GoModule进行管理。
 
 
### 2.解决依赖的问题
因此可能出现下面两种问题：
1. 依赖消失的情况[由于 Golang 在 GoModule之前是没有一套官方定义的统一方法去管理第三方的依赖，只会把代码托管在第三方的平台上(eg:github)，并且在历史上没有一个类似于其他语言社区有一个中心化的库管理的工具（如Python中的Pypi, Java中的Maven]
 
- 解决方案  
  对上面的这种情况，像项目中之前依赖的
  ```
   github.com/xiao100/redis-pattern
   github.com/xiao100/compoent
  ```
  两个类库的情况下，因为类库不算复杂并且比较小，我们目前的做法是直接把Vendor中的旧代码挪出来进入项目中自己进行管理.
2. 本项目的依赖和依赖项目的引用了同一个包的不同版本的问题
- 解决方案  
  本项目的依赖版本和依赖的依赖的项目版本不一致的情况
  在上图提及 Magpie 的项目中用到 xiao100/cast 版本与 penguin 中用到的版本不一样，但是因为只是Cast的new方法是比较小的差异，新版本多返回了一个 error类 型，所以只要手动处理。（其实也可以理解为倒逼一些引用了的包比较旧的版本升级，只要测试充分，其实对系统的伤害不大，而且依赖库解决了一些潜在的 Bug)
 
 
### 3.获取私有依赖的新版本的问题
 
1. 配置好相关的环境变量后, 在所在目录上面直接执行
 
```
go get gitlab.xinghuovip.com/birds-backend/$project@($branch/$gitTag/$commitNumber)
```
执行成功后，go.mod 文件会自动修改相关的依赖。
 
### 4. Replace 在开发中的使用
因为我们在项目中采用的是 GitFlow 的功能分支开发模式进行代码管理，因此每个人切换或者合并分支的频率是比较高的，因此，如果每次都要手动去 Go get 下层依赖的改动其实会特别繁琐，因此我们在开发中，如果会修改到下层依赖的代码时候，如 turkey 的代码，我们默认会把 go.mod 中 turkey 的引用修改成相对位置的引用,这样就能减少开发切换的成本，降低使用心智负担。
```
// go.mod
module gitlab.xinghuolive.com/birds-backend/phoenix
 
go 1.13
 
require (
    github.com/360EntSecGroup-Skylar/excelize/v2 v2.1.0
    github.com/Shopify/sarama v1.27.2
    github.com/aliyun/aliyun-oss-go-sdk v2.1.4+incompatible
    github.com/bradfitz/gomemcache v0.0.0-20190913173617-a41fca850d0b // indirect
    github.com/bxcodec/faker v2.0.1+incompatible
    github.com/bxcodec/faker/v3 v3.5.0
    github.com/cep21/circuit v3.0.0+incompatible
    github.com/dgrijalva/jwt-go v3.2.0+
   ...
)
replace gitlab.xinghuolive.com/birds-backend/turkey => ../turkey
```
 
### 5. Jenkins构建脚本的修改
1. 修改了 build 函数里面的拉取的函数直接通过Go Get 去进行获取对应库的分支，以前采用的方式是使用 git clone 下来对应版本的私有代码到GOPATH来进行构建。
2. 为了方便开发时功能分支的切换，而且降低开发人员对GoMod 文件管理的心智负担，我们在构建的时候会把 go mod 中的 replace 相关的内容在构建中通过shell命令给替换掉，保证了上线代码的版本必定是指定构建依赖的分支
 
 
## 迁移GoModule后遇到的坑
- Jenkins构建上面的问题
   遇到一个比较大的坑是，实现的第一版的时候，想把pull本项目的代码与Go get 依赖放到同一个 Stage 中，build单独放到另外一个 Stage 中。但是发现在 Jenkin s中如果把 go get 放到 build 之外的 Stage 中，实际上并没有go get 依赖库新版本成功。因此最终是把go get 自己私有依赖和 build 放到了同一个 stage 就可以解决问题
- 依赖库版本管理的问题  
   有一次在开发版本国中发生比较大的代码合入之后，直接使用 Go Mod tidy 整理依赖后，go.mod 文件重新计算依赖后，直接把我们的excel解析库的版本进行了升级，然后api发生了变动，经检查后，我们直接通过下面的go get 的命令来直接锁死了该库的版本
   ```
   go get  github.com/360EntSecGroup-Skylar/excelize/v2@<=v2.1.0
   ```
 
## 本地切换 GoModule 使用方法
### Windows桌面开发环境
 
1. 安装Go1.13<br />
   1. 在Goland中点击 File-> Setting -> GO -> GOROOT
   <!-- ![image.png](https://cdn.nlark.com/yuque/0/2019/png/554199/1575364784373-2d4a8391-e531-4702-af07-8fc4dfc58daf.png#align=left&display=inline&height=438&margin=%5Bobject%20Object%5D&name=image.png&originHeight=876&originWidth=1286&size=68526&status=done&style=none&width=643) -->
   {% asset_img golang-setting-00000.png 图1 %}
   1. 点击加号 -> Download
   <!-- ![image.png](https://cdn.nlark.com/yuque/0/2019/png/554199/1575364793577-2d942479-b6ff-4108-b388-91ed23aebfc2.png#align=left&display=inline&height=368&margin=%5Bobject%20Object%5D&name=image.png&originHeight=735&originWidth=1295&size=53950&status=done&style=none&width=647.5) -->
   {% asset_img golang-setting-0000.png 图2 %}
   1. 点击对应版本和选择安装这个Go版本的路径即可
   <!-- ![image.png](https://cdn.nlark.com/yuque/0/2019/png/554199/1575364801584-5e91b22b-a586-4f37-9ed1-c57db92d83bc.png#align=left&display=inline&height=242&margin=%5Bobject%20Object%5D&name=image.png&originHeight=484&originWidth=844&size=29828&status=done&style=none&width=422) -->
   {% asset_img golang-setting-0.png 图3 %}
   1. 然后使用go1.13.4 的sdk 即可
   <!-- ![image.png](https://cdn.nlark.com/yuque/0/2019/png/554199/1575364812491-04b111dd-c6b5-41fe-9919-ba0458b2ad17.png#align=left&display=inline&height=334&margin=%5Bobject%20Object%5D&name=image.png&originHeight=668&originWidth=1200&size=52419&status=done&style=none&width=600) -->
    {% asset_img golang-setting.png 图4 %}
 
2. 配置Go相关的变量<br />
   1.  把项目路径从GOPATH中删除（与添加GOPATH是相反的操作，此处不再展示）
   2.  执行下面的命令组,配置 GOPROXY 和 GOPRIVATE 环境变量（我的操作是在git shell 中，理论上在 cmd 也可以执行相同的命令）
         ```
         go env -w GOPROXY=https://goproxy.cn,direct
         go env -w GOPRIVATE=gitlab.xinghuolive.com
 
         ```
 
3. 配置git相关的配置
   1. 配置默认使用ssl 代替https仓库。在git shell中执行下面命令
      ```
      vim ~/.gitconfig
      输入下面的内容
      [url "ssh://git@gitlab.xinghuolive.com/"]
         insteadOf = https://gitlab.xinghuolive.com/
      [url "ssh://git@github.com/"]
         insteadOf = https://github.com/
      ```
 
   2. 配置github的用户（方便拉取github源的情况免输入用户密码，如果本机有在github做公钥的话可以忽略这个）
      ```
      // 在bash中执行
      vim ~/.netrc
      输入下面内容
      machine github.com login username $github_username password $github_password
      ```
 
      此处 $github_password 可以直接输入github账号的密码或者使用github生成的token
 
4. 在  IDE 项目中打开 GoModule 选项
   1. 在 Goland 中点击 File-> Setting -> GO -> Go Module
          ![image.png](https://cdn.nlark.com/yuque/0/2019/png/554199/1575364828721-a5e60d60-bf10-4d8d-9aa3-965a1c8b4f77.png#align=left&display=inline&height=458&margin=%5Bobject%20Object%5D&name=image.png&originHeight=916&originWidth=1348&size=89362&status=done&style=none&width=674)
 
   Enable 勾选 -> Proxy选项选择direct ->  VendoringMode 选择关闭即可
 
5. 然后进行愉快的Build即可，跟以前项目的使用没有任何差异。
 
 
 
### Linux环境配置
 
1. 如果是使用Goland IDE 按上面的操作即可，上面Windows的操作在git shell中执行的，可以直接使用shell执行即可
 
 
## FAQ
### 开发中依赖变动导致build 失败情况
 
1. 直接更新 go.mod  依赖版本
在目前开发中如果依赖被改动导致build失败的情况下（如phoenix中的引用swan，swan被改动的情况下）目前的的处理方法是切到swan，然后拉取代码。  
但是切到GoModule后，对于上面的情况我们可以直接在phoenix项目中，使用
```
go get gitlab.xinghuolive.com/birds-backend/swan@develop
```
 
即可以解决上面的依赖问题。不用切换到swan去进行代码拉取。  
 
2. 使用 Replace 来进行
```
go mod edit -replace github.com/pselle/bar=/Users/pselle/Projects/bar
```
 
 
##  Reference
 
1. [GOMODULE-Wiki](https://github.com/golang/go/wiki/Modules)
2. [GIT-HTTPS](https://golang.org/doc/faq#git_https)
3. [GoModule提案](https://go.googlesource.com/proposal/+/master/design/24301-versioned-go.md)
4. [GoModule小结](https://mp.weixin.qq.com/s?__biz=MzAwNzEzNDMyNg==&mid=2247483722&idx=1&sn=4646d810dc43889ad72366b431a49609&chksm=9b038c53ac74054545baa60b5f1699dcfaebbada7846a92c8d6ebbc8086abfbd52ced2bd3823&mpshare=1&scene=1&srcid=&sharer_sharetime=1574906293192&sharer_shareid=9fbe1d93883d7c2a4204dcd9c62de80c&key=ee95545b6f7acc6e511bb0e9cb24d1742dc89a893cb5347267b56017b59057adc17c7a8177e405b3cc7e486cf1305c84565a9a4713040e05befb2c41c79d6ce2ca5dc08639bb1dd11a9204e47082d47c&ascene=1&uin=MjIyMDMyNTU4MA%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=ha1kk5e3DGz7oclEnTJnFqdqZBTlQCiYC73jFMAQnHWsgxruuE5pBu2d9rQ6iF%2FQ)
5. [七牛云GOPROXY的实现](https://goproxy.cn)
6. [阿里云GOPROXY的实现](https://https://mirrors.aliyun.com/goproxy)