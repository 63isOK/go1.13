# io包中的multi

## 源码分析

开始就描述了两个结构体:
eofReader,用于模拟一个空Reader;
multiReader,实现了Reader,可以从多个Reader中读.

    func MultiReader(readers ...Reader) Reader {
      r := make([]Reader, len(readers))
      copy(r, readers)
      return &multiReader{r}
    }

可变参readers的类型可理解为`[]Reader`,所以copy可以直接用.
构造函数MultiReader返回的是对象指针.

接下来还有个multiWriter,实现了Writer接口,
行为是将缓冲写到多个Writer中.

io包中还定义了StringWriter接口

    type StringWriter interface {
      WriteString(s string) (n int, err error)
    }

multiWriter也实现了StringWriter接口.指针接受者实现的,
在test文件中出现一句可以触发编译期接口合理性检查的句子.

    var _ StringWriter = (*multiWriter)(nil)

这种触发方式也是编码规范推荐使用的.

之后也提供了一个构造函数MultiWriter.

io包的mutil.go文件,暴露了两个函数.

## test分析

multi_test.go,里面的层次或许不如io.go,但里面的写法更加大胆,值得观摩.

### TestMultiReader()分析

这个测试函数使用了很多新的写法.

    func TestMultiReader(t *testing.T) {
      var mr Reader
      var buf []byte
      nread := 0
      withFooBar := func(tests func()) {
        r1 := strings.NewReader("foo ")
        r2 := strings.NewReader("")
        r3 := strings.NewReader("bar")
        mr = MultiReader(r1, r2, r3)
        buf = make([]byte, 20)
        tests()
      }
      expectRead := func(size int, expected string, eerr error) {
        nread++
        n, gerr := mr.Read(buf[0:size])
        if n != len(expected) {
          t.Errorf("#%d, expected %d bytes; got %d",
            nread, len(expected), n)
        }
        got := string(buf[0:n])
        if got != expected {
          t.Errorf("#%d, expected %q; got %q",
            nread, expected, got)
        }
        if gerr != eerr {
          t.Errorf("#%d, expected error %v; got %v",
            nread, eerr, gerr)
        }
        buf = buf[n:]
      }
      withFooBar(func() {
        expectRead(2, "fo", nil)
        expectRead(5, "o ", nil)
        expectRead(5, "bar", nil)
        expectRead(5, "", EOF)
      })
      withFooBar(func() {
        expectRead(4, "foo ", nil)
        expectRead(1, "b", nil)
        expectRead(3, "ar", nil)
        expectRead(1, "", EOF)
      })
      withFooBar(func() {
        expectRead(5, "foo ", nil)
      })
    }

先不看具体的测试功能,先看看写法.
里面定义了两个函数,一个用于准备测试数据,一个用于具体的测试执行.
这两个函数都是在测试函数中定义的,属于闭包的范围.
这种写法,特别适合对一组数据的多场景测试,数据的初始化和测试的预期结果
分别放在两个函数,属于表格测试的一种简化.

其次,对数据做了3个场景测试(得益于闭包),所以调用了withFooBar 3次.
再看看数据的初始化和预期结果,数据变量放在测试函数,
数据的初始化放在第一个内置函数里,预期结果作为第二个之函数的参数.

这种测试方式是对表格测试的一种补充.写法大胆而不失优雅.

再看看具体的功能.

TestMultiReader,测试对象是MultiReader().Read().
multiReader是一个Reader树,每次调用Read()就是从Reader树中找一个进行Read,
直到读到东西,或遍历完Reader树.其中有几个细节:

- 读完的Reader不再读
- 只要能读到数据就返回,不管这个Reader是不是已经EOF
- 所有Reader都读完了,返回(0,EOF)

withFooBar(),构造了一个multiReader,里面包含3个strings.Reader.

expectRead(),对应一次Read,指定了读缓冲大小和预期结果.

### multiWriter测试分析

第一个测试是分析内存申请次数

    func TestMultiWriter_WriteStringSingleAlloc(t *testing.T) {
      var sink1, sink2 bytes.Buffer
      type simpleWriter struct { // hide bytes.Buffer's WriteString
        Writer
      }
      mw := MultiWriter(simpleWriter{&sink1}, simpleWriter{&sink2})
      allocs := int(testing.AllocsPerRun(1000, func() {
        WriteString(mw, "foo")
      }))
      if allocs != 1 {
        t.Errorf("num allocations = %d; want 1", allocs)
      }
    }

testing.AllocsPerRun有两个参数,一个是要执行的函数,另一个是执行次数,
最后返回的是内存平均申请次数.

