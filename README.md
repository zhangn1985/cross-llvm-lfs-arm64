# cross-llvm-lfs-arm64
## 前言

* 有一天晚上做了一个梦，梦见纯LLVM编译的Linux跑起来了，醒来后才发现是一场梦，在探索一番后，发现LLVM目前编译glibc还有点问题。在网上搜罗一番后，发现了 https://12101111.github.io/llvm-cross-compile/ 。这才有开始LLVM-lfs的起点。

* 本项目现在还在非常早期，可能随时会失败，如果失败就当作流水账，成功会修改成LFS模式的文档发布，这也是本项目的目标。

## 一些思考

* LFS 和 本项目的一些区别：
> LFS使用的是GCC，而GCC天然不是交叉编译器，而LLVM天然是交叉编译器。
>
> LFS是在X86_64上编译x86_64，为了和宿主系统隔离，多次编译GCC。而cross-llvm-lfs-arm64是在x86_64上编译arm64，天然隔离。

     基于以上，可以直接交叉编译出临时工具链，而不需要多次编译LLVM。
 
 * 要尽可能用LLVM编译系统，除非LLVM不支持。
 
 # 准备工作
 
 ## 准备宿主系统
 
 本次实验使用debian 11作为宿主系统
 
  ### 基本环境变量
 
 ```
 export LFS=~/llfs
 export LFS_SRC=~/llfs-sources
 export LFS_BLD=~/llfs-build

 ```
 
 ### 安装编译host LLVM使其成为真正的cross compile
 
 ```
 sudo apt install build-essential llvm clang lld cmake linux-libc-dev-arm64-cross libc6-dev-arm64-cross
 ```
 
 这个步骤是安装llvm，并且编译生成：`libclang_rt.builtins-aarch64.a`，使其真正成为cross compiler.
 
 source: https://github.com/llvm/llvm-project/releases/download/llvmorg-9.0.1/compiler-rt-9.0.1.src.tar.xz
 
 ```
 cd ${LFS_SRC}
 wget https://github.com/llvm/llvm-project/releases/download/llvmorg-9.0.1/compiler-rt-9.0.1.src.tar.xz
 cd ${LFS_BLD}
 tar xvf ${LFS_SRC}/compiler-rt-9*
 cd compiler-rt*
 mkdir build
 cd build
 cmake ../ -DCOMPILER_RT_BUILD_BUILTINS=ON -DCOMPILER_RT_INCLUDE_TESTS=OFF -DCOMPILER_RT_BUILD_SANITIZERS=OFF -DCOMPILER_RT_BUILD_XRAY=OFF -DCOMPILER_RT_BUILD_LIBFUZZER=OFF -DCOMPILER_RT_BUILD_PROFILE=OFF -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" -DCMAKE_C_COMPILER_TARGET=aarch64-linux-gnu -DCMAKE_ASM_COMPILER_TARGET=aarch64-linux-gnu -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON -DCMAKE_INSTALL_PREFIX="/usr/lib/clang/9.0.1/" -DCMAKE_C_COMPILER_WORKS=1 -DCMAKE_CXX_COMPILER_WORKS=1 -DCMAKE_SIZEOF_VOID_P=8 -DCMAKE_SYSROOT=/usr/aarch64-linux-gnu/
 make
 sudo make install
 ```
 
 一些说明：
 -DCMAKE_C_COMPILER_WORKS=1 -DCMAKE_CXX_COMPILER_WORKS=1 -DCMAKE_SIZEOF_VOID_P=8 ： 这时候cmake 并不能有效检查编译器，但我们的编译器的确能工作，除去这些选项会出错。
 
 sudo make install: 我们的确需要安装在host，使host clang能编译出aarch64的目标代码。
 
 思考：
 
 在clang 编译glibc时，也遇到了configure检查编译器失败，看来也应该bypass编译器检查。

 
### 在LFS中创建文件系统布局
```
mkdir -pv $LFS/{bin,etc,lib,sbin,usr,var}
```

### 准备cross工具链

