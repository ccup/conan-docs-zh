# 开发包

## 包开发流程

在先前的例子中，我们使用`conan create`命令创建了包，每次运行这条命令，Conan执行如下步骤：

- 将源码拷贝到一个新创建的干净的build目录下；
- 从源码构建完整的库；
- 当构建成功后，将库打包；
- 构建test_package样例并运行测试；

有的时候，我们构建的库很大，每次重复上述过程有些消耗和浪费。本节描述详细的包开发流程，深入的理解可以对该过程做优化。

### `conan source`

我们可以从`conan source`命令开始。这条命令会按照你的包配置下载源码文件，将其放到临时的子目录(`source-folder`参数指定)中。

```sh
$ cd example_conan_flow
$ conan source . --source-folder=tmp/source

PROJECT: Configuring sources in C:\Users\conan\example_conan_flow\tmp\source
Cloning into 'hello'...
```

### `conan install`

`conan install`用于激活所有和依赖相关的操作。 参数`install-folder`指定安装文件的生成目录。

```sh
$ conan install . --install-folder=tmp/build [--profile XXXX]

PROJECT: Installing C:\Users\conan\example_conan_flow\conanfile.py
Requirements
Packages
...
```

上面我们通过`conan install`在“tmp/build”子目录下生成了conaninfo.txt以及conanbuildinfo.cmake文件。

### `conan build`

build方法需要源码的路径以及install目录（获得依赖以及包配置文件），然后它就可以执行构建了。

```sh
$ conan build . --source-folder=tmp/source --build-folder=tmp/build

Project: Running build()
...
Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:03.34
```

这里如果我们忽略`--install-folder=tmp/build`，它将默认取`--build-folder`参数的值。

如果你只需要修改代码重新构建包，其实只用像这个例子这样调用`conan build`就可以了。

### `conan package`

`conan package`将会调用包配置文件中的`package()`函数。它需要所有的其它目录信息（源码中的头文件、install目录下的包依赖信息、构建的结果），以便能够执行打包。

```sh
$ conan package . --source-folder=tmp/source --build-folder=tmp/build --package-folder=tmp/package

PROJECT: Generating the package
PROJECT: Package folder C:\Users\conan\example_conan_flow\tmp\package
PROJECT: Calling package()
PROJECT package(): Copied 1 '.h' files: hello.h
PROJECT package(): Copied 2 '.lib' files: greet.lib, hello.lib
PROJECT: Package 'package' created
```

### `conan export-pkg`

当你检查你的包没有问题，这时你可以使用`conan export-pkg`将其导出到你的本地conan包缓存。这个命令需要的参数和`package()`一样，它会完整的重新执行一遍打包过程，以确认包的产生是可以重复的。

```sh
$ conan export-pkg . user/channel --source-folder=tmp/source --build-folder=tmp/build --profile=myprofile

Packaging to 6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7
Hello/1.1@user/channel: Generating the package
Hello/1.1@user/channel: Package folder C:\Users\conan\.conan\data\Hello\1.1\user\channel\package\6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7
Hello/1.1@user/channel: Calling package()
Hello/1.1@user/channel package(): Copied 2 '.lib' files: greet.lib, hello.lib
Hello/1.1@user/channel package(): Copied 2 '.lib' files: greet.lib, hello.lib
Hello/1.1@user/channel: Package '6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7' created
```

- 使用`source-folder`和`build-folder`将会调用`package()`函数从这些目录里抽取需要的文件在本地缓存里面创建包。这里不需要提前执行`conan package`。但是你在开发的时候可能需要`conan package`命令做中间的调试；

- 如果在上例中使用`package-folder`参数，那么将不会调用`package()`函数，它将假定包已经通过之前的`conan package`创建好了，并直接从提供的目录中进行拷贝生成包。

### `conan test`

最后一步就是测试包。

```sh
$ conan test test_package Hello/1.1@user/channel

Hello/1.1@user/channel (test package): Installing C:\Users\conan\repos\example_conan_flow\test_package\conanfile.py
Requirements
    Hello/1.1@user/channel from local
Packages
    Hello/1.1@user/channel:6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7

Hello/1.1@user/channel: Already installed!
Hello/1.1@user/channel (test package): Generator cmake created conanbuildinfo.cmake
Hello/1.1@user/channel (test package): Generator txt created conanbuildinfo.txt
Hello/1.1@user/channel (test package): Generated conaninfo.txt
Hello/1.1@user/channel (test package): Running build()
```

