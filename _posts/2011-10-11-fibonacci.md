---
layout: post
title: 青蛙跳台阶的问题——Fibonacci
categories: Algorithm
description: 使用 Fibonacci 来解决青蛙跳台阶的问题。
keywords: 算法，Fibonacci
---

## 01 实验过程

### 0x1  AFL简介

` tar xvf afl-latest.tgz`

<img src="https://img1.imgtp.com/2023/08/05/ei42j3Np.png" alt="image-20230731140037021" style="zoom:80%;" />

`cd afl-2.52b
sudo make && sudo make install`

执行完三条命令后

<img src="https://img1.imgtp.com/2023/08/05/IEpgbmN0.png" alt="image-20230731140207486" style="zoom:80%;" />

可以看到afl成功下载

<img src="https://img1.imgtp.com/2023/08/05/P7KkyF0d.png" alt="image-20230731141408035" style="zoom:80%;" />

再下载XPDF

<img src="https://img1.imgtp.com/2023/08/05/rCQ1lzn8.png" alt="image-20230731184215275" style="zoom:67%;" />

后面出现问题 改成ubuntuFuzz继续做

先下载afl

<img src="https://img1.imgtp.com/2023/08/05/wKxGdbzJ.png" alt="image-20230804221002348" style="zoom:80%;" />

再下载xpdf

<img src="https://img1.imgtp.com/2023/08/05/6iSpLQnF.png" style="zoom:67%;" />

执行`./configure --prefix="$HOME/fuzzing_xpdf/install/"`

<img src="https://img1.imgtp.com/2023/08/05/m1ILuqF0.png" alt="image-20230804165910916" style="zoom:67%;" />

下载几个文件进行验证

<img src="https://img1.imgtp.com/2023/08/05/Sz9C8dyv.png" style="zoom:80%;" />

后面发现-s无法进行 于是检查发现需要下载afl++

### 0x2 下载AFL++

环境：

```basic
sudo apt-get update
sudo apt-get install -y build-essential python3-dev automake git flex bison libglib2.0-dev libpixman-1-dev python3-setuptools
sudo apt-get install -y lld-11 llvm-11 llvm-11-dev clang-11 || sudo apt-get install -y lld llvm llvm-dev clang 
sudo apt-get install -y gcc-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-plugin-dev libstdc++-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-dev
```

检查并下载AFL++

```basic
cd $HOME
git clone https://github.com/AFLplusplus/AFLplusplus && cd AFLplusplus
export LLVM_CONFIG="llvm-config-11"
make distrib
sudo make install
```

下载成功

使用AFL++编译器编译代码 首先清理没用的文件

```
rm -r $HOME/fuzzing_xpdf/install
cd $HOME/fuzzing_xpdf/xpdf-3.02/
make clean
```

### 0x3 使用 **afl-clang-fast** 编译器构建 xpdf

```
export LLVM_CONFIG="llvm-config-11"
CC=$HOME/AFLplusplus/afl-clang-fast CXX=$HOME/AFLplusplus/afl-clang-fast++ ./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

### 0x4 模糊测试

运行fuzzer

```
AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1  afl-fuzz -i $HOME/fuzzing_xpdf/pdf_examples/ -o $HOME/fuzzing_xpdf/out/ -s 123 -- $HOME/fuzzing_xpdf/install/bin/pdftotext @@ $HOME/fuzzing_xpdf/output
```

解释：-i 表示我们必须放置输入用例的目录 -o 表示 AFL + + 将存储变异文件的目录

<img src="https://img1.imgtp.com/2023/08/05/nztNxZl0.png" alt="9d64cbc5501f607c00d5f3839f3a1d8" style="zoom: 67%;" />

等待找到漏洞

<img src="https://img1.imgtp.com/2023/08/05/fVvfCSvd.png" alt="d2c779b1034bc010bebe3d9c7748119" style="zoom: 67%;" />

### 0x5 漏洞分析

进行漏洞分析 这是运行后找到的其中一个漏洞

![image-20230805110110846](https://img1.imgtp.com/2023/08/05/Tr5PgXOp.png)

可以看出再次使用xpdf运行该文件时错误是堆栈溢出

![image-20230805110524898](https://img1.imgtp.com/2023/08/05/jkbLJvBE.png)

再使用gdb调试

![](https://img1.imgtp.com/2023/08/05/wKxGdbzJ.png)

运行后查看函数栈

![](https://img1.imgtp.com/2023/08/05/rsicPbL8.png)

可以看出一直不停循环 本次漏洞CVE-2019-13288通过使程序中每个被调用的函数都在堆栈上分配一个堆栈帧，如果一个函数被递归调用太多次，就会导致堆栈内存耗尽和程序崩溃。

漏洞修复https://github.com/Ruanxingzhi/Fuzzing101/commit/57d37c2a1ea91a30fa825068d14db48335190672

详解：

https://blog.csdn.net/qq_40025866/article/details/127823491

https://www.ruanx.net/fuzzing101-wp-1/ 关于复现更加详细的解释

## 02 实验心得

### 0x1 代码应用方面

每个实验的文件夹

```
cd $HOME
mkdir fuzzing_xpdf && cd fuzzing_xpdf/
```

初始时更新环境基础工具如make,gcc

```
sudo apt install build-essential
```

下载目标应用Xpdf

```
wget https://dl.xpdfreader.com/old/xpdf-3.02.tar.gz
tar -xvzf xpdf-3.02.tar.gz
```

并构建

```
cd xpdf-3.02
sudo apt update && sudo apt install -y build-essential gcc
./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

