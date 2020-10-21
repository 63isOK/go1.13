# context test

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
