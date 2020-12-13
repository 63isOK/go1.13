# encoding/binary

这个包有两个功能:

- 数值和字节序之间的转换
- 可变长的编解码

这个包追求的是简单,而不是高性能,如果场景需要高性能,
尤其是针对大数据结构体,应该考虑encoding/gop或者pb.

## 源码分析

这个包包含两个源文件 binary.go和varint.go,分别承载binary包的两个功能.
那一一来分析.

### reflect包中的常用函数

- reflect.VauleOf()
  - 用参数接口对象的值去初始化一个reflect.Value
- reflect.Indirect()
  - 获取参数reflect.Value实际指向的值
  - 如果参数是指针,就返回指针指向的值;如果不是,就返回参数本身
- reflect.Value.Type()
  - 获取Value的类型reflect.Type
  - relfect.Type是一个接口
- reflect.Type.Kind()
  - 返回reflect.Type对应的reflect.Kind
  - reflect.Kind是一个常量枚举,这个枚举里包含了Go中定义的各种类型
  - reflect.Kind.String()会返回对应类型的字符串
- reflect.Type.Elem()
  - 类型Type的元素类型
  - 下列类型中才会包含元素类型
    - 数组/切片/指针/信道/map

### binary.go

先确定分析顺序.需要看对外暴露的:

- 接口类型ByteOrder,两个实现类型 BigEndian和LittleEndian,分别是大小端
- 3个函数 Read/Write/Size, 类型ByteOrder就作为读写函数的参数

这个文件最重要就是Read/Write函数,这两个函数的参数又使用了ByteOrder,
所以需要先分析大小端字节序,同时Read/Write的实现中还用到了decoder/encoder,
这两个类型都包含了ByteOrder字段.所以分析顺序如下:

- ByteOrder接口类型
- BigEndian/LittleEndian类型
- coder/decoder/encoder
- Read/Write
- Size

ByteOrder接口类型分析:

    // A ByteOrder specifies how to convert byte sequences into
    // 16-, 32-, or 64-bit unsigned integers.
    type ByteOrder interface {
      Uint16([]byte) uint16
      Uint32([]byte) uint32
      Uint64([]byte) uint64
      PutUint16([]byte, uint16)
      PutUint32([]byte, uint32)
      PutUint64([]byte, uint64)
      String() string
    }

这个ByteOrder字节序类型只定义了`[]byte`如何转换为无符号数值.
看接口定义,ByteOrder也定义了无符号数值如何转换成`[]byte`.

我们看下网上对大小端的定义:

- 大端:高字节保存在低地址,低字节保存在高地址
- 小端:高字节保存在高地址,低字节保存在低地址

下面是一个例子:

    var a uint32 = 0x12345678;
    var buf [4]byte;
    // 大端, 符合阅读习惯
    // buf[0] = 0x12
    // buf[1] = 0x34
    // buf[2] = 0x56
    // buf[3] = 0x78
    // 小端, 符合逻辑
    // buf[0] = 0x78
    // buf[1] = 0x56
    // buf[2] = 0x34
    // buf[3] = 0x12

ByteOrder对16/32/64位的数值和`[]byte`做了一个转换.

    // 空结构体,表示只关注方法,不在乎数据
    type littleEndian struct{}

    // 小端模式将`[]byte`转换为uint16
    // 边界检查丢给了编译器
    // 测试发现,其实并不是在编译期将错误抛出来
    // 而是写了一个边界检查,还是在运行时报错
    // issue14808提出了一个解决方法,用以兼容不满足边界检查的场景
    // 小端,低位放低地址,16表示只处理16位
    func (littleEndian) Uint16(b []byte) uint16 {
      _ = b[1] // bounds check hint to compiler; see golang.org/issue/14808
      return uint16(b[0]) | uint16(b[1])<<8
    }

    func (littleEndian) PutUint16(b []byte, v uint16) {
      _ = b[1] // early bounds check to guarantee safety of writes below
      b[0] = byte(v)
      b[1] = byte(v >> 8)
    }

    func (littleEndian) Uint32(b []byte) uint32 {
      _ = b[3] // bounds check hint to compiler; see golang.org/issue/14808
      return uint32(b[0]) | uint32(b[1])<<8 | uint32(b[2])<<16 | uint32(b[3])<<24
    }

    func (littleEndian) PutUint32(b []byte, v uint32) {
      _ = b[3] // early bounds check to guarantee safety of writes below
      b[0] = byte(v)
      b[1] = byte(v >> 8)
      b[2] = byte(v >> 16)
      b[3] = byte(v >> 24)
    }

    func (littleEndian) Uint64(b []byte) uint64 {
      _ = b[7] // bounds check hint to compiler; see golang.org/issue/14808
      return uint64(b[0]) | uint64(b[1])<<8 | uint64(b[2])<<16 |
      uint64(b[3])<<24 | uint64(b[4])<<32 | uint64(b[5])<<40 |
      uint64(b[6])<<48 | uint64(b[7])<<56
    }

    func (littleEndian) PutUint64(b []byte, v uint64) {
      _ = b[7] // early bounds check to guarantee safety of writes below
      b[0] = byte(v)
      b[1] = byte(v >> 8)
      b[2] = byte(v >> 16)
      b[3] = byte(v >> 24)
      b[4] = byte(v >> 32)
      b[5] = byte(v >> 40)
      b[6] = byte(v >> 48)
      b[7] = byte(v >> 56)
    }

    // 仅仅是输出和字节序相关的信息
    func (littleEndian) String() string { return "LittleEndian" }
    func (littleEndian) GoString() string { return "binary.LittleEndian" }

至此小端已经分析完了,大端只是对应不同而已,就不贴代码了.

coder分析:
