# 如何编译 Go 语言

> 本文写于 2022-10

Go 语言是可以自举的，意味着你可以编译一个属于你自己的 Go 语言版本！本篇文章将会从零开始，讲述如何一步步编译 Go 语言。读者可以使用编译出来的版本用于线下开发测试，但是强烈不建议在线上环境使用它。

参考的官方文档: https://go.dev/doc/install/source



# 环境准备

**本文基于** **Linux (Ubuntu 22.04)** 系统，以及 **amd64** **CPU架构**，即我们常说的 `linux/amd64` 环境。因为其他的操作系统可能由于更新出现各种各样的问题，我们这里限制了环境以便讲解。

其他的系统也无需担心，因为 Go 语言是跨平台编译的，所以即便在 `linux/amd64` 也可以编译出可以给 `darwin/arm64` （即 M1 macbook）使用的 Go !



## 环境网络要求

本文忽略了对于网络代理的配置，默认可以高速连接 github、apt 源、Google 等网站。请有需要的同学自行配置。

## 提前安装的组件

我们其实只需要安装如下两个组件

### build-essential

安装方式 `apt install -y build-essential`

### Go 语言

> 如果本身环境就有 Go 语言，并且 GOROOT 和 GOPATH 也设置正确，可以忽略此步骤。如果想重新安装 Go 语言，请按照**原来的方式**移除 Go 语言再按照此步骤安装。如果没有按照原来的方式卸载，会有不确定的错误发生。

如果你没有安装过 Go 语言，我们可以使用一个简易的命令来一键安装

```Bash
wget "https://go.dev/dl/go1.18.7.linux-amd64.tar.gz" -O bytedance-go.tar.gz && sudo tar -C /usr/local -xzf bytedance-go.tar.gz && export GOROOT=/usr/local/go && export PATH=$PATH:/usr/local/go/bin && sudo sh -c "echo 'export GOROOT=/usr/local/go' >> /etc/profile" && sudo sh -c "echo 'export PATH=\$PATH:/usr/local/go/bin' >> /etc/profile" && rm bytedance-go.tar.gz
```

这里我们安装的是 Go 1.18.7 的 `linux/amd64` 版本，建议使用较高的版本来编译，早期的版本面对复杂环境可能会有一些奇怪的问题。

> Tips: 
>
> 1. 如果需要卸载这个 Go 语言，只需要执行 `sudo rm -rf /usr/local/go` ，然后将 `/etc/profile` 中加入的相关路径注释即可。
> 2. 我们在命令中手动对 `/etc/profile` 注入了相关路径，这是因为在某些环境的 GOROOT 或者 GOPATH 非空值，可能会对我们的 Go 安装造成影响。

# 编译流程

## 克隆 Go 仓库

> 这里的 Go 仓库地址换成任何一个 Go 语言版本都是可行的

这里我们演示的是使用 Google 的 Go 仓库，我们需要在目录中执行

```Bash
git clone https://go.googlesource.com/go golang-go
```

即可将 Go 语言源码克隆到`golang-go`这个文件夹，进入到 `golang-go` 后，你可以切换到对应的分支，例如 `release-branch.go1.18` 这样的稳定版分支来获得更稳定的版本（可选）。

> 这里我们并没有去克隆 github 上的 golang/go，因为这只是一个镜像。克隆 github 的版本也没有问题，因为从代码层面两者几乎是一致的。

## 复制源码仓库

我们不希望编译的行为对我们原本的仓库有任何影响，所以我们尽可能地将克隆下来的仓库复制一份。

```Bash
$ cp -r golang-go base-go
```

后续我们将在 base-go 这个目录下开始我们的编译过程。

## 开始编译

进入到 `base-go` 这个目录后，我们即将开始我们的编译过程。整体的过程其实非常简单，但是分为两种方案:

### 不运行测试的编译

```Bash
cd src && ./make.bash
```

这个方式不会进行任何测试，而是直接尝试编译 Go 语言。在一个性能较好的电脑上，运行时间大概为 1 min。

