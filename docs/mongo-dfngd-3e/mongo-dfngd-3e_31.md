# 附录 A. 安装 MongoDB

MongoDB 二进制文件适用于 Linux、macOS、Windows 和 Solaris。这意味着在大多数平台上，您可以从[MongoDB 下载中心页面](https://www.mongodb.com/download-center)下载一个存档文件，解压缩并运行该二进制文件。

MongoDB 服务器需要一个可以写入数据库文件的目录和一个可以监听连接的端口。本节涵盖了两种系统变体的完整安装过程：Windows 和其他系统（Linux/Unix/macOS）。

当我们谈论“安装 MongoDB”时，通常指的是设置 *mongod*，核心数据库服务器。*mongod* 可以作为独立服务器使用，也可以作为复制集的成员使用。大多数情况下，这将是您使用的 MongoDB 进程。

# 选择版本

MongoDB 使用一种相当简单的版本号方案：偶数点版本是稳定版，奇数点版本是开发版本。例如，以 4.2 开头的任何版本都是稳定发布，如 4.2.0、4.2.1 和 4.2.8。以 4.3 开头的任何版本都是开发版本，如 4.3.0、4.3.2 或 4.3.12。让我们以 4.2/4.3 版本发布为例，演示版本时间轴的工作方式：

1.  MongoDB 4.2.0 已经发布。这是一个重要的版本，将有一个详尽的变更日志。

1.  开发人员开始着手为 4.4（下一个主要稳定版本）制定里程碑计划后，他们发布了 4.3.0。这是新的开发分支，与 4.2.0 相似，但可能会增加一两个额外的功能，也可能有一些 bug。

1.  随着开发人员继续添加功能，他们将发布 4.3.1、4.3.2 等版本。这些版本不应该在生产环境中使用。

1.  一些次要的 bug 修复可能会被反向移植到 4.2 分支，这将导致发布 4.2.1、4.2.2 等版本。开发人员对反向移植非常谨慎；几乎不会向稳定版添加新功能。通常只会移植 bug 修复。

1.  在 4.4.0 的所有主要里程碑达成后，4.3.7（或者最新的开发版本）将被转换为 4.4.0-rc0。

1.  在对 4.4.0-rc0 进行了广泛测试后，通常会发现几个需要修复的小 bug。开发人员修复这些 bug 并发布 4.4.0-rc1。

1.  开发人员重复第 6 步，直到没有新的 bug 显现，然后 4.4.0-rc2（或者最新版本）被重命名为 4.4.0。

1.  开发人员从第 1 步重新开始，将所有版本号增加 0.2。

您可以通过浏览[**MongoDB bug tracker**](http://jira.mongodb.org)上的核心服务器路线图来查看生产版本发布的时间。

如果您正在生产环境中运行，应该使用稳定版。如果您计划在生产环境中使用开发版，请先在邮件列表或 IRC 上咨询开发人员的建议。

如果您刚开始开发项目，可能更好选择使用开发版本。到生产阶段时，可能会有一个带有您使用的功能的稳定版本（MongoDB 尝试每 12 个月发布一次稳定版本）。但是，您必须权衡这一点，因为可能会遇到服务器错误，这可能会对新用户产生影响。

# Windows 安装

要在 Windows 上安装 MongoDB，请从 [MongoDB 下载中心页面](https://oreil.ly/nZZd0) 下载 Windows *.msi*。使用上一节中的建议选择正确的 MongoDB 版本。单击链接后，将下载 *.msi*。双击 *.msi* 文件图标启动安装程序。

现在您需要创建一个目录，MongoDB 可以在其中写入数据库文件。默认情况下，MongoDB 尝试在当前驱动器上使用 *\data\db* 目录作为其数据目录（例如，如果您在 Windows 的 *C:* 上运行 *mongod*，它将使用 *C:\Program Files\MongoDB\Server\&<VERSION>\data*）。安装程序将自动为您创建此目录。如果选择使用除 *\data\db* 以外的目录，则需要在启动 MongoDB 时指定路径，稍后将进行介绍。

现在您有了数据目录，请打开命令提示符（*cmd.exe*）。导航到您解压 MongoDB 二进制文件的目录，并运行以下命令：

```
$ C:\Program Files\MongoDB\Server\&<VERSION>\bin\mongod.exe
```

如果选择的目录不是 *C:\Program Files\MongoDB\Server\&<VERSION>\data*，则必须在此处指定，使用 `--dbpath` 参数：

```
$ C:\Program Files\MongoDB\Server\&<VERSION>\bin\mongod.exe \
        --dbpath C:\Documents and Settings\Username\My Documents\db
```

有关更常见选项，请参阅第二十一章，或运行 `mongod.exe` `--help` 查看所有选项。

## 安装为服务

MongoDB 也可以作为 Windows 的服务安装。要执行此操作，只需使用完整路径运行，并转义任何空格，并使用 `--install` 选项。例如：

```
$ C:\Program Files\MongoDB\Server\4.2.0\bin\mongod.exe \
          --dbpath "\"C:\Documents and Settings\Username\My Documents\db\"" \
          --install
```

然后可以通过控制面板启动和停止它。

# POSIX（Linux 和 Mac OS X）安装

根据“选择版本”部分的建议选择 MongoDB 的版本。前往 [MongoDB 下载中心](https://oreil.ly/XEScg) 并选择适合您操作系统的正确版本。

###### 警告

如果您使用的是 macOS Catalina 10.15+ 的 Mac，应该使用 */System/Volumes/Data/db*，而不是 */data/db*。此版本进行了更改，使根文件夹为只读，并在重新启动时重置，这将导致 MongoDB 数据文件夹丢失。

您必须创建一个目录，用于存放数据库文件。默认情况下，数据库将使用 */data/db*，但您可以指定任何其他目录。如果创建默认目录，请确保具有正确的写权限。您可以通过运行以下命令创建目录并设置权限：

```
      $ mkdir -p /data/db
      $ chown -R $USER:$USER /data/db
```

`mkdir -p` 创建目录及其所有父目录（如果需要的话，即如果 */data* 目录不存在，它将创建 */data* 目录，然后创建 */data/db* 目录）。 `chown` 更改 */data/db* 的所有权，以便您的用户可以向其写入。当然，您也可以只在您的主目录中创建一个目录，并在启动数据库时指定 MongoDB 应使用该目录，以避免任何权限问题。

解压从 MongoDB 下载中心下载的 *.tar.gz* 文件：

```
      $ tar zxf mongodb-linux-x86_64-enterprise-rhel62-4.2.0.tgz
      $ cd mongodb-linux-x86_64-enterprise-rhel62-4.2.0
```

现在您可以启动数据库了：

```
      $ bin/mongod
```

或者，如果您想使用替代的数据库路径，请使用 `--dbpath` 选项指定：

```
$ bin/mongod --dbpath ~/db
```

您可以运行 `mongod.exe` `--help` 查看所有可能的选项。

## 从软件包管理器安装

也有许多软件包管理器可用于安装 MongoDB。如果您喜欢使用其中之一，Red Hat、Debian 和 Ubuntu 都有官方软件包，还有许多其他系统的非官方软件包。如果您使用非官方版本，请确保它安装的是相对较新的版本。

在 macOS 上，有适用于 Homebrew 和 MacPorts 的非官方软件包。要使用[MongoDB Homebrew tap](https://oreil.ly/9xoTe)，您首先需要安装该 tap，然后通过 Homebrew 安装所需版本的 MongoDB。以下示例突出显示如何安装最新的 MongoDB Community Edition 生产版本。您可以在 macOS 终端会话中添加自定义 tap：

```
        $ brew tap mongodb/brew
```

然后使用以下命令安装最新的可用生产版本 MongoDB Community Server（包括所有命令行工具）：

```
        $ brew install mongodb-community
```

如果您选择 MacPorts 版本，请注意：编译所有 MongoDB 先决条件 Boost 库需要几个小时的时间。开始下载并让其在夜间运行。

无论您使用哪个软件包管理器，找出它将 MongoDB 日志文件放在何处是一个好主意，在出现问题并需要找到它们之前。确保在可能出现问题之前将其正确保存是非常重要的。
