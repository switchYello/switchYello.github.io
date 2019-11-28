---
layout:     post
title:      "netty buttyBuf的三种复制方式"
subtitle:   ""
date:       2019-09-17
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - netty
---


# netty buttyBuf的三种复制方式

#### copy     物理复制，复制数据构成新buf，且与原buf无关，大小等于数据长度
#### duplicate 逻辑复制，复制原buf的索引构成新buf，和原数据一模一样
#### slice  切片，获取原buf指定范围的映射，大小等于切片的数据长度

```
	@Test
	public void test2() {
		ByteBuf b = getBuf();
		System.out.println("原始数据:" + b);
		//原始数据:UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeHeapByteBuf(ridx: 4, widx: 12, cap: 1024)


		b = getBuf();
		System.out.println("copy:" + b.copy());
		//copy:UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeHeapByteBuf(ridx: 0, widx: 8, cap: 8)


		b = getBuf();
		System.out.println("duplicate：" + b.duplicate());
		//duplicate：UnpooledDuplicatedByteBuf(ridx: 4, widx: 12, cap: 1024, unwrapped: UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeHeapByteBuf(ridx: 4, widx: 12, cap: 1024))


		b = getBuf();
		System.out.println("slice;" + b.slice());
		//slice;UnpooledSlicedByteBuf(ridx: 0, widx: 8, cap: 8/8, unwrapped: UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeHeapByteBuf(ridx: 4, widx: 12, cap: 1024))

	}

```

```
	//返回一个总长度为1024，数据长度为12，读索引为4，写索引为12的bytebuf
	public static ByteBuf getBuf() {
		ByteBuf b = Unpooled.buffer(1024);
		b.writeInt(1);
		b.writeInt(2);
		b.writeInt(3);
		b.readInt();
		return b;
	}


```