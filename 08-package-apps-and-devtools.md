# 打包应用与开发工具

使用Conan可以打包和部署应用程序，同样支持打包和部署开发工具，如编译器（例如MinGW）或者构建系统（如CMake）。

本节描述如何打包以及运行可执行程序以及开发工具，以及如何依据`build_requires`的描述从源码构建各种开发工具或者库（如测试框架等）。

## 运行以及部署包

使用Conan也可以对包含动态库的可执行程序进行分发和部署。相比较使用其他部署工具，使用Conan具有如下优点：

- Conan可以作为统一的跨系统和平台的开发和分发工具；
- 以统一的方式（包管理的方式）管理大量不同的部署配置；
- 可以使用Conan服务端存储各种系统、平台以及目标的应用程序和运行时；

具体Conan可以使用以下几种不同的方式，对应用程序进行分发和部署。

### 使用虚拟环境（virtual environments）

我们可以创建包含可执行程序的包。我们以默认的`conan new`产生的包模板举例：`conan new Hello/0.1`。

这个源码会产生一个叫做`greet`的可执行程序，但是可执行程序默认并不会被打包。我们可以修改包配置的`package()`函数将可执行程序也打包起来：

```python
def package(self):
    self.copy("*greet*", src="bin", dst="bin", keep_path=False)
```

现在我们可以像以往一样创建包，但是当我们想要运行可执行程序的时候会发现找不到：

```sh
$ conan create . user/testing
...
Hello/0.1@user/testing package(): Copied 1 '.h' files: hello.h
Hello/0.1@user/testing package(): Copied 1 '.exe' files: greet.exe
Hello/0.1@user/testing package(): Copied 1 '.lib' files: hello.lib

$ greet
> ... not found...
```

默认情况下，Conan并不会修改环境，它仅将包创建在本地缓存，而对应的路径不会加入到系统PATH里面，所以greet的可执行程序系统是找不到的。

使用`virtualrunenv`生成器可以产生对应的文件，能够将包的默认二进制路径加到以下需要的路径中：

- 将依赖的lib子目录加到`DYLD_LIBRARY_PATH`环境变量中（为OSX系统中的共享库）；
- 将依赖的lib子目录加入到`LD_LIBRARY_PATH`环境中（为Linux系统中的共享库）；
- 将依赖的bin子目录加入到系统的PATH环境变量中（为可执行程序）；

我们在安装包的时候，指定`virtualrunenv`：

```sh
$ conan install Hello/0.1@user/testing -g virtualrunenv
```

这样就会产生一些文件，可以激活或者去激活需要的环境变量：

```sh
$ activate_run.sh # $ source activate_run.sh in Unix/Linux
$ greet
> Hello World!
$ deactivate_run.sh # $ source deactivate_run.sh in Unix/Linux
```

### Imports

同样可以自定义conanfile(txt或者py的都可以)，在里面使用`imports`段，这样就会把本地缓存中需要的文件拷贝出来。`imports`的具体细节会在后面给出示例。

### 可部署的包

使用`deploy`函数可以定义将包对应的文件或者构建产物拷贝到系统其它地方的用户空间中。我们给前面的例子加上`deploy()`方法：

```python
def deploy(self):
    self.copy("*", dst="bin", src="bin")
```

这时运行`conan create . user/testing`。可以看到Conan将包的可执行程序拷贝到了当前的bin目录下：

```sh
$ conan install Hello/0.1@user/testing
...
> Hello/0.1@user/testing deploy(): Copied 1 '.exe' files: greet.exe
$ bin\greet.exe
> Hello World!
```

在部署的过程中，Conan创建了一个deploy_manifest.txt文件，里面记录了所有部署的文件及其内容的哈希值。

有的时候部署如果不关心构建时的编译器的话，可以为此调整包的package ID：

```python
def package_id(self):
    del self.info.settings.compiler
```

