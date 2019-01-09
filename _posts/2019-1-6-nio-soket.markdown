---
layout:     post
title:      "NIO Socket (一)"
subtitle:   " \"nio和socket\""
date:       2019-1-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - nio
    - socket
---

> “Come on!. ”

## 缓冲区API   
```
    ByteBuffer
    CharBuffer
    FloutBuffer
    DoubleBuffer
    LongBuffer
    IntBuffer
    ShortBuffer
    
    父类 Buffer 且为抽象类 用warp()方法放入缓冲区
    ByteBuffer bu = ByteBuffer.warp(new byte[]{1,2,3});
```
ps: (没BooleanBuffer,StringBuffer)

### 四大核心技术
```
    - capacity  缓冲区容量
    - limit  限制
    - position  位置
    - mark  标记
```
关系如下

0<= Mark <= position <= limit <= capacity

``` java
clear(){ //恢复默认
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}

flip(){ //缩小limit范围 h 
    limit = position;
    position = 0;
    mark = -1;
    return this;
}

rewind(){ //重绕缓冲区并将位置设为0,标识丢弃(-1),limit不变
    position = 0;
    mark = -1;
    return this;
}
```

1:缓冲区的capacity,limit,position不能为负数
2:position不能大于limit,如果position大于limit时,position代替limit,limit不能大于capacity
3:如果定义了mark,如果limit,position小于mark时 mark值被丢弃 即为-1
4:如果未定义mark,调reset()方法是 抛出 InvalidMarkException的一场
5:当limit,position值一样时,position位置写入数据时 会抛出异常 
___
判断是否制度 boolean isReadOnly()
___
直接缓冲区 boolean isDirect()
非直接缓冲区储存数据时会先把数据放入jvm中的中间缓冲区中,再写入磁盘,使用直接缓冲区,省略了中间缓冲区的步骤,提高了数据的吞吐量,提高内存占用率
___
判断是否有底层数组实现 final boolean hasArray()
___
获取偏移量 final int arrayOffset()
一般情况为0
___
创建缓冲区
创建非直接缓冲区 allocate(int capacity) 缓冲区类型为 HeapByteBuffer
创建直接缓冲区 allocateDirect(int capacity) 缓冲区类型为 DiretByteBuffer
直接缓冲区效率更高
___
获取剩余空间大小  final int remaining() 
ps : limit - position
___
判断当前位置与现值之间是否有剩余元素 final boolean hasRemaining()
___
warp的数据处理 
warp(byte[] array) 将数组包装到缓冲区中 capacity,limit为array.length,position = 0 ,底层实现为数组,arrayOffset = 0;
wrap(byte[] array,int offset, int length)
offset为0,底层实现为传入数组
参数说明:
array 字节数组
offset position值 非负不大与array.length
length limit = offset + length; 非负且不大与array.length - offset
___
abstract Buffer put(byte[] b) 
将数组写入当前位置 且 position递增
abstract byte get() 
读取当前位置的数据 且position递增
___
put(byte[] src,int offset, int length)相对批量put方法 
参数说明: 
src 字节数组
offset 要读取的第一个字节在数组中的偏移量(不是缓冲区的偏移)非负且大于 src.length
length 给定数组读取的数量 非负大于src.length-offset

##### FAQ:
1: offset + length > src.length 会抛出 INdexOutBoundsException
2: length > buffer.remaining 会抛出 BufferOverFlowException

get(byte[] dst, int offset, int length)
相对批量 get方法
参数说明:
dst 同上
offset 同上 
length 写入给给定数组的最大字节数量 非负大于src.length-offset

##### FAQ:
1: offset + length > dst.length 会抛出 INdexOutBoundsException
2: length > buffer.remaining 会抛出 **BufferUnderFlowException**

##### ps:
``` java
    byte[] b1 = {1,2,3,4,5,6,7}
    byte[] b2 = {55,66,77,88}
    ByteBuffer byteBuffer = ByteBuffer.allocate(10);
    byteBuffer.put(b1);
    byteBuffer.position(2);
    byteBuffer.put(b2,1,3);

    //byteBuffer: 1,2,66,77,88,6,7,8,0,0

    byteBuffer.position(1);
    byte[] byteArrayOut = new byte[byteBuffer.capacity()];
    byteBuffer.get(byteArrayOut,3,4);

    //byteBuffer: 0,0,0,2,66,77,88,0,0,0
```
___
put(byte[] src) 将数组写入当前位置
get(byte[] dst) 读取数组到当前位置
##### FAQ:
缓冲区remaining小于数组
put抛出: BufferOverFlowException
get抛出: BufferUnderFlowException
___
put(int index,byte b) 绝对put方法,将字节写入索引指定位置 position位置不变
get(int index) 绝对get方法,读取指定位置字节 position位置不变
___
put(ByteBuffer src) 将给定缓冲区的数据写入缓冲区当前位置,两个缓冲区位置都加给定缓冲区的remaininhg
``` java
    byte[] array1 = {1,2,3,4,5,6,7,8};
    ByteBuffer b1 = ByteBuffer.wrap(array1);
    byte[] array2 = {55,66,77};
    ByteBuffer b2 = ByteBuffer.wrap(array2);
    b1.position(4);
    b2.position(1);
    b1.put(2);
    
    // b1.position: 6
    // b2.position: 3
    // b1: 1,2,3,4,66,77,7,8
```
___
##### putType(),getType
**位置+(基本类型字节数)**

