# io

## 包的说明和定位

提供基础的io原语接口.
因为是基于底层操作实现的,所以,如果没有特别说明,都不应该认为是并行安全.

## 接口或结构体的关系

第一部分是io包的核心部分,包括四个接口:
Reader/Writer/Closer/Seeker,分别对应io的读写关闭和偏移.

第一部分的扩展部分,是基于核心4接口的组合:
ReadWriter/ReadCloser/WriterCloser/ReadWriteCloser/
ReadSeeker/WriteSeeker/ReadWriteSeeker.
组合的方式是接口内嵌接口.

到目前为止,包含了常规的11种接口.

第二部分是基于第一部分的11接口,扩展出的一些功能性ds/func.
io包用的最多的还是第一部分的核心接口.

### 详细说明(包含精彩片段欣赏)

## 眼前一亮的写法

## 可能的应用场景