进一步了解更多关于`deploy`函数的用法，参见[deploy文档](https://docs.conan.io/en/latest/reference/conanfile/methods.html#method-deploy)。


### 使用部署生成器（deploy generator）

部署生成器负责生成文件，用来记录所有被拷贝部署的文件和其哈希值。这使得部署的过程变得可重复。下面的命令会将所有的依赖项移除conan缓冲区，将其汇集到一个独立的空间：`conan install . -g deploy`

### 使用json生成器（json generator）

一个更好的方式是使用json生成器：这个生成器在部署时不会将文件拷贝到一个目录中，而是产生一个JSON文件(conanbuildinfo.json)记录所有的依赖信息，包含每个文件在Conan缓冲区中的位置。

```sh
$ conan install . -g json
```

conanbuildinfo.json文件是一个为机器生成的文件，可以使用脚本处理它。下面的代码演示了如何从该文件中读取库以及库的目录：

```python
import os
import json

data = json.load(open("conanbuildinfo.json"))

dep_lib_dirs = dict()
dep_bin_dirs = dict()

for dep in data["dependencies"]:
    root = dep["rootpath"]
    lib_paths = dep["lib_paths"]
    bin_paths = dep["bin_paths"]

    for lib_path in lib_paths:
        if os.listdir(lib_path):
            lib_dir = os.path.relpath(lib_path, root)
            dep_lib_dirs[lib_path] = lib_dir
    for bin_path in bin_paths:
        if os.listdir(bin_path):
            bin_dir = os.path.relpath(bin_path, root)
            dep_bin_dirs[bin_path] = bin_dir
```

Json生成器的好处在于它只记录部署依赖的文件，但是并不会自行拷贝，将选择权交给用户，用户可以根据需要进行选择并将文件拷贝成想要的目录布局。上面的脚本很容易修改得去执行各种过滤，并完成目标任务。

另外，你也可以自己写一些简单的启动脚本，为你的应用程序进行各种信息配置：

```python
executable = "MyApp"  # just an example
varname = "$APPDIR"

def _format_dirs(dirs):
    return ":".join(["%s/%s" % (varname, d) for d in dirs])

path = _format_dirs(set(dep_bin_dirs.values()))
ld_library_path = _format_dirs(set(dep_lib_dirs.values()))
exe = varname + "/" + executable

content = """#!/usr/bin/env bash
set -ex
export PATH=$PATH:{path}
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:{ld_library_path}
pushd $(dirname {exe})
$(basename {exe})
popd
""".format(path=path,
       ld_library_path=ld_library_path,
       exe=exe)
Note
```

### 从包开始运行

如果想要在conanfile中运行某个依赖包中的可执行程序，可以在包配置文件中使用`run_environment=True`参数。它会间接调用`RunEnvironment()`帮助函数。例如我们想要在构建Consumer包的时候执行greet程序：

```python
from conans import ConanFile, tools, RunEnvironment

class ConsumerConan(ConanFile):
    name = "Consumer"
    version = "0.1"
    settings = "os", "compiler", "build_type", "arch"
    requires = "Hello/0.1@user/testing"

    def build(self):
        self.run("greet", run_environment=True)
```

现在为Consumer执行`conan install`和`conan build`，可以看到greet被执行了。

```sh
$ conan install . && conan build .
...
Project: Running build()
Hello World!
```

当然也可以显示的通过对应依赖的路径访问可执行程序，如下例。但是这种方法对于有动态库存在的情况下是不行的。

```python
def build(self):
    path = os.path.join(self.deps_cpp_info["Hello"].rootpath, "bin")
    self.run("%s/greet" % path)
```

因此，使用`run_environment=True`是一个更完整的解决方案。

最后，还有另一种做法，那就是可以将包的可执行程序的bin目录直接添加到系统的PATH中。例如在Hello包的配置中如下修改：

```python
def package_info(self):
    self.cpp_info.libs = ["hello"]
    self.env_info.PATH = os.path.join(self.package_folder, "bin")
```

使用这种方法，如果可执行程序需要的话，我们同样可以定义`DYLD_LIBRARY_PATH`和`LD_LIBRARY_PATH`。

这样消费方的包就会比较简单，可以直接调用可执行程序：

```python
def build(self):
    self.run("greet")
```

### Runtime packages and re-packaging

可以创建只包含可以运行的二进制的包，而将供编译时依赖的头文件和库文件等文件都去掉。比如对于Hello包，通过如下方式就可以做到：

```python
from conans import ConanFile

class HellorunConan(ConanFile):
    name = "HelloRun"
    version = "0.1"
    build_requires = "Hello/0.1@user/testing"
    keep_imports = True

    def imports(self):
        self.copy("greet*", src="bin", dst="bin")

    def package(self):
        self.copy("*")
```

这样的包配置文件具有如下特点：

- 它将`Hello/0.1@user/testing`设置为`build_requires`。这意味着Hello包仅仅被用于构建HelloRun包，一旦构建结束就不再需要Hello包了；

- 它使用`imports()`将所有依赖的可执行文件拷贝出来；

- 它通过设置`keep_imports=True`定义将构建阶段（`build()`函数没有定义，所以用默认的）的产物在构建结束后保留下来；

- `package()`函数将build目录下import出来的文件进行打包；

以下是创建并上传包：

```python
$ conan create . user/testing
$ conan upload HelloRun* --all -r=my-remote
```

安装及运行这个包，可以使用我们前面介绍过的任一方式，例如

```sh
$ conan install HelloRun/0.1@user/testing -g virtualrunenv
# You can specify the remote with -r=my-remote
# It will not install Hello/0.1@...
$ activate_run.sh # $ source activate_run.sh in Unix/Linux
$ greet
> Hello World!
$ deactivate_run.sh # $ source deactivate_run.sh in Unix/Linux
```

### 部署挑战

当部署一个C/C++应用程序的时候，有一些特殊的挑战需要应对。这里是一些常见的挑战以及建议的解决方式。

#### C标准库

一个常见的挑战是，应用程序（无论是C还是C++写的）都可能依赖C的标准库，最常见的就是GNU的C库：glibc。

Glibc其实不仅是C标准库，它包含以下内容：

- C函数，如`malloc()`、`sin()`等，以及语言标准，如C99；
- POSIX函数，如`pthread`库；
- BSD函数，如BSD套接字（socket）；
- 对操作系统特定API的封装，如Linux的系统调用；

及时你的应用程序没有直接使用这些函数，但是有大量的库在使用这些函数，因此很可能你的库间接依赖到了glibc。

这里的问题是glibc在不同的Linux发行版本中是不兼容的！

例如我们在新的Ubuntu系统上构建的hello world程序在Centos 6上就会出错：

```sh
$ /hello
/hello: /lib64/libc.so.6: version `GLIBC_2.14' not found (required by /hello)
```

可以看到，两个Linux系统上glibc的版本是不同的。

还有其它一些C标准库的实现，例如面向嵌入式开发的[newlib](https://sourceware.org/newlib/)和[musl](https://www.musl-libc.org/)，也会有一样的挑战。

有几种针对该问题的解决方案：
- [LibcWrapGenerator](https://github.com/AppImage/AppImageKit/tree/stable/v1.0/LibcWrapGenerator)
- [glibc_version_header](https://github.com/wheybags/glibc_version_header)
- [bingcc](https://github.com/sulix/bingcc)

一些人建议使用glibc的静态库，但是强烈建议不要这样做。一个原因是新的glibc可能使用早期版本不存在的系统调用，如果你的应用程序依赖了新的glibc，可能在某些系统上出现随机的运行时失败，这种问题非常难以调试和定位。

可以通过在Conan包配置里面把不同glibc版本的Linux发行名称作为Conan的子settings（通过在settings.yml文件中定义），这和我们前面讲的通过`package_id()`和`build_id()`为包定义兼容性是一样的。这样不同glibc版本的linux将会获取不同的包或者触发自行从源码构建。具体为Conan增加子settings的方式见[文档](https://docs.conan.io/en/latest/extending/custom_settings.html#add-new-settings)。

#### C++标准库

一般默认的C++标准库是`libstdc++`，但是`libc++`和`stlport`也是常用的实现版本。

和标准C库glibc类似，应用程序和老系统上的libstdc++链接，也可能会发生错误：

```sh
$ /hello
/hello: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.21' not found (required by /hello)
/hello: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.26' not found (required by /hello)
```

幸运的是，对于C++我们可以简单的为编译增加`-static-libstdc++`的编译标记即可，这是因为C++标准库不会直接使用系统调用，往往是通过libc库帮助其封装。

#### 编译器运行时

除了C和C++运行时库，应用程序可能还会用编译器的运行时库。这些编译器运行时库为应用程序提供了一些低层次的函数，例如支持处理异常的编译器指令。编译器运行时库中的函数往往不会被代码直接调用，大多数时候是由编译器隐式的插入到代码中的。例如下面的可执行程序就依赖了libgcc_s.so。

```sh
$ ldd ./a.out
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f6626aee000)
```

可以使用`-static-libgcc`的编译期标记来对其静态链接而避免各种应用程序分发问题。另外，可以通过[GCC手册](https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html)尝试找到其它更具体的解决方案。

#### 系统调用（syscall）

Linux内核经常会提供新的系统调用，如果应用程序或者第三方库想要使用这些新的能力，有的时候可能会直接调用这些系统API（而非使用glibc的封装）。

上述行为的结果是，应用程序需要在对应的内核上编译，当其发布到其它老的内核系统上执行可能会运行失败。

所以建议尽量使用glibc，或者根据linux内核的版本为定义Conan定义新的settings和子settings，然后用其保证二进制兼容性。

## 创建Conan包，用于安装开发工具

Conan 1.0引入了两个新的settings，`os_build`以及`arch_build`。这两个配置表示运行Conan的机器（而非表示包的目标机器的`os`和`arch`），这样就使得我们可以打包一些构建阶段使用的工具，例如编译器或者构建系统等。

如下示例一个打包汇编编译工具`nasm`的包配置：

```python
import os
from conans import ConanFile
from conans.client import tools


