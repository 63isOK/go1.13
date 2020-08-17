# pipe.go分析

这个文件使用到了errors包,也是用到了sync库.

文件说明:pipe是一个适配器,用于连接Reader和Writer.

## 结构分析

对外暴露的是一个构造函数和构造的两个对象.
两个对象分别暴露了方法,同时这两个对象还有一个共同的底层对象.
实际上,这两个对象暴露的方法是直接调用底层对象的,
那么核心还是在底层对象上,只是通过两个对象和一个构造方法将底层对象的细节隐藏了.

## pipe sruct分析

pipe的方法不多,新的写法却不少.

    type atomicError struct{ v atomic.Value }

    func (a *atomicError) Store(err error) {
      a.v.Store(struct{ error }{err})
    }
    func (a *atomicError) Load() error {
      err, _ := a.v.Load().(struct{ error })
      return err.error
    }

atomicError提供了error的原子读写.

    type pipe struct {
      wrMu sync.Mutex // Serializes Write operations
      wrCh chan []byte
      rdCh chan int

      once sync.Once // Protects closing done
      done chan struct{}
      rerr atomicError
      werr atomicError
    }

可以看到pipe结构体中主要分两块:

- 读写信道
  - 两个无缓冲信道
  - 一个互斥量(保护暴露的写函数)
- 结束标识
  - once保证done的关闭只执行一次
  - done标志整个读写的结束
  - 剩下两个用于存储读写错误

## PipeReader/PipeWriter的分析

PipeReader对外暴露的是读/关闭

    type PipeReader struct {
      p *pipe
    }

    func (r *PipeReader) Read(data []byte) (n int, err error) {
      return r.p.Read(data)
    }

    func (r *PipeReader) Close() error {
      return r.CloseWithError(nil)
    }

    func (r *PipeReader) CloseWithError(err error) error {
      return r.p.CloseRead(err)
    }

PipeWriter对外暴露的是写/关闭

     type PipeWriter struct {
       p *pipe
     }

    func (w *PipeWriter) Write(data []byte) (n int, err error) {
      return w.p.Write(data)
    }

    func (w *PipeWriter) Close() error {
      return w.CloseWithError(nil)
    }

    func (w *PipeWriter) CloseWithError(err error) error {
      return w.p.CloseWrite(err)
    }

她们的方法集都是指针接收者.具体方法的实现是通过pipe的方法完成的.
pipe的方法更加明确:读/获取读错误/结束读写并设置读错误;
写/获取写错误/结束读写并设置写错误.思路相当明确.

下面主要分析pipe的读写

    func (p *pipe) Read(b []byte) (n int, err error) {
      select {
      case <-p.done:
        return 0, p.readCloseError()
      default:
      }

      select {
      case bw := <-p.wrCh:
        nr := copy(b, bw)
        p.rdCh <- nr
        return nr, nil
      case <-p.done:
        return 0, p.readCloseError()
      }
    }

    func (p *pipe) Write(b []byte) (n int, err error) {
      select {
      case <-p.done:
        return 0, p.writeCloseError()
      default:
        p.wrMu.Lock()
        defer p.wrMu.Unlock()
      }

      for once := true; once || len(b) > 0; once = false {
        select {
        case p.wrCh <- b:
          nw := <-p.rdCh
          b = b[nw:]
          n += nw
        case <-p.done:
          return n, p.writeCloseError()
        }
      }
      return n, nil
    }

读写都是利用两个阶段的select来完成,第一个阶段的select是判断读写有没有结束,
第二阶段处理实际的读写.

- Read
  - 每次将读的数量写到读信道
- Write
  - 先将缓冲写到写信道,再从读信道中获取读字节数,最后调整缓冲
  - 如果缓冲太大,一次读没读完,就将写的过程多来几遍,知道缓冲全部写完

## 写法

PipeWriter/PipeReader对外暴露的关闭,其实只可以保留一个CloseWithError,
但是为了方便客户(调用者),还是拆成两个,其实可以做测试比较一下.
性能测试发现拆成两个或写成一个可选参函数,性能上差别不大,
那这种写法的主要作用是让暴露的方法更加清晰易懂.

pipe.Write中,for循环带有once参数,可以保证循环至少来一次,
算是do while的一种实现.

## 总结

不管是PipeReader/PipeWriter,还是pipe,都对Reader/Writer有(部分)实现.

另外还有一些细节没有说道:读写错误和EOF.

反思:本次阅读是先理代码后看文档,才发现关于error部分没有留心到,
后面还是先文档后代码,这样效率会高一点.
