# Hyperscan简介

## 概述
Hyperscan是一款可以同时匹配多个正则表达式的高性能正则引擎。

## 依赖
|依赖|版本|说明|
|:-:|:-:|:-:|
|Cmake|>=2.8.1||
|Ragel|6.9||
|Python|2.7||
|Boost|>=1.57|只要头文件|
|Pcap|>=0.8|只有示例代码需要|

## 编译安装
1. 下载Hyperscan

		cd where-you-store-hyperscan-source
		git clone git://github.com/intel/hyperscan

2. 配置Hyperscan

		cd hyperscan-source-dir
		mkdir build-dir
		cd build-dir

3. 构建Hyperscan

		cmake .. - 构建makefile
		make -j jobs - 并行编译

4. 检查Hyperscan

		bin/unit-hyperscan     - 运行Hyperscan单元测试

5. 安装Hyperscan

		make install   -  安装 .so .a .h 文件到系统目录

## 匹配模式
根据使用场景的不同，hyperscan有**流模式**、**块模式**和**向量模式**3种不同的匹配方式。

1. 流模式

		目标数据是一条连续的流，正则可以跨越流中的多个块。
        在流模式下，每个流需要一块内存保存扫描状态。

2. 块模式

		目标数据是一个离散的连续块，可以在一次函数调用中将其全部扫描完毕，不需要保留状态。

3. 向量模式

		目标数据是一个连续的数据块，不需要保留状态。

## 使用场景

1. 流模式

		对于分包的HTTP响应，每来一个包扫描一次，不需要存储数据包，但是要针对这条流存储扫描状态。

2. 块模式

		对于分包的HTTP响应，组包后再扫描。

3. 向量模式

		对于分包的HTTP响应，将其存储成数组，然后对数组进行扫描。

摘自 http://intel.github.io/hyperscan/dev-reference/runtime.html

	From the caller’s perspective, vectored mode will produce the same matches as if the set of data blocks were scanned in sequence with a series of streaming mode scans, or copied in sequence into a single block of memory and then scanned in block mode.

