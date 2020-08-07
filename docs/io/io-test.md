# io 测试

## 测试的安排

下面针对copy系列说的扩展是指:Writer/Reader的实现者,
同时实现了WriterTo/ReaderFrom接口,
或是实现了Writer/Reader的具体类型添加了具体的场景.

第一,针对Copy/CopyBuffer/CopyN的测试,
主要集中在Writer/Reader是否有扩展的情况.
这一部分被成为"simple test",验证copy系列是否能正常工作.

这部分还分成了几个阶段来验证:

- 无扩展,验证所有的返回值是否符合预期
- 带扩展,验证是否调用了扩展逻辑
  - Writer/Reader都带扩展,验证扩展优先级是否按预期执行
- 验证读写逻辑错误的优先级
- CopyN针对带扩展/不带扩展的情况
- CopyN针对拷贝大小的基准测试
- CopyN针对EOF的情况

第二,针对ReadAtLeast的测试,
主要集中在error/EOF/常规的功能测试.

第三,针对功能性结构体的测试.

## 具体分析

### copy系列,无扩展时的测试

bytes.Buffer实现了Reader/Writer/ReaderFrom/WriterTo,很适合做测试.

    type Buffer struct {
      bytes.Buffer
      ReaderFrom
      WriterTo
    }

这种写法是利用语言的机制,屏蔽了bytes.Buffer的两个接口.

    func main() {
      var b io.Reader = &Buffer{}
      if _, ok := b.(io.ReaderFrom); !ok {
        fmt.Println("not a ReaderFrom")
      }
    }

通过类型断言可以看出,如果Buffer类型不另外实现ReadFrom()方法,
那么她就不是属于ReaderFrom接口的实现者,即使结构体中内嵌了接口类型.

上面的写法是通过内嵌接口不实现来屏蔽部分行为.
还有一种interface{},任何类型都实现了这个接口,
其底层处理手法应该是比较方法集的包含关系.
如果将空接口内嵌到Buffer,最后发现Buffer还是属于空接口的实现,
这里算是一个特例吧,毕竟实现与否看方法集的包含关系,
而内嵌接口是利用了语言的冲突机制,隐去了级别较低的访问,
从而造成了屏蔽效果.

dlv test, b io_test.TestCopy, c, 可调试第一个测试函数.

- TestCopy
  - 测试Copy,Writer/Reader无扩展
- TestCopyNegative
  - 测试Copy,Reader扩展成LimitedReader
    - 测试LimitedReader的最大可读字节数是负数的情况
  - 测试CopyN,也是测试最大可读字节数为负数的情况
- TestCopyBuffer
  - 测试CopyBuffer,设定中间缓冲区,只是决定了每次读写的颗粒度
- TestCopyBufferNil
  - 测试CopyBuffer,不指定缓冲区,会自己生产中间缓冲区
- TestCopyReadFrom/TestCopyWriteTo
  - 测试Copy系列函数,最后是测试对扩展的处理
  - 这也算是代码复用,以后要修改只修改一处

### 扩展逻辑的优先级,用测试结果来说明

源码中用writeToChecker结构体来验证,具体做法是在WriteTo中加了标志位.

之前的测试已经验证了会走扩展逻辑,io包的源码指定了会优先走Writer扩展,
所以在TestCopyPriority中,只需要检查标志位是否是true.

### 读写错误优先级

io包源码中写错误的优先级高.
TestCopyReadErrWriteErr配合两个特别设计的Writer/Reader,测试Copy()的返回值,
这种写法注重于测试异常流程.

### CopyN

CopyN在Copy的基础上做了一个封装,限定了Reader读的最大字节数.

测试例子中提供了3个测试函数:

- TestCopyN, 常规测试
- TestCopyNReadFrom, 测试Writer对ReadFrom的扩展
- TestCopyNWriteTo, 测试Reader对WriteTo的扩展

dlv单步之后,发现TestCopyNWriteTo没有走扩展流程,具体原因如下:
测试函数传入的Reader是bytes.Buffer,
在CopyN中将bytes.Buffer封装到LimitedReader中了,但LimitedReader
是没有实现WriterTo接口的,所以不会州扩展流程.

如果要修改,可copyBuffer中的扩展逻辑下面添加如下代码:

    if l,ok := src.(*LimitedReader); ok {
      if wt,ok := src.R.(WriterTo); ok {
        return wt.WriteTo(dst)
      }
    }

这里仅仅是针对CopyN的改造,但放在了公共函数copyBuffer上面,
肯定是不合适的,或许可以放在LimitedReader上,问题是:
万一LimitedReader的R没有实现WriterTo,就会影响到copyBuffer
的正常流程,综合考虑,还是应该修改公共函数copyBuffer.

### CopyN的基准测试

这里有很多新东西,可以学习.

    func BenchmarkCopyNSmall(b *testing.B) {
      bs := bytes.Repeat([]byte{0}, 512+1)
      rd := bytes.NewReader(bs)
      buf := new(Buffer)
      b.ResetTimer()

      for i := 0; i < b.N; i++ {
        CopyN(buf, rd, 512)
        rd.Reset(bs)
      }
    }

首先是创建一个bytes.Buffer,Repeat的实现很有趣,后面也会分析到,
b.ResetTimer()消除了准备阶段对测试的影响,