class NasmConan(ConanFile):
    name = "nasm"
    version = "2.13.01"
    license = "BSD-2-Clause"
    url = "https://github.com/conan-community/conan-nasm-installer"
    settings = "os_build", "arch_build"
    build_policy = "missing"
    description="Nasm for windows. Useful as a build_require."

    def configure(self):
        if self.settings.os_build != "Windows":
            raise Exception("Only windows supported for nasm")

    @property
    def nasm_folder_name(self):
        return "nasm-%s" % self.version

    def build(self):
        suffix = "win32" if self.settings.arch_build == "x86" else "win64"
        nasm_zip_name = "%s-%s.zip" % (self.nasm_folder_name, suffix)
        tools.download("http://www.nasm.us/pub/nasm/releasebuilds/"
                       "%s/%s/%s" % (self.version, suffix, nasm_zip_name), nasm_zip_name)
        self.output.warn("Downloading nasm: "
                         "http://www.nasm.us/pub/nasm/releasebuilds"
                         "/%s/%s/%s" % (self.version, suffix, nasm_zip_name))
        tools.unzip(nasm_zip_name)
        os.unlink(nasm_zip_name)

    def package(self):
        self.copy("*", dst="", keep_path=True)
        self.copy("license*", dst="", src=self.nasm_folder_name, keep_path=False, ignore_case=True)

    def package_info(self):
        self.output.info("Using %s version" % self.nasm_folder_name)
        self.env_info.path.append(os.path.join(self.package_folder, self.nasm_folder_name))
