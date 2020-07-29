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

## 详细说明(包含精彩片段欣赏)

针对核心4接口会重点分析,针对功能性部分会大致过一下.

### Reader接口

只包含一个方法`Read`,接口名叫Reader,完全是符合编码规范的.

Reader接口封装的是基础的io读.

    type Reader interface {
      Read(p []byte) (n int, err error)
    }

Read方法说明:

- 读数据到切片p,长度最多是len(p)
  - 返回值是具体读到的字节数,或任何错误
  - 如果要读的数据长度不足len(p),会返回实际长度，不会等待
- 不管是读中出错,还是遇到文件尾,第一个返回值n会返回实际已读字节数
  - 对于第二个返回值err就有些不同了
    - 可能会返回EOF,也可能返回non-nil
    - 再次调用Read的返回值是确定的: 0,EOF
- 调用者正确的处理返回值姿势应该是这样的
  - 先判断n是否大于0,再判断err
- 对于实现
  - 除非len(p)是0,否则不推荐返回 0,nil
  - 因为0,nil表示什么都没发生
  - 实际处理中，EOF就是EOF，不能用0,nil来代替

额外说明,Read方法的参数是切片,是引用,所以是可以在内部修改值的.
当然是有前提的:实现类型的指针方法集匹配Reader.
值方法集实现Reader接口是没有意义的.
就像LimitedReader对Reader的实现:

    func (l *LimitedReader) Read(p []byte) (n int, err error) {
      if l.N <= 0 {
        return 0, EOF
      }
      if int64(len(p)) > l.N {
        p = p[0:l.N]
      }
      n, err = l.R.Read(p)
      l.N -= int64(n)
      return
    }

### Writer接口

基础的io写操作.

    type Writer interface {
      Write(p []byte) (n int, err error)
    }

- 将切片数据p写道相关的流中,写的字节数是len(p)
  - 返回值n表示实际写的字节数，如果n<len(p),err就是non-nil
- 方法是不会改变切片p的数据，临时性的也不行
- 在实现上
  - 要确保不会修改切片p

额外说明,对Writer的实现,最好是通过值方法集来匹配,
如果非要用指针方法集来匹配,那就一定要注意不能修改切片数据.

### Closer接口

和io的读写类似,Closer封装的是关闭操作.

    type Closer interface {
      Close() error
    }

- 关闭之后,再次调用关闭,行为是未定义的
- 实现上,如果有特别操作,应该在文档上有所体现

### Seeker接口

io的偏移操作封装.

    type Seeker interface {
      Seek(offset int64, whence int) (int64, error)
    }

- 这个偏移是针对下次读或下次写
- 参数offset和whence确定了具体的偏移地址
  - offset定义了三个值(SeekStart/SeekCurrent/SeekEnd)
  - 第一个返回值是基于文档头的偏移量
- 偏移到文件头之前,是非法的,err就是non-nil
  - 偏移位置是任何正整数都是ok的
- 实现上
  - 偏移位置是任何正整数是不合逻辑的
  - 具体要表现出什么行为,和实现相关

## 眼前一亮的写法

## 可能的应用场景
