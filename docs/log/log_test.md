# log库的测试例子分析

例子分两类:test和eaxmple

文档上也说了: these tests are too simple

## 单元测试

    const (
      Rdate         = `[0-9][0-9][0-9][0-9]/[0-9][0-9]/[0-9][0-9]`
      Rtime         = `[0-9][0-9]:[0-9][0-9]:[0-9][0-9]`
      Rmicroseconds = `\.[0-9][0-9][0-9][0-9][0-9][0-9]`
      Rline         = `(57|59):`
      // must update if the calls to l.Printf / l.Print below move
      Rlongfile     = `.*/[A-Za-z0-9_\-]+\.go:` + Rline
      Rshortfile    = `[A-Za-z0-9_\-]+\.go:` + Rline
    )

这是用于正则匹配的,针对这次的测试,做了很多定制.

    type tester struct {
      flag    int
      prefix  string
      pattern string  // regexp that log output must match
                      //we add ^ and expected_text$ always
    }

    var tests = []tester{
      // individual pieces:
      {0, "", ""},
      {0, "XXX", "XXX"},
      {Ldate, "", Rdate + " "},
      {Ltime, "", Rtime + " "},
      {Ltime | Lmicroseconds, "", Rtime + Rmicroseconds + " "},
      {Lmicroseconds, "", Rtime + Rmicroseconds + " "}, // microsec implies time
      {Llongfile, "", Rlongfile + " "},
      {Lshortfile, "", Rshortfile + " "},
      {Llongfile | Lshortfile, "", Rshortfile + " "}, // shortfile overrides longfile
      // everything at once:
      {Ldate | Ltime | Lmicroseconds | Llongfile, "XXX",
        "XXX" + Rdate + " " + Rtime + Rmicroseconds + " " + Rlongfile + " "},
      {Ldate | Ltime | Lmicroseconds | Lshortfile, "XXX",
        "XXX" + Rdate + " " + Rtime + Rmicroseconds + " " + Rshortfile + " "},
    }

这里面是做了一些测试数据,如果flag设置为0,表示没有日期/文件等,
因为枚举是从1开始的,后面的测试数据中,组合了flag和prefix的各种情况.

这些测试数据是用在testPrint的,testPrint是用在TestAll中的,
在上面的测试中,使用默认日志器std,测试了各种flag/prefix的组合,
也测试了Printf和Println两种不同写法的最终输出.

TestOutput 单元测试, TestOutputRace是对并发进行测试

TestFlagAndPrefixSetting 测试了Logger属性的读写操作

TestUTCFlag 测试utc, 考虑到跨秒,所以做了两次

TestEmptyPrintCreatesLine 测试自动添加换行功能

## 性能测试

- BenchmarkItoa
- BenchmarkPrintln
- BenchmakrPrintlnNoFlags

## 分析

- 单元测试覆盖的范围是针对一个小特征的,针对核心函数或重要的辅助函数
- 单元测试,不应该是针对每个方法/函数来的
- 单元测试大部分类似的函数,挑一两个测试即可
- 单元测试的例子应该覆盖大多数方法和函数
- 确保库中所有的特性都覆盖到了
- 从多个函数抽出的公共部分(一般也会是一个函数),需要针对性进行测试
- 对于模型复杂的辅助函数,即使提供的功能很简单,也要做性能测试
  - 性能的提高就是从这里来的,就如本例中的itoa
- 对于提供类似功能的方法,可提供性能测试

如果用tdd的思想来看:

- 要先做好log库的设计工作
  - 做一个Logger来封装所有底层的逻辑
    - 辅助功能不暴露
    - 对标Print系列
    - 暴露Output,提供更加精细的控制
  - 做一个默认的对象,不暴露,通过函数来暴露Logger提供的功能
- 在做设计工作时,确定好每个特征的边界
- tdd做test,覆盖每个特征边界的测试
- tdd绿灯
- 重构,丰富Logger对Printx系列的支持
- 写example

从testing来看:

- TestAll对所有测试数据进行测试,并不局限于t.Run
- 并不是每个测试都需要使用t.Errorf
