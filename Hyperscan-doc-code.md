# Hyperscan编码简介

# 流模式

- 编译正则数据库
		hs_compile(...,HS_MODE_STREAM)

- 分配scratch空间
		hs_alloc_scratch()

- 扫描数据
		hs_open_stream() ,hs_scan_stream(),hs_close_stream()

# 块模式

- 编译正则数据库
		hs_compile(...,HS_MODE_BLOCK)

- 分配scratch空间
		hs_alloc_scratch()

- 扫描数据
		hs_scan()

# 向量模式

- 编译正则数据库
		hs_compile(...,HS_MODE_VECTORED)

- 分配scratch空间
		hs_alloc_scratch()

- 扫描数据
		hs_scan_vector()
