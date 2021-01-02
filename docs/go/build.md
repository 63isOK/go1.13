# go/build

这个包用于获取Go包的信息.

## 先看文档

先分析导出元素

- 变量ToolDir
- 函数 IsLocalImport,判断import路径是否是本地导入
- 5个type
  - Context,struct,是构建是支持的上下文,默认对象是Default
  - ImportMode,枚举,控制导入方式的行为
  - MultiplePackageError,错误类型,描述一个目录的源码可构建多个不同的包
  - NoGoError,错误类型,描述目录下没有可构建的源码(可能只包含了测试代码)
  - Package,struct,对应Go包的信息