最后完整的过程如下：

```sh
$ git clone git@github.com:memsharded/example_conan_flow.git
$ cd example_conan_flow
$ conan source .
$ conan install . -pr=default
$ conan build .
$ conan package .
# So far, this is local. Now put the local binaries in cache
$ conan export-pkg . Hello/1.1@user/testing -pr=default
# And test it, to check it is working in the local cache
$ conan test test_package Hello/1.1@user/testing
...
Hello/1.1@user/testing (test package): Running test()
Hello World!
```

### `conan create`

现在我们知道了包配置的详细执行步骤。`conan create`是对上面命令的合并，除了`conan test`。

```sh
$ conan create . user/channel
```

即使使用这个命令，包创建者仍然可以在本地包缓存中迭代执行（当调试的时候），可以通过`--keep-source`和`--keep-build`参数。

如果在过程中看到`source()`方法执行成功了，但是包的构建执行失败了，则再次执行可以使用`--keep-source`。

```sh
$ conan create . user/channel --keep-source

Hello/1.1@user/channel: A new conanfile.py version was exported
Hello/1.1@user/channel: Folder: C:\Users\conan\.conan\data\Hello\1.1\user\channel\export
Hello/1.1@user/channel (test package): Installing C:\Users\conan\repos\example_conan_flow\test_package\conanfile.py
Requirements
    Hello/1.1@user/channel from local
Packages
    Hello/1.1@user/channel:6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7

Hello/1.1@user/channel: WARN: Forced build from source
Hello/1.1@user/channel: Building your package in C:\Users\conan\.conan\data\Hello\1.1\user\channel\build\6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7
Hello/1.1@user/channel: Configuring sources in C:\Users\conan\.conan\data\Hello\1.1\user\channel\source
Cloning into 'hello'...
remote: Counting objects: 17, done.
remote: Total 17 (delta 0), reused 0 (delta 0), pack-reused 17
Unpacking objects: 100% (17/17), done.
Switched to a new branch 'static_shared'
Branch 'static_shared' set up to track remote branch 'static_shared' from 'origin'.
Hello/1.1@user/channel: Copying sources to build folder
Hello/1.1@user/channel: Generator cmake created conanbuildinfo.cmake
Hello/1.1@user/channel: Calling build()
```

如果你看到构建成功执行了，则可以使用`--keep-build`来跳过`build()`函数的执行。

```sh
$ conan create . user/channel --keep-build`
```

## 可编辑模式(editable mode)的包

注意：这是个实验特性，未来发布的版本可能会发生不兼容的修改。

当在有多个内在功能互相关联的大型项目中工作时，建议不要将工程组织成一个大的单体项目。推荐将项目分解成多个库，每个库专门完成一组内聚的任务，甚至分配给专门的开发团队维护。这种方式有助于隔离和重用代码，缩短编译时间，并减少include了错误头文件的可能性。

然而，在某些情况下，同时工作在多个库上，能够及时看到一个修改后对其它的影响，这点也是很有用的。我们前面介绍的工作流：`conan source`,`conan install`, `conan build`, `conan package`, 以及`conan export-pkg`，会将包构建后发布到本地缓存，为使用者做好准备。

但是，使用可编辑模式包，你能够告诉Conan从本地工作目录下查找包的头文件以及构建产物，而无需将包导出。

我们用一个例子来看看这个特性。假设开发者创建了一个`CoolApp`的应用，紧密依赖了一个`cpp/version@user/dev`的库。包`cpp/version@user/dev`已经可以工作了，开发者在本地目录下开发了代码，他们随时可以构建并执行`conan create . cool/version@user/dev`来创建这个包。同时，`CoolApp`里面有一个conanfile.txt(或者conanfile.py)，描述了它依赖于`cpp/version@user/dev`。当构建这个程序的时候，它将从Conan的本地缓存中查找依赖的`cool`包。

### 将包设置为可编辑模式

避免每次修改`cpp/version@user/dev`都需要在Conan缓存中创建包，我们可以将包设置为可修改模式：通过创建一个从Conan缓存到本地工作目录的引用连接。

```sh
$ conan editable add <path/to/local/dev/libcool> cool/version@user/dev
# you could do "cd <path/to/local/dev/libcool> && conan editable add . cool/version@user/dev"
```

执行上述命令后，本地的包或者项目每次使用`cpp/version@user/dev`，都将会重镜像到`<path/to/local/dev/libcool>`目录下，不会再从Conan缓存中去获取。

Conan的包配置文件通过`package_info()`函数定义了包的布局（layout）。如果不做修改，默认的布局如下述代码所示：

```python
def package_info(self):
    # default behavior, doesn't need to be explicitly defined in recipes
    self.cpp_info.includedirs = ["include"]
    self.cpp_info.libdirs = ["lib"]
    self.cpp_info.bindirs = ["bin"]
    self.cpp_info.resdirs = ["res"]
