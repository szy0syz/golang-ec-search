# golang-ec-search

> "eCommerce search"
>
> Golang Elasticsearch Kafka MongoDB MySQL
>
> 还是搞点实战玩玩，要不然看不懂了

## 项目基础

- 如何设计可扩展可维护的工程目录？
  - 坏处：凌乱无序、不易扩张、不可维护
  - 好处：可读性、扩展性、可交流性

标准Go项目的基本文件布局：`cmd`、`internal`、`pkg`，目前这是Go生态中常见的布局形式。

- 工标准目录结构：
- `https://github.com/golang-standards/project-layout`
- 后端代码
  - `/internal` `/cmd`
    - `internal`:
      - 用来分离应用中共享和非共享的内部代码
      - 限制公开程序实体只能被其父目录下的包或者子包引用
    - 为啥 `internal` 限制包导入的目的
      - 讲运营管理服务相关的代码和用户层代码隔离避免误调用 -> 防止安全事故
      - 从使用者来说，目录辨识度非常高 -> 有效杜绝使用者随意乱导入的问题
    - `/cmd`:
      - 程序入口代码，不含业务逻辑
      - 负责程序的启动和关闭以及配置初始化等
      - **cmd下面的子目录名跟你期望生成可执行程序的名需一致**
  - `/vendor` `/third_party`
    - `go mod vendor`
    - `/third_party`
      - 魔改过的第三方包
      - 跟 `vendor` 区分，方便更新
  - `/pkg` `/api`
    - `/pkg` 与 `internal` 是相对的
      - 即外部项目可以直接导入的
      - 可以沉淀整个企业的业务包
      - 还可以作为独立的仓库提供给各个业务组使用
    - `/api`
      - 存放接口定义的文件
      - 例如存放IDL、YML
      - 以及通过这些定义文件生成的client代码
- 项目工具、构建、部署相关的目录
  - Makefile、scripts
    - Makefile
      - 编译工程代码的指令入口
      - 方便使用者对工程进行编译
    - scripts
      - 存放脚本文件，完成构建，安装，分析检查等功能
      - Makefile文件中各个指令的具体实现
  - tools
    - 项目的一些脚本工具
    - 可以调用 /pkg 和 /internal 下面的代码
  - build
    - 主要存放安装包和CI/CD相关的文件
    - 像是Dockerfile
  - deployments
    - 存放系统和容器编排部署配置和模板
    - docker-compose
  - init
    - 应用初始化
    - 比如systemd和进程管理配置

> 真心比看书安逸很多

在Go语言中不建议使用的目录

- `/src`
  - 因为在`GOPATH`模式下，代码会被放到`$GOPATH`下面的src目录下去
  - 则会在导入路径中包含了两个src目录！
- `/model`
  - 不建议将实体或类型定义都放到model目录里
  - 按照业务领域来划分
  - 这里有个标准的划分目录
  - `https://github.com/go-kratos/kratos-layout/tree/main/internal`
-`/common` `/util`
  - 也不推荐以上两个目录
  - 无法看出包的具体功能
  - 而且还容易变成大杂烩

- 如何设计我们的API接口？
- 如何管理项目中的配置？
- 如何做Go工程的包管理？
- 如何解决单元测试中中间件的依赖问题？

业务错误码设计

- 不推荐使用全局错误码 -> 因为有可能跨团队
- 按照项目，服务，模块，错误类型一次编号
- 模块的错误码建议不超过99个
- 错误代码：06100325
  - 06 项目组
  - 10 服务号
  - 03 模块号
  - 25 错误代码

接口的兼容性设计

- `GET /product/v1/search?keyword=abc`
- `GET /product/v2/search?q=abc`

<img width="949" alt="image" src="https://user-images.githubusercontent.com/10555820/197109847-80cbb8e5-fa7a-4d85-a702-505a314b1e8a.png">

Go Module 的包校验

- 为了防止Go Module中的包被篡改
- go.sum文件保存了依赖包的hash值
- GOPRIVATE包将不会做checksum校验

如果想魔改那就 `go module vendor`，在项目根目录生成vendor依赖

### 如何结合Go语言特色优雅的处理错误？

```go
// error的本质
type error interface {
  Error() string
}

type New(text string) error {
  return &errorString{text}
}

type errorString struct {
  s string
}
```

#### 实际项目中对错误处理的一些经验

- 使用errors.Wrap或者errors.Wrapf来保存堆栈信息，包装具体的文件文件路径信息到错误中

```go
path := "path/to/file"
f, error := os.Open(path)
if err != nil {
  return errors.Wrapf(err, "open file %s error", path)
}
```

- 一旦函数确定了错误的处理方案以后，错误就不再是错误
  - 比如出错后我们使用降级方案
  - 则在降级方法执行成功后我们不再返回错误
- errors.Wrap或者errors.Wrapf一般在我们应用中才能使用
  - 一些重用性很高的基础包，一般只能返回最原始的错误
  - "原来用第三方包搞的"，可以从Wrap的错误中解析原始错误
- 一般在程序的最外层才回考虑将错误通过日志的形式保存下来
  - 在程序调用链中，我们直接通过返回error来传递错误
  - "还真是和koa或express里的一样"
- 那么就 "github.com/pkg/errors" 包再进一步进行封装

## ElasticSearch

### `Dynamic Mapping`是特性也是毒性

Dynamic Mapping 特性

- es的mapping类似数据库中的表结构定义
- 向一个不存在的索引写入数据，会根据写入的字段类型创建mapping
- 数据写入失败也会自动创建索引，但不会根据数据字段类型创建mapping

dynamic属性设置不同值，对索引存放有很大的区别

<img width="809" alt="image" src="https://user-images.githubusercontent.com/10555820/197314676-0e4f7314-c3b7-469a-9d36-d4b0b508fc68.png">
<img width="843" alt="image" src="https://user-images.githubusercontent.com/10555820/197314699-2a1b0613-f134-4ceb-854c-b5ad2d17510d.png">

### dynamic特性引发的问题

- 字段数量不可控

> ❌ 默认情况下dynamic的特性会让“索引”根据写入的文档中新增的字段来增加mapping中的字段，对于日志场景中产生大量字段！

- 误操作写入引入无关字段

> ❌ 混淆了写操作和读操作，就会导致检索语句中的查询语义被设置为索引字段

- 集群性能杀手

> ❌ 如果索引字段的大量激增，则原信息会存储到集群的每个节点上，当索引新增字段时，ES必须为每个字段更新集群状态，并且必须将集群中字段传递给所有的节点，对于大规模的集群来说，由于跨节点的集群状态传输是单线程，因此需要更新的字段映射mapping越多完成更新所需要的时间就越长！
> 修改字段数量限制，超过1000个字段的索引就是非常差的！
> "index.mapping.total_fields.limit: 1000"

解决方案

- 开启索引和mapping的严格模式
  - 限制只有系统级的索引可以自动创建
  - dynamic设置为`strict`
- 使用flattened类型
  - 不管多少层都只会当做keyword处理
  - flattened类型支持修改子字段
  - 只能使用term做精确查找

> 没时间了，先pass这部分。
