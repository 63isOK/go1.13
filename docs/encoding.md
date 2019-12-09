# package encoding

说明： 从文档上看，这个包提供一些接口给其他包用，
这些接口做的事是将字节流转换成某种具有表现力的文本，反之亦然。

通俗地讲，就是序列化和反序列化。

这个包提供了4个接口：

    // 序列化成二进制
    type BinaryMarshaler interface {
      MarshalBinary() (data []byte, err error)
    }

    // 二进制格式反序列化成字节流
    type BinaryUnmarshaler interface {
      UnmarshalBinary(data []byte) error
    }

    // 序列化成某种文本格式
    ype TextMarshaler interface {
      MarshalText() (text []byte, err error)
    }

    // 文本格式反序列化成字节流
    type TextUnmarshaler interface {
      UnmarshalText(text []byte) error
    }

像这种反序列化，如果要读取反序列化的结果，需要将入参拷贝出来。
