---
layout:     post
title:      "vertx buffer类"
subtitle:   ""
date:       2019-08-05
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - vertx
---


# vertx buffer类
	

## Buffer类

	vertx的Buffer接口实现类 BufferImpl 对netty的ByteBuf进行封装，隐藏了复杂的读写指针操作，改写了一些基本操作进行改写，如
	
		read 操作  -> 移除了
		write 操作 -> append
		get ->  get
		set -> set	
		copy -> copy
		slice-> slice
		
	当然方便操作的同时带来的是一点点性能损失。封装的ByteBuf都是Unpoold的，享受不到ByteBuf池带来的好处。

## 不同类型变量的内存占用
	byte 1字节
	short 2字节
	medium 3字节
	int 4字节
	long 8字节
	double 8字节

	其中
	appendIntLE 表示将 int 的bit反着插入buffer中，
	appendUnsignedInt 表示插入无符号位的int，将int符号位当成普通bit使用	
	这样getUnsignedInt就必须以long类型来接受了。
	
	

## 1.创建buffer
	
	创建buffer 由Buffer接口的静态方法实现。	有以下几个

	构造工厂方法：
	Buffer.buffer()；  默认大小为 integer.max的空buffer
	static Buffer buffer(int initialSizeHint)； 指定大小的buffer
	static Buffer buffer(String string)； 将string以u8的编码转成byte[] 再调用下面的构造方法
	static Buffer buffer(String string, String enc)；将string以指定编码转成byte[]，再调用下面的构造方法
	static Buffer buffer(byte[] bytes)； 默认integer.max的buffer，写入指定数据
	static Buffer buffer(ByteBuf byteBuf)； 从netty的bufferBuf中拷贝数据，构造新的buffer

## 2.拷贝和切片

 ### 2.1 拷贝
	
	拷贝函数，底层调用netty的ByteBuf的copy方法，复制一个ByteBuf ，再new一个Buffer。
	新建的Buffer 和原来的没有关系，只是写指针相同，大小相同，内容相同。

```java
 public Buffer copy() {
    return new BufferImpl(buffer.copy());
  }
```


### 2.2 切片
	
	切片函数，返回一个原buffer的切片，切片buffer的大小等于切片范围， 底层和原buffer是同一个ButeBuf，
	因为切片的大小在调用切片函数时，就确定了，所以无法在对切片返回的buffer进行append操作，
	但可以修改它，这样原buffer也会被修改。
	
``` 底层实现

 public Buffer slice() {
    return new BufferImpl(buffer.slice());
  }


```



``` java


	public static void main(String[] args) {

		Buffer buffer = Buffer.buffer("abc");
		System.out.println("新创建的:" + buffer); //此时底层Bytebuf大小为Integer.max，内容为abc

		Buffer copy = buffer.slice(1, 3); //创建切片， 此切片的大小为2 内容为 bc
		System.out.println("切片:" + copy);

		copy.setString(1, "d"); //修改切片第1位，也就是原Buffer的第二位，为d此时切片大小为2 ，内容为bd
		System.out.println("修改后的原buffer:" + buffer.toString());  //原buffer的内容也跟着变成abd了。
		System.out.println("修改后的切片:" + copy); //切片变成 bd

		copy.appendString("e"); //这句会报错，因为切片的大小被固定为2，只能修改无法再添加

	}


```

	当然一般用切片只是为了提高性能，不要原buffer 和 切片都进行修改，不然容易把自己搞混了，还有获取的切片大小固定，不能再添加内容了，不然会报错。