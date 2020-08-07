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

## 眼前一亮的手法

## 总结