```
export XDG_BIN=$HOME/.local/bin
mkdir -pv $XDG_BIN
ln -s `which lld` $XDG_BIN/${CROSS_COMPILE}ld
ln -s `which lld` $XDG_BIN/${CROSS_COMPILE}ld.lld
ln -s `which clang` $XDG_BIN/${CROSS_COMPILE}gcc
ln -s `which clang` $XDG_BIN/${CROSS_COMPILE}clang
ln -s `which clang++` $XDG_BIN/${CROSS_COMPILE}g++
ln -s `which clang++` $XDG_BIN/${CROSS_COMPILE}clang++
for i in ar nm objcopy objdump ranlib strip;do
  ln -s `which llvm-$i` $XDG_BIN/${CROSS_COMPILE}$i
done
export PATH="$XDG_BIN:$PATH"
```

### 额外的环境变量
```
export COMMON_FLAGS=" -O2 -pipe --sysroot=${LFS}"
export CFLAGS="${COMMON_FLAGS}"
export CXXFLAGS="${COMMON_FLAGS} -stdlib=libc++"
export LDFLAGS="-fuse-ld=lld -rtlib=compiler-rt -flto=thin"
```


### 环境变量总结
```
export LFS=~/llfs
export LFS_SRC=~/llfs-sources
export LFS_BLD=~/llfs-build
export LFS_TGT=aarch64-llfs-linux-musl
export PATH="$HOME/.local/bin:$PATH"
export COMMON_FLAGS=" -O2 -pipe --sysroot=${LFS}"
export CFLAGS="${COMMON_FLAGS}"
export CXXFLAGS="${COMMON_FLAGS} -stdlib=libc++"
export LDFLAGS="-fuse-ld=lld -rtlib=compiler-rt -flto=thin"
```

如果需要重新编译： `libclang_rt.builtins-aarch64.a` 需要删掉这里的环境变量。

## 构建临时工具链

至此，我们已经有能够工作的交叉编译器，能交叉编译目标文件。本章的目的是建立临时工具，以便`chroot`到`$LFS`，生成真正的目标。

### kernel headers

source: https://mirrors.tuna.tsinghua.edu.cn/kernel/v5.x/linux-5.9.9.tar.xz

```
cd ${LFS_SRC}
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v5.x/linux-5.9.9.tar.xz
cd ${LFS_BLD}
tar xvf ${LFS_SRC}/linux-5.9.9.tar.xz
cd linux-5.9.9
make mrproper
make headers
find usr/include -name '.*' -delete
rm usr/include/Makefile
mkdir -pv $LFS/usr/include/
cp -rv usr/include $LFS/usr
```

### musl libc

source: https://musl.libc.org/releases/musl-1.2.1.tar.gz

```
cd $LFS_SRC
wget https://musl.libc.org/releases/musl-1.2.1.tar.gz
cd $LFS_BLD
cd musl*
mkdir build
cd build
./configure --prefix=/ --target=${LFS_TGT} LIBCC=/usr/lib/clang/9.0.1/lib/linux/libclang_rt.builtins-aarch64.a
DESTDIR=$LFS/usr make install-headers
DESTDIR=$LFS/ make install-libs
```

### libunwind

source: https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.1/libunwind-10.0.1.src.tar.xz

```
cd $LFS_SRC
wget https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.1/libunwind-10.0.1.src.tar.xz
cd $LFS_BLD
tar xvf $LFS_SRC/libunwind-10.0.1.src.tar.xz
cd libunwind-10.0.1.src
sed -i 's/include("${LLVM/#include("${LLVM/g' CMakeLists.txt
mkdir build
cd build
cmake ../  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_CXX_COMPILER_TARGET=$LFS_TGT -DCMAKE_C_COMPILER_TARGET=$LFS_TGT -DCMAKE_SYSROOT=$LFS -DCMAKE_INSTALL_PREFIX=$LFS/usr -DCMAKE_CXX_FLAGS="$CXXFLAGS" -DCMAKE_CXX_COMPILER_WORKS=1
make -j8
make install
```

### libc++abi/libcxx
source: https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.1/libcxxabi-10.0.1.src.tar.xz

source: https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.1/libcxx-10.0.1.src.tar.xz

