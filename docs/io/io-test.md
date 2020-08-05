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
- CopyN针对大小缓冲区的基准测试
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

## 测试遵循的细节

## 令人眼前一亮的写法

测试很少对接口进行测试,之前一直以为接口是核心,
应该会有大篇幅来测试Writer/Reader接口,实际上不是.
大部分篇幅都是针对暴露的函数,少部分针对暴露的功能性结构体.

## 如何写测试
