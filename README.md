## Conan Tutorial

![conan logo](./images/jfrog_conan_logo.png)

由于工作原因，常常会在各种编程语言技术栈下切换。每次切回到C/C++技术栈下，都会为C/C++语言缺乏一个好用的包管理器而不适应好一阵。

包管理器的存在可以让程序功能单元的组织满足闭包化（隐藏源码、依赖和构建细节）、契约化（显示的API导出、变更和版本管理）以及标准化（体验一致的本地客户端、中央仓、及其在工具约束下的标准开发活动等等），因此能够让软件功能的复用变得更加黑盒和简单，降低了程序的复用成本。

这些优点对于社区化开发是非常重要的。离散的社区团队之间需要有一种标准和契约，可以低成本的信任和依赖别人发布的代码和更新。关于C/C++语言需要一个好用的包管理器这件事，已经被社区呼吁很久了。

C/C++程序员大多都羡慕过NodeJS下有npm、RUST下有cargo，就连一贯采用单一代码库的google也因为社区的需要从golang的1.11版本开始引入了`go modules`机制来支持包管理。回过头来我们不禁会问，为C/C++语言做一款好用的包管理器真的有那么难吗？

答案是确实挺难！这里面涉及到多方面的原因，而且不仅仅是技术的原因。

首先是C/C++语言自身的问题。C/C++程序的构建是底层相关的，这导致当你创建一个包的时候，必须考虑目标的操作系统、体系架构、以及构建时使用的编译器类型和版本、构建类型（Debug/Release）等一系列影响依赖方能否正常使用的因素。

此外，你还需要关注一些包自身的属性：是纯头文件库、静态库还是动态库，以及包的构建参数（比如优化级别、是否开启exception和rtti的编译选项...）、还有指定裁剪性(特性宏)等配置。

