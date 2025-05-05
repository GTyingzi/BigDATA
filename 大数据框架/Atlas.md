&emsp;<a href="#0">Atlas</a>  
&emsp;&emsp;<a href="#1">1.Atlas概述</a>  
&emsp;&emsp;<a href="#2">2.Atlas架构原理</a>  
## <a name="0">Atlas




### <a name="1">1.Atlas概述


​		Apache Atlas为组织提供开放式元数据管理和治理功能，用以构建其数据资产目录，对这些资产进行分类和管理，服务于数据分析师和数据治理团队，提供围绕这些数据资产的协作功能。

​		Atlas的具体功能如下：

| 元数据分类 | 支持对元数据进行分类管理，例如个人信息，敏感信息等           |
| ---------- | ------------------------------------------------------------ |
| 元数据检索 | 可按照元数据类型、元数据分类进行检索，支持全文检索           |
| 血缘依赖   | 支持表到表和字段之间的血缘依赖，便于进行问题回溯和影响分析等 |

1）表与表之间的血缘依赖

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151612804.png)
2)字段与字段之间的血缘依赖

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151612898.png)
### <a name="2">2.Atlas架构原理

![](https://img-blog.csdnimg.cn/8ef1ac937d3c4ad38b26495b9ea69c28.png)

