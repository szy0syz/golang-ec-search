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

## 微服务

为什么要用微服务

- 大厂应用几乎全是基于微服务构建的
- 对微服务的了解碎片化，缺少整体理解
- 微服务构架设计思想能帮我们解决实际问题

设计思想，设计原则，演进过程，拆分方式等

BFF层的由来，AKF扩展立方体

隔离，限流，降级，过载保护，超时控制等

- 提高架构意识
  - 不断的去学习并实践
  - 架构设计是业务驱动的
  - 强化意识，提前考虑

### 在架构设计中需要考虑的因素有哪些？

- 团队
  - 团队人员技术能力
  - 人力资源 -> 其实就是💰
- 技术
  - 当前已有的技术体系
    - 说实话，微服务也不是“银弹”
  - 技术成熟度和掌控能力
- 业务
  - 体量和增量
    - 毛三没几个用户，你微了给谁用呢？
  - 发展方向

### 架构的设计目标

- 解决实际问题
- 提升可用性、可扩展性，控制成本
  - 可用性和成本是相互对立的，那就冗余机制上

### 原则1——避免过度设计

- 避免设计超过实际需求的架构
  - 老哥，必须考虑降低复杂性
  - 又不是越复杂的架构就越好
- 提升可扩展性
- 缩短实现周期节省成本
  - 对了，记得有个 “二八定律”
  - 都说了80%的产出源自20%的投入
  - 所以说，我们要先圈定范围，然后投入80%的时间去设计🚨**它**🚨

### 为什么要强调避免过度设计？

- 实现成本和维护成本都过高，还限制了可扩展性！
  - 不仅是为服务了，啥子项目单体、一张网页也是
- 会脱离当前业务的实际需求，浪费大量人力成本和硬件资源
  - 你要知道，业务永远是不断变化的，架构方案必须跟着业务不断优化和迭代演变
  - 最大可能是推翻重来
- 影响发布计划，高成本低收益
  - 这个么不消说了，眼高手低

> Q: 那么问题来了，如何验证自己的架构方案是否存在过度设计呢？😂 真tm是个灵魂质问
>
> A: 有个方法就是，讲自己的解决方案展示给公司内，不同的技术团队，那参与者是不同经验水平和不同任期的代表来审阅。如果每个技术团队，都能轻松理解这个方案，并且可以在没有人协助的情况下，向其他不知道这个方案的人描述这个方案，而这个其他人可以轻松理解。
>
> 那么我们就可以认为，这个解决方案通过评审。

### 原则2——优先使用成熟的技术

- 没有经过充分验证的技术通常会踩坑
- 学习成本高，实施周期长，解决问题更困难
- 求稳，求快，求低成本
  - 这个求快，我感觉应该是说就 “快速落地” 吧

### 原则3——可扩展原则

#### 扩展性的本质

- 表象是应对不断变化的业务
- 设计合理的模块关系来控制系统的复杂度
  - 也就是将业务变化所带来的系统调整降到最低

#### AFK扩展立方体

<img width="582" alt="image" src="https://user-images.githubusercontent.com/10555820/197338219-88c66d1a-5b72-48e9-b507-b4dc69ac9748.png">

X轴的扩展

- 水平扩展，通过复制实例，负载均衡，分摊整体压力
  - 分摊整体压力位目的就是X轴扩展
- 这个轴的扩展仅适用于产品初期。架构简单，实施快速，研发成本低
- 有状态的服务并不适用

Y轴的扩展

- 根据服务或资源扩展，也是微服务架构演进的思想
- 适用于业务逻辑复杂，团队规模大的场景
- 故障隔离性好、部署快、沟通效率高。实现成本也比较高
  - 这样比较方便实现业务复杂度分解
  - 但也有缺陷，拆分时对于工具的依赖非常高，资源消耗也比较多，运维复杂度也高，移植型的实现成本也比较高

Z轴的扩展