```

上面的包配置中有几个值得注意的点：

- `configure()`函数只支持Windows，其它的配置将抛出异常；
- `build()`函数下载对应的文件以及unzip它；
- `package()`函数将所有解压的文件拷贝到包目录下；
- `package_info()`函数使用`self.env_info`将包的bin目录加入到系统的环境变量path中；

这个包有两个不同于一般Conan包的地方：

- 没有`source()`函数。这是因为当你编译一个库，需要从源码执行构建。而在本例中，我们直接下载二进制程序，build函数仅仅完成下载以及解压，并不需要源码。当然，如果你需要从源码构建工具，你也可以像之前那样创建包配置；

- `package_info()`函数使用了`self.env_info`对象。使用`self.env_info`对象，包可以在依赖这个构建工具的包执行`build()`、`package()`和`imports()`之前先自动的声明环境变量。这样工具的消费者就可以方便的使用工具，而不用自己先得去设置好系统path；

### 在其它的包配置文件中使用工具包（tool package）

当你依赖一个声明了`self.env_info`变量的包配置文件的时候，`self.env_info`中的环境变量将会自动应用。例如看看MinGW的包配置文件 conanfile.py (https://github.com/conan-community/conan-mingw-installer)。

```python
 class MingwInstallerConan(ConanFile):
     name = "mingw_installer"
     ...

     build_requires = "7zip/19.00"

     def build(self):
         keychain = "%s_%s_%s_%s" % (str(self.settings.compiler.version).replace(".", ""),
                                     self.settings.arch_build,
                                     self.settings.compiler.exception,
                                     self.settings.compiler.threads)

         files = {
            ...        }

         tools.download(files[keychain], "file.7z")
         self.run("7z x file.7z")

     ...