```

这意味着conan将会使用路径`path/to/local/dev/libcool/include`查找`cool`的头文件，通过`path/to/local/dev/libcool/lib`查找对应的库文件。

但是，实际情况是，大部分包在开发过程中经常做增量构建，构建目录的布局和包最终发布的布局不会完全一样。当然每次增量构建后可以执行`conan package`进行打包，使得布局满足包布局要求，但是这样不够优雅。Conan提供了几种为可编辑模式下的包自定义包布局的方式。

### 可编辑模式的包布局

可编辑模式的包的布局有几种不同的自定义方式：

- 通过包配置文件定义

通过包配置文件中的`package_info()`函数进行定义：

```python
from conans import ConanFile

class Pkg(ConanFile):
    settings = "build_type"
    def package_info(self):
        if not self.in_local_cache:
            d = "include_%s" % self.settings.build_type
            self.cpp_info.includedirs = [d.lower()]
```

上述代码通过构建类型来配置头文件的目录布局。如果`build_type=Debug`，则include目录是`path/to/local/dev/libcool/include_debug`，否则是`path/to/local/dev/libcool/include_release`。同样，其它的目录（libdirs，bindirs等）都可以如此自定义。

- 通过布局文件（layout files）配置

除了通过包配置文件自定义布局，还可以使用独立的布局文件。这对于有很多的库共享相同的布局的时候很有用。

布局文件是一些ini文件，但是Conan做了扩展，可以使用[Jinja2](https://palletsprojects.com/p/jinja/)模板引擎。在配置文件里面使用`settings`、`options`以及当前的`reference`对象，可以为布局配置文件增加逻辑：

```ini
[includedirs]
src/core/include
src/cmp_a/include

[libdirs]
build/{{settings.build_type}}/{{settings.arch}}

[bindirs]
{% if options.shared %}
build/{{settings.build_type}}/shared
{% else %}
build/{{settings.build_type}}/static
{% endif %}

[resdirs]
{% for item in ["cmp1", "cmp2", "cmp3"] %}
src/{{ item }}/resouces/{% if item != "cmp3" %}{{ settings.arch }}{% endif %}
{% endfor %}
```

Jinja的语法可以查看它的[官方文档](https://palletsprojects.com/p/jinja/)。

布局描述文件也可以对指定的包做配置，下面例子中设置`cool`包的include目录为`src/core/include`，而其它的则是`src/include`。

```ini
[includedirs]
src/include

