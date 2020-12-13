# context test

测试对象是context包,测试包的包名是context_test

## 测试例子分析

example_test.go,展示了With-系列的4个例子.

    func ExampleWithCancel() {
      gen := func(ctx context.Context) <-chan int {
        dst := make(chan int)
        n := 1
        go func() {
          for {
            select {
            case <-ctx.Done():
              return // returning not to leak the goroutine
            case dst <- n:
              n++
            }
          }
        }()
        return dst
      }

      ctx, cancel := context.WithCancel(context.Background())
      defer cancel() // cancel when we are finished consuming integers

      for n := range gen(ctx) {
        fmt.Println(n)
        if n == 5 {
          break
        }
      }
      // Output:
      // 1
      // 2
      // 3
      // 4
      // 5
    }

结构分析,gen是一个函数,返回值是一个信道, for range channel是有特殊意义的,
for会循环从channel读数据,直到channel被close(),不然就是无限循环.

gen内部的协程就是典型的闭包,for range会不断触发读,gen内部的for select
会不断触发写,主协程读5次之后,会结束main函数,会触发defer函数,
也就是取消操作对应的回调,此时done信道会被close,gen内部的协程会正常退出.

这个例子是测试支持取消信号的上下文,取消函数的调用放在了main的defer函数中.

    const shortDuration = 1 * time.Millisecond
    func ExampleWithDeadline() {
      d := time.Now().Add(shortDuration)
      ctx, cancel := context.WithDeadline(context.Background(), d)

      // Even though ctx will be expired, it is good practice to call its
      // cancellation function in any case. Failure to do so may keep the
      // context and its parent alive longer than necessary.
      defer cancel()

      select {
      case <-time.After(1 * time.Second):
        fmt.Println("overslept")
      case <-ctx.Done():
        fmt.Println(ctx.Err())
      }

      // Output:
      // context deadline exceeded
    }

deadline的这个例子,在main的defer中也有主动调用取消函数的.
实际上通过打印可以显示deadline是否按预期工作.

    func ExampleWithTimeout() {
      ctx, cancel := context.WithTimeout(context.Background(), shortDuration)
      defer cancel()

      select {
      case <-time.After(1 * time.Second):
        fmt.Println("overslept")
      case <-ctx.Done():
        fmt.Println(ctx.Err()) // prints "context deadline exceeded"
      }

      // Output:
      // context deadline exceeded
    }

timeout只是deadline的一种简写.

    func ExampleWithValue() {
      type favContextKey string

      f := func(ctx context.Context, k favContextKey) {
        if v := ctx.Value(k); v != nil {
          fmt.Println("found value:", v)
          return
        }
        fmt.Println("key not found:", k)
      }

      k := favContextKey("language")
      ctx := context.WithValue(context.Background(), k, "Go")

      f(ctx, k)
      f(ctx, favContextKey("color"))

      // Output:
      // found value: Go
      // key not found: color
    }

context.WithValue和Context.Value()是存取操作,
取的时候,如果key没找到,会返回nil.

## 单元测试

context_text.go,x_test.go是单元测试,
example_test.go是示例,benchmark_test.go是基准测试,
net_test.go展示了deadline对net包的支持.

