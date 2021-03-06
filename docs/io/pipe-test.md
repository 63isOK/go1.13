# pipe 测试

结构分析,主要包括以下几部分的测试:

- 非并发读写
- 写长缓冲,测试多次调用读
- 读写时遇到关闭
- 并发(并发写/并发读)

Pipe()构造的两个对象是同步读写的.

## 非缓冲读写

最简单的场景:

    func checkWrite(t *testing.T, w Writer, data []byte, c chan int) {
      n, err := w.Write(data)
      if err != nil {
        t.Errorf("write: %v", err)
      }
      if n != len(data) {
        t.Errorf("short write: %d != %d", n, len(data))
      }
      c <- 0
    }

    func TestPipe1(t *testing.T) {
      c := make(chan int)
      r, w := Pipe()
      var buf = make([]byte, 64)
      go checkWrite(t, w, []byte("hello, world"), c)
      n, err := r.Read(buf)
      if err != nil {
        t.Errorf("read: %v", err)
      } else if n != 12 || string(buf[0:12]) != "hello, world" {
        t.Errorf("bad read: got %q", buf[0:n])
      }
      <-c
      r.Close()
      w.Close()
    }

两个协程,一个读一个写,各自检查自己的结果,
这里面有个控制协程不退出的小技巧:利用信道.

下面是多次写多次读(非并发):

    func reader(t *testing.T, r Reader, c chan int) {
      var buf = make([]byte, 64)
      for {
        n, err := r.Read(buf)
        if err == EOF {
          c <- 0
          break
        }
        if err != nil {
          t.Errorf("read: %v", err)
        }
        c <- n
      }
    }

    func TestPipe2(t *testing.T) {
      c := make(chan int)
      r, w := Pipe()
      go reader(t, r, c)
      var buf = make([]byte, 64)
      for i := 0; i < 5; i++ {
        p := buf[0 : 5+i*10]
        n, err := w.Write(p)
        if n != len(p) {
          t.Errorf("wrote %d, got %d", len(p), n)
        }
        if err != nil {
          t.Errorf("write: %v", err)
        }
        nn := <-c
        if nn != n {
          t.Errorf("wrote %d, read got %d", n, nn)
        }
      }
      w.Close()
      nn := <-c
      if nn != 0 {
        t.Errorf("final read got %d", nn)
      }
    }

这里也是分两个协程来处理,不过读写都是多次.
读退出的唯一条件是遇到EOF.
只有地用pipe.CloseWrite(nil)才会出现EOF,也就是PipeWriter.Close().
这个测试中,读缓冲长度是64,写缓冲长度是逐次增加的.

值得一提的是,每次读完都会将结果告诉写协程,让写协程去作判断.

## 写长缓冲

不断调整读缓冲大小的场景:

    func writer(w WriteCloser, buf []byte, c chan pipeReturn) {
      n, err := w.Write(buf)
      w.Close()
      c <- pipeReturn{n, err}
    }

    func TestPipe3(t *testing.T) {
      c := make(chan pipeReturn)
      r, w := Pipe()
      var wdat = make([]byte, 128)
      for i := 0; i < len(wdat); i++ {
        wdat[i] = byte(i)
      }
      go writer(w, wdat, c)
      var rdat = make([]byte, 1024)
      tot := 0
      for n := 1; n <= 256; n *= 2 {
        nn, err := r.Read(rdat[tot : tot+n])
        if err != nil && err != EOF {
          t.Fatalf("read: %v", err)
        }

        expect := n
        if n == 128 {
          expect = 1
        } else if n == 256 {
          expect = 0
          if err != EOF {
            t.Fatalf("read at end: %v", err)
          }
        }
        if nn != expect {
          t.Fatalf("read %d, expected %d, got %d", n, expect, nn)
        }
        tot += nn
      }
      pr := <-c
      if pr.n != 128 || pr.err != nil {
        t.Fatalf("write 128: %d, %v", pr.n, pr.err)
      }
      if tot != 128 {
        t.Fatalf("total read %d != 128", tot)
      }
      for i := 0; i < 128; i++ {
        if rdat[i] != byte(i) {
          t.Fatalf("rdat[%d] = %d", i, rdat[i])
        }
      }
    }

写的协程很简单,写一个128长度的数据,
整个读的过程是读一步检查一步,最后将所有的结果进行检查.
包括了读长度/写长度/返回值/读到的每个数据.
基本上能检查的全部做了检查.

## 关闭的场景

读写过程中的关闭是可能出现的,所以需要做测试.

TestPipeReadClose(),表格测试,测试了两种情况:
pipe关闭后读;读时遇到pipe关闭.(分别对应同步和异步的场景).

这里面有个缓冲信道,用于保证delayClose执行完之后再做检查.
这步的目的是先检查delayClose的结果,后检查读的结果.
不然测试函数执行完了,再检查delayClose逻辑就不对了.

上面的关闭是通过PipeWriter来关闭,返回的error是EOF,
TestPipereadClose2(),是通过PipeReader来关闭,返回的error是ErrClosePipe.

TestPipeWriteClose()系列则是测试关闭后写和写时关闭,
以及通过不同PipeWriter来关闭.场景和读时关闭的场景一致.

TestWriteEmpty/TestWriteNil 没有做结果检查.

## 并发

TestPipeConcurrent()分成两部分的读写,
一个是并发写,一个协程读;另一个是并发读,一个协程写.

## 写法

一个测试函数的执行过程中,要确保每个检查做做完了再结束,
这种只针对多协程场景.此时缓冲信道可以派上用场.

针对包提供的函数做了全面的测试,包括流程分支,和各种边界.