基准测试一定要有对比,这里对比的因素是CopyN中每次读写的颗粒度.
Benchmark测试叫基准测试,也叫性能测试,后面都用性能测试来表述.

    ➜  io go test -bench=. -run=^$ -benchmem
    goos: linux
    goarch: amd64
    pkg: io
    BenchmarkCopyNSmall-4   2238950   8258 ns/op    2462 B/op   2 allocs/op
    BenchmarkCopyNLarge-4   1300      2222590 ns/op 135716 B/op 2 allocs/op
    PASS
    ok  io  23.226s

可以看出,读写字节数越大性能越差,(废话?).
Large读写的字节数是Small的64倍,但性能却差了三个数量级.
所以说宁可多调CopyN几次,也不要让CopyN来一次大的.

### CopyN对各种EOF的处理

分别覆盖了以下场景:

- Reader扩展,Writer不扩展,读缓冲3字节,最大读写数是3
  - 预期,写3字节,err为nil
- Reader扩展,Writer不扩展,读缓冲3字节,最大读写数是4
  - 预期,写3字节,err为EOF
- Reader扩展,Writer扩展,读缓冲3字节,最大读写数是3
  - 预期,写3字节,err为nil
- Reader扩展,Writer扩展,读缓冲3字节,最大读写数是4
  - 预期,写3字节,err为EOF
- Reader报错,Writer扩展,最大读写数是5
  - 预期,写5字节,err为nil
- Reader报错,Writer不扩展,最大读写数是5
  - 预期,写5字节,err为nil

前面4个测试函数好了解,后面两个为啥也是正确的.
CopyN内部,Copy返回的是(5,error),5正好满足CopyN的要求,
所以错误会被忽略掉.
倒数第二个测试函数走的是strings.Buffer.ReadFrom;倒数第一个走的是常规逻辑.

这里有一点值得思考:LimitedReader.N到底有哪些意思:

对于非CopyN的拷贝函数:

- 如果Writer/Reader有扩展,N具体是多少就没关系了
- 如果走常规逻辑,N只表示最大读写数
  - 这里的条件非常严:
    - 第一需要非CopyN系列函数
    - 第二需要Reader要支持扩展 LimitedReader

对于CopyN:

- N表示最大读写数
  - 此时不管Writer/Reader是否还有其他扩展

再次总结:CopyN的N和Reader的LimitedReader扩展,都限定了最大读写的字节数.

### ReadAtLeast系列

有三个测试函数,底层调用的都是testReadAtLeast,
ReadAtLeast函数是从Reader中最少读xx字节的数据到制定切片.

这个测试的手法,是将ReadAtLeast的所有边界都包含了.
对bytes.Buffer的实现细节不清楚,所以不好分析具体的流程,待后续对bytes包的分析.

### TeeReader的测试

TeeReader实现了Reader,具体行为是从Reader读数据到buf切片,后再写回到Writer,
TestTeeReader前一部分是常规的测试,后一部分,Writer是通过Pipe()生成的.

Pipe()是io包的其余部分,后面分析.

### SectionReader

SectionReader模拟了一个文件读,测试ReadAt时用了表格测试.
实现ReaderAt接口的是strings.Reader类型.

    struct {
      data   string
      off    int
      n      int
      bufLen int
      at     int
      exp    string
      err    error
    }

前面3个data/off/n用于描述SectionReader结构体,bufLen描述ReadAt()第一个参数,
也就是读到那个缓冲(这里是字节切片), at表示读时的偏移,exp/err是读之后的结果.

表格测试,主要是变化前5个参数,来预测结果.测试函数中也包含了各种场景.

对于seek:

- for range的容器,是字面量来体现的
- 测试不再是具体的预期作对比,而是和另一个测试对象作对比
  - 毕竟SectionReader是模拟文件读,而bytes.Reader就是一个优秀的文件读例子

对与Size,也是采用和另一个测试对象作对比.

## 测试遵循的细节

- 针对非接口进行测试
- 一个测试函数只针对一个主题
  - 正常流程
  - 错误参数
  - 边界
  - 返回值
  - 逻辑分支,包括先后顺序
  - 入参的不同组合
- 对调用层次最高的那个函数做性能测试
- 主动构建异常对象(读写错误,或部分错误)来测试各种场景
- 多边界的推荐使用表格测试(ReadAtLeast也适合用表格测试)
- 函数行为是否ok,可通过判断多个测试点来判断
- 对于一些模仿性函数,可将预期结果和被模仿函数的执行结果进行对比

## 令人眼前一亮的写法

测试很少对接口进行测试,之前一直以为接口是核心,
应该会有大篇幅来测试Writer/Reader接口,实际上不是.
大部分篇幅都是针对暴露的函数,少部分针对暴露的功能性结构体.

## 如何写测试

对于异常流程或预期的返回值,可单独用一个测试函数来测试.

对函数或方法的约束可用一个功能函数(非导出,意味着可能是功能函数),
再针对不同的场景,使用多个测试函数来调用.

表格测试,包含了被测试函数的入参,最后的结果(可能是返回值可能是引用入参),错误等.
对于有多个边界条件的函数,应该引入表格测试.

对于模拟性的实现,针对其的测试应该是直接和被模拟对象的执行结果对比.

测试永远不是为了覆盖率,不是做完单点单元测试就可以了,
应该要有设计.测试是确保一个场景范围,在这个范围内,
包的实现的行为是符合预期,且可靠的.

如果说包的设计是为了实现某个功能,那测试的设计就是为了保证正确性,
从Go的源码中看出,这两者应该是同时进行,同时发展.
两者都是先有了设计,再用代码实现的.