- 其是根据查询或者计算结果的拆分
  - 简单来说，就是 **分片的思想**
  - 当数据规模非常大时，我们就可以通过分片降低整体的压力
  - 比如果ES的分片、Kafka的分区等等
- 适用于大型分布式系统，you并发压力，X轴、Y轴扩展无法解决
- 扩展性更强，架构复杂，实现成本很高
  - 可突破单张表的容量限制，数据迁移也非常复杂

### 原则4——高可用设计原则

- 故障发生前，考虑如何避免
- 故障发生时，考虑怎么做故障转移
- 故障发生后，需要做好复盘总结
  - 复盘不是批斗会，不是追责的，要以开放、包容的心态共同探讨如何不让故障再次发生的行之有效的方案
- 可回滚
  - 噢哟，在架构设计之初还得考虑可回滚方案，有难度哦
- 可禁用
  - 必要时，需要给功能提供禁用的能力
- 限流
- 降级
  - 当发生故障时，放弃非核心的功能，尽量保障核心功能的可用性
- 熔断
  - 保险丝，当某个服务过载故障时，能够自动断开，避免占用大量资源

### 原则5——隔离原则

- 控制故障影响范围。减少服务之间的相互影响

### 原则6——自动化驱动原则

- 70%的生产事故来源于部署变更
- 几乎任何重复性工作都应该自动化
  - **todo**

- 总结
  - 尽可能使用简单的架构来解决问题
  - 架构是不断演化的
    - 架构设计绝不是一步到位

#### 单体架构本身存在哪些问题？

- 模块间耦合度太高
- 架构部署缓慢，维护成本较高
- 迭代进度难统一，边界职责难划分，协作开发成本高

#### 单体架构 VS 微服务架构

- 交互速度
- 故障隔离范围
- 持续演进灵活度
- 沟通效率
- 技术栈选择
- 可扩展性
- 可重用性
- 对于复杂问题分解难度

#### 单体架构怎么发展到微服务架构？

- 对单体应用进行服务化拆分
- 将单体应用中的本地调用抽象成单独的模块后变成远程调用
  - 楼上这句有点不错哦
- 服务化在很大程度上解决了单体应用的不断膨胀导致的问题

#### 什么是微服务

- 在服务化的基础上进一步演化，服务原子化，数据独立化
- 围绕业务能力通过多个小型服务组合构建单个应用的架构风格
- 轻量化通信，自动化部署和运维

#### 什么时候开始微服务化

- 产品初期可以先从单体应用架构开始
- 微服务化试业务发展到一定阶段后被迫去做的
- 确保基础设施及公共基础服务已准备好了

#### 微服务拆分粒度

- 团队规模，决策占比50%
  - 参考三个火枪手的原则，三个人负责一个微服务，
  - 团队规模变大，会出现决策平衡点：所有的决策都要通过某个会议或某个人，没有人愿意承担责任，效率十分低下
- 微服交付速度要求，决策占比30%
- 其他方面，占比20%
  - 对占用资源的要求、对性能的要求、对一致性的要求、对架构运营速度的要求、对创建速度的要求等等

#### 如何衡量拆分的粒度是合适的？

通过复杂度衡量

- 内部复杂度
  - 又称为 “单体复杂度”，指的是单个对象内部的复杂度
  - 可以用参与开发的人数来衡量，三个火枪手原则
- 外部复杂度

> 为什么是三个人？

- 系统的复杂度刚好达到每个人都能全面理解整个系统
- 3个人可以形成一个稳定的备份
- 3个人的技术小组既能够形成有效的讨论，又能够快速达成一致的意见
  - 2个人就有可能你一样，我一样
  - 1个人就形成思维盲区了，都没人跟他讨论么
  - 4+人可能有人摸鱼或只是划水而已 😂
- 这个讨论的是某个微服务的开发阶段，如果稳定了，一个人维护多个微服务也是可以的
