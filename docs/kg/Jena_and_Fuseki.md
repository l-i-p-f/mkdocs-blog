# Apache Jena
Apache Jena是一个开源的Java语义网框架，用于构建语义网和链接数据应用。
关键词：

- **Jena**：提供了RDFS、OWL和通用规则推理机。
- **TDB**：是Jena用于存储RDF的组件（Native Tuple Store），是属于存储层面的技术。在单机情况下，它能够提供非常高的RDF存储性能。目前TDB的最新版本是TDB2，且与TDB1不兼容。
- **Fuseki**：是Jena提供的SPARQL服务器，也就是SPARQL Endpoint。其提供了**4种运行模式**：单机运行、作为系统的一个服务运行、作为web应用运行或作为一个嵌入式服务器运行。

![image.png](images\Jena_architecture.png)
<font color=Gray><center>Jena框架图</center></font>

# Fuseki
Fuseki单机运行是最常用的使用方式，通过命令挂起一个服务在后台，通过浏览器前端访问http://localhost:3030/，使用其sparql查询功能。
单机运行是通过以下命令挂起fuseki服务



| cmd                                | description                                                  |
| ---------------------------------- | ------------------------------------------------------------ |
| fuseki-server --mem /DBName        | Create an empty, in-memory (non-persistent) dataset.         |
| fuseki-server --file=FILE /DBName  | Create an empty, in-memory (non-persistent) dataset, then load FILE into it. |
| fuseki-server --loc=DBPath /DBName | Use an existing TDB database. Create an empty TDB2 dataset if it does not exist. |
| fuseki-server --config=ConfigFile  | Construct one or more service endpoints based on the configuration description. |


`fuseki-server` 是安装包中的工具

`/DBName` 是自定义数据库名称，它前面的 `/` 不能遗漏，浏览器是通过 `localhost:3030/DBName/query` 接口进行SPARQL查询的
可添加以下可选参数：

- `--port PORT` 指定端口，默认3030
- `--update` 允许更新（通过SPARQL Update或SPARQL HTTP Update）

<br>

## 服务器配置fuseki遇到的问题

**问题**
在Linux服务器，第一次开启fuseki服务时，前端界面显示fuseki管理页面但无法操作数据库

**原因**
初始配置限制了只能本机访问

**解决**
为使本地可通过ip访问服务器的fuseki服务，在服务器上运行一次根目录下的fuseki-server，会在运行目录下（不一定是根目录）得到/run文件夹，修改/run文件夹下的shiro.ini把localhostfilter修改成任何人可访问。
使用##把localhostFilter那行注释掉，把anon那行的##去掉。
![image.png](images\fuseki_server_properties.png)