putChar(char value) 相对方法,按照当前字节顺序写入缓冲区当前位置,位置加字节数(2)
putChar(int index, char value) 绝对方法,按照当前字节顺序写入缓冲区指定索引位置,位置加字节数(2)

putDouble(double value)
putDouble(int index,double value)
putFloat(float value)
putFloat(int index,float value)
putInt(int value)
putInt(int index,int value)
putLong(long value)
putLong(int index,long value)
putShort(short value)
putShort(int index,short value)
#### **同char**
___
**slice()** 创建新的缓冲区,从当前缓冲区的位置开始,该缓冲区有独立的位置,限制,标记,位置默认为0,缓冲区性质依照当前缓冲区(直接,只读)
使用slice()方法后arrayOffset()值不为0,如下
``` java
    byte[] array1 = {1,2,3,4,5,6,7,8};
    ByteBuffer b1 = ByteBuffer.wrap(array1);
    b1.position(5);
    ByteBuffer b2 = b1.slice();
    //此时arrayOffset=5
```
___
asCharBuffer() 中文处理
"中文".getBytes("utf-8");
CharBuffer cf = Charset.forName("utf-8").decode(**youByteBuffer**);
___
转换为其他类型缓冲区
asDoubleBuffer()
asFloatBuffer()
asIntBuffer()
asLongBuffer()
asShortBuffer()
缓冲区位置从当前缓冲区位置开始,位置,限制,标记是相互独立的,新缓冲区位置为0,容量和限制为缓冲区剩余字节数的1/(类型字节),状态也由原缓冲区决定(直接缓冲区,是否只读)
___
order() 设置字节顺序 默认为顺序
顺序: ByteOrder.BIG_ENDIAN
逆序: ByteOrder.LITTLE_ENDIAN
因cpu不同,读取顺序不同可从高位或者低位读取,一般字节从中间分开,左边为高位,右边为低位
___
asReadOnlyBuffer() 创建共享此缓冲区内容为只读状态的缓冲区
___
压缩缓冲区 compact()
根绝 opsotion 位置进行压缩
获取位置之后的数据,并且根据位置的值读取缓冲区最后的数据
ps: 1,2,3,4,5 position = 2;
压缩后的值 3,4,5,4,5
___
boolean equals(),int compareTo() 比较两个缓冲区的内容
equals:
1: 是否为自身
2: 是否为ByteBuffer实例
3: remaining是否一样
4: 两个缓冲区position,limit之间的数据是否一样
capacity可以不同

compareTo:
比较两个缓冲区的方法是按照字典顺序比较他们的剩余元素顺序,而不是每个序列在其对应缓冲区的起始位置
1: 判断两个ByteBuffer的范围是从当前ByteBuffer对象的前位置开始,以两个ByteBuffer对象最小的remaining()值结束说明判断的范围是remaining的交集
2: 如果在开始和结束范围之间有一个字节不同,则返回两者的减数
3: 如果开始结束范围内字节都相同返回remaining()的减数
capacity可以不同
___
ByteBuffer duplicate() 复制缓冲区
同使用一个原数组,数组改变,原缓冲区和复制的缓冲区数据都改变
___
对缓冲区的扩容
等于创建新的缓冲区 
allocate(youbyteBuffer.capavity + size) size扩容大小
___
重载
CharBuffer append(char c) 相当于 dst.put(c)
CharBuffer append(CharSequence csq) 相当于 dst.put(csq.toString())  可能没添加整个序列 取决于缓冲区的位置
CharBuffer append(CharSequence csq,int start ,int end) 相当于 dst.put(csq.subSequence(start,end).toString())//截取
___
final char charAt(int index) 读取指定索引位置字符
___
put(String src) 将给定字符串复制到当前位置
int read(CharBuffer target) 将当前缓冲区字符写到指定缓冲区的当前位置
subSequence(int start,int end) 创建此缓冲区的指定序列,和原缓冲区数据共享,位置为原position + start,limit为position + end
___
static CharBuffer wrap(CharSequence csq,int start,int end) 
将字符串顺序包装到缓冲区中,字符串内容长度为csq.length,位置为start,限制end,标记未定义
___
final int length() 获取字符串长度
