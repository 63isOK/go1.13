# errors包

在builtin包中指定了很多内置类型和一些内置函数,其中使用非常广的一个是error接口.

    type error interface {
      Error() string
    }

在errors包中有对error接口的实现.

    package errors

    func New(text string) error {
      return &errorString{text}
    }

    type errorString struct {
      s string
    }

    func (e *errorString) Error() string {
      return e.s
    }

整个errors包对error的实现非常简单,只是扩展了一个错误信息.
对外暴露的就是构造函数New().

## 扩展

Unwrap()扩展

    func Unwrap(err error) error {
      u, ok := err.(interface {
        Unwrap() error
      })
      if !ok {
        return nil
      }
      return u.Unwrap()
    }

如果一个错误(这里指实现了error接口的类型)同时还实现了interface{Unwrap()error},
那么调用Unwarp()就可以获取到更深层次的错误.

这里有趣的是Unwrap()返回的是error类型,所以可以用于链式扩展.
调用一次Unwrap(),就是解开一层,直到最后一层原始error时,Unwrap返回nil.

Is()扩展

    func Is(err, target error) bool {
      if target == nil {
        return err == target
      }

      isComparable := reflectlite.TypeOf(target).Comparable()
      for {
        if isComparable && err == target {
          return true
        }
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
          return true
        }
        if err = Unwrap(err); err == nil {
          return false
        }
      }
    }

这个是比较两个error是否相等(这里的相等也是需要定义的),
要么是通过反射来进行比较,要么是实现 interface { Is(error) bool }来确定比较,
如果这两种方式都不行,就将第一个比较值(对应代码中的err)进行Unwrap,
这里比较适合链式错误的比较.

As()扩展,是一个赋值,也支持链式.

## errors测试

- TestNewEqual
  - 测试构造出的error是否相等
  - 不同的New是不同的
  - 同一个New出来的变量是可以和自己做比较的,而且应该相等
- 其他单元测试

## 扩展测试

这里面的测试对象之一是wrapped结构体

    type wrapped struct {
      msg string
      err error
    }

    func (e wrapped) Error() string { return e.msg }

    func (e wrapped) Unwrap() error { return e.err }

实现了error接口和interface { Unwarp() error }接口.

另一个测试对象是poser结构体

    type poser struct {
      msg string
      f   func(error) bool
    }

    var poserPathErr = &os.PathError{Op: "poser"}

    func (p *poser) Error() string     { return p.msg }
    func (p *poser) Is(err error) bool { return p.f(err) }
    func (p *poser) As(err interface{}) bool {
      switch x := err.(type) {
      case **poser:
        *x = p
      case *errorT:
        *x = errorT{"poser"}
      case **os.PathError:
        *x = poserPathErr
      default:
        return false
      }
      return true
    }

poser实现了error接口,以及Is/As对应的接口.

先分析最基础的TestUnwrap().

    func TestUnwrap(t *testing.T) {
      err1 := errors.New("1")
      erra := wrapped{"wrap 2", err1}

      testCases := []struct {
        err  error
        want error
      }{
        {nil, nil},
        {wrapped{"wrapped", nil}, nil},
        {err1, nil},
        {erra, err1},
        {wrapped{"wrap 3", erra}, erra},
      }
      for _, tc := range testCases {
        if got := errors.Unwrap(tc.err); got != tc.want {
          t.Errorf("Unwrap(%v) = %v, want %v", tc.err, got, tc.want)
        }
      }
    }

erra对err1封装了一层,再利用表格测试来测试不同的场景.
具体的测试功能是解封一层,即对error调用Unwrap().

再看看TestIs().也是表格测试,写法上稍微有些区别,
就是这里的测试将每个测试项都作为一个测试点.
这里分3块看:

- wrapped类的数据
  - 这类数据只包含错误链
  - 而且只那第一个参数的错误链和第二个参数做比较
- poser类的数据
  - 这类数据实现了Is()方法集对应的接口
  - 测试函数中Is()的逻辑是判断err1/err3
  - 如果poser实现了Unwrap(),那结果又不一样了
- errorUncomparable类的数据
  - 这个类型的Is,只判断参数是不是errorUncomparable值类型
  - 而且结构体里的数据是字符串切片,不可比较

接下来看TestAs().反射库用的较多,后面再分析.

## 总结

对于errors包:

- errors.New构造的两个错误对象是不相等的,即使错误消息是一样的
- errors.Is(err,targe),是拿err错误链和targe比较,targe是不解封的(层层剥开)

写法上:

- 函数或结构体等引用的部分会在后面出现
  - c/c++中的函数必须前置申明,在Go并不要求如此
- 对于简单的函数的单元测试,主要以表格测试为主(此处主要测试边界)
- 对于待测试的包,相应的测试包名是在后面添加"\_test"
  - eg: io_test  errors_test
- 测试包对于待测试包的引用,可以使用别名".",也可以不用
  - io包用".", erros包就没有用别名