### 运行测试的编译

```Bash
cd src && ./all.bash
```

这个方式会进行一次快速测试，然后再编译 Go 语言。在一个性能较好的电脑上，运行时间大概为 1 min ~ 10 min。

### 跨平台编译

跨平台编译指的是你需要一个在其他平台使用的 Go SDK，只需要在运行编译的时候加入对应环境变量即可。**如果在当前平台编译在当前平台使用的 Go SDK，按照上面两种方式即可。**

例如我们想要在一个 M1 的 macbook 上使用的 Go SDK，只需要这样：

```Bash
env GOOS=darwin GOARCH=arm64 cd src && ./make.bash
```

可以看到 GOOS 指的是需要的系统，而 GOARCH 指的是对应 CPU 架构，darwin 指的是 macOS 系统，而 arm64 指的是 M 系列芯片（他们都是 arm v8 架构的）。

同理可知，Intel 芯片的 macbook 则可以替换为 `env GOOS=darwin GOARCH=amd64`，amd64 是 64-bit intel 或者 amd CPU 的架构。

我们常用的开发环境一般是 `env GOOS=linux GOARCH=amd64`，即我们常说的 `linux/amd64` 架构。它们是运行在 amd/Intel CPU 上的 linux 系统环境。

## 移除不必要的文件（可选）

> 此过程中，我们位于 base-go/ 目录

如果我们不进行这一步，将会导致我们打包后的 Go SDK 非常庞大，甚至是 GB 级别。

### 移除杂物

这里主要是移除 .git 仓库文件以及中间产物。

```Bash
find . -name ".git"  | xargs rm -Rf
find . -name ".idea"  | xargs rm -Rf
find . -name ".DS_Store"  | xargs rm -Rf
rm -rf pkg/obj
rm -rf pkg/bootstrap
rm -rf pkg/linux_amd64/cmd
```

### 移除跨平台产物

如果你进行了跨平台编译，还可以移除这些不必要的中间产物。

将 {BUILD_OS} 和 {BUILD_ARCH} 替换为你编译时的跨平台编译环境变量，例如给 M1 的 macbook 使用的 Go SDK 的 {BUILD_OS} 为 `darwin`，{BUILD_ARCH} 为 `arm64`。

```Bash
rm -rf pkg/linux_amd64
rm -rf pkg/tool/linux_amd64
rm -rf bin/{BUILD_OS}_{BUILD_ARCH}
rm -rf pkg/{BUILD_OS}_{BUILD_ARCH}/cmd
```

## 移动跨平台编译的产物

> 此过程中，我们位于 base-go/ 目录

**如果不是跨平台编译，可以忽略此过程。**

将 {BUILD_OS} 和 {BUILD_ARCH} 替换为你编译时的跨平台编译环境变量，例如给 M1 的 macbook 使用的 Go SDK 的 {BUILD_OS} 为 `darwin`，{BUILD_ARCH} 为 `arm64`。

```Bash
mv bin/{BUILD_OS}_{BUILD_ARCH}/* bin/
```

## 打包 Go SDK

这里我们编译已经完成了，现在让我们回到有 `golang-go` 和 `base-go` 这两个文件夹的主目录，使用如下命令，将我们编译好的 Go SDK 打包

```Bash
mv base-go go && tar -zcf go.tar.gz go && rm -rf go
```

> 这里会将 base-go 重命名为 go，方便以后直接注入到 /usr/local 的目录中

