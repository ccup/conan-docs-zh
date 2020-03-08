# Conan介绍

[Conan](https://docs.conan.io/en/latest/introduction.html)是一个可以帮C/C++进行依赖管理的包管理器。它是免费、开源、跨平台的。目前支持Windows, Linux, OSX, FreeBSD, Solaris等平台。同时也支持嵌入式、移动端（IOS，Andriod）、或者直接基于裸机之上的目标程序开发。它当前支持各种构件系统，例如CMake，Visual Studio（MSBuild），Makefiles，Scons等等。

Conan是分布式的，它允许运行自己私有的包管理器托管自己私有的包和二进制文件。Conan是基于二进制管理的，它可以为包生成各种不同配置、不同体系架构或者编译器版本的二进制文件。

Conan相对比较成熟和稳定，有一个全职团队在维护它。Conan保证前向兼容，社区相对成熟，从开源到商业公司都有使用。它有一个官方的中央仓 [ConanCenter](https://conan.io/center/)。

Conan的源码遵循 MIT license，位于[github : https://github.com/conan-io/conan](https://github.com/conan-io/conan)。

## 分布式的包管理器

Conan是分布式的，遵循C/S架构。客户端可以从不同的远端server上获取或上传包。和git的`git pull`和`git push`类似。

Conan的服务端主要负责包的存储，并不构建和产生包。包产生于Conan的客户端，包括包中的二进制也是在客户端编译而成。

![conan system](./images/conan-systems.png)

上图描述了conan的主要组成：

- Conan Client：Conan的客户端。它是一个基于命令行的程序，支持包的创建和使用。Conan客户端有一个包的本地缓存，因此你可以完全离线的创建和测试和使用本地的包。

- JFrog Artifactory Community Edition (CE)：[JFrog Artifactory Community Edition (CE)](https://conan.io/downloads.html)是官方推荐的用于私有部署的Conan服务器程序。这个是JFrog Artifactory的免费社区版，包含了WebUI、LDAP协议、拓扑管理、REST API以及能够存储构建物的通用仓库。下载Docker Image：`docker pull docker.bintray.io/jfrog/artifactory-cpp-ce`，运行方式参见[文档](https://www.jfrog.com/confluence/display/RTF5X/Installing+with+Docker)。

- Conan Server：这是一个与Conan Client一起发布的小的服务端程序。它是一个Conan服务端的开源实现，只包含服务端的基本功能，没有WebUI以及其它高级功能。

- [ConanCenter](https://conan.io/center/)：这是官方的中央仓，用于管理社区贡献的各种流行开源库，例如Boost，Zlib，OpenSSL，Poco等。

## 基于二进制的包管理

Conan最强大的特性使它能为任何可能的平台和配置生成和管理预编译的二级制文件。使用预编译的二进制文件可以避免用户反复的从源码进行构建，节省大量的开发以及持续集成服务器用于构建的时间，同时也提高了交付件的可重现性和可跟踪性。

Conan中的包由一个"conanfile.py"定义。该文件定义了包的依赖、包含的源码、以及如何从源码构建出二进制文件。一个包的"conanfile.py"配置可以生成任意数量的二进制文件，每个二进制可以面向不同的平台和配置（操作系统、体系结构、编译器、以及构件类型等等）。二进制的创建和上传，在所有平台上使用相同的命令，并且都是基于一套包的源码产生的。使用Conan不用为不同的操作系统提供不同的解决方案。

![Conan二进制管理](./images/conan-binary_mgmt.png)

使用Conan从服务器安装包也很高效。只用从服务端下载所需平台和配置对应的二进制文件即可，不用下载所有的二进制。如果所有的二进制都不可用，也可以用客户端的源码重新构建包。

## 支持所有的平台，构建系统以及编译器

Conan可以工作在Windows, Linux (Ubuntu, Debian, RedHat, ArchLinux, Raspbian), OSX, FreeBSD, 以及 SunOS系统上。因为它是可移植的，其实它可以运行在所有可以运行python的平台上。它的目标是针对所有存在的平台，从裸机到桌面端、移动端、嵌入式以及交叉编译。

Conan支持当前所有的构建系统。它内建了与当前最流行的构建系统的集成，例如CMake、Visual Studio (MSBuild)、 Autotools、 Makefiles, SCons等等。Conan并不强制所有的包都是用相同的构建系统，每个包可以使用自己的构架系统，并且可以依赖于使用不同构建系统的其它包。Conan也支持与其它构建系统继承，包括一些专有的构建系统。

Conan支持管理任何编译器的任何版本，包含主流的gcc、cl.exe、clang、apple-clang、intel等，支持它们的各种配置、版本、运行时和C++标准库。Conan也支持各种客户自定义的编译器配置。

## 稳定性

从Conan1.0之后，开发团队保证Conan的前向兼容性，新的版本不破坏之前的用户配置和包构建发布。而使用老的Conan版本并不保证可以兼容新的包定义。

Conan需要Python3来运行，对Python2的支持到2020年1月1日。从Conan 1.22.0版本开始，不再保证Python2的支持。

## 社区

Conan当前被 Audi、 Continental、 Plex、 Electrolux、 Mercedes-Benz等公司以及全球数万开发者用于生产环境中。

Conan在[github](https://github.com/conan-io/conan)上拥有3.6k的星，许多客户为[ConanCenter](https://conan.io/center/)创建各种流行的开源库的包，超过1000名Conan用户在slack上的CppLang Slack#Conan频道中帮助回答问题。

