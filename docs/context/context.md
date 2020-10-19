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

Context接口的实现类型是emptyCtx.

    type emptyCtx int
    func (*emptyCtx) Deadline() (deadline time.Time, ok bool) { return }
    func (*emptyCtx) Done() <-chan struct{} { return nil }
    func (*emptyCtx) Err() error { return nil }
    func (*emptyCtx) Value(key interface{}) interface{} { return nil }
    func (e *emptyCtx) String() string {
      switch e {
      case background:
        return "context.Background"
      case todo:
        return "context.TODO"
      }
      return "unknown empty Context"
    }

可以看到emptyCtx除了实现了context.Context接口,还实现了context.stringer接口,
注意下面的String不是实现的fmt.Stringer接口,而是未暴露的context.stringer接口.
正如empty的名字,对Context接口的实现都是空的,后续需要针对emptyCtx做扩展.

    var (
      background = new(emptyCtx)
      todo       = new(emptyCtx)
    )
    func Background() Context {
      return background
    }
    func TODO() Context {
      return todo
    }
    
这里通过两个暴露的函数创建两个空的emptyCtx实例,后续会根据不同场景来扩展.
在注释中,background实例的使用场景是:main函数/初始化/测试/或者作为top-level
的Context(派生其他Context);todo实例的使用场景是:不确定时用todo.
到此emptyCtx的构造就理顺了,就是Background()/TODO()两个函数,之后是针对她们
的扩展和Context派生.

Context派生是基于With-系列函数实现的,我们先看对emptyCtx的扩展,
这些扩展至少会覆盖一部分函数,让空的上下文变成支持某些功能的上下文,
取消信号/截至日期/值,3种功能的任意组合.

从源码中可以看出,除了emptyCtx,还有cancelCtx/myCtx/myDoneCtx/otherContext/
timeCtx/valueCtx,他们有个共同特点:基于Context组合的新类型,
我们寻找的对emptyCtx的扩展,就是在这些新类型的方法中.

小技巧:emptyCtx已经实现了context.Context,如果要修改方法的实现,
唯一的方法就是利用Go的内嵌进行方法的覆盖.简单点说就是内嵌到struct,
struct再定义同样签名的方法,如果不需要数据,内嵌到接口也是一样的.

### cancelCtx

支持取消信号的上下文

    type cancelCtx struct {
      Context

      mu       sync.Mutex
      done     chan struct{}
      children map[canceler]struct{}
      err      error
    }

看下方法:

    var cancelCtxKey int
    func (c *cancelCtx) Value(key interface{}) interface{} {
      if key == &cancelCtxKey {
        return c
      }
      return c.Context.Value(key)
    }
    func (c *cancelCtx) Done() <-chan struct{} {
      c.mu.Lock()
      if c.done == nil {
        c.done = make(chan struct{})
      }
      d := c.done
      c.mu.Unlock()
      return d
    }
    func (c *cancelCtx) Err() error {
      c.mu.Lock()
      err := c.err
      c.mu.Unlock()
      return err
    }
    
cancelCtxKey默认是0,Value()要么返回自己,要么调用上下文Context.Value(),
具体使用后面再分析;Done()返回cancelCtx.done;Err()返回cancelCtx.err;

    func contextName(c Context) string {
      if s, ok := c.(stringer); ok {
        return s.String()
      }
      return reflectlite.TypeOf(c).String()
    }
    func (c *cancelCtx) String() string {
      return contextName(c.Context) + ".WithCancel"
    }

internal/reflectlite.TypeOf是获取接口动态类型的反射类型,
如果接口是nil就返回nil,此处是获取Context的类型,
从上面的分析可知,顶层Context要么是background,要么是todo,
cancelCtx实现的context.stringer要么是context.Background.WithCancel,
要么是context.TODO.WithCancel.这里说的只是顶层Context下的,
多层派生Context的结构也是类似的.

值得注意的是String()不属于Context接口的方法集,而是emptyCtx对
context.stringer接口的实现,cancelCxt内嵌的Context,所以不会覆盖
emptyCtx对String()的实现.

    var closedchan = make(chan struct{})
    func init() {
      close(closedchan)
    }

    func (c *cancelCtx) cancel(removeFromParent bool, err error) {
      if err == nil {
        panic("context: internal error: missing cancel error")
      }
      c.mu.Lock()
      if c.err != nil {
        c.mu.Unlock()
        return // already canceled
      }
      c.err = err
      if c.done == nil {
        c.done = closedchan
      } else {
        close(c.done)
      }
      for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
      }
      c.children = nil
      c.mu.Unlock()

      if removeFromParent {
        removeChild(c.Context, c)
      }
    }

cancel(),具体的取消信令对应的操作,err不能为nil,err会存到cancelCtx.err,
如果已经存了,表示取消操作已经执行.关闭done信道,如果之前没有调用Done()
来获取done信道,就返回一个closedchan(这是要给已关闭信道,可重用的),
之后是调用children的cancel(),最后就是在Context树上移除当前派生Context.

    func parentCancelCtx(parent Context) (*cancelCtx, bool) {
      done := parent.Done()
      if done == closedchan || done == nil {
        return nil, false
      }
      p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
      if !ok {
        return nil, false
      }
      p.mu.Lock()
      ok = p.done == done
      p.mu.Unlock()
      if !ok {
        return nil, false
      }
      return p, true
    }
    func removeChild(parent Context, child canceler) {
      p, ok := parentCancelCtx(parent)
      if !ok {
        return
      }
      p.mu.Lock()
      if p.children != nil {
        delete(p.children, child)
      }
      p.mu.Unlock()
    }