`./configure --prefix="$HOME/fuzzing_xpdf/install/"`: 在源代码目录中运行 `configure` 脚本，这个脚本会根据你的系统环境进行一些配置。`--prefix` 参数指定了安装的路径前缀，这将决定软件最终被安装到哪个目录下。在这里，软件将被安装到你的主目录下的 `fuzzing_xpdf/install/` 目录。

下载示例，对软件进行模糊测试需要软件先运行例子进行预热，可以建立一个初始的代码覆盖率基线。这有助于确定目标应用程序的主要执行路径和代码区域，减少随机性。

```
cd $HOME/fuzzing_xpdf
mkdir pdf_examples && cd pdf_examples
wget https://github.com/mozilla/pdf.js-sample-files/raw/master/helloworld.pdf
wget http://www.africau.edu/images/default/sample.pdf
wget https://www.melbpc.org.au/wp-content/uploads/2017/10/small-example-pdf-file.pdf
```

下面进行测试pdfinfo 二进制文件

```
$HOME/fuzzing_xpdf/install/bin/pdfinfo -box -meta $HOME/fuzzing_xpdf/pdf_examples/helloworld.pdf
```

AFL++的安装实验过程中已经介绍，直接运行代码

```
sudo apt-get update
sudo apt-get install -y build-essential python3-dev automake git flex bison libglib2.0-dev libpixman-1-dev python3-setuptools
sudo apt-get install -y lld-11 llvm-11 llvm-11-dev clang-11 || sudo apt-get install -y lld llvm llvm-dev clang 
sudo apt-get install -y gcc-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-plugin-dev libstdc++-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-dev
```

```
cd $HOME
git clone https://github.com/AFLplusplus/AFLplusplus && cd AFLplusplus
export LLVM_CONFIG="llvm-config-11"
make distrib
sudo make install
```

输入afl-fuzz进行验证

验证成功后，开始正式进行模糊测试

清理所有先前编译的目标文件和可执行文件，得到干净的编译环境。

```
rm -r $HOME/fuzzing_xpdf/install
cd $HOME/fuzzing_xpdf/xpdf-3.02/
make clean
```

使用**afl-clang-fast**编译器构建 xpdf

```
export LLVM_CONFIG="llvm-config-11"
CC=$HOME/AFLplusplus/afl-clang-fast CXX=$HOME/AFLplusplus/afl-clang-fast++ ./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

运行模糊器

```
afl-fuzz -i $HOME/fuzzing_xpdf/pdf_examples/ -o $HOME/fuzzing_xpdf/out/ -s 123 -- $HOME/fuzzing_xpdf/install/bin/pdftotext @@ $HOME/fuzzing_xpdf/output
```

遇到关于core问题 解决方法如下：

```
sudo su
echo core >/proc/sys/kernel/core_pattern
exit
```

得出结果。

### 0x2 工具理解方面

1. AFL作为覆盖率模糊测试工具，基于代码覆盖率指导模糊测试过程，也就是模糊测试是通过输入异常稀奇古怪的输入检测潜在漏洞，覆盖率模糊测试是通过**生成测试用例来尝试覆盖尚未执行过的代码路径**

2. 提供测试案例是为了建立初始覆盖率基线的优势是确保你开始测试时已经了解了目标代码的主要执行路径和一些关键区域。当开始进行模糊测试时，确保测试环境是干净的，以避免之前测试的影响。在这个阶段，重点可能会更多地放在随机性和多样性的测试用例生成上，以全面地探索代码路径。
3. **afl-clang-fast**编译器，对clang进行优化，在代码插桩，性能改进等方面更多优化。

