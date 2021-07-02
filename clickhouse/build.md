# clickhouse build

> 本文介绍一下 `clickhouse` 整个的build过程

1. `git clone --recursive https://gitee.com/mirrors/clickhouse.git` 这个已经是下载好的第三包的中国镜像(每天一同步)
2. 可能会出现这个问题；需要手动改一下 `objcopy` 路径
```diff
brew install binutils
mdfind -name objcopy
// /usr/local/Cellar/binutils/2.35/bin/objcopy
// 然后改变路径即可
- find_program (OBJCOPY_PATH NAMES "llvm-objcopy" "llvm-objcopy-12" "llvm-objcopy-11" "llvm-objcopy-10" "llvm-objcopy-9" "llvm-objcopy-8" "objcopy")
+ find_program (OBJCOPY_PATH NAMES "llvm-objcopy" "llvm-objcopy-12" "llvm-objcopy-11" "llvm-objcopy-10" "llvm-objcopy-9" "llvm-objcopy-8" "objcopy" PATHS "/usr/local/Cellar/binutils/2.36.1/bin/")
```
3. `git submodule update --init --recursive` 即可(这一步如果在第一步下载好的情况下，其实可以不进行的)
4. 按照下面依次进行，采用的是`Debug`：
```shell
mkdir build
cd build
cmake -DCMAKE_C_COMPILER=$(brew --prefix llvm)/bin/clang -DCMAKE_CXX_COMPILER=$(brew --prefix llvm)/bin/clang++ -DCMAKE_BUILD_TYPE=Debug ..
cmake --build . --config Debug
cd ..
```

## calltree & cpptree

- `calltree` -> 观测 `cpp` 代码的调用栈
- `cpptree` -> 观测 `cpp class` 的类对象属性，主要是找继承关系

这个是学习大型 `cpp` 项目最有力的工具，可以说没有之一。当然还有需要你去 `gdb` debug。

### 用例

```cpp
cpptree.pl "IStorage" "Merge" 0 3
// 观察 `IStorage` 这个顶层接口中 `MergeTree` 的继承关系
// 很快也能发现：MergeTree -> MergeTreeData，插入的数据也是在 MergeTreeData 中准备的
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92201e7d0c7d49eaba469e09f4fae19e~tplv-k3u1fbpfcp-watermark.image)

```cpp
calltree.pl "processInsertQuery" "" 1 1 3
// processInsertQuery 是要从 TCP 控制insert的入口
// 我们发现上面调用他的是 `KeeperTCPHandler::run`
// KeeperTCPHandler 这个就是 Server::main 中被调用的
// 那么从 main -> insert 的路径就通了
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7224b96701c4316881b827e8d5787e7~tplv-k3u1fbpfcp-watermark.image)

此外还有很多使用方式。可以参考下面的资料。。。

[cpptree.pl](https://github.com/satanson/cpp_etudes/blob/master/cpptree.pl) <br />
[calltree.pl](https://github.com/satanson/cpp_etudes/blob/master/calltree.pl) <br />

## 参考资料

Build ClickHouse on MacOSX [https://clickhouse.tech/docs/en/development/build-osx/](https://clickhouse.tech/docs/en/development/build-osx/) <br />
objcopy解决方式 [https://github.com/ClickHouse/ClickHouse/issues/13597](https://github.com/ClickHouse/ClickHouse/issues/13597) <br />
