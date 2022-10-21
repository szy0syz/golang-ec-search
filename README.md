# golang-ec-search

> "eCommerce search"
>
> Golang Elasticsearch Kafka MongoDB MySQL
>
> 还是搞点实战玩玩，要不然看不懂了

## Notes

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

- 如何设计我们的API接口？
- 如何管理项目中的配置？
- 如何做Go工程的包管理？
- 如何结合Go语言特色优雅的处理错误？
- 如何解决单元测试中中间件的依赖问题？
