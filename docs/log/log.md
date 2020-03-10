# log包

这是标准库中的一个简单日志库,这个库提供了一个类型Logger,
就称她是日志器,这个Logger还带有一些方法,通过这些可以格式化输出日志.

这个包还定义了一个标准的的日志器,叫std,通过这个std还扩展出了一些列辅助函数,
这些函数的定义和Printf等是一致的.当然通过Logger自定义也是非常方便的.

日志器可以输出到stderr,可以给每条日志消息加上时间日期.

一条日志消息,就是单独一行,如果消息中没有换行,日志器也会自动加上.
Fatal/Panic函数会在写完日志后掉用系统对应的函数.

因为标准库的文档写的好,所以看完文档就知道大体实现的流程:

- [x] Logger类型分析
- [x] std默认日志器的分析
- [x] 基于std的辅助函数分析
- [x] 总结

## 源码分析

### 常量分析

    const (
      Ldate = 1 << iota   // the date in the local time zone: 2009/01/23
      Ltime               // the time in the local time zone: 01:23:23
      Lmicroseconds       // microsecond resolution: 01:23:23.123123.  assumes Ltime.
      Llongfile           // full file name and line number: /a/b/c/d.go:23
      Lshortfile          // final file name element and line number: d.go:23.
                          // overrides Llongfile
      LUTC                // if Ldate or Ltime is set
                          // use UTC rather than the local time zone
      LstdFlags     = Ldate | Ltime // initial values for the standard logger
    )

首先,是个枚举,枚举值是 1<<iota, 表示这些枚举值是可以做或运算的,
也就是说这些枚举值是可以进行任意组合的.

从注释上可以看到,日期/时分秒/微秒是可以任意组合的;
长短文件名是带路径和不带路径的二选一,如果都设置了,短文件名优先级高;
如果日期/时分秒都设置了,那么还可额外设置utc时间格式;
最后LstdFlags是一个新的常量,为默认std日志器用的,
只显示日期和时分秒,这点也和文档的说明一致.

### Logger分析

    type Logger struct {
      mu     sync.Mutex // ensures atomic writes; protects the following fields
      prefix string     // prefix to write at beginning of each line
      flag   int        // properties
      out    io.Writer  // destination for output
      buf    []byte     // for accumulating text to write
    }

一个Logger表示一个日志器,她的目的是将日志写入到io.Writer,
有互斥量,可以支持多个协程的并发操作,互斥量会保证顺序访问.

### Logger的构造

    func New(out io.Writer, prefix string, flag int) *Logger {
      return &Logger{out: out, prefix: prefix, flag: flag}
    }

构造方法,支持配置日志器的3个属性,buf字段要在写日志才会用到,
所以构造不用设置buf.

    func (l *Logger) SetPrefix(prefix string) {
      l.mu.Lock()
      defer l.mu.Unlock()
      l.prefix = prefix
    }

不止SetPrefix,还有SetFlags/SetOutput都是类似写法,用互斥来保证修改.
这3个属性对应的获取是Prefix/Flags/Writer,都是用互斥量来保证读.

注意Logger.out的写是SetOutput,读是Writer,而不是用统一的Output,
因为Output函数用于日志的写操作了,这点确实很别扭,
猜测应该是向后兼容问题,或者是io.Writer的输出默认用Writer,
等后面分析多了就清楚了.

### Logger 日志的输出

调用栈因该是这样的:

- 各种格式化写日志函数
  - Output 具体的写操作
    - formatHeader 日志头部
      - itoa 数字的自动填充

下面就按从下到上的顺序来分析:

itoa已经在[这里](https://github.com/63isOK/kata/blob/master/codewars/padding/README.md)
做了详细的分析,所以就不讨论了.

formatHeader,文档上写的是:处理前缀/日期/事件/文件路径/行号.源码也是这么处理的.

Output,简单的按日志器Logger中设置的属性去组合消息(包含格式化头信息的),之后执行写操作.
值得注意的是,在加锁期间,获取文件名和行数时,释放了锁,理由是开销太大.
这种处理方式是值得借鉴的.

之后各式各样的写日志函数,都是基于Output来做的,她们更多的是针对日志消息的本身内容.

### std

    var std = New(os.Stderr, "", LstdFlags)

默认日志器,特征是:

- 输出到标准错误
- 没有前缀
- 只显示日期和时间
- 不对外暴露,对外暴露的是辅助函数

### 其他辅助函数

就是针对默认日志器,将方法转变为了函数,好处是默认日志器没有暴露.

最后还暴露了一个Output

    func Output(calldepth int, s string) error {
      return std.Output(calldepth+1, s) // +1 for this frame.
    }

如果不满足log库提供的几个格式化输出函数,还可以直接调用Output,
颗粒度更加精细.

## 总结

- 标准库都是经过优化的,性能非常高
- 为了性能甚至不调用其他标准库
- 不止标准库,很多开源的库都是以下套路:
  - 暴露一个类型,类型中提供了很多功能方法
  - 设置一个默认的对象(不对外暴露),使用常用配置
  - 基于默认对象将功能方法转换为对外暴露的函数
  - 提供一个颗粒度很小的核心功能(一般是最小的原子功能),暴露给外面
- 利用锁来保证多协程的并发使用
- 对于可配置项,可按枚举的方式来处理