```

上面的文件中声明了`build_requires`，依赖了`7zip`，用于在下载了MinGW安装器之后执行解压。因此，下载了MinGW安装器后，7z的可执行程序将会在PATH中，因为它依赖的7zip包的`package_info()`里面对此做了声明。

注意：一些构建依赖有可能需要设置`os`、`compiler`或者`arch`来从源码构建它们。这些情况下，包的配置文件可能如下：

```python
    settings = "os_build", "arch_build", "arch", "compiler"
    ...

    def build(self):
        cmake = CMake(self)
        ...

    def package_id(self):
        self.info.include_build_settings()
        del self.info.settings.compiler
        del self.info.settings.arch
```

上面`package_id()`删除只为从源码构建的settings，保留了用于包分发的`os_build`和`arch_build`选项。

### 在你的系统中使用工具包（tool packages）

可以使用[virtualenv generator](https://docs.conan.io/en/latest/reference/generators/virtualenv.html#virtualenv-generator)来使用安装到你机器上的工具包。例如，基于windows上使用MinGW和CMake：

- 在你的工程外创建一个独立的目录，用于保存和配置全局开发环境:

```sh
$ mkdir my_cpp_environ
$ cd my_cpp_environ
```

- 创建一个conanfile.txt文件：

```toml
[requires]
mingw_installer/1.0@conan/stable
cmake/3.16.3

[generators]
virtualenv
```

- 安装依赖：`conan install`；

- 在shell中激活虚拟环境：

```sh
$ activate
(my_cpp_environ)$
```

- 检查工具是否在path中：

```sh
(my_cpp_environ)$ gcc --version

> gcc (x86_64-posix-seh-rev1, Built by MinGW-W64 project) 4.9.2

 Copyright (C) 2014 Free Software Foundation, Inc.
 This is free software; see the source for copying conditions.  There is NO
 warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

(my_cpp_environ)$ cmake --version

> cmake version 3.16.3

  CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

- 可以通过去激活脚本将虚拟环境去激活：

```sh
(my_cpp_environ)$ deactivate
```

## 构建时依赖

有一些包并不适合增加到包的依赖里面，例如你有一个`Zlib`包依赖于包`Cmake/3.4`，这时你需要考虑依赖CMake仅是为了构建`Zlib`？

- 有一些依赖仅仅是为了从源码构建包时需要；
- 这些大多都是一些开发工具、编译器、构建系统、代码分析、测试库等等；
- 这些依赖往往和包的交付后使用无关，它们仅仅用于包的生成过程；
- 对于这些工具，你并不会添加太多的版本，对这些依赖的修改也希望能尽可能简单；
- 有一些工具链甚至不被包创建时考虑在内，例如将zlib交叉编译到Android系统，虽然在这种场景下Android工具链也是构建时依赖；

为了满足上述需求，Conan实现了`build_requires`。

### 声明构建时依赖

构建时依赖可以通过profile文件来声明：

```toml
 [build_requires]
 Tool1/0.1@user/channel
 Tool2/0.1@user/channel, Tool3/0.1@user/channel
 *: Tool4/0.1@user/channel
 MyPkg*: Tool5/0.1@user/channel
 &: Tool6/0.1@user/channel
 &!: Tool7/0.1@user/channel
```

在profile中构建时依赖可以通过pattern来指定。不同的包可以指定不同的构建时依赖。上例中Tool1、Tool2、Tool3以及Tool4将会被用于所有的包（当执行`conan install`或者`conan create`时）。Tool5将会被应用于以“MyPkg”开头的包。`&`用于没有在conanfile中指明名称和版本的包，而`&!`则正好相反。

不要忘了，包的消费者的conanfile可能在test_package目录下，或者通过`conan install`传入，这些也会使用上述profile中的规则。

包的配置文件中的`build_requires`属性以及`build_requirements()`函数也能够用来指定构建时依赖。

```python
class MyPkg(ConanFile):
    build_requires = "ToolA/0.2@user/testing", "ToolB/0.2@user/testing"

    def build_requirements(self):
        # useful for example for conditional build_requires
        # This means, if we are running on a Windows machine, require ToolWin
        if platform.system() == "Windows":
            self.build_requires("ToolWin/0.1@user/stable")
```