另外，由于C/C++的标准库（glibc和libstdc++）存在版本兼容性问题，以及C++存在ABI兼容性问题，这会让包的版本管理超越语义化版本([SemVer](https://semver.org/))所能解决的问题范围，这导致包的创建者需要在发包的时候为包的兼容性做更多的考虑。

最后，由于C/C++语言在语法上缺乏模块化机制，会让包的符号冲突以及依赖解决变得困难。

如果上述还不够，那么在这些的基础上，再加上交叉编译的场景，绝对会让一个通用C/C++包管理器的复杂度超过其它任何语言。

上述问题，整个C/C++社区中的组织和开发者一直都在努力解决。然而不像别的语言（golang属于google，rust属于社区），C/C++是由标准委员会和各个编译器工具背后的商业组织共同推动的（主要是C++，但是C受制于不同的Linux发行版本和工具链），所以无论是从效率还是结果上都不是那么好。

所以这个社区是分裂的，只用看看有多少种编译构建系统就知道了：gcc、clang、intel、qcc、Visual Studio（MSBuild）、Makefiles、Ninja、Scons、CMake...。同样，在缺少通用包管理的情况下，大家对于代码复用的解决方式也发展出了各种模式。

首先是基于源码的复用方式。项目只要划好模块，定义好各自的模块目录以及share的头文件目录，然后就可以分工合作了。

这种方式的问题是代码都在单一代码库中，可以直接看到对方的源码。由于互相之间的依赖是隐式的，导致不容易对代码做溯源和裁剪。当然这首先是个设计问题，但是这种复用方式让工具不容易对现状作出有效的可视化和约束管理。

在这种方式下，大家很容易商量出一个公共的common头文件目录，将每个模块公开的头文件都放里面（因为成本很低，无论是手动还是构建过程中自动完成）。任何一个模块依赖别人似乎都很简单，但是最后所有模块都耦合到了一起。

而且这种方式下，代码库会膨胀的很快，所有变更最终都会拥挤到一条效率不高的持续集成流水线上。由于依赖的隐式化，为持续集成流水线做分层和优化需要花费比较大的精力。

后来围绕着Git，人们发展出了一些能够优化“基于源码复用”的工具，如`git submodule`、`git subtree`、`git repo`等。这些工具可以把代码分布到不同的git仓库和分支中，能为每个代码仓搭建自己的CI流水线。但是这些方式没有从根本上解决依赖白盒化的问题。由于在使用这些工具的时候，大家仍然优先倾向将所有源码拉到一起后再进行构建，因此每个库的独立构建、测试和发布其实是缺乏原动力的。

一些构建工具的发展，为C/C++的代码复用引入了更好的方式。例如[CMake](https://cmake.org/)从3.0版本开始引入了target的概念，以及基于target的依赖可见性和传播控制。这些都更好的支持了代码在构建上的模块化，号称“everything is a (self-contained) target”。另外，借助[ExternalProject](https://cmake.org/cmake/help/latest/module/ExternalProject.html)和[find_package](https://cmake.org/cmake/help/latest/command/find_package.html)特性，使得我们可以从指定的http或者git分支下载、构建、安装和引用代码库。由于CMake的广泛流行，目前这已经成为C/C++开源社区的事实标准。

Google的[Bazel](https://bazel.build/)也支持类似的模块化特性，某些方面它还要更强大，而且支持云构建和缓存。但是由于其它一些原因，并没有大规模流行。我的好朋友刘光聪写过系列文章对bazel做过分析和介绍，具体可以看看： [《Bazel是把双刃剑》](https://www.jianshu.com/p/ab5ef02bfa2c)。

上述构建工具提供的代码复用能力，使得C/C++从代码从白盒复用往黑盒复用上迈进了一大步：代码的发布方至少要保证自己代码库的构建闭包性。但是这种复用方式，对于间接依赖的管理仍旧是不足的。我们需要一种能力，可以通过全链条的依赖解析，进行依赖溯源、冲突判决，以及基于变更进行最小范围的重构建和发布管理。

所以，包管理器在C/C++社区很早就有了。包管理器通过让包显示化的描述自己的元信息：名称、版本、构建方式、以及所有的依赖包的版本信息，标准化了包的构建、发布和复用方式，以及自动化的对依赖和变更做管理。

遗憾的是如我们前面所说，C/C++的构建以及二进制兼容性的外部影响因素太多，所以现有被广泛使用的包管理器往往是局限于某种系统类型内的。例如Linux下主流的rpm和deb就分别面向不同的linux发行版（如Fedora和Ubuntu，当然可以扩展）。这种方式简化了C/C++的构建和二进制兼容性的管理（还包括标准库的兼容性管理），因此让包管理器的设计和使用变得容易。遗憾的是，这样的包管理器对于更广泛的社区化开发是不够的。

不过，社区一直没有停止过努力的脚步。[biicode](https://biicode.github.io/biicode/)是一款探索以源码发包的现代化C/C++包管理器，但遗憾的是这个项目由于经营原因在2015年关闭了。随后[conan](https://docs.conan.io/en/latest/introduction.html)接过了接力棒，继续探索解决通用C/C++包管理器的各种挑战，并默默的在运营壮大中。

借用Conan文档中的介绍：“Conan is a dependency and package manager for C and C++ languages. It is free and open-source, and it works in all platforms，also integrates with all build systems...”。

Conan支持交叉编译，如果获取匹配的二进制包失败会尝试从源码进行构建。除了基本的包管理能力外，conan试图内置以包管理为中心的开发最佳实践，包括内置的代码布局(layout)、构建、包测试、发布、以及与Git、IDE、CI和部署工具的集成。这些都让C/C++开发逐渐有了类似于在RUST下使用Cargo的感觉。我把这些归为是现代化包管理器应有的能力，当然conan还有一些工作要做，包括语言自身的完善（例如C++20标准引入的module机制），但目前的使用体验已经不错了。唯独可能会对使用者造成门槛的是，conan的包配置描述需要使用python。

[https://ccup.github.io/conan-docs-zh/](https://ccup.github.io/conan-docs-zh/)是我在官网学习Conan的过程中，一边学习一边翻译记录的结果。最初的目的是通过翻译让自己对看过的东西加深印象，虽然还没有完全完成，但是还是先稍加整理提供给有需要的同学吧。

在翻译记录的过程中，我根据个人的感觉对内容做了些取舍。中间有很小的部分加了点个人的理解，以使得整体更加易懂。因此，这个手册不保证更新以及和官网完全一致，有精力的同学还是推荐大家尽可能阅读[官网文档](https://docs.conan.io/)。

最后，还想讨论一个话题，那就是在大型C/C++项目中有没有必要将类似于Conan这样的包管理能力内置于开发过程中。

和社区化开发不同，大型项目可以通过集中的项目管理手段协调内部的协作和复用，再加上一些我们前面提的源码和模块化构建的技术手段，大多数时候确实可以不需要包管理器。但是我见过很多大型C/C++项目，代码动辄百万、千万，涉及很多可复用的功能单元，由于缺乏包管理器对依赖进行显示化管理，最后内部依赖混乱复杂，以至于源码的追溯性和构建的可重复性都变得困难。这些项目为了解决问题，会自行制定代码标准，开发内部工具，但是做的很多工作在我看来都是使用一款包管理器就可以解决的。

当然，这些项目宁愿自行定义标准和开发工具，而不使用包管理，是有原因的。引入包管理会改造现有的开发和协作模式，改造成本可能会比较大，另外采用包管理还可能会让跨模块的变更变得更加低效。

大多数项目在初期时候，变化方向不明确，因此系统内部结构是不稳定的。在中期结构稳定后，往往又缺乏设计和重构能力，对软件结构的划分未必能保证低耦合。如果当大多数变更都需要跨越多个包的时候，采用包管理这种隔离性强的方式，反而会增大协作沟通成本，降低效率。幸运的是，conan提供了[editable mode package](https://docs.conan.io/en/latest/developing_packages/editable_packages.html)和[workspace](https://docs.conan.io/en/latest/developing_packages/workspaces.html)的特性(RUST的cargo也提供了这个特性)，来让多包协作的修改变得容易。

许多语言都把包管理器作为一个抓手，围绕着包开发来打造贯穿整个开发态的最佳实践和辅助工具。包管理的引入会将原有的软件模块团队的交付终点，从仅仅将代码合入到代码库，延长到了需要保证构建、测试、打包和发布成功，并且满足包版本的发布契约（验收测试和契约测试），从而真正意义上的使能团队独立流水线，推动了团队的devops能力。

因此我觉得，随着C/C++包管理器的成熟，以及对软件开发过程支持的更加完善，会有越来越多的C/C++新项目逐步开始使用包管理器。而那些改造负担大，或者已经有自己的标准和工具来替代包管理能力的项目，也不妨多关注C/C++社区现代化包管理的现状和进展，从中学习和借鉴一些经验，让自己的标准和工具做的更好。

最后，再说一句，包管理只是一系列工具以及基于这些工具所构建的公共能力，它早已被证明不是银弹！包划分的好不好依然依赖于软件设计能力，这在某种程度上和微服务是一样的。你一定听说过很多关于服务拆分不好带来问题的故事吧，幸运的是包划分不好的成本比这低一些，但仍旧是有成本的。

---

本系列文档的github库地址：[https://github.com/ccup/conan-docs-zh/](https://github.com/ccup/conan-docs-zh/)。