# io.ioutil 分析

这是io下的子包,通过两个源码文件来提供了一些功能性函数.

这里的功能性函数都是将和io包相关,而且蛮常用的功能进行了封装.
ioutil.go提供了:

- 读目录下的文件信息 ReadDir
- 对指定文件的读写 ReadFile/WriteFile
- NopCloser,特殊场景下需要的函数功能(后面分析测试文件时描述其适用场景)

tempfile.go提供了:

- 生成临时目录或临时文件的函数 TempFile/TempDir

## iotil.go分析

空关闭Reader.实现了Closer接口.

    type nopCloser struct {
      io.Reader
    }

    func (nopCloser) Close() error { return nil }

    func NopCloser(r io.Reader) io.ReadCloser {
      return nopCloser{r}
    }

ReadAll, 将Reader里的内容全部读出.

    func ReadAll(r io.Reader) ([]byte, error) {
      return readAll(r, bytes.MinRead)
    }

    func readAll(r io.Reader, capacity int64) (b []byte, err error) {
      var buf bytes.Buffer
      defer func() {
        e := recover()
        if e == nil {
          return
        }
        if panicErr, ok := e.(error); ok && panicErr == bytes.ErrTooLarge {
          err = panicErr
        } else {
          panic(e)
        }
      }()
      if int64(int(capacity)) == capacity {
        buf.Grow(int(capacity))
      }
      _, err = buf.ReadFrom(r)
      return buf.Bytes(), err
    }

