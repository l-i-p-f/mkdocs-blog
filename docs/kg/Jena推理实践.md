# Jena推理实践

使用Jena内置的推理机进行本体推理和规则推理。

Jena提供了**RDFS规则推理机**、**OWL推理机**和**通用规则推理机**。

本体推理可用RDFS或OWL推理机；自定义规则推理可用通用规则推理机，支持前向链、表反向链和混合执行模式。



## Jena推理机制

Jena提供的推理机制是，上层应用通过提供的API将数据集与某个推理机关联，从而创建出一个新的Model提供访问。所有的查询建立在新的Model上，查询的内容包括原有数据及推理机推理出的内容。

![reasoner-overview](images\reasoner-overview.png)

<center>推理机制结构</center>

通过*ModelFactory*可建立Ontology/Model和注册推理机。

*Reasoner*通过绑定调用将推理机附加到不同的实例数据集上。

所有查询是通过*Model API*调用在*Reasoner*推理出的*InfGraph*上进行的。



## 推理实践

假设Ontology只有网元和槽两个实体，且有一个hasSlot关系。

目的是根据hasSlot推导出逆关系inNetworkElement。

![](images\ontology2.png)



### 模型构建/实例数据导入

模型构建或数据导入有三种方式，取其一即可。

1. 直接使用 *ModelFactory* API构建默认模型

```java
public static final String NS = "http://base#";

public static Model constructModelManually() {
    Model model = ModelFactory.createDefaultModel();
    
    // model实体
    Resource NetworkElement = model.createResource(NS + "NetworkElement");
    Resource Slot = model.createResource(NS + "Slot");
    // model关系
    Property hasSlot = model.createProperty(NS + "hasSlot");
    
    // 实例数据
    Resource ne = model.createResource(NS + "9701-清水苑小区西北");
    Resource slot = model.createResource(NS + "9701-清水苑小区西北-slot-2");

    // 构造三元组
    model.add(ne, RDF.type, NetworkElement);
    model.add(ne, hasSlot, slot);
    return model;
}
```

（不关心其他实现，可直接跳过2、3）

2. 先构建Ontology，再构造数据

   适用于工程化，提前构建Ontology（定义实体和关系）

```java
@Getter
public class BaseOntology {
    private final OntModel baseRdfModel;
    private final OntClass networkElement;

    BaseOntology() {
        baseRdfModel = ModelFactory.createOntologyModel(OntModelSpec.OWL_DL_MEM);
        networkElement = baseRdfModel.createClass(NS + "NetworkElement");
    }
}

public class LinkOntology extends BaseOntology {
    private final OntModel linkRdfModel;
    private final OntClass slot;
    
    private final OntProperty hasSlot;

    public LinkOntology() {
        super();
        linkRdfModel = getBaseRdfModel();
        slot = linkRdfModel.createClass(NS + "Slot");
        hasSlot = linkRdfModel.createOntProperty(NS + "hasSlot");
        hasSlot.addDomain(getNetworkElement());
        hasSlot.addRange(slot);
    }
}

public static OntModel constructModel() {
    // 获取model
    LinkOntology linkOnt = new LinkOntology();
    OntModel linkModel = linkOnt.getLinkRdfModel();

    // 实例数据
    Resource ne = linkModel.createResource(NS + "9701-清水苑小区西北");
    Resource slot = linkModel.createResource(NS + "9701-清水苑小区西北-slot-2");

    // 构造三元组
    linkModel.add(ne, RDF.type, linkOnt.getNetworkElement());
    linkModel.add(ne, linkOnt.getHasSlot(), slot);
    return linkModel;
}
```

3. 从文件读入数据

   使用*RDFDataMgr* API读取文件，相当于直接构建三元组

```
# data.ttl文件名称

PREFIX : <http://base#>

:9701-清水苑小区西北 
	:hasSlot :9701-清水苑小区西北-slot-2 ;
	.
```

```java
String dataPath = "D:\\...\\data.ttl";
Model model = RDFDataMgr.loadModel(dataPath);
```



### 导入规则

```
# ne_demo.rules文件名称

@prefix : <http://base#>
# @include is unnecessary
@include <OWL>.

[ruleSlotInNe: (?n :hasSlot ?s) -> (?s :inNetworkElement ?n)]
```

```JAVA
String rulesPath = "D:\\...\\ne_demo.rules";
List<Rule> rules = Rule.rulesFromURL(rulesPath);
```



### 定义推理机

推理机的选择很多，初始化方式也不同，比如可以使用

- *ReasonerRegistry* API中的`getTransitiveReasoner`,`getRDFSReasoner`, `getRDFSSimpleReasoner`, `getOWLReasoner`, `getOWLMiniReasoner`, `getOWLMicroReasoner` 方法构建不同的推理机

