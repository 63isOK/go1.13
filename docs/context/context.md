# context包

**这个包分析的是1.15**

## 包说明分析

context包定义了一个Context类型(接口类型),通过这个Context接口类型,
就可以跨api边界/跨进程传递一些deadline/cancel信号/request-scoped值.

发给server的请求中需要包含Context,server需要接收Context.
在整个函数调用链中,Context都需要进行传播.
期间是可以选择将Context替换为派生Context(由With-系列函数生成).
当一个Context是canceled状态时,所有派生的Context都是canceled状态.

With-系列函数(不包含WithValue)会基于父Context来生成一个派生Context,
还有一个CancelFunc函数,调用这个CancelFun函数可取消派生对象和
"派生对象的派生对象的...",并且会删除父Context和派生Context的引用关系,
最后还会停止相关定时器.如果不调用CancelFunc,直到父Context被取消或
定时器触发,派生Context和"派生Context的派生Context..."才会被回收,
否则就是泄露leak. go vet工具可以检测到泄露.

使用Context包的程序需要遵循以下以下规则,目的是保持跨包兼容,
已经使用静态分析工具来检查context的传播:

- Context不要存储在struct内,直接在每个函数中显示使用,作为第一个参数,名叫ctx
- 即使函数允许,也不要传递nil Context,如果实在不去确定就传context.TODO
- 在跨进程和跨api时,要传request-scoped数据时用context Value,不要传函数的可选参数
- 不同协程可以传递同一Context到函数,多协程并发使用Context是安全的

## 包结构分析

核心的是:

    func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
    func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
    func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
    type CancelFunc
    type Context

从上可以看出,核心的是Context接口类型,围绕这个类型出现了With-系列函数,
针对派生Context,还有取消函数CancelFunc.

还有两个暴露的变量:

- Canceled
  - context取消时由Context.Err方法返回
- DeadlineExceeded
  - context超过deadline时由Context.Err方法返回

## Context接口类型分析

context也称上下文.

    type Context interface {
      Deadline() (deadline time.Time, ok bool)
      Done() <-chan struct{}
      Err() error
      Value(key interface{}) interface{}
    }

先看说明:

跨api时,Context可以携带一个deadline/一个取消信号/某些值.
并发安全.

方法集分析:

- Deadline
  - 返回的是截至时间
    - 这个时间表示的是任务完成时间
    - 到这个时间点,Context的状态已经是be canceled(完成状态)
  - ok为false表示没有设置deadline
  - 连续调用,返回的结果是相同的
- Done
  - 返回的只读信道
  - 任务完成,信道会被关闭,Context状态是be canceled
  - Conetext永远不be canceled,Done可能返回nil
  - 连续调用,返回的结果是相同的
  - 信道的关闭会异步发生,且会在取消函数CancelFunc执行完之后发生
  - 使用方面,Done需要配合select使用
  - 更多使用Done的例子在[这个博客](https://blog.golang.org/pipelines)
- Err
  - Done还没关闭(此处指Done返回的只读信道),Err返回nil
  - Done关闭了,Err返回non-nil的error
    - Context是be canceled,Err返回Canceled(这是之前分析的一个变量)
    - 如果是超过了截至日期deadline,Err返回DeadlineExceeded
  - 如果Err返回non-nil的error,后续再次调用,返回的结果是相同的
- Value
  - 参数和返回值都是interface{}类型(这种解耦方式值得学习)
  - Value就是通过key找value,如果没找到,返回nil
  - 连续调用,返回的结果是相同的
  - 上下文值,只适用于跨进程/跨api的request-scoped数据
  - 不适用于代替函数可选项
  - 一个上下文中,一个key对应一个value
  - 典型用法:申请一个全局变量来放key,在context.WithValue/Context.Value中使用
  - key应该定义为非暴露类型,避免冲突
  - 定义key时,应该支持类型安全的访问value(通过key)
  - key不应该暴露
    - 表示应该通过暴露函数来进行隔离(具体可以查看源码中的例子)

## 后续分析规划

看完Context的接口定义后,还需要查看With-系列函数才能知道context的定位,
在With-系列中会涉及到Context的使用和内部实现,那就先看WithCancel.

withCancel:

- CancelFunc
- newCancelCtx
  - cancelCtx
    - canceler
- propagateCancel
  - parentCancelCtx

以下是分析出的通过规则:很多包对外暴露的是接口类型和几个针对此类型的常用函数.
接口类型暴露意味可扩展,但是想扩展之后继续使用常用函数,那扩展部分就不能
修改常用函数涉及的部分,当然也可以通过额外的接口继续解耦.
针对"暴露接口和常用函数"这种套路,实现时会存在一个非暴露的实现类型,
常用函数就是基于这个实现类型实现的.在context.go中的实现类型是emptyCtx.
如果同时需要扩展接口和常用函数,最好是重新写一个新包.

下面的分析分成两部分:基于实现类型到常用函数;扩展功能以及如何扩展.

## 基于实现类型到常用函数

Context接口的实现类型是emptyCtx,通过常用函数会封装到cancelCtx.

## 扩展功能以及如何扩展