WriteString()是io包暴露的一个函数

    func WriteString(w Writer, s string) (n int, err error) {
      if sw, ok := w.(StringWriter); ok {
        return sw.WriteString(s)
      }
      return w.Write([]byte(s))
    }

如果实现了StringWriter接口,就调用StringWriter接口,不然就走常规的Write.

    func (t *multiWriter) WriteString(s string) (n int, err error) {
      var p []byte // lazily initialized if/when needed
      for _, w := range t.writers {
        if sw, ok := w.(StringWriter); ok {
          n, err = sw.WriteString(s)
        } else {
          if p == nil {
            p = []byte(s)
          }
          n, err = w.Write(p)
        }
        if err != nil {
          return
        }
        if n != len(s) {
          err = ErrShortWrite
          return
        }
      }
      return len(s), nil
    }

可以看到multiWriter是实现了StringWriter接口的.
实际上,multiWriter.WriteString也是优先使用各个Writer的StringWriter接口,
因为这个效率是最高的(这点可以后面分析).

回到我们的测试函数,最终是调用到bytes.Buffer.WriteString(),
这里面的内存申请次数就不细究了,等分析到bytes包时再说.

第二个是测试如果Writer实现了StringWriter,是否真的会调用StringWriter接口,
TestMultiWriter_StringCheckCall函数,就是用来测试是否正确调用.

第三个是测试multiWriter.Write(),最后利用的是testMultiWriter(),
这个测试是利用Copy系列的Copy函数来测试,其中一个有意思的是:
sha1.digest也作为Writer参与其中.在这里展示了一种新的接口屏蔽方式.

    func TestMultiWriter(t *testing.T) {
      sink := new(bytes.Buffer)
      // Hide bytes.Buffer's WriteString method:
      testMultiWriter(t, struct {
        Writer
        fmt.Stringer
      }{sink, sink})
    }

spec对struct的定义中,有一段是这么描述的:
如果struct中有内嵌字段,只需要指明内嵌类型即可,
内嵌类型的字段和方法自动晋升为struct的字段和方法,
如果有冲突,按最外层覆盖最内层的原则走.

多么通俗,但不那么易懂,直到我们撞墙多次后,再翻开spec,才恍然大悟,
原来结构体类型和其他类型都是类型,包括接口类型只是类型而以.
如果struct内嵌接口类型,在实际运用结构体的过程中,
会有一个满足接口的具体类型会赋值给这个内嵌字段.

所以上面才有一个struct内嵌两个接口类型,而实际初始化的过程可以用同一个sink来满足,
这种玩弄技巧的目的在哪?sink是bytes.Buffer,通过这个结构体包一层后,
对外暴露的方法集中,只包含(Writer/fmt.Stringer),其他所有的bytes.Buffer实现的接口,
都被屏蔽掉了.

    func TestMultiWriter_String(t *testing.T) {
      testMultiWriter(t, new(bytes.Buffer))
    }

这个里面的new(bytes.Buffer))可是没有屏蔽任何接口.

再回头说说io_test.go中的屏蔽手法.

    type Buffer struct {
      bytes.Buffer
      ReaderFrom // conflicts with and hides bytes.Buffer's ReaderFrom.
      WriterTo   // conflicts with and hides bytes.Buffer's WriterTo.
    }

bytes.Buffer内部是实现了ReaderFrom/WriterTo的,
strcut内嵌类型的字段和方法晋升机制,导致bytes.Buffer对这两个接口的实现被屏蔽,
实际上这个Buffer有3个字段,后两个是接口类型,
rb := new(Buffer)时接口字段被初始化为nil.

说完了接口屏蔽,回到测试函数:

    func testMultiWriter(t *testing.T, sink interface {
      Writer
      fmt.Stringer
    }) {...}

第二个参数是一个接口类型,不管是上面的struct,还是new(bytes.Buffer)均可支持.

### 一些特别的类型

TestMultiReaderFlatten测试的是实现了Reader接口的函数类型.
配合runtime的堆栈信息做一些测试.

还有按字节读的结构体配合ioutil包作测试,放后面分析.

## 眼前一亮的手法

测试数据的初始化和预期结果可以放在两个内置函数中,
可以代替表格测试.

像这种测试逻辑分支的场景,可以通过模拟返回值来避开业务,
通过标识符flag等来判断具体走的是哪个分支.

屏蔽接口可使用struct内嵌接口,而要屏蔽对象可放在struct中,
也可以放在结构体的初始化字面量中,io包的测试例子关于这两种写法都提供了.

## 总结