上面ToolA和ToolB将会在构建这个包的时候获取，而ToolWin则只会在Windows上使用。

如果`build_requirements()`和`build_requires`中定义了相同的包名，则`build_requirements()`优先。

根据规则，如果profile中定义了编译期依赖，包配置文件中的定义的编译期依赖如果具有相同的包名，则会覆盖profile中的。

### 构建时依赖的特点

无论构建时依赖定义在`build_requires`中还是定义在profile中，它们都具有相同的特性：

- 构建时依赖，在使用它们的包从源码开始构建的时候，并且和定义的pattern相匹配的时候，构建时依赖的包才会被获取；否则都不会检查这些构建时依赖的包是否存在；

- 通过Profile或者通过命令行传入的Options以及环境变量都会影响到包的构建时依赖的选择。例如你可以通过profile或者命令行指定需要安装cmake/3.16.3版本；

- 如果构建时依赖包满足匹配，则`deps_cpp_info`和`deps_env_info`的成员将会被激活。`deps_cpp_info`的成员有include目录、library名称、编译参数（CFLAGS、CXXFLAGS，LINKFLAGS）、sysroot等，将会从构建时依赖包的`self.cpp_info`的值中应用；而`deps_env_info`的成员，如PATH、PYTHONPATH等将作为环境变量被激活；

- 构建时依赖同样会被传递。每个依赖可以继续声明自己的依赖，包括普通依赖和构建时依赖。构建时依赖的冲突解决和依赖覆写规则和普通依赖是一样的；

- Conan一样会为匹配的构建时依赖创建依赖图谱，并将其缓存下来。构建时依赖被安装后缓存在Conan的本地缓存区；

- 构建时依赖不影响包二进制的package ID。如果使用一个不同的构建时依赖产生出一个不同的二进制，你应当考虑增加options或者settings以反映出构建时依赖对二进制兼容性的影响；

- `conan info`不会列出构建时依赖的包；

### 测试框架库

一个使用构建时依赖的例子就是测试框架。测试框架往往被实现为库，我们下面的例子中假设有一个叫做`mytest_framework`的测试框架库，并且已经存在其对应的Conan包。

下面的例子中，在包配置文件中使用逻辑判断检查构建时依赖是否存在：

```python
def build(self):
    cmake = CMake(self)
    enable_testing = "mytest_framework" in self.deps_cpp_info.deps
    cmake.configure(defs={"ENABLE_TESTING": enable_testing})
    cmake.build()
    if enable_testing:
        cmake.test()
```

同时，包的CMakeLists.txt如下：

```cmake
project(PackageTest CXX)
cmake_minimum_required(VERSION 2.8.12)

include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup()
if(ENABLE_TESTING)
    add_executable(example test.cpp)
    target_link_libraries(example ${CONAN_LIBS})

    enable_testing()
    add_test(NAME example
              WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/bin
              COMMAND example)
endif()
```

此时我们执行`conan install`，包的配置文件并不会获取`mytest_framework`也不会构建测试。

但是如果定义如下的profile （例如mytest_profile）：

```ini
 [build_requires]
 mytest_framework/0.1@user/channel
```

然后调用下面的命令，将会获取`mytest_framework`，并且构建并运行测试。

```sh
$ conan install . --profile=mytest_profile
```

### 共用的python代码

构建时依赖也可以用来注入或者重用包配置文件中的python代码。

如下Conan包用来封装和重用mypythontool.py文件：

```python
import os
from conans import ConanFile

class Tool(ConanFile):
    name = "PythonTool"
    version = "0.1"
    exports_sources = "mypythontool.py"

    def package(self):
        self.copy("mypythontool.py")

    def package_info(self):
        self.env_info.PYTHONPATH.append(self.package_folder)
```

然后，如果在profile中定义了构建时依赖：

```python
[build_requires]
PythonTool/0.1@user/channel
```

这样包里封装的Python代码就能被其它包配置文件复用：

```python
def build(self):
    self.run("mytool")
    import mypythontool
    self.output.info(mypythontool.hello_world())
```

注意：这种重用python代码的方式，将会被Conan新提供的`python_requires`的方式替代，具体请参考[Python Requires的文档](https://docs.conan.io/en/latest/extending/python_requires.html#python-requires)。