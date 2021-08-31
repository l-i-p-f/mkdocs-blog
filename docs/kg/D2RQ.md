# D2RQ

The D2RQ Platform is a system for **accessing relational databases as virtual, read-only RDF graphs**. It offers RDF-based access to the content of relational databases without having to replicate it into an RDF store. Using D2RQ you can:

- query a non-RDF database using [SPARQL](http://www.w3.org/TR/sparql11-query/)
- access the content of the database as [Linked Data](http://en.wikipedia.org/wiki/Linked_Data) over the Web
- create custom  dumps of the database in RDF formats for loading into an RDF store
- access information in a non-RDF database using the [Apache Jena API](http://incubator.apache.org/jena/)

D2RQ is Open Source software and published under the [Apache license](http://www.apache.org/licenses/LICENSE-2.0.html).



## Supported Databases ##

官方支持数据库

- Oracle
- MySQL
- PostgreSQL
- SQL Server
- HSQLDB
- Interbase/Firebird

我对接的高斯数据库不在以上列表，但调研以后有说法高斯数据库是在PostgreSQL9.2基础上的魔改，尝试以后发现可以使用D2RQ。

| 数据库信息                                                   |
| :----------------------------------------------------------- |
| DBMS:  Zenith (ver. V3R1)<br />Case  sensitivity: plain=upper, delimited=upper<br />Driver:  Zenith JDBC Driver (ver. V3R1, JDBC4.0) <br />Effective  version: UNKNOWN (ver. 3.1)<br>Ping:  108 ms |
| 高斯数据库 GaussDB<br />高斯数据库并非完全自研，可以算是在PostgreSQL9.2基础上的魔改。<br />ZenithDriver是高斯数据库的驱动 |



## 生成默认mapping

使用D2RQ自带的generate-mapping工具生成默认映射文件

```shell
generate-mapping [-u user] [-p password] [-d driver]
        [-l script.sql] [--[skip-](schemas|tables|columns) list]
        [--w3c] [-v] [-b baseURI] [-o outfile.ttl]
        [--verbose] [--debug]
        jdbcURL
```

`jdbcURL`

JDBC connection URL for the database. Refer to your JDBC driver documentation for the format for your database engine. Examples:

MySQL: `jdbc:mysql://servername/databasename`
PostgreSQL: `jdbc:postgresql://servername/databasename`
Oracle: `jdbc:oracle:thin:@servername:1521:databasename`
HSQLDB: `jdbc:hsqldb:mem:databasename` (in-memory database)
Microsoft SQL Server: `jdbc:sqlserver://servername;databaseName=databasename` (due to the semicolon, the URL must be put in quotes when passed as a command-line argument in Linux/Unix shells)

`-d driver`

The fully qualified Java class name of the database driver. For MySQL, PostgreSQL, and HSQLDB, this argument can be omitted as drivers are already included with D2RQ. For other databases, a driver has to be downloaded from the vendor or a third party. **The jar file containing the JDBC driver class has to be in D2RQ's /lib/db-drivers/ directory**. To find the driver class name, consult the driver documentation. 

Examples:

- Oracle: oracle.jdbc.OracleDriver

- Microsoft SQL Server: com.microsoft.sqlserver.jdbc.SQLServerDriver

**Notice**：**需要使用驱动包连接数据库**

综上，得到命令如下（创建mapping文件夹存放映射文件）：

```shell
generate-mapping -u CMDBCORESVRDB -p *** -o .\mapping\mydb.ttl -d com...jdbc.xxxDriver jdbc:zenith:@ip:port
```



## Mapping Language

### Introduction

The mapping defines a **virtual RDF graph** that contains information from the database. This is similar to the concept of views in SQL, except that the virtual data structure is an RDF graph instead of a virtual relational table. 

The virtual RDF graph can be accessed in various ways, depending on what's offered by the implementation. The D2RQ Platform provides SPARQL access, a Linked Data server, an RDF dump generator, a simple HTML interface, and Jena API access to D2RQ-mapped databases.

**Structure of example D2RQ map**

![mapping](images\mapping.png)

The database is mapped to RDF terms, shown on the right, using `d2rq:ClassMap`s and `d2rq:PropertyBridge`s. The most important objects within the mapping are the class maps. A class map represents a class or a group of similar classes of the ontology. A class map specifies how URIs (or blank nodes) are generated for the instances of the class. It has a set of property bridges, which specify how the properties of an instance are created.

### Mapping与rdf

详细内容请戳[官方文档](http://d2rq.org/d2rq-language#example-refers)，这里仅做基本的简单的入门介绍。

Mapping语法中最基本的是`d2rq:ClassMaps`和`d2rq:PropertyBridge`，ClassMaps用于关联本体和数据表，PropertyBridge用于关联边和实例数据值。用法如下：

```turtle
# 一般mapping的两个步骤：
# 	把数据表定义为ClassMap，对应Ontology中的一个实体对象
#	把表字段的值，对应Ontology中对象的关系和值

# 定义实体
map:ClassMap名称 a d2rq:ClassMap;
	d2rq:dataStorage map:database;
	d2rq:uriPattern "唯一标识符";			 # Ontology实体名称
	d2rq:class ns:Ontology实体;			   # 对应Ontology实体
	d2rq:classDefinitionLabel "实体描述";
	.
# 定义边
map:边名称 a d2rq:PropertyBridge;
	d2rq:belongsToClassMap map:ClassMap名称;	# 对应Ontology实体head
    d2rq:property ns:Ontology关系;			# 对应Ontology关系
    d2rq:propertyDefinitionLabel "关系描述";
    d2rq:column "指定表字段";				  # 对应Ontology实体tail
    .
```

### Demo

假设有如下Ontology需要建立与数据库的联系。

![ontology](images\ontology.png)

数据库名称为CMDBCORESVRDB，涉及两张表I_FIXEDNETWORKELEMENT和I_FIXEDNETWORKCARD，其中hasCard关系涉及**跨表取值**。

有如下mapping映射：

```turtle
# 自动生成
@prefix map: <#> .
@prefix db: <> .
@prefix vocab: <http://my#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix d2rq: <http://www.wiwiss.fu-berlin.de/suhl/bizer/D2RQ/0.1#> .
@prefix jdbc: <http://d2rq.org/terms/jdbc/> .

# 自定义前缀，与Ontology一致
@prefix ns: <http://base#> .

# 自动生成的数据库信息
map:database a d2rq:Database;
	d2rq:jdbcDriver "com...Driver";
	d2rq:jdbcDSN "jdbc:zenith:@ip:port";
	d2rq:username "your_name";
	d2rq:password "***";
	.

# Table CMDBCORESVRDB.I_FIXEDNETWORKELEMENT
map:NetworkElementMap a d2rq:ClassMap;
	d2rq:dataStorage map:database;
	d2rq:uriPattern "http://base#@@CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.name@@";
	# d2rq:uriPattern "CMDBCORESVRDB/I_FIXEDNETWORKCARD/@@CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.name@@";
	d2rq:class ns:NetworkElement;
	d2rq:classDefinitionLabel "Network Element";
	.
# ne has id
map:NetworkElementResId a d2rq:PropertyBridge;
	d2rq:belongsToClassMap map:NetworkElementMap;
    d2rq:property ns:hasNeId;
    d2rq:propertyDefinitionLabel "Network Element Id";
    d2rq:column "CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.nativeId";
    .
# ne has ip
map:NeIp a d2rq:PropertyBridge;
	d2rq:belongsToClassMap map:NetworkElementMap;
	d2rq:property ns:hasNeIp;
	d2rq:propertyDefinitionLabel "Network Element ipAddress";
	d2rq:column "CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.ipAddress";
	.
# ne has card
map:Card a d2rq:PropertyBridge;
	d2rq:belongsToClassMap map:NetworkElementMap;
	d2rq:property ns:hasCard;
	d2rq:propertyDefinitionLabel "Card in NetworkElement";
	# 跨表取值
	d2rq:column "CMDBCORESVRDB.I_FIXEDNETWORKCARD.name";
	d2rq:join "CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.nativeId <= CMDBCORESVRDB.I_FIXEDNETWORKCARD.neNativeId";
	.

# Table CMDBCORESVRDB.I_FIXEDNETWORKCARD
map:CardMap a d2rq:ClassMap;
	d2rq:dataStorage map:database;
	d2rq:uriPattern "http://base#@@CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.name@@-card-@@CMDBCORESVRDB.I_FIXEDNETWORKCARD.name@@";
	d2rq:class ns:Card;
	d2rq:classDefinitionLabel "Card";
	# 跨表取值
	d2rq:join "CMDBCORESVRDB.I_FIXEDNETWORKCARD.neNativeId <= CMDBCORESVRDB.I_FIXEDNETWORKELEMENT.nativeId";
	.
map:CardResId a d2rq:PropertyBridge;
	d2rq:belongsToClassMap map:CardMap;
    d2rq:property ns:hasCardId;
    d2rq:propertyDefinitionLabel "Card Id";
    d2rq:column "CMDBCORESVRDB.I_FIXEDNETWORKCARD.nativeId";
    .

```

定义完成以下mapping，保存即可。可用于导出RDF或启动D2R服务器。



## 导出rdf

### 命令解析

```shell
dump-rdf [-f format] [-b baseURI] [-o outfile.ttl]
        [--verbose] [--debug]
        mapping-file.ttl
```

`mapping-file.ttl`：

​		mapping文件

`-f format` 

保存rdf的格式，支持

- TURTLE
- RDF/XML
- RDF/XML-ABBREV
- N3
- N-TRIPLE（默认），数据库较大时推荐使用N-TRIPLE方式

但N-TRIPLE方式[不显示中文](https://github.com/d2rq/d2rq/issues/312)，中文使用编码存储；存储时逐条显示且前缀不使用@prefix定义

```turtle
# 存储的一条数据示例
<http://base#20868-\u4E1C\u533A\u6C11\u751F\u94F6> <http://base#hasCard> "D3EM8F" .
```

使用TURTLE方式，且不会有以上问题

```turtle
<http://base#45199-登封宣化镇政府集客>
      a       ns:NetworkElement ;
      ns:hasCard "TPM1CXPL" , "CARD PTN 905E GE" , "CARD PTN 905E E1" ;
      ns:hasNeId "4101HWCSANELdba7e2b6d7deb740" ;
      ns:hasNeIp "129.111.177.241" .
```

`--verbose` 

​		打印运行信息（个人建议使用）

`-o outfile` 

​		输出文件

综上，得到导出命令

```shell
dump-rdf.bat -f TURTLE --verbose -o demo_turtle.ttl .\mapping\demo.ttl
```

### 效率

时间效率问题：测试了几次导出rdf时间均有点过长，可能与导出方式（官方推荐N-Triples）和限制[内存](https://github.com/d2rq/d2rq/issues/304)使用有关

效率不高的原因个人认为如下：

1. 数据量过大：如一张LINK表就有12万条数据，仅构建只有5条关系的ontology就导出数据35MB
2. 底层通过SQL查询数据：跨表查询时，表连接的计算量十分大
3. 图谱复杂度

**一个提速方案**

- 把内存限制由1G提升为10G，修改`dump-rdf.bat`文件里的`-Xmx1G`为`-Xmx10G`

```bash
# Before
java -cp %CP% -Xmx1G "-Dlog4j.configuration=%LOGCONFIG%" d2rq.dump_rdf %*

# After
java -cp %CP% -Xmx10G "-Dlog4j.configuration=%LOGCONFIG%" d2rq.dump_rdf %*
```



## 启动D2R SERVER

执行命令

```shell
./d2r-server.bat demo.tll
```

打开 http://localhost:2020/，可以查看数据信息（截图的是默认mapping数据）

![D2R_server](images\D2R_server.PNG)

支持虚拟知识图谱查询，SPARQL Endpoint：（截图的是自定义ontology数据）

http://localhost:2020/snorql/

![sparql_endpoint](images\sparql_endpoint.PNG)



## 总结

关于D2RQ个人认为有以下优缺点：

- 优
  - 支持虚拟知识图谱查询
  - 允许用户通过配置映射表的方式构建RDF

- 缺
  - mapping过程花费精力较多：需要学习mapping规则，根据相应语法建立ontology与数据库的映射关系，需要各种查表找value
  - 导出rdf效率不高：主要是数据量，图谱复杂度，表连接数量的影响

但从技术通用性和用户使用体验角度，个人认为优大于缺。