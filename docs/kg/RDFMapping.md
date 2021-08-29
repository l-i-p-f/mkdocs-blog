# RDF Mapping

知识图谱中的Mapping是指把源结构化/半结构化数据映射为RDF表示。

Mapping的研究方向是

- 以更少的时间和资源（指性能）实现转换
- 更自动化，减少人的干预



## Mapping Method

**关系型数据**转换为RDF形式的标准W3C方式有两种：

1. [Direct Mapping](https://www.w3.org/TR/rdb-direct-mapping/) — 直接映射方式。定义关系数据库与RDF对象的映射关系进行直接映射
2. [R2RML](https://www.w3.org/TR/r2rml/) — RDB to RDF Mapping Language。统一映射语言，定制化Mapping映射

**半结构化数据**转换为RDF表示：

3. [RML](https://www.w3.org/TR/csv2rdf/) — R2RML的扩展，支持半结构化数据，如CSV、JSON、XML等到RDF的映射

<center><b>R2RML vs RML</b></center>

| language              | R2RML                             | RML                                                          |
| --------------------- | --------------------------------- | ------------------------------------------------------------ |
| prefix                | rr                                | rml                                                          |
| URI                   | http://www.w3.org/ns/r2rml#       | http://semweb.mmlab.be/ns/rml#                               |
| relational DBs        | multiple tables<br>one DB         | multiple tables<br/>multiple DBs                             |
| access interfaces     | only ODBC                         | multiple                                                     |
| other data structures | -                                 | tabluar(e.g., CSV, TSV, XLS)<br>hierarchical(e.g., XML, JSON)<br>pair-valued(e.g., wikitext)<br>graphs(e.g., RDF), etc. |
| integration           | materialisation<br>virtualisation | materialisation<br/>virtualisation                           |
| data transformation   | pre-processing                    | pre-processing<br>inline processing                          |



## Mapping Tools

Direct Mapping

- [D2RQ](http://d2rq.org/)：全栈的解决方案。但执行过多的原子SQL查询，逐个查数据库记录，Mapping速度慢

R2RML

- [Ontop](https://ontop-vkg.org/)：可作为`Protégé `插件，也可通过命令行方式使用；Mapping速度快
- [XSPARQL]([http://xsparql.deri.org](http://xsparql.deri.org/))：Intermediate translation into XQuery is used, very slow.
- Virtuoso：商用
- [Ultrawrap](http://www.capsenta.com/)
- [Morph-RDB](https://github.com/jpcik/morph)：[官方测试样例](https://www.w3.org/TR/2012/NOTE-rdb2rdf-implementations-20120814/) 部分不通过。
- Karma

RML

- [RMLMapper](https://github.com/RMLio/rmlmapper-java)：
- [CARML](https://github.com/carml/carml)：尚不支持DB（v0.3.0）

Others

- [Tarql](https://tarql.github.io/)：CSV转换为RDF的工具



## 比较

|                   | D2RQ                                                         | Ontop                                                        | RMLMapper（RML.io）                                          |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据库支持        | Oracle MySQL PostgreSQL SQL Server HSQLDB Interbase/Firebird，支持高斯 | PostgreSQL, MySQL, SQL server, Oracle and DB2，**不支持高斯**，但可扩展 | MySQL, PostgreSQL, Oracle, and SQLServer                     |
| 半结构化数据支持  | 不支持                                                       | 不支持                                                       | CSV files (including CSVW)<br>JSON files (JSONPath)<br>XML files (XPath) |
| Mapping语言通用性 | 不通用                                                       | R2RML                                                        | 通用                                                         |
| Mapping性能       | 慢                                                           | seems：SQL查询优化，快                                       |                                                              |
| API               | Java API                                                     | 弃用                                                         | Java API                                                     |
| SPARQL Endpoint   | 包含                                                         | 包含                                                         | 包含                                                         |
| 虚拟知识图谱      | 支持                                                         | 支持                                                         | 支持                                                         |
| 开源及License     | 开源 Apache-2.0 License                                      | 开源 Apache-2.0 License                                      | 开源 MIT License                                             |
| 社区文档          | 通俗易懂，但很久未维护                                       | 难用；社区较活跃                                             | 较少                                                         |
| 其他              |                                                              | 支持Protege插件                                              |                                                              |