```
cd $LFS_SRC
wget https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.1/libcxxabi-10.0.1.src.tar.xz
wget https://github.com/llvm/llvm-project/releases/download/llvmorg-10.0.1/libcxx-10.0.1.src.tar.xz
cd $LFS_BLD
tar xvf $LFS_SRC/libcxxabi-10.0.1.src.tar.xz
tar xvf $LFS_SRC/libcxx-10.0.1.src.tar.xz
mv libcxxabi-10.0.1.src libcxxabi
mv libcxx-10.0.1.src libcxx
cd libcxxabi
mkdir build
cd build
cmake .. -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_CXX_COMPILER_TARGET=$LFS_TGT -DCMAKE_C_COMPILER_TARGET=$LFS_TGT -DCMAKE_SYSROOT=$LFS -DCMAKE_INSTALL_PREFIX=$LFS/usr -DLIBCXXABI_USE_COMPILER_RT=YES -DLIBCXXABI_USE_LLVM_UNWINDER=YES  -DCMAKE_CXX_COMPILER_WORKS=1
make -j8
make install
cd ../../libcxx/
mkdir build
cd build
cmake ../  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_CXX_COMPILER_TARGET=$LFS_TGT -DCMAKE_C_COMPILER_TARGET=$LFS_TGT -DCMAKE_SYSROOT=$LFS -DCMAKE_INSTALL_PREFIX=$LFS/usr -DLIBCXX_CXX_ABI=libcxxabi -DLIBCXX_USE_COMPILER_RT=YES -DLIBCXX_HAS_MUSL_LIBC=ON -DLIBCXX_CXX_ABI_INCLUDE_PATHS=$LFS_BLD/libcxxabi/include -DCMAKE_CXX_COMPILER_WORKS=1
make -j8
make install
```

### m4

source: http://ftp.gnu.org/gnu/m4/m4-1.4.18.tar.xz

```
cd $LFS_SRC
wget http://ftp.gnu.org/gnu/m4/m4-1.4.18.tar.xz
cd $LFS_BLD
tar xvf $LFS_SRC/m4-1.4.18.tar.xz
cd m4-1.4.18
mkdir build
cd build
../configure --prefix=/usr --host=$LFS_TGT --build=$(build-aux/config.guess)
make -j8
make DESTDIR=$LFS install
```

### ncurses

source: ftp://ftp.invisible-island.net/ncurses/ncurses-6.2.tar.gz

```
../configure --prefix=/usr   --host=$LFS_TGT  --build=$(./config.guess)  --mandir=/usr/share/man  --with-manpage-format=normal  --with-shared   --without-debug           --without-ada  --without-normal   --enable-widec --disable-stripping 
make -j16
make DESTDIR=$LFS  install
echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so
mv -v $LFS/usr/lib/libncursesw.so.6* $LFS/lib
ln -sfv ../../lib/$(readlink $LFS/usr/lib/libncursesw.so) $LFS/usr/lib/libncursesw.so
```

### bash

source: http://ftp.gnu.org/gnu/bash/bash-5.0.tar.gz

need to remove all `examples` in configure and Makefile.in

```
./configure --prefix=/usr                   \
            --build=$(support/config.guess) \
            --host=$LFS_TGT                 \
            --without-bash-malloc
make -j8
make DESTDIR=$LFS install
mkdir $LFS/bin/
mv $LFS/usr/bin/bash $LFS/bin/bash
ln -sv bash $LFS/bin/sh
```

At this point, install `qemu-user-static` run `sudo chroot $LFS` enter target, success.

### coreutils

source: http://mirrors.ustc.edu.cn/gnu/coreutils/coreutils-8.32.tar.xz

need apply patches from debian bullseys


```
./configure --prefix=/usr                     \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --enable-install-program=hostname \
            --enable-no-install-program=kill,uptime
make -j8
make DESTDIR=$LFS install
mkdir $LFS/usr/sbin
mv -v $LFS/usr/bin/chroot                                     $LFS/usr/sbin
```

### diffutils

source: http://mirrors.ustc.edu.cn/gnu/diffutils/diffutils-3.7.tar.xz

```
./configure --prefix=/usr --host=$LFS_TGT
make -j8
make DESTDIR=$LFS install
```

### file

source: https://mirrors.tuna.tsinghua.edu.cn/debian/pool/main/f/file/file_5.39.orig.tar.gz

***fail to build, try to skip***

### find

source: http://mirrors.ustc.edu.cn/gnu/findutils/findutils-4.7.0.tar.xz

```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make 
make DESTDIR=$LFS install
```

### gawk

source:http://mirrors.ustc.edu.cn/gnu/gawk/gawk-5.1.0.tar.xz

### status update

 目前停在交叉编译LLVM。可能要研究一段时间了。
