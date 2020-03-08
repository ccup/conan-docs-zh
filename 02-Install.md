# 安装

-  通过pip安装（官方推荐）

需要Python版本大于等于3.5。Python 3.4以及Python2的支持将被废弃。新的Python都会预装pip，所以应该可以直接运行pip。

```sh
pip install conan
```

如果你的机器同时有python2和python3，可能需要运行：

```sh
pip3 install conan
```

- OSX安装

MAC上可以使用brew直接安装：

```sh
brew update
brew install conan
```

- 二进制安装

也可以从conan官网上直接下载二进制安装：https://conan.io/downloads.html

- 源码安装

确保你预装了python和pip。

```sh
git clone https://github.com/conan-io/conan.git
cd conan
pip install -r conans/requirements.txt
```

创建一个脚本，将其加入你的path路径中：

```sh
#!/usr/bin/env python

import sys

conan_repo_path = "/home/your_user/conan" # ABSOLUTE PATH TO CONAN REPOSITORY FOLDER

sys.path.append(conan_repo_path)
from conans.client.command import main
main(sys.argv[1:])
```

- 测试conan安装OK：

```sh
$ conan
```

打印类似如下，说明安装OK

```sh
Consumer commands
  install    Installs the requirements specified in a recipe (conanfile.py or conanfile.txt).
  config     Manages Conan configuration.
  get        Gets a file or list a directory of a given reference or package.
  info       Gets information about the dependency graph of a recipe.
  ...
```

- 升级

最简单的方式：

```sh
$ pip install conan --upgrade
```