我使用的是*GenericRuleReasoner*构建通用推理机

```java
GenericRuleReasoner reasoner = new GenericRuleReasoner(rules);
# 以下两行是特殊处理，解释见补充
reasoner.setOWLTranslation(true);	// not needed in RDFS case
reasoner.setTransitiveClosureCaching(true);
```



### 推理

推理直接把推理机与model绑定即可

```java
InfModel infOwl = ModelFactory.createInfModel(reasoner, model);
```



## 结果

```java
# 获取推理后模型数据
List<Statement> infOwlData = infOwl.listStatements().toList();
```

我使用的是上述的第2种方式构建model。

从以下运行结果分析，代码运行后实际我们可以获得两个模型，分别是推理前和推理后的。且能够根据hasSlot关系和自定义规则推理出inNetworkElement关系。

![reason-result](images\reason-result.png)



## 冲突检测

Jena推理机基于本体推理还提供了冲突检测的功能。

但有两点需要注意：

1. 规则文件中需要包含 ```@include <OWL>.```

2. 推理机需要导入OWL规则，即

   ```java
   GenericRuleReasoner reasoner = new GenericRuleReasoner(rules);
   # 以下两行
   reasoner.setOWLTranslation(true);
   reasoner.setTransitiveClosureCaching(true);
   ```

准备好即可开始进行冲突定义。



对于以上数据，我自己构造了如下冲突过程：

1. 通过OWL规范的`disjointWith`逻辑，构造网元与槽是互斥关系
2. 把`ne`既定义为网元，又定义为槽

```java
# 定义冲突
    
public static OntModel constructModel() {
    LinkOntology linkOnt = new LinkOntology();
    OntModel linkModel = linkOnt.getLinkRdfModel();

    // 实例数据
    Resource ne = linkModel.createResource(NS + "9701-清水苑小区西北");
    Resource slot = linkModel.createResource(NS + "9701-清水苑小区西北-slot-2");

    // 构造三元组，构造冲突
    linkModel.add(linkOnt.getNetworkElement(), OWL.disjointWith, linkOnt.getSlot());
    linkModel.add(ne, RDF.type, linkOnt.getNetworkElement());
    linkModel.add(ne, RDF.type, linkOnt.getSlot());
    linkModel.add(ne, linkOnt.getHasSlot(), slot);
    return linkModel;
}
```

```java
public static void consistency_dec(InfModel inf_owl) {
    /**
     * 一致性检测
     */
    System.out.println(" >>> INFO: Start consistency detection");
    ValidityReport validity = inf_owl.validate();
    if (validity.isValid()) {
        System.out.println(" >>> INFO: Consistency validation success");
    } else {
        System.out.println(" >>> ERROR: Inconsistency");
        System.out.println(String.join("", Collections.nCopies(20, "--")));
        for (Iterator<ValidityReport.Report> i = validity.getReports(); i.hasNext(); ) {
            System.out.println(i.next());
        }
        System.out.println(String.join("", Collections.nCopies(20, "==")));
    }
}
```

​	**结果**

![inconsistency](images\inconsistency.png)



## RDFS/OWL与规则同时使用

Jena内置的推理机是只能单独使用的，即只能单独使用RDFSReasoner、OWLReasoner或GenericRuleReasoner 。**因此若需要同时结合RDFS或OWL本体推理与自定义规则推理是需要特殊方式的**。

官方提供了两种方式结合自定义规则推理与RDFS/OWL本体推理。

1. **层级推理**。即使用两次推理机进行推理，把一个推理模型结果作为另一个推理机的输入，从而实现本体与规则的推理。该方式的优点是推理过程独立，可以根据需求定义推理顺序。但缺点是，**只有外层推理模型能看到内层模型的结果而不能反过来**，因此对于推理顺序不同解耦的情况不适用。其次，查询时涉及内外两层数据及推理过程，**提高了查询成本**。
2. **规则覆盖**。创建通用规则推理机`GenericRuleReasoner `，同时包含RDFS或OWL和自定义规则。该方式表面上能解决层级推理的问题。但因为Jena中RDFS和OWL规则集使用的是混合规则引擎，其本身是层级推理的，前向推理规则结果不能看到后向规则的。因此层级推理的问题依然存在。



本案例中使用的是第二种方式。因此，当需要自定义规则推理，同时又需要满足OWL规范时，需要在`rules`文件中使用`@inculde`导入OWL规则。同时，开启推理机的OWL开关。



## 参考

1. [Reasoners and rule engines: Jena inference support](https://jena.apache.org/documentation/inference/#reasonerAPI)

