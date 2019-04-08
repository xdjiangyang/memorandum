# Hyperscan简介

## 概述
Hyperscan是一款可以同时匹配多个正则表达式的高性能正则引擎。

## 安装
1. 下载Hyperscan
		cd where-you-store-hyperscan-source
		git clone git://github.com/intel/hyperscan

2. 配置Hyperscan
		cd where-you-store-hyperscan-source
		mkdir build-di
		cd build-dir

3. 构建Hyperscan
		cmake --build .. - 构建makefile
		make -j jobs - 并行编译

4. 检查Hyperscan
		bin/unit-hyperscan     - 运行Hyperscan单元测试

5. 安装Hyperscan
		make install   -  安装 .so .a .h 文件到系统目录
