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

## 安装
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
