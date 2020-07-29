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

### abc

基础的io写操作.

    type Writer interface {
      Write(p []byte) (n int, err error)
    }

分析

- 将切片数据p写道相关的流中,写的字节数是len(p)
  - 返回值n表示实际写的字节数，如果n小于len(p),err就是non-nil
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
  - whence定义了三个值(SeekStart/SeekCurrent/SeekEnd)
  - 第一个返回值是基于文档头的偏移量
- 偏移到文件头之前,是非法的,err就是non-nil
  - 偏移位置是任何正整数都是ok的
- 实现上
  - 偏移位置是任何正整数是不合逻辑的
  - 具体要表现出什么行为,和实现相关

### io.LimitReader分析

这属于io包第二部分的一些扩展,属于功能性的,
LimitReader就属于功能性的结构体.

    func LimitReader(r Reader, n int64) Reader { return &LimitedReader{r, n} }

    type LimitedReader struct {
      R Reader // underlying reader
      N int64  // max bytes remaining
    }

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

从名字LimitedReader可以看出是一个实现了io.Reader接口的类型,
从源码上可以看出LimitedReader是限制了读的最大字节数.
当超过阀值时,返回的是(0,EOF).

其次,之前提到过的"Read使用指针接收者,writer使用值接收者",
在这里也有体现.

### io.SectionReader分析

io包第二部分的功能性扩展.

    func NewSectionReader(r ReaderAt, off int64, n int64) *SectionReader {
      return &SectionReader{r, off, off, off + n}
    }

    type SectionReader struct {
      r     ReaderAt
      base  int64
      off   int64
      limit int64
    }

同样是功能型的结构体,SectionReader和LimitReader是有很大不一样的.

- LimitReader的构造函数不是Newxxx
  - 作者的意图应该是构造一个满足Reader的对象
  - 而Newxxx应该是用于构造新类型,NewSectionReader返回的就不是Reader类型
- LimitedReader结构体的字段是暴露的
  - 那作者的意图是构造之后,可以走类型断言,通过字段来查属性
  - SectionReader的字段是不暴露的,说明访问全都得通过方法来

下面就将目光放在SectionReader身上.
构造函数的第一个参数类型是ReaderAt,接口类型.

    type ReaderAt interface {
      ReadAt(p []byte, off int64) (n int, err error)
    }

方法ReadAt()相比Read来说,参数中了一个off,
表示先偏移off字节数再读len(p)的字节数到切片.
下面来看下ReaderAt接口的信息:

- 接口的功能是从输入源的偏移off处开始读
- 如果没有读到len(p),返回non-nil错误
  - 在错误中描述为啥少读了,这点上比Reader接口更加严格
- 如果ReadAt没有读到len(p)字节数,要么阻塞等待,要么返回error
  - 这点上是和Reader完全不一样,Reader不会阻塞
- 就算ReadAt读了len(p)字节数,err也有可能是EOF,其他情况是nil
- ReadAt不受seek offset(头/当前位置/结尾)的影响,也不影响seek因数
- 并行读是ok的
- 实现上,不得存p的引用

从这里看,ReadAt是对Read的一种补充,她们的相同点是调用的姿势:
先处理返回值n,后处理err.

回头看看SectionReader的构造函数.因为结构体的字段是不暴露的,
所以在其他包中只能通过构造函数NewSectionReader来正确构造.
从文档上可以看出,SectionReader对Reader有两个扩展:
添加了偏移量;添加了最大读取的字节数.

先看对Reader的实现:

    func (s *SectionReader) Read(p []byte) (n int, err error) {
      if s.off >= s.limit {
        return 0, EOF
      }
      if max := s.limit - s.off; int64(len(p)) > max {
        p = p[0:max]
      }
      n, err = s.r.ReadAt(p, s.off)
      s.off += int64(n)
      return
    }

如果字节数已经超过了,返回(0,EOF),
其他的就是调用构造函数第一个参数ReaderAt接口变量来读.
因为ReaderAt的方法ReadAt是并发安全的,
所以SectionReader.Read也是并发安全的.

SectionReader除了实现了Reader,还实现了Seeker:

    func (s *SectionReader) Seek(offset int64, whence int) (int64, error) {
      switch whence {
      default:
        return 0, errWhence
      case SeekStart:
        offset += s.base
      case SeekCurrent:
        offset += s.off
      case SeekEnd:
        offset += s.limit
      }
      if offset < s.base {
        return 0, errOffset
      }
      s.off = offset
      return offset - s.base, nil
    }

这个实现是非常有意思的.我们先看两个地方,再来讨论这个实现.

实现Reader时用的时ReadAt方法,这个方法和普通的Read对偏移的副作用是不一样的,
ReadAt完全不影响偏移的那些因子,而普通的Read/Seek是会更新偏移因子的,
为什么叫偏移因子不叫偏移变量,因为这些因子是由io操作自己底层维护的,
这也是为啥SectionReader结构体有(base/off/limit)三个字段的原因,
因为这三个字段就是用来模拟io底层的偏移因子的.

