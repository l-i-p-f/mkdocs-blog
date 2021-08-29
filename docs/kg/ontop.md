# Ontop使用

Ontop与D2RQ功能相似，提供结构化数据的Mapping功能，提供SparqlEndpoint，支持虚拟知识图谱查询。

熟悉Ontop，建议按 [官网流程](https://ontop-vkg.org/tutorial/) 走一遍，h2数据库很轻量级，数据和mapping文件都已经准备好。

## 使用流程

### 1. 配置数据库信息

- 把数据库驱动放在`..\ontop-cli-4.0.3\jdbc`文件夹内

- 准备配置文件`db.properties`，内容如下

```
#Wed Dec 02 11:21:44 CST 2020

jdbc.name=d6ba3997-f6e4-438e-8f8b-b0703e495c80
jdbc.driver=com.huawei.gauss.jdbc.ZenithDriver
jdbc.url=jdbc:zenith:@ip:32081
jdbc.user=CMDBCORESVRDB
jdbc.password=***
```



### 2. 准备mapping文件

Ontop提供了两种mapping映射语言，自定义的obda方式及R2RML。

`mapping.obda`

```
[PrefixDeclaration]
:		http://PTT/faultDiagnosis#
rdf:		http://www.w3.org/1999/02/22-rdf-syntax-ns#


[MappingDeclaration] @collection [[
mappingId	1
target		:{name} a :NetworkElement ; :hasNeId {nativeId} ; :hasNeIp {ipAddress}; :hasCard :{cardName}. 
source		SELECT name, nativeId, ipAddress, (SELECT name FROM I_FIXEDNETWORKCARD WHERE I_FIXEDNETWORKCARD.nativeId=I_FIXEDNETWORKELEMENT.nativeId) AS cardName FROM CMDBCORESVRDB.I_FIXEDNETWORKELEMENT

mappingId	2
target		:{name} a :Card ; :hasCardId :{nativeId} . 
source		SELECT * FROM CMDBCORESVRDB.I_FIXEDNETWORKCARD
]]

```

<br>

`mapping.ttl` ：基于[R2RML](https://www.w3.org/TR/r2rml/)语法。

```
@prefix : <http://PTT/faultDiagnosis#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rr: <http://www.w3.org/ns/r2rml#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

<urn:1_1> a rr:TriplesMap;
  rr:logicalTable [ a rr:R2RMLView;
      rr:sqlQuery "SELECT name, nativeId, ipAddress, (SELECT name FROM I_FIXEDNETWORKCARD WHERE I_FIXEDNETWORKCARD.nativeId=I_FIXEDNETWORKELEMENT.nativeId) AS cardName FROM CMDBCORESVRDB.I_FIXEDNETWORKELEMENT"
    ];
  rr:predicateObjectMap [ a rr:PredicateObjectMap;
      rr:objectMap [ a rr:ObjectMap, rr:TermMap;
          rr:template "http://PTT/faultDiagnosis#{cardName}";
          rr:termType rr:IRI
        ];
      rr:predicate :hasCard
    ], [ a rr:PredicateObjectMap;
      rr:objectMap [ a rr:ObjectMap, rr:TermMap;
          rr:column "nativeId";
          rr:termType rr:Literal
        ];
      rr:predicate :hasNeId
    ], [ a rr:PredicateObjectMap;
      rr:objectMap [ a rr:ObjectMap, rr:TermMap;
          rr:column "ipAddress";
          rr:termType rr:Literal
        ];
      rr:predicate :hasNeIp
    ];
  rr:subjectMap [ a rr:SubjectMap, rr:TermMap;
      rr:class :NetworkElement;
      rr:template "http://PTT/faultDiagnosis#{name}";
      rr:termType rr:IRI
    ] .

<urn:2_1> a rr:TriplesMap;
  rr:logicalTable [ a rr:R2RMLView;
      rr:sqlQuery "SELECT * FROM CMDBCORESVRDB.I_FIXEDNETWORKCARD"
    ];
  rr:predicateObjectMap [ a rr:PredicateObjectMap;
      rr:objectMap [ a rr:ObjectMap, rr:TermMap;
          rr:template "http://PTT/faultDiagnosis#{nativeId}";
          rr:termType rr:IRI
        ];
      rr:predicate :hasCardId
    ];
  rr:subjectMap [ a rr:SubjectMap, rr:TermMap;
      rr:class :Card;
      rr:template "http://PTT/faultDiagnosis#{name}";
      rr:termType rr:IRI
    ] .

```

<br>

两种格式mapping转换

``` 
# obda to r2rml
ontop.bat mapping to-r2rml -i ./demo/mapping.obda -o ./demo/mapping.ttl

# r2rml to obda
ontop.bat mapping to-obda -i ./demo/mapping.ttl -o ./demo/mapping.obda
```



### 3. 准备Ontology文件（可选）

如果有Ontology文件，mapping会检查当前规则是否与Ontology定义的一致。

如Ontology中定义一个object property关系，若mapping中指定其为data property则会报错。



### 4. 启动Endpoint / 执行Mapping

**启动Endpoint**

```
ontop.bat endpoint -m ./demo/mapping.obda -p ./demo/db.properties --cors-allowed-origins=*
```

<br>
其他参数

```
--port <port>
        port of the SPARQL endpoint. Default: 8080

-t <ontology file>
        OWL ontology file   
```

<br>

**执行Mapping**

```
ontop.bat materialize -m ./demo/mapping.obda -f ntriples -o out.ttl -p ./demo/db.properties
```

<br>

参数
```
-f <outputFormat>, --format <outputFormat>
      The format of the materialized ontology. Default: rdfxml

      This options value is restricted to the following set of values:
          rdfxml
          turtle
          ntriples
```

