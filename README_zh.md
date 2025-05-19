# go-selfupdate

[![GoDoc](https://godoc.org/github.com/liuhengloveyou/go-selfupdate/selfupdate?status.svg)](https://godoc.org/github.com/liuhengloveyou/go-selfupdate/selfupdate)
![CI/CD](https://github.com/liuhengloveyou/go-selfupdate/actions/workflows/ci.yml/badge.svg)

使您的 Golang 应用程序能够实现自动更新。灵感来自 Chrome，基于 Heroku 的 [hk](https://github.com/heroku/hk) 项目。

## 特性

* 在 Mac、Linux、Arm 和 Windows 平台上经过测试
* 使用 [bsdiff](http://www.daemonology.net/bsdiff/) 创建二进制差异文件，实现小型增量更新
* 如果差异文件的 SHA 校验失败，会自动回退到完整二进制更新

## 快速开始

### 安装库和更新/补丁创建工具

`go install github.com/liuhengloveyou/go-selfupdate/cmd/go-selfupdate@latest`

### 为您的应用启用自动更新功能

`go get -u github.com/liuhengloveyou/go-selfupdate/...`

```go
var updater = &selfupdate.Updater{
    CurrentVersion: version,     // 当前应用版本，用于判断是否需要更新
    // 以下端点可以相同，如果所有内容都托管在同一位置
    ApiURL:         "http://updates.yourdomain.com/", // 获取更新清单的端点
    BinURL:         "http://updates.yourdomain.com/", // 获取完整二进制文件的端点
    DiffURL:        "http://updates.yourdomain.com/", // 获取二进制差异/补丁的端点
    Dir:            "update/",                        // 存储自动更新相关临时状态文件的目录（相对于应用程序）
    CmdName:        "myapp",                         // 您的应用名称（必须与托管更新的应用名称相对应）
    // 应用名称允许您在同一服务器/端点上为多个应用提供更新服务
}

// 在应用启动时检查更新
go updater.BackgroundRun()
// 您的应用继续运行...
```

### 推送更新

```bash
go-selfupdate path-to-your-app the-version
go-selfupdate myapp 1.2
```

默认情况下，这将在您的项目中创建一个名为 *public* 的文件夹。然后您可以通过 rsync 或其他方式将其传输到您的网络服务器或 S3。使用 `-o` 标志可以更改输出目录。

如果您在进行交叉编译，可以指定一个目录：

```bash
go-selfupdate /tmp/mybinares/ 1.2
```

该目录应包含以 $GOOS-$ARCH 命名的文件。例如：

```
windows-386
darwin-amd64
linux-arm
```

如果您使用 [goxc](https://github.com/laher/goxc)，可以通过指定以下配置来输出符合此命名格式的文件：

```
"OutPath": "{{.Dest}}{{.PS}}{{.Version}}{{.PS}}{{.Os}}-{{.Arch}}",
```

## 更新协议

更新从 HTTP(s) 服务器获取。可以使用 AWS S3 或静态托管。首先获取 JSON 清单文件，该文件指向所需版本（通常是最新版本）和相应的元数据。SHA256 哈希目前是唯一的元数据，但可能会在此处添加新字段，如签名。`go-selfupdate` 不了解任何版本控制方案。它不知道主版本/次版本。它只通过名称知道目标版本，并且可以基于当前版本和您希望移动到的版本应用差异。例如从 1.0 到 5.0 或从 1.0 到 1.1。您甚至不需要使用点号版本。您可以使用哈希值、日期等作为版本。

```
GET yourserver.com/appname/linux-amd64.json

200 ok
{
    "Version": "2",
    "Sha256": "..." // base64
}

然后

GET patches.yourserver.com/appname/1.1/1.2/linux-amd64

200 ok
[bsdiff data]

或

GET fullbins.yourserver.com/appname/1.0/linux-amd64.gz

200 ok
[gzipped executable data]
```

唯一必需的文件是 `<appname>/<os>-<arch>.json` 和 `<appname>/<latest>/<os>-<arch>.gz`，其他都是可选的。如果您愿意，可以跳过使用 go-selfupdate CLI 工具，手动或使用其他工具生成这两个文件。

## 配置

更新器配置选项：

```go
type Updater struct {
    CurrentVersion string    // 当前运行的版本。这里 `dev` 是一个特殊版本，会导致更新器永不更新。
    ApiURL         string    // API 请求的基础 URL（JSON 文件）。
    CmdName        string    // 命令名称会附加到 ApiURL 后面，如 http://apiurl/CmdName/。这代表一个二进制文件。
    BinURL         string    // 完整二进制下载的基础 URL。
    DiffURL        string    // 差异文件下载的基础 URL。
    Dir            string    // 存储自动更新状态的目录。
    ForceCheck     bool      // 无论 cktime 时间戳如何，都检查更新
    CheckTime      int       // 下次检查前的时间（小时）
    RandomizeTime  int       // 与 CheckTime 一起随机化的时间（小时）
    Requester      Requester // 可选参数，用于覆盖现有的 HTTP 请求处理程序
    Info           struct {
        Version string
        Sha256  []byte
    }
    OnSuccessfulUpdate func() // 可选函数，在更新成功后运行
}
```

### 更新后重启

应用程序在更新后通常需要重启以应用更新。`go-selfupdate` 为您提供了一个钩子来实现这一点，但由于每个应用程序的重启方式都不同，具体如何重启以及何时重启则由您决定。如果您有像 Docker 或 systemd 这样的服务重启应用程序，您可以简单地退出并让上游应用程序启动/重启您的应用程序。只需设置 `OnSuccessfulUpdate` 钩子：

```go
u.OnSuccessfulUpdate = func() { os.Exit(0) }
```

或者您可能有一个优雅重启的库/函数：

```go
u.OnSuccessfulUpdate = func() { gracefullyRestartMyApp() }
```

## 状态

go-selfupdate 会在 `Updater.Dir` 指定的文件夹中保存一个名为 `cktime` 的文件，其中包含 Go time.Time 格式的时间戳。这对于调试下次更新时间或允许其他应用程序操作它很有用。