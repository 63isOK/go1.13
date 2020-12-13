# package encoding/json

## 官方博客加持 json and Go

### 编码

也就是将数据序列化成json格式

    func Marshal(v interface{}) ([]byte, error)

入参可以是Go的结构体等

序列化如果要成功，还有几个前提：

- json对象的key只支持string，Go中的map的key要是string
- channel/complex/function类型不能参与序列化
- 数据结构不能循环，防止无限循环
- 指针参与序列化时，会以指针指向值的身份参与，nil会解析为null

结构体的序列化还有一点，只有暴露的字段才会参与序列化

### 解码

    func Unmarshal(data []byte, v interface{}) error

反序列化，第二个空接口参数需要我们提前申请,入参要是传址

如果`[]byte`里的json数据可以解析为v，默认会按顺序优先进行一一赋值。
当然也是能修改的，现在json有个key是Foo，如果反序列化到那个字段呢：

- 如果字段有个tag叫Foo，那优先匹配
- 如果字段名是Foo
- 如果字段名是FOO或foO，这种情况是大小写有差异

Unmarshal只会反序列化类型能匹配上的字段，如果json数据的结构无法匹配上Go类型，
那就不会参与反序列化, 都会被忽略掉。
这种行为正好符合只想从大json数据中只取小部分数据的场景。

### 使用interface{}来接收反序列化数据

这种很适用于用反序列化时不知道json结构的情况

encoding/json包用 `map[string]interface{}` 和 `[]interface{}` 来存储json的对象和数组，
bool对应json的booleans，
float64对应json的数值，
string对应json的string，
nil对应json的null

### 反序列化任意数据

接收反序列化的对象，可使用空接口类型，在后面的使用，可使用类型断言，
对于具体的值，可使用switch type来处理。

这种方式适合不清楚json数据格式的情况。

### 引用类型

反序列化时，如果Go结构体中有引用类型(map/slice/pointer)，
在申请结构体时不需要初始化这些引用类型，在反序列化的过程中会处理这个。

如果json数据中没有和引用字段对应的数据，那这个引用字段的值还是nil。

利用这点，可以在很多场景中使用

eg: 下面就是支持接收信令和消息

    type IncomingMessage struct {
        Cmd *Command
        Msg *Message
    }

而我们需要做的就是判断cmd或msg是否为空再处理即可，接收反序列化的结构保持不变。

### 流式序列化和反序列化

    func NewDecoder(r io.Reader) *Decoder
    func NewEncoder(w io.Writer) *Encoder

## 序列化分析