我们要看第一个地方就是结构体中的这三个变量.
base表示文件头,limit表示文件尾,off表示当前的偏移量(或者叫当前游标).
初始化看构造函数的设置,Read()/Seek()都有对偏移的更新.

第二个地方是Seeker接口,Seeker是通过两个参数来更新偏移的,
而且对偏移的一些处理也是有规定的:不能更新到文件头去.
SectionReader对Seeker的实现中,就判断了新的偏移不能在base之前.

明白这两点后,再看SectionReader对Seeker的实现就很简单了.
过程就不多了,只说两个细节:

- 第一个返回值是基于文件头的偏移,SectionReader在这点上也做的很好
- switch的default可以提前,特别是处理一些错误时

接下来看看对ReaderAt的实现:

    func (s *SectionReader) ReadAt(p []byte, off int64) (n int, err error) {
      if off < 0 || off >= s.limit-s.base {
        return 0, EOF
      }
      off += s.base
      if max := s.limit - off; int64(len(p)) > max {
        p = p[0:max]
        n, err = s.r.ReadAt(p, off)
        if err == nil {
          err = EOF
        }
        return n, err
      }
      return s.r.ReadAt(p, off)
    }

ReaderAt接口的读是不受偏移因子的影响的,也不影响偏移因子,
ReaderAt只受io文件的头和尾的影响,
SectionReader是用base和limit来模拟头和尾的.

SectionReader.ReadAt()对于要读的数据不足len(p)的情况,
ReaderAt对这种情况的规定是阻塞等或返回error,
这里是通过一次正常的读,然后修改返回值err来实现,
当然这里如果出现其他错误,也是可以处理的.

这里不得不佩服作者在给出ReadAt接口的定义后,
又利用模拟偏移的SectionReader来完美实现了ReaderAt.

### io.teeReader分析

这是第三个功能性结构体.

从名字就很有启示性,tee用于改变流向.

    func TeeReader(r Reader, w Writer) Reader {
      return &teeReader{r, w}
    }

    type teeReader struct {
      r Reader
      w Writer
    }

    func (t *teeReader) Read(p []byte) (n int, err error) {
      n, err = t.r.Read(p)
      if n > 0 {
        if n, err := t.w.Write(p[:n]); err != nil {
          return n, err
        }
      }
      return
    }

先说一下文档上的介绍,再分析细节.

teeReader实现的是Reader接口,处理逻辑是:
从一个Reader读,写到一个Writer中.
如果出现error,都作为读错误来返回.
从源码上看,就是一个纯粹的将读到的数据写到Writer中.

细节分析:

- 构造函数不是Newxxx,因为不是构造一个新类型,而是Reader
- 同时结构体teeReader没有暴露
  - 意味着作者不希望调用方走类型断言,通过字段来获取信息
  - 完美地将teeReader的相关信息隐藏起来,这是封装
- 当多个操作都可能出现错误时,以其中一种为主

不管是LimitReader还是SectionReader/teeReader,共同点是结构体,
还有对io接口的使用:带名字段而不是内嵌.

另外对构造函数的额命名也归纳一下:

- 如果是构造新类型,用Newxxx
- 如果是返回接口类型,用和实现类型接近的名字

## io.CopyN

这是功能性函数.采用层层分析的方式,最后再做总结.

    func CopyN(dst Writer, src Reader, n int64) (written int64, err error) {
      written, err = Copy(dst, LimitReader(src, n))
      if written == n {
        return n, nil
      }
      if written < n && err == nil {
        // src stopped early; must have been EOF.
        err = EOF
      }
      return
    }

文档描述:

- 从src拷贝n字节数的内容到dst
  - 第一个返回值是具体拷贝的字节数
  - 第二个返回值err特指最先出现的错误
  - 当err为nil时,具体拷贝的字节数和参数指定的字节数n是一致的
- 如果dst(Writer)实现了ReaderFrom,内部实现会用到ReaderFrom

还好先分析的功能性结构体,所以又可以有一些猜测.
我们分析了三个功能性结构体:LimitReader扩展了读到的最大字节数,
SectionReader除了读的最大字节数,还扩展了读哪一块,
最后的teeReader扩展了Reader和Writer的连接枢纽.
CopyN除了扩展连接枢纽,还扩展了最大字节数.
所以CopyN里用到了LimitReader是很自然的事.

    func Copy(dst Writer, src Reader) (written int64, err error) {
      return copyBuffer(dst, src, nil)
    }

文档描述:

## 眼前一亮的写法

## 可能的应用场景

LimitReader可用在对读的最大字节数有限制的场景;
SectionReader可用在即对最大字节数有限制,又对读的区域有限制的场景.
