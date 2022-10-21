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

- 如何设计我们的API接口？
- 如何管理项目中的配置？
- 如何做Go工程的包管理？
- 如何结合Go语言特色优雅的处理错误？
- 如何解决单元测试中中间件的依赖问题？
