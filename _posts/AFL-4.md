---
title: AFL-4
date: 2023-08-23 12:47:05
tags:
---

exercise-4

libtiff

## 01 实验过程

### 0x1 下载并解压libtiff

```
wget https://download.osgeo.org/libtiff/tiff-4.0.4.tar.gz
tar -xzvf tiff-4.0.4.tar.gz

cd tiff-4.0.4/
./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```

对`tiffinfo`位于`/bin`文件夹中的二进制文件进行模糊测试

![](https://img1.imgtp.com/2023/08/23/83sMVaS4.png)

上面的“-j -c -r -s -w” 是为了提高**代码覆盖率**并增加发现错误的机会，

下面看一看此程序的代码覆盖率，使用html输出查看一下

![](https://img1.imgtp.com/2023/08/23/1ERF1MGB.png)

下面在启用 ASAN 的情况下编译 libtiff进行模糊测试：

`afl-fuzz -m none -i $HOME/fuzzing_tiff/tiff-4.0.4/test/images/ -o $HOME/fuzzing_tiff/out/ -s 123 -- $HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w @@`

![](https://img1.imgtp.com/2023/08/23/e8liZLOp.png)

输入命令查看漏洞跟踪结果：

![](https://img1.imgtp.com/2023/08/23/9XSNK2Tq.png)

详细显示

![](https://img1.imgtp.com/2023/08/23/KvEyKDpd.png)

## 02 实验心得

### 0x1 代码应用方面

创建一个新目录：

```
cd $HOME
mkdir fuzzing_tiff && cd fuzzing_tiff/
```

下载并解压 libtiff 4.0.4：

```
wget https://download.osgeo.org/libtiff/tiff-4.0.4.tar.gz
tar -xzvf tiff-4.0.4.tar.gz
```

构建并安装 libtiff：

```
cd tiff-4.0.4/
./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```

测试

```
$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
```

通过使用代码覆盖率，我们将了解模糊器已到达代码的哪些部分，并可视化模糊测试过程。

安装lcov

```
sudo apt install lcov
```

使用标志（编译器和链接器）重建 libTIFF `--coverage`：

```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
  
CFLAGS="--coverage" LDFLAGS="--coverage" ./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```

通过输入以下内容来收集代码覆盖率数据：

```
cd $HOME/fuzzing_tiff/tiff-4.0.4/
lcov --zerocounters --directory ./
lcov --capture --initial --directory ./ --output-file app.info
$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
lcov --no-checksum --directory ./ --capture --output-file app2.info
```

生成 HTML 输出：

```
genhtml --highlight --legend -output-directory ./html-coverage/ ./app2.info
```

清理所有先前编译的目标文件和可执行文件

```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
```

在调用 make 之前设置 AFL_USE_ASAN=1

```
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
AFL_USE_ASAN=1 make -j4
AFL_USE_ASAN=1 make install
```

运行模糊器

```
afl-fuzz -m none -i $HOME/fuzzing_tiff/tiff-4.0.4/test/images/ -o $HOME/fuzzing_tiff/out/ -s 123 -- $HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w @@
```

### 0x2 工具理解方面