[cool/version@user/dev:includedirs]
src/core/include
```

可编辑模式下的包的客赔目录有`includedirs`, `libdirs`, `bindirs`, `resdirs`, 以及`builddirs`。所有这些都在包描述文件的`cpp_info`的字典字段中，而该字典中其它值都是不能修改的，例如`cflags`、`defines`等。

默认所有的路径都是相对于conanfile.py的相对路径，也可以用绝对路径。

为包指定布局文件使用`conan editable add`命令，例如：

```sh
$ conan editable add . cool/version@user/dev --layout=win_layout
```

`win_layout`文件首先会在当前文件夹下查找，所以可将将其和源码放到一起，一起发布和被共享使用。如果在当前目录下没有找到，则会在本地缓存`~/.conan/layouts`目录下查找。可以定义布局文件共享给团队，然后通过`conan config install`进行安装。

如果`conan editable add`命令没有参数，将会默认使用`~/.conan/layouts/default`文件中描述的布局。

对于布局，Conan会按照以下优先级选择：

- 首先会执行包配置文件中的`package_info()`函数。这会定义各种标记参数（例如`cflags`）、definitions（例如`-D`的宏参数）以及布局目录: `includedirs`, `libdirs`等等；

- 如果布局文件存在，或者显示的应用了默认的`.conan/layouts/default`文件，Conan将会查找对应的匹配布局文件；

- 如果找到匹配的布局文件，则会使用布局文件中的定义替换(includedirs, libdirs, resdirs, builddirs, bindirs)；

- 布局文件的匹配，按照从特殊到一般的顺序 （布局文件可以为指定的包指定特殊的目录布局）；

- 如果没有匹配的，则仍旧使用原来定义在`package_info()`中的目录结构；

- 如果在`conan editable add`之后，手动增加了`.conan/layouts/default`文件，它将不会被使用；

### 使用可编辑模式的包

一旦对一个包建立了可编辑模式的引用，它就会对整个系统生效：所有本机（使用相同的conan缓存）的对其有依赖的工程或者包，都会重镜像到可编辑模式包的真实工作目录。

一个可编辑模式的包，对它的用户来说是透明的。使用可编辑模式的包的工作流如下：

- 通过`git/svn clone ... && cd folder`获得`cool/version@user/dev`的源码；

- 将包设置为可编辑模式：`conan editable add . cool/version@user/dev --layout=mylayout`;

- 现在可以正常编码和构建。注意你本地工作目录下的'cool'包的目录结构布局要和前面命令中的`mylayout`文件制定的一样；

- 对于消费`cool`的工程`CoolApp`，和以前一样，使用`conan install`并执行构建；

- 回到`cool/version@user/dev`的源码目录，修改代码并构建，这里不需要执行任何Conan命令以及打包；

- 回到`CoolApp`目录，重新构建它，测试`cool`的修改对其的影响；

可以看到，可以同时开发`cool`和`CoolApp`，不用反复打包发包。

注意：当一个包处于可编辑模式，大多数的工作也不会工作。可编辑模式的包不能使用`conan upload`, `conan export`或者`conan create`命令。

### 取消包的可编辑模式

取消包的可编辑模式，使用如下命令：

```sh
$ conan editable remove cool/version@user/dev
```

当执行上述命令后，所有对包的消费将会重新需要从本地包缓存中获取。

## 工作区（Workspaces）

注意：这个特性在实验中，当前的预览特性是为了收集反馈以便于改进。未来的版本中，文件格式、命令、以及工作流都有可能发生变化。

有的时候，需要同时开发处理多个包，理论上，每个包都需要是一个独立的工作单元(work unit)。但有的时候，确实存在一些修改需要同时跨越多个包，按照之前的本地开发流程，需要使用`export-pkg`将包发布到本地缓存，以便其它正在开发的包可以消费它。

Conan工作区（workspace）允许多个包存在于用户目录下，互相之间直接依赖，而不用将它们导出到本地缓存。更进一步，允许包含多个包的大工程执行增量构建。

我们用一个例子来介绍这个特性：

```sh
$ git clone https://github.com/conan-io/examples.git
$ cd features/workspace/cmake
```

可以看到这个目录下包含两个文件conanws_gcc.yml以及conanws_vs.yml。conanws_gcc.yml示例了一个基于Makefile的单配置构建环境（Single configuration build environments）；而conanws_vs.yml示例了一个基于MSBuild的多配置构建环境（MSBuild, multi-configuration build environment)。

### Conan工作区定义

工作区使用yaml文件来定义，用户可以随意为配置文件命名。文件结构如下：

```yaml
editables:
    say/0.1@user/testing:
        path: say
    hello/0.1@user/testing:
        path: hello
    chat/0.1@user/testing:
        path: chat
layout: layout_gcc
workspace_generator: cmake
root: chat/0.1@user/testing
```

`editables`段定义了多个包和其相对路径。每一个等价于下面的`conan editable add`命令，但是你不用执行这些命令，工作区稍后会自动完整配置。

```sh
$ conan editable add say say/0.1@user/testing --layout=layout_gcc
$ conan editable add hello hello/0.1@user/testing --layout=layout_gcc
$ conan editable add chat chat/0.1@user/testing --layout=layout_gcc
```

工作区中的包的可编辑状态仅对工作区内有效，不会影响到其它工程和包。其它不在工作区内的工程和包仍需要通过本地包缓存消费say、hello以及chat包。

工作区配置文件中的`layout: layout_gcc`会影响到对应的所有的包，也可以单独为某个包定义不同的目录布局，例如：

```yaml
editables:
    say/0.1@user/testing:
        path: say
        layout: custom_say_layout
