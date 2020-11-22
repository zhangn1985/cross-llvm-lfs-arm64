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
 
 > 本次实验使用debian 11作为宿主系统
 > 环境变量
 
 ```
 export LFS=~/llfs
 export LFS_TGT=aarch64-llfs-linux-musl

 ```
 
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

### 编译host LLVM使其成为真正的cross compile



### 环境变量总结
```
export LFS=~/llfs
export LFS_TGT=aarch64-llfs-linux-musl
export XDG_BIN=$HOME/.local/bin
export PATH="$XDG_BIN:$PATH"
export COMMON_FLAGS=" -O2 -pipe --sysroot=${LFS}"
export CFLAGS="${COMMON_FLAGS}"
export CXXFLAGS="${COMMON_FLAGS} -stdlib=libc++"
export LDFLAGS="-fuse-ld=lld -rtlib=compiler-rt -flto=thin"
```
