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

## 眼前一亮的手法

测试数据的初始化和预期结果可以放在两个内置函数中,
可以代替表格测试.

## 总结