```

关于可编辑模式的包以及其布局文件的介绍在上一节。

`workspace_generator`定义了将会为顶级工程产生的文件。当前配置为`cmake`，将会产生如下的`conanworkspace.cmake`文件：

```cmake
set(PACKAGE_say_SRC "<path>/examples/workspace/cmake/say/src")
set(PACKAGE_say_BUILD "<path>/examples/workspace/cmake/say/build/Debug")
set(PACKAGE_hello_SRC "<path>/examples/workspace/cmake/hello/src")
set(PACKAGE_hello_BUILD "<path>/examples/workspace/cmake/hello/build/Debug")
set(PACKAGE_chat_SRC "<path>/examples/workspace/cmake/chat/src")
set(PACKAGE_chat_BUILD "<path>/examples/workspace/cmake/chat/build/Debug")

macro(conan_workspace_subdirectories)
    add_subdirectory(${PACKAGE_say_SRC} ${PACKAGE_say_BUILD})
    add_subdirectory(${PACKAGE_hello_SRC} ${PACKAGE_hello_BUILD})
    add_subdirectory(${PACKAGE_chat_SRC} ${PACKAGE_chat_BUILD})
endmacro()
```

这个文件可以被包含在你自定义的CMakeLists.txt中。你可以看到示例工程中的CMakeLists.txt是如下使用的：

```cmake
cmake_minimum_required(VERSION 3.0)

project(WorkspaceProject)

include(${CMAKE_BINARY_DIR}/conanworkspace.cmake)
conan_workspace_subdirectories()
```

`root: chat/0.1@user/testing`定义了工作区的根工程，一般会是些可执行程序。你可以提供多个，用逗号分开。所有的根工程的依赖不能出现同一个库的不同版本。

```yaml
editables:
    say/0.1@user/testing:
        path: say
    hello/0.1@user/testing:
        path: hello
    chat/0.1@user/testing:
        path: chat

root: chat/0.1@user/testing, say/0.1@user/testing
# or
root: ["HelloA/0.1@lasote/stable", "HelloB/0.1@lasote/stable"]
# or
root:
    - HelloA/0.1@lasote/stable
    - HelloB/0.1@lasote/stable
```

### 单配置构建环境（Single configuration build environments）

对于一些构建系统（如Make），需要开发者在不同的build目录下管理不同的配置，然后在目录间切换以改变配置。conan_gcc.yml文件定义了一个Conan工作区，工作在基于MinGW/Unix Makefiles的gcc环境的CMake工程上。

如下使用并安装工作区：

```sh
$ mkdir build_release && cd build_release
$ conan workspace install ../conanws_gcc.yml --profile=my_profile
```

这里我们假定你有一个`my_profile`的profile文件，定义了linux gcc的相关构建目标配置（MAC下使用apple-clang也是可以执行这个示例的）。上述命令的输出如下：

```sh
[settings]
...
build_type=Release
compiler=gcc
compiler.libcxx=libstdc++
compiler.version=4.9
...

Requirements
    chat/0.1@user/testing from user folder - Editable
    hello/0.1@user/testing from user folder - Editable
    say/0.1@user/testing from user folder - Editable
Packages
    chat/0.1@user/testing:df2c4f4725219597d44b7eab2ea5c8680abd57f9 - Editable
    hello/0.1@user/testing:b0e473ad8697d6069797b921517d628bba8b5901 - Editable
    say/0.1@user/testing:80faec7955dcba29246085ff8d64a765db3b414f - Editable

say/0.1@user/testing: Generator cmake created conanbuildinfo.cmake
...
hello/0.1@user/testing: Generator cmake created conanbuildinfo.cmake
...
chat/0.1@user/testing: Generator cmake created conanbuildinfo.cmake
```

可以看到每个包下都产生了一个conanbuildinfo.cmake文件，目录结构定义在layout_gcc文件中：

```ini
# This helps to define the location of CMakeLists.txt within package
[source_folder]
src