**此时，我们会得到一个** **`go.tar.gz`** **文件，这个文件和我们从 Go 官网(****https://go.dev/dl/****)下载的以 .tar.gz 结尾的安装文件是基本一致的，你可以将其当做 Go 语言官方安装包来使用（前提是对应环境一致）！**

例如，如果你不是跨平台编译，你可以将这个 SDK 安装到本机，首先我们先卸载原本的 Go 语言。

```Bash
sudo rm -rf /usr/local/go
```

然后使用这个命令将其解压安装到对应位置

```Bash
sudo tar -C /usr/local -xzf go.tar.gz
```

运行 go version，可以看到你使用了自行编译的 Go 版本

```Bash
$ go version
go version devel go1.20-4c61e079c0 Thu Oct 20 23:11:47 2022 +0000 linux/amd64
```

这里我们使用的是 .tar.gz 这种打包形式，其实还有很多其他的形式，例如 .deb 等。他们适用在不同的环境，但是由于这些形式会依赖其他的组件，很容易造成其他的问题，所以个人更推荐使用这种源码打包的形式进行安装（官方提供的大部分安装包也是这样的形式）。

# 快速编译流程

> **此项目原本仅打算个人使用，使用方式可能发生改变，使用不当可能造成任意有害行为（例如误删文件等），建议大家参考项目的写法自制一套脚本**
>
> 快速编译流程仍然需要进行环境准备，请先准备好对应环境再进行操作。

上面的流程如果逐步操作确实比较复杂，所以我们通常可以用脚本来解决这个问题，此方式只支持 `linux/amd64` 平台。

首先我们先克隆脚本仓库 `git clone ``https://github.com/zhangyunhao116/gocompile.git`` `，接着移动到此仓库 `cd gocompile`，然后执行 `sh init.sh`，此时会自动克隆 Go 语言官方仓库到当前目录，其文件夹名称为 `golang-go`。当然你也可以克隆任意一个 Go 语言仓库将其命名为 `golang-go`。

我们可以对 `golang-go` 进行任何改动，使用脚本打包或测试不会对 `golang-go` 造成任何改动，之后通过如下命令执行不同操作。

## 编译并安装 Go 到当前环境

```
sh auto_install.sh golang-go
```

此命令会将 golang-go 安装到当前的系统中，**同时删除之前通过同种方式安装的 Go**，编译过程不包含测试。

> 如果之前没有注入过 PATH，你还需要执行这个命令来注入 PATH
>
> ```
> export GOROOT=/usr/local/go && export PATH=$PATH:/usr/local/go/bin && sudo sh -c "echo 'export GOROOT=/usr/local/go' >> /etc/profile" && sudo sh -c "echo 'export PATH=\$PATH:/usr/local/go/bin' >> /etc/profile"
> ```

## 编译并打包 Go SDK

```
sh compile.sh golang-go
```

此命令会将 golang-go 的内容编译，**但是不会运行测试**，打包的结果位于 `gocompile/results/go.tar.gz`。

> 如果需要跨平台编译，则可以直接将对应 GOOS 和 GOARCH 加入在目录变量后边即可
>
> 例如跨平台编译 darwin/amd64 的 Go SDK 可以使用如下命令
>
> ```
> sh compile.sh golang-go darwin amd64
> ```

## 编译测试 Go SDK

```
sh build.sh golang-go
```

此命令会尝试将 golang-go 编译并进行测试，但是不会打包，主要用于测试我们对于 Go 的代码修改是否正确。

> 如果需要跨平台编译，则可以直接将对应 GOOS 和 GOARCH 加入在目录变量后边即可
>
> 例如跨平台编译 darwin/amd64 的 Go SDK 可以使用如下命令
>
> ```
> sh compile.sh golang-go darwin amd64
> ```

## 安装打包好的 Go SDK

```
sh install go.tar.gz
```

此命令会将一个打包好的 Go SDK 安装到本地环境，可以将 go.tar.gz 换成任何合法的打包好的 Go SDK。

> 如果之前没有注入过 PATH，你还需要执行这个命令来注入 PATH
>
> ```
> export GOROOT=/usr/local/go && export PATH=$PATH:/usr/local/go/bin && sudo sh -c "echo 'export GOROOT=/usr/local/go' >> /etc/profile" && sudo sh -c "echo 'export PATH=\$PATH:/usr/local/go/bin' >> /etc/profile"
> ```