先看单元测试的context_text.go.

    type testingT interface {}

    type otherContext struct {}
    func quiescent(t testingT) time.Duration {}
    func XTestBackground(t testingT) {}
    func XTestTODO(t testingT) {}
    func XTestWithCancel(t testingT) {}
    func contains(m map[canceler]struct{}, key canceler) bool {}
    func XTestParentFinishesChild(t testingT) {}
    func XTestChildFinishesFirst(t testingT) {}
    func testDeadline(c Context, name string, t testingT) {}
    func XTestDeadline(t testingT) {}
    func XTestTimeout(t testingT) {}
    func XTestCanceledTimeout(t testingT) {}
    func XTestValues(t testingT) {}
    func XTestAllocs(t testingT, testingShort func() bool, testingAllocsPerRun func(int, func()) float64) {}
    func XTestSimultaneousCancels(t testingT) {}
    func XTestInterlockedCancels(t testingT) {}
    func XTestLayersCancel(t testingT) {}
    func XTestLayersTimeout(t testingT) {}
    func XTestCancelRemoves(t testingT) {}
    func XTestWithCancelCanceledParent(t testingT) {}
    func XTestWithValueChecksKey(t testingT) {}
    func XTestInvalidDerivedFail(t testingT) {}
    func recoveredValue(fn func()) (v interface{}) {}
    func XTestDeadlineExceededSupportsTimeout(t testingT) {}
    type myCtx struct {}
    type myDoneCtx struct {}
    func (d *myDoneCtx) Done() <-chan struct{} {}
    func XTestCustomContextGoroutines(t testingT) {}

这暴露的大多测试函数的参数类型是testingT接口类型,但这个源文件中没有实现testingT接口的,

    func TestBackground(t *testing.T)                      { XTestBackground(t) }
    func TestTODO(t *testing.T)                            { XTestTODO(t) }
    func TestWithCancel(t *testing.T)                      { XTestWithCancel(t) }
    func TestParentFinishesChild(t *testing.T)             { XTestParentFinishesChild(t) }
    func TestChildFinishesFirst(t *testing.T)              { XTestChildFinishesFirst(t) }
    func TestDeadline(t *testing.T)                        { XTestDeadline(t) }
    func TestTimeout(t *testing.T)                         { XTestTimeout(t) }
    func TestCanceledTimeout(t *testing.T)                 { XTestCanceledTimeout(t) }
    func TestValues(t *testing.T)                          { XTestValues(t) }
    func TestAllocs(t *testing.T)                          { XTestAllocs(t, testing.Short, testing.AllocsPerRun) }
    func TestSimultaneousCancels(t *testing.T)             { XTestSimultaneousCancels(t) }
    func TestInterlockedCancels(t *testing.T)              { XTestInterlockedCancels(t) }
    func TestLayersCancel(t *testing.T)                    { XTestLayersCancel(t) }
    func TestLayersTimeout(t *testing.T)                   { XTestLayersTimeout(t) }
    func TestCancelRemoves(t *testing.T)                   { XTestCancelRemoves(t) }
    func TestWithCancelCanceledParent(t *testing.T)        { XTestWithCancelCanceledParent(t) }
    func TestWithValueChecksKey(t *testing.T)              { XTestWithValueChecksKey(t) }
    func TestInvalidDerivedFail(t *testing.T)              { XTestInvalidDerivedFail(t) }
    func TestDeadlineExceededSupportsTimeout(t *testing.T) { XTestDeadlineExceededSupportsTimeout(t) }
    func TestCustomContextGoroutines(t *testing.T)         { XTestCustomContextGoroutines(t) }

这是x_test.go的内容,直接是用testing.T类型来实现testingT接口.

那先分析一下testing.T对testingT接口的实现.

    type T struct {
      common
      isParallel bool
      context    *testContext
    }
    func (t *T) Deadline() (deadline time.Time, ok bool) {
      deadline = t.context.deadline
      return deadline, !deadline.IsZero()
    }

注意,testing.T.context不是context.Context的实现类型,
Deadline()返回了t.context中存储的deadline信息.

testing.T内嵌了testing.common,大部分方法集都来至common:

    Error(args ...interface{})
    Errorf(format string, args ...interface{})
    Fail()
    FailNow()
    Failed() bool
    Fatal(args ...interface{})
    Fatalf(format string, args ...interface{})
    Helper()
    Log(args ...interface{})
    Logf(format string, args ...interface{})
    Name() string
    Skip(args ...interface{})
    SkipNow()
    Skipf(format string, args ...interface{})
    Skipped() bool
  
Parallel()是由testing.T实现,某个测试用例多次重复执行时,
可启用并发参数.