# This defines where the conanbuildinfo.cmake will be written to
[build_folder]
build/{{settings.build_type}}
```

我们可以如下构建以及运行这个工程了：

```sh
$ cmake .. -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release
$ cmake --build . # or just $ make
$ ../chat/build/Release/app
Release: Hello World!
Release: Hello World!
Release: Hello World!
```

现在，到某一个包下面修改代码并执行构建，可以看看它可以执行更快的增量构建。另外，可以看到，包并没有安装到本地的conan缓存中，调用`conan search say`会返回查找失败。

注意，本例中每个包的conanfile.py和普通的包配置文件并没有什么差异，但是CMakeLists.txt有些不同：`conan_basic_setup(NO_OUTPUT_DIRS）`。这时因为默认的`conan_basic_setup()`会为工件(例如bin、lib等)定义输出目录，这些目录和本地工程期望的布局可能不同。你需要检查以确定你的构建脚本和包配置与工程期望的目录布局（定义在layout文件中的）是匹配的。

下面是为上例构建具有独立目录的debug模式：

```sh
$ cd .. && mkdir build_debug && cd build_debug
$ conan workspace install ../conanws_gcc.yml --profile=my_gcc_profile -s build_type=Debug
$ cmake .. -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Debug
$ cmake --build . # or just $ make
$ ../chat/build/Debug/app
Debug: Bye World!
Debug: Bye World!
Debug: Bye World!
```

### 多配置构建环境（Multi configuration build environments）

一些构建系统，如Visual Studio（MSBuild），使用多配置环境（multi-configuration）。这意味着工程被配置一次，然后你可以通过IDE在多个配置间（Debug/Release）切换。

上面的例子中Conan为每个构建目录（Debug/Release）产生一个独立的conanbuildinfo.cmake。对于Visual Studio，我们希望能欧在Debug和Release之间切换，我们重新在一个干净的构建目录上执行`conan workspace install`。

Conan有一个[cmake_multi](https://docs.conan.io/en/latest/integrations/build_system/cmake/cmake_multi_generator.html#cmake-multi)的生成器，允许直接通过IDE在Debug和Release之间切换。这个示例的工作区配置如下：

```yaml
editables:
say/0.1@user/testing:
    path: say
hello/0.1@user/testing:
    path: hello
chat/0.1@user/testing:
    path: chat
layout: layout_vs
generators: cmake_multi
workspace_generator: cmake
root: chat/0.1@user/testing
```

注意上面`generators: cmake_multi`这句，定义了使用`cmake_multi`作为CMake生成器。因此，Conan将不会生成conanbuildinfo.cmake，而是生成conanbuildinfo_multi.cmake文件。可以看到“hello/src/CMakeLists.txt”文件中，会包含对应的cmake文件。

```cmake
project(Hello)

if(EXISTS ${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo_multi.cmake)
    include(${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo_multi.cmake)
else()
    include(${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo.cmake)
endif()

conan_basic_setup(NO_OUTPUT_DIRS)

add_library(hello hello.cpp)
conan_target_link_libraries(hello)
```

上面文件最后的`conan_target_link_libraries(hello)`是一个帮助函数，可以让目标链接到正确的Debug或者Release库上。

根据需要安装对应的Debug或者Release配置。下面的配置将会产生一个Visual Studio的solution，你可以打开它选择执行程序。你可以同时为工程生成Release版本，然后在Visual Studio中进行Debug和Release的切换。

```sh
$ mkdir build && cd build
$ conan workspace install ../conanws_vs.yml
$ conan workspace install ../conanws_vs.yml -s build_type=Debug
$ cmake .. -G "Visual Studio 15 Win64"
```

如果我们打开工程的build目录，可以看到多个版本的cmake文件都会在同一个build目录下，而不会有各自独立的目录。可以看看layout_vs文件，构建目录布局是`[build_folder]`而不是`[build/{settings.build_type}]`:

```ini
[build_folder]
build
```

### 源码之外构建

前面的例子都是使用可编辑模式的包的源码里面的build目录，也可以使用源码之外的build目录布局，依靠相对路径和`reference`参数。下面的布局定义了如何使用源码外的build目录存放所有包的构建结果：

```ini
[build_folder]
../build/{{reference.name}}/{{settings.build_type}}

[includedirs]
src

[libdirs]
../build/{{reference.name}}/{{settings.build_type}}/lib
```

### 补充

本节讲的包的开发方式最后并非真正的发布包（你仍然可以尝试使用`conan export-pkg`），推荐使用`conan create`创建并发布完整的包（最好是在持续集成服务器上）。

目前，工作区仅支持CMake的生成器，Visual Studio的目前在考虑中，尚不完全支持。


