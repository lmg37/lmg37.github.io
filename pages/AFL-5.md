---
title: AFL-5
date: 2023-08-23 13:18:19
tags:
---

exercise-5

LibXML2 XML 解析库

下载并安装libxml2-2.9.4.tar.gz

```
wget http://xmlsoft.org/download/libxml2-2.9.4.tar.gz
tar xvf libxml2-2.9.4.tar.gz && cd libxml2-2.9.4/
#安装
sudo apt-get install python-dev
CC=afl-clang-lto CXX=afl-clang-lto++ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --prefix="$HOME/Fuzzing_libxml2/libxml2-2.9.4/install" --disable-shared --without-debug --without-ftp --without-http --without-legacy --without-python LIBS='-ldl'
make -j$(nproc)
make install
```

通过`./xmllint --memory ./test/wml.xml`检测后

![](https://img1.imgtp.com/2023/08/23/hxYovSjF.png)

下面创建XML字典例子

```
mkdir afl_in && cd afl_in
wget https://raw.githubusercontent.com/antonio-morales/Fuzzing101/main/Exercise%205/SampleInput.xml
cd ..
```

进行模糊测试

`afl-fuzz -m none -i ./afl_in -o afl_out -s 123 -x ./dictionaries/xml.dict -D -M master -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@`

但是进行三天仍未成功

![未完成](https://img1.imgtp.com/2023/08/23/wmt3LqdA.png)