removeChild首先判断父Context是不是cancelCtx类型,
再判断done信道和当前Context的done信道是不是一致的,
(如果不一致,说明:done信道是diy实现的,就不能删掉了).

到此,cancelCtx覆盖了cancelCtx.Context的Done/Err/Value,
同时实现了自己的打印函数String(),还实现了cancel().
也就是说cancelCtx还实现了接口canceler:

    type canceler interface {
      cancel(removeFromParent bool, err error)
      Done() <-chan struct{}
    }
    // cancelCtx.children的定义如下:
    // children map[canceler]struct{}

执行取消信号对应的操作时,其中有一步就是执行children的cancel(),
children的key是canceler接口类型,所以有对cancel()的实现.
cancelCtx实现了canceler接口,那么在派生Context就可以嵌套很多层,
或派生很多个cancelCtx.

    func newCancelCtx(parent Context) cancelCtx {
      return cancelCtx{Context: parent}
    }

非暴露的构造函数.

回顾一下:cancelCtx添加了Context对取消信号的支持.
只要触发了"取消信号",使用方只需要监听done信道即可.

myCtx myDoneCtx otherContext属于测试,等分析测试的时候再细说.

### timerCtx

前面说到了取消信号对应的上下文cancelCtx,timerCtx就是基于取消信号上下扩展的

    type timerCtx struct {
      cancelCtx
      timer *time.Timer

      deadline time.Time
    }

注释说明:内嵌cancelCtx是为了复用Done和Err,扩展了一个定时器和一个截至时间,
在定时器触发时触发cancelCtx.cancel()即可.

    func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
      return c.deadline, true
    }
    func (c *timerCtx) String() string {
      return contextName(c.cancelCtx.Context) + ".WithDeadline(" +
        c.deadline.String() + " [" +
        time.Until(c.deadline).String() + "])"
    }
    func (c *timerCtx) cancel(removeFromParent bool, err error) {
      c.cancelCtx.cancel(false, err)
      if removeFromParent {
        removeChild(c.cancelCtx.Context, c)
      }
      c.mu.Lock()
      if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
      }
      c.mu.Unlock()
    }

timerCtx内嵌了cancelCtx,说明timerCtx也实现了canceler接口,
从源码中可以看出,cancel()是重新实现了,String/Deadline都重新实现了.

cancel()中额外添加了定时器的停止操作.

这里没有deadline设置和定时器timer开启的操作,会放在With-系列函数中.

回顾一下: Context的deadline是机会取消信号实现的.

### valueCtx

valueCtx和timerCtx不同,是直接基于Context的.

    type valueCtx struct {
      Context
      key, val interface{}
    }

一个valueCtx附加了一个kv对.实现了Value和String.

    func stringify(v interface{}) string {
      switch s := v.(type) {
      case stringer:
        return s.String()
      case string:
        return s
      }
      return "<not Stringer>"
    }
    func (c *valueCtx) String() string {
      return contextName(c.Context) + ".WithValue(type " +
        reflectlite.TypeOf(c.key).String() +
        ", val " + stringify(c.val) + ")"
    }
    func (c *valueCtx) Value(key interface{}) interface{} {
      if c.key == key {
        return c.val
      }
      return c.Context.Value(key)
    }
    
因为valueCtx.val类型是接口类型interface{},所以获取具体值时,
使用了switch type.

## With-系列函数

支持取消信号:

    var Canceled = errors.New("context canceled")
    func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
      if parent == nil {
        panic("cannot create context from nil parent")
      }
      c := newCancelCtx(parent)
      propagateCancel(parent, &c)
      return &c, func() { c.cancel(true, Canceled) }
    }
 
派生一个支持取消信号的Context,类型是cancelCtx,CancelFunc是取消操作,
具体是调用cancelCtx.cancel()函数,err参数是Canceled.

    func propagateCancel(parent Context, child canceler) {
      done := parent.Done()
      if done == nil {
        return // parent is never canceled
      }

      select {
      case <-done:
        // parent is already canceled
        child.cancel(false, parent.Err())
        return
      default:
      }

      if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
          // parent has already been canceled
          child.cancel(false, p.err)
        } else {
          if p.children == nil {
            p.children = make(map[canceler]struct{})
          }
          p.children[child] = struct{}{}
        }
        p.mu.Unlock()
      } else {
        atomic.AddInt32(&goroutines, +1)
        go func() {
          select {
          case <-parent.Done():
            child.cancel(false, parent.Err())
          case <-child.Done():
          }
        }()
      }
    }
 
传播取消信号.如果父Context不支持取消信号,那就不传播.
如果父Context的取消信号已经触发(就是父Context的done信道已经触发或关闭),
之后判断父Context是不是cancelCtx,如果是就将此Context丢到children中,
如果父Context不是cancelCtx,那就起协程监听父子Context的done信道.

小技巧:

    select {
    case <-done:
      child.cancel(false, parent.Err())
      return
    default:
    }

不加default,会等到done信道有动作;加了会立马判断done信道,done没操作就结束select.

    select {
    case <-parent.Done():
      child.cancel(false, parent.Err())
    case <-child.Done():
    }
    
这个会等待,因为没有加default.

因为顶层Context目前只能是background和todo,不是cancelCtx,
所以顶层Context的直接派生Context不会触发propagateCancel中的和children相关操作,
至少得3代及以后才有可能.

WithCancel的取消操作会释放相关资源,所以在上下文操作完之后,最好尽快触发取消操作.
触发的方式是:done信道触发,要么有数据,要么被关闭.

WithDeadline函数:



## 扩展功能以及如何扩展