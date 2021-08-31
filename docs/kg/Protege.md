# Protege

Protege是一个用于知识图谱OWL本体建模的工具。

OWL是W3C制定的关于知识图谱的标准语义网知识表示语言，提供逻辑推理能力。（RDF只具表示能力）

OWL在RDF(S)基础上，增加了：

- 对于局部值域的属性定义
- 类、属性、个体的等价性
- 不相交类的定义
- 基数约束
- 关于属性特性的描述



## 操作教程

推荐按序阅读

1. 这里有篇 [博客](https://zhuanlan.zhihu.com/p/136570668) 写得挺细致的，就不再赘述

2. 也可以看 [官方入门使用文档](http://protegeproject.github.io/protege/) 
3. 进阶英文版使用手册见文末附件



## Main Point

主要使用Protege中**Entities**的**Classes**标签来定义类对象，OWL中的类是指一系列个体（Individual）的集合。

在类定义过程中，通常会划分子类，如下图Figure 4.11所示。

其意义是什么？

简单而言就是，类的分级使得定义的事物更直观。而在OWL中其更重要的意义是，表示必要的含义。如图，VegetableTopping是PizzaTopping的子类，那么无须声明，VegetableTopping的所有实例同时也是PizzaTopping的实例。

![](images\meaningOfSubclass.png)



## Class Description

在类定义的过程中，可以使用Description各字段来对类作明确的说明或提供约束。

![](images\ClassDescription.PNG)

各字段作用：

- **Equivalent To** - 指定等价对象。指定后，当前选定类对象与指定对象享有共同的个体。即两个类只是名字不同，表达含义相同。

  同时，OWL允许对对象加以约束/限制，因此，每行可以为一个`class experssion`（[类表达式](http://protegeproject.github.io/protege/class-expression-syntax/)），如上图所示，通过some等关键字实现。**一般，是对类定义有条件限制需求才使用约束。而不是为了加约束，强行增加条件。**

  如上图假设有类定义：` MeatyPizza`是以肉为topping的pizza，则可以使用**已经定义好**的`Pizza`和`MeatTopping`类与`hasTopping`关系定义其为 `Pizza and (hasTopping some MeatTopping)`

  附类表达式语法：

  - `some` -  hasPet `some` Dog - Things that have a pet that is a Dog.
  - `value` - hasPet `value` Tibbs - Things that have a pet that is Tibbs.
  - `and` - Person `and` (hasPet some Cat) - People that have a pet that's a Cat.
  - `or` -  (hasPet some Cat) `or` (hasPet some Dog) - Things that have a pet that's a Cat or have a pet that's a Dog
  - `not` - `not` (hasPet some Dog) - Things that do not have a pet that's a dog.
  - `only` - hasPet `only` Cat -  Things that have pets that are only Cats.
  - `min` \ `max` - hasPet `min` \ `max` 3 Cat -  Things that have at least \ most three pets that are Cats.
  - `exactly` -  hasPet `exactly` 2 GoldFish -  Things that have exactly 2 GoldFish as pets.

- **SubClass Of** - 指定父类。一般自动生成，也可使用类表达式加以约束。

  ![](images\subclassOf.png)

  `NamedPizza`是自动生成的直接父类，其它为约束条件。表示`Cajun`是以`Mozzarella`或`Onion`…为顶的Pizza。（`Pizza`类底下还有很多其他子类，因此不能直接定义`Cajun`为`Pizza`子类）

- **General class axioms** - 通用类公理。

  ```
  官方解释：Each row shows a General Class Axiom that contains the current selected class in its signature (i.e. mentions the current selected class).
  
  This section does not display inferred information.
  ```

- **SubClass Of (Anonymous Ancestor)** - 继承自匿名类。表示该类下所有个体共有的属性。在其父类的`SubClass Of`中定义后自动生成。

  如在`Pizza`父类中定义了`SubClass Of`后，其所有子类将自动生成`SubClass Of (Anonymous Ancestor): hasBase some PizzaBase`

  ![](images\subclassOfAnonymous.png)

- **Instances** - 类实例。其选项为在Individuals中定义好的个体。可用于验证类定义的正确性：推理时，会把满足条件的个体生成到相应Class的Instances选项中。

  如下Individuals中有两个Example对象。其中`ExampleMargherita`的`hasCalorificContentValue`为263，`ExampleQuattroFormaggio`为723，因此`ExampleQuattroFormaggio`满足`HighCaloriePizza`的定义。

  ![](images\instances.png)

- **Target for key**

  ```
  官方解释：Specifies a mixed list of object and data properties that act as a key for instances of the current selected class. Keys are a new feature in OWL 2 and consist of a set of properties. For a given individual the particular values of these properties taken together imply distinctness. For example, a key consisting of hasSurname and hasDateOfBirth could be used (in a limited setting) to imply distinctness of the individuals in the class Person.
  ```

- **Disjoint With** - 指定不相交的类。实例数据的类型只能为当前类型与disjoint with类型中的其一。

- **Disjoint Union Of** - 不交并。与指定类集合的元素不相交。即当前类的个体不可能是Disjoint Union中其他类的个体。

## Object Properties

Protege中对象属性是指连接的两端为两个类对象的属性，即Domain和Range均为class。

![OBJECTPROPERTIES](images\objectProperties.png)

### [Object Property Characteristics](http://protegeproject.github.io/protege/views/object-property-characteristics/) - OWL对象属性特征

OWL通过属性特征来使得属性的意义更加丰富

- **Functional 功能属性：range唯一 ** 

  ![OBJECTPROPERTIES](images\funtional.png)

  <center>多值等同为一个</center>

- **Inverse functional 转置功能属性：domain唯一** 

  ![](images\inverseFunctional.png)

  <center>源结点值唯一</center>

- **Transitive 传递性** 

  ![](images\transitive.png)

- **Symmetric 对称性：关系可逆** ，如朋友、同事、同学…

  ![](images\symmetric.png)

- **Asymmetric 非对称性：关系不能逆** 

  ![](images\asymmetric.png)

  <center>孩子不能hasChild父母</center>

- **Reflexive 自反属性：允许自身到自身的关系** 

  ![reflexive](images\reflexive.png)

  <center>自己认识自己</center>

- **Irreflexive 反自反属性：不允许自已到自己** 

  ![irreflexive](images\irreflexive.png)

  <center>自己不能为自己的母亲/孩子</center>

### [Object Property Description](http://protegeproject.github.io/protege/views/object-property-description/)

之前讨论过的就不再重复。

字段说明：

- **Equivalent To** - 等价边。共享属性定义。

- **Inverse Of** - 指定互逆关系。可以指定多个，有时候自身就是自己的转置属性，如`marry`关系。

- **Doamins(Intersection)** - 边的头结点class类型

  注意：domain和range主要用于推理实例的类型，**不能将其视为一种约束**。同时，当domain和range定义不正确时，会推理出不正确的结果，且不好定位。

  ![](images\domainRange.png)

- **Ranges(Intersection)** - 边的尾结点class类型

  多个Range，似乎会把实例的类型推断为多个Range的交集。

  ![](images\multiRange.png)

- **Disjoint with** - 不相交。如`hasParent`与`hasChild`则为互斥关系，A不能是B孩子的同时又是B的父母。

- **SuperProperty Of(Chain)** 

  ```
  官方解释：The selected property is a super property (i.e. implied by) each chain of properties listed in this section. Note that each property in a given chain is written separated by “o” (lowercase letter O). To add a chain click the add button and in the dialog that appears enter the chain using an alternating pattern of property names and lowercase letter O. Autocompletion may be used to complete property names.
  ```

  

## [Data Properties](http://protegeproject.github.io/protege/views/data-property-hierarchy/)

Protege中数据属性是指Range为数值、字符串、布尔值等**数据类型**的属性。

字段说明略。

### 数据类型

```
Datatype properties link an individual to an XML Schema Datatype(XSD) value or an rdf literal.`
```

XSD是早期XML使用的数据类型，RDF直接沿用了；同时提出RDF Literal来直接表示strings，numbers和dates。

目前，XSD使用较多，可能因其数据类型表达明显。



## 推理机

定义好OWL规范的Ontology后，是可以使用protege的推理机功能的。这也是使用OWL定义Ontology的目的。

推理机主要的两个作用是：

1. 验证当前子类所属父类是否正确，如果不正确会自动调整位置。如把lion定义在bird底下，推理机可以把lion归类回爬行动物或其它定义的类，前提是定义和约束正确的情况下。
2. 一致性检测。推理机能够根据类描述推理出符合该类的实例。同时，能检测出定义不一致/冲突的地方。