# 《ElasticSearch: 权威指南》学习笔记

Elasticsearch 是一个分布式、可扩展、实时的搜索与数据分析引擎。 

Elasticsearch 是一个开源的搜索引擎，建立在一个全文搜索引擎库 [Apache Lucene™](https://lucene.apache.org/core/) 基础之上。 Lucene 可以说是当下最先进、高性能、全功能的搜索引擎库。

Elasticsearch 也是使用 Java 编写的，它的内部使用 Lucene 做索引与搜索，但是它的目的是使全文检索变得简单， 通过隐藏 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API。

然而，Elasticsearch 不仅仅是 Lucene，并且也不仅仅只是一个全文搜索引擎。 它可以被下面这样准确的形容：

- 一个分布式的实时文档存储，*每个字段* 可以被索引与搜索
- 一个分布式实时分析搜索引擎
- 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据



Elasticsearch 尽可能地屏蔽了分布式系统的复杂性。这里列举了一些在后台自动执行的操作：

- 分配文档到不同的容器 或 *分片* 中，文档可以储存在一个或多个节点中
- 按集群节点来均衡分配这些分片，从而对索引和搜索过程进行负载均衡
- 复制每个分片以支持数据冗余，从而防止硬件故障导致的数据丢失
- 将集群中任一节点的请求路由到存有相关数据的节点
- 集群扩容时无缝整合新节点，重新分配分片以便从离群节点恢复



## 先来感受一些例子

匹配字段 `last_name` 为 `Smith` 的对象

```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```



搜索姓氏为 Smith 年龄大于 30  的员工

```
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```



全文检索，搜索所有喜欢攀岩（rock climbing）的员工：

```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}

# 结果
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            ...
            "_score":         0.16273327, 
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_score":         0.016878016, 
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```



短语搜索

仅匹配同时包含 “rock” *和* “climbing” ，并且二者以短语 “rock climbing” 的形式紧挨着的雇员记录

```
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}

# 结果
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         }
      ]
   }
}
```



高亮搜索

增加一个新的 `highlight` 参数

```
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
```

当执行该查询时，返回结果与之前一样，与此同时结果中还多了一个叫做 `highlight` 的部分。这个部分包含了 `about` 属性匹配的文本片段，并以 HTML 标签 `<em></em>` 封装：

```
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>" 
               ]
            }
         }
      ]
   }
}
```



_all字段

这个简单搜索返回包含 `mary` 的所有文档：

```
GET /_search?q=mary
```

以上命令执行时，当索引一个文档的时候，Elasticsearch 取出所有字段的值拼接成一个大的字符串，作为 `_all` 字段进行索引。例如，当索引这个文档时：

```
{
    "tweet":    "However did I manage before Elasticsearch?",
    "date":     "2014-09-14",
    "name":     "Mary Jones",
    "user_id":  1
}

等价于：
"_all": "However did I manage before Elasticsearch? 2014-09-14 Mary Jones 1"
```



## 分析与分析器

*分析* 包含下面的过程：（标准化、归一化）

- 首先，将一块文本分成适合于倒排索引的独立的 *词条* ，
- 之后，将这些词条统一化为标准格式以提高它们的“可搜索性”，或者 *recall*

分析器执行上面的工作。 *分析器* 实际上是将三个功能封装到了一个包里：

- **字符过滤器**

  首先，字符串按顺序通过每个 *字符过滤器* 。他们的任务是在分词前整理字符串。一个字符过滤器可以用来去掉HTML，或者将 `&` 转化成 `and`。

- **分词器**

  其次，字符串被 *分词器* 分为单个的词条。一个简单的分词器遇到空格和标点的时候，可能会将文本拆分成词条。

- **Token 过滤器**

  最后，词条按顺序通过每个 *token 过滤器* 。这个过程可能会改变词条（例如，小写化 `Quick` ），删除词条（例如， 像 `a`， `and`， `the` 等无用词），或者增加词条（例如，像 `jump` 和 `leap` 这种同义词）。

### 内置分析器

但是， Elasticsearch还附带了可以直接使用的预包装的分析器。接下来我们会列出最重要的分析器。为了证明它们的差异，我们看看每个分析器会从下面的字符串得到哪些词条：

```
"Set the shape to semi-transparent by calling set_trans(5)"
```

- **标准分析器**

  标准分析器是Elasticsearch默认使用的分析器。它是分析各种语言文本最常用的选择。它根据 [Unicode 联盟](http://www.unicode.org/reports/tr29/) 定义的 *单词边界* 划分文本。删除绝大部分标点。最后，将词条小写。它会产生

  `set, the, shape, to, semi, transparent, by, calling, set_trans, 5`

- **简单分析器**

  简单分析器在任何不是字母的地方分隔文本，将词条小写。它会产生

  `set, the, shape, to, semi, transparent, by, calling, set, trans`

- **空格分析器**

  空格分析器在空格的地方划分文本。它会产生

  `Set, the, shape, to, semi-transparent, by, calling, set_trans(5)`

- **语言分析器**

  特定语言分析器可用于 [很多语言](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-lang-analyzer.html)。它们可以考虑指定语言的特点。例如， `英语` 分析器附带了一组英语无用词（常用单词，例如 `and` 或者 `the` ，它们对相关性没有多少影响），它们会被删除。 由于理解英语语法的规则，这个分词器可以提取英语单词的 *词干* 。`英语` 分词器会产生下面的词条：`set, shape, semi, transpar, call, set_tran, 5`

  注意看 `transparent`、 `calling` 和 `set_trans` 已经变为词根格式。

### 什么时候使用分析器

当 *索引* 一个文档，它的全文域被分析成词条以用来创建倒排索引。 但是，当我们在全文域 *搜索* 的时候，我们需要将查询字符串通过 *相同的分析过程* ，以保证我们搜索的词条格式与索引中的词条格式一致。

全文查询，理解每个域是如何定义的，因此它们可以做正确的事：

- 当你查询一个 *全文* 域时， 会对查询字符串应用相同的分析器，以产生正确的搜索词条列表。
- 当你查询一个 *精确值* 域时，不会分析查询字符串，而是搜索你指定的精确值。

### 默认分析器

虽然我们可以在字段层级指定分析器，但是如果该层级没有指定任何的分析器，那么我们如何能确定这个字段使用的是哪个分析器呢？

分析器可以从三个层面进行定义：按字段（per-field）、按索引（per-index）或全局缺省（global default）。Elasticsearch 会按照以下顺序依次处理，直到它找到能够使用的分析器。索引时的顺序如下：

- 字段映射里定义的 `analyzer` ，否则
- 索引设置中名为 `default` 的分析器，默认为
- `standard` 标准分析器

在搜索时，顺序有些许不同：

- 查询自己定义的 `analyzer` ，否则
- 字段映射里定义的 `analyzer` ，否则
- 索引设置中名为 `default` 的分析器，默认为
- `standard` 标准分析器

有时，在索引时和搜索时使用不同的分析器是合理的。我们可能要想为同义词建索引（例如，所有 `quick` 出现的地方，同时也为 `fast` 、 `rapid` 和 `speedy` 创建索引）。但在搜索时，我们不需要搜索所有的同义词，取而代之的是寻找用户输入的单词是否是 `quick` 、 `fast` 、 `rapid` 或 `speedy` 。

为了区分，Elasticsearch 也支持一个可选的 `search_analyzer` 映射，它仅会应用于搜索时（ `analyzer` 还用于索引时）。还有一个等价的 `default_search` 映射，用以指定索引层的默认配置。

如果考虑到这些额外参数，一个搜索时的 *完整* 顺序会是下面这样：

- 查询自己定义的 `analyzer` ，否则
- 字段映射里定义的 `search_analyzer` ，否则
- 字段映射里定义的 `analyzer` ，否则
- 索引设置中名为 `default_search` 的分析器，默认为
- 索引设置中名为 `default` 的分析器，默认为
- `standard` 标准分析器

## 映射

> 映射就是定义每个字段的type，决定了该字段的索引方式。

为了能够将时间域视为时间，数字域视为数字，字符串域视为全文或精确值字符串， Elasticsearch 需要知道每个域中数据的类型。这个信息包含在映射中。

索引中每个文档都有 *类型* 。每种类型都有它自己的 *映射* ，或者 *模式定义* 。映射定义了类型中的域，每个域的数据类型，以及Elasticsearch如何处理这些域。映射也用于配置与类型有关的元数据。

### 核心简单域类型

Elasticsearch 支持如下简单域类型：

- 字符串: `string`
- 整数 : `byte`, `short`, `integer`, `long`
- 浮点数: `float`, `double`
- 布尔型: `boolean`
- 日期: `date`

### 自定义域映射

尽管在很多情况下基本域数据类型已经够用，但你经常需要为单独域自定义映射，特别是字符串域。自定义映射允许你执行下面的操作：

- 全文字符串域和精确值字符串域的区别
- 使用特定语言分析器
- 优化域以适应部分匹配
- 指定自定义数据格式
- 还有更多

域最重要的属性是 `type` 。对于不是 `string` 的域，一般无需太多关注。

默认， `string` 类型域会被认为包含全文。就是说，它们的值在索引前，会通过一个分析器，针对于这个域的查询在搜索前也会经过一个分析器。

`string` 域映射的两个最重要属性是 `index` 和 `analyzer` 。

#### index

`index` 属性控制怎样索引字符串。它可以是下面三个值：

- **`analyzed`**

  首先分析字符串，然后索引它。换句话说，以全文索引这个域。

- **`not_analyzed`**

   索引这个域，所以它能够被搜索，但索引的是精确值。不会对它进行分析。

- **`no`**

  不索引这个域。这个域不会被搜索到。

`string` 域 `index` 属性默认是 `analyzed` 。如果我们想映射这个字段为一个精确值，我们需要设置它为 `not_analyzed` ：

```
{
    "tag": {
        "type":     "string",
        "index":    "not_analyzed"
    }
}
```

其他简单类型（例如 `long` ， `double` ， `date` 等）也接受 `index` 参数，但有意义的值只有 `no` 和 `not_analyzed` ， 因为它们永远不会被分析。

#### analyzer

对于 `analyzed` 字符串域，用 `analyzer` 属性指定在搜索和索引时使用的分析器。默认， Elasticsearch 使用 `standard` 分析器， 但你可以指定一个内置的分析器替代它，例如 `whitespace` 、 `simple` 和 `english`：

```
{
    "tweet": {
        "type":     "string",
        "analyzer": "english"
    }
}
```

### 更新映射

当你首次创建一个索引的时候，可以指定类型的映射。你也可以使用 `/_mapping` 为新类型（或者为存在的类型更新映射）增加映射。

尽管你可以 *增加* 一个存在的映射，你不能 *修改* 存在的域映射。如果一个域的映射已经存在，那么该域的数据可能已经被索引。如果你意图修改这个域的映射，索引的数据可能会出错，不能被正常的搜索。

### 内部对象数组

考虑包含内部对象的数组是如何被索引的。 假设我们有个 `followers` 数组：

```js
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
```

这个文档索引前会被扁平化处理，结果如下所示：

```js
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}
```

`{age: 35}` 和 `{name: Mary White}` 之间的相关性已经丢失了，因为每个多值域只是一包无序的值，而不是有序数组。这足以让我们问，“有一个26岁的追随者？”

但是我们不能得到一个准确的答案：“是否有一个26岁 *名字叫 Alex Jones* 的追随者？”



## 最重要的查询

虽然 Elasticsearch 自带了很多的查询，但经常用到的也就那么几个。接下来对最重要的几个查询进行简单介绍。

### match_all 查询

`match_all` 查询简单的匹配所有文档。在没有指定查询方式时，它是默认的查询：

```
{
  "query": {
    "match_all": {}
  }
}
```

它经常与 filter 结合使用—例如，检索收件箱里的所有邮件。所有结果被认为具有相同的相关性，所以都将获得分值为 `1` 的中性 `_score`。

###  match 查询

无论你在任何字段上进行的是全文搜索还是精确查询，`match` 查询是你可用的标准查询。

如果你在一个全文字段上使用 `match` 查询，在执行查询前，它将用正确的分析器去分析查询字符串：

```
{
  "query": {
    "match": {
      "neName": "spine2"
    }
  }
}
```

对于精确值的查询，你可能需要使用 filter 语句来取代 query，因为 filter 将会被缓存。

> 过滤查询（Filtering queries）只是简单的检查包含或者排除，这就使得计算起来非常快。考虑到至少有一个过滤查询（filtering query）的结果是 “稀少的”（很少匹配的文档），并且经常使用不评分查询（non-scoring queries），结果会被缓存到内存中以便快速读取，所以有各种各样的手段来优化查询结果。
>
> 相反，评分查询（scoring queries）不仅仅要找出匹配的文档，还要计算每个匹配文档的相关性，计算相关性使得它们比不评分查询费力的多。同时，查询结果并不缓存。
>
> 通常的规则是，使用查询（query）语句来进行 *全文* 搜索或者其它任何需要影响 *相关性得分* 的搜索。除此以外的情况都使用过滤（filters)。

### multi_match 查询

`multi_match` 查询可以在多个字段上执行相同的 `match` 查询：

```
{
  "query": {
    "multi_match": {
      "query": "pod11-mlagleaf eth-trunk",
      "fields": [
        "neName",
        "name"
      ]
    }
  }
}
```

###  range 查询

`range` 查询找出那些落在指定区间内的数字或者时间：

```
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

### term 查询

`term` 查询被用于精确值匹配，这些精确值可能是数字、时间、布尔或者那些 `not_analyzed` 的字符串：

```
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```

`term` 查询对于输入的文本不 *分析* ，所以它将给定的值进行精确查询。

### terms 查询

`terms` 查询和 `term` 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：

```
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

和 `term` 查询一样，`terms` 查询对于输入的文本不分析。它查询那些精确匹配的值（包括在大小写、重音、空格等方面的差异）。

###  exists 查询和 missing 查询

`exists` 查询和 `missing` 查询被用于查找那些指定字段中有值 (`exists`) 或无值 (`missing`) 的文档。这与SQL中的 `IS_NULL` (`missing`) 和 `NOT IS_NULL` (`exists`) 在本质上具有共性：

```
{
    "exists":   {
        "field":    "title"
    }
}
```

这些查询经常用于某个字段有值的情况和某个字段缺值的情况。



## 组合多查询

你可以用 `bool` 查询来实现你的需求。这种查询将多查询组合在一起，成为用户自己想要的布尔查询。它接收以下参数：

- **`must`**

  文档 *必须* 匹配这些条件才能被包含进来。

- **`must_not`**

  文档 *必须不* 匹配这些条件才能被包含进来。

- **`should`**

  如果满足这些语句中的任意语句，将增加 `_score` ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。

- **`filter`**

  *必须* 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。（功能上等于must，但不计算score）

每一个子查询都独自地计算文档的相关性得分。一旦他们的得分被计算出来， `bool` 查询就将这些得分进行合并并且返回一个代表整个布尔操作的得分。

下面的查询用于查找 `title` 字段匹配 `how to make millions` 并且不被标识为 `spam` 的文档。那些被标识为 `starred` 或在2014之后的文档，将比另外那些文档拥有更高的排名。如果 *两者* 都满足，那么它排名将更高：

```
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}
```

如果没有 `must` 语句，那么至少需要能够匹配其中的一条 `should` 语句。但，如果存在至少一条 `must` 语句，则对 `should` 语句的匹配没有要求。

实现以下SQL

```
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
```

```
GET /my_store/products/_search
{
   "query" : {
      "filtered" : { 
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}}, 
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} 
              ],
              "must_not" : {
                 "term" : {"price" : 30} 
              }
           }
         }
      }
   }
}
```



## 验证查询

查询可以变得非常的复杂，尤其和不同的分析器与不同的字段映射结合时，理解起来就有点困难了。不过 `validate-query` API 可以用来验证查询是否合法。



## 排序与相关性

对于 `filter` 或相关性评分相同的情况，文档是随机返回的。

### 按照字段的值排序

在这个案例中，通过时间来对 tweets 进行排序是有意义的，最新的 tweets 排在最前。 我们可以使用 `sort` 参数进行实现：

```
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}

# 结果
"hits" : {
    "total" :           6,
    "max_score" :       null, 
    "hits" : [ {
        "_index" :      "us",
        "_type" :       "tweet",
        "_id" :         "14",
        "_score" :      null, 
        "_source" :     {
             "date":    "2014-09-24",
             ...
        },
        "sort" :        [ 1411516800000 ] 
    },
    ...
}
```

在该案例中，

_score 不被计算, 因为它并没有用于排序。


date 字段的值表示为自 epoch (January 1, 1970 00:00:00 UTC)以来的毫秒数，通过 sort 字段的值进行返回。

如果需要计算 `_score` ， 可以将 `track_scores` 参数设置为 `true` 。

### 多级排序

假定我们想要结合使用 `date` 和 `_score` 进行查询，并且匹配的结果首先按照日期排序，然后按照相关性排序：

```
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```

排序条件的顺序是很重要的。结果首先按第一个条件排序，仅当结果集的第一个 `sort` 值完全相同时才会按照第二个条件进行排序，以此类推。

多级排序并不一定包含 `_score` 。你可以根据一些不同的字段进行排序，如地理距离或是脚本计算的特定值。

### 多值字段的排序

一种情形是字段有多个值的排序， 需要记住这些值并没有固有的顺序；一个多值的字段仅仅是多个值的包装，这时应该选择哪个进行排序呢？

方案是对每个字段指定排序方式

```
"sort": {
    "dates": {
        "order": "asc",
        "mode":  "min"
    }
}
```

### 字符串排序与多字段

被解析的字符串字段也是多值字段，如果你想分析一个字符串，如 `fine old art` ， 这包含 3 项。

按多值字段排序方法，需要对每一个字段指定排序方式，如 `min` 或 `max`，但是这会导致排序以 `art` 或是 `old` 排在最前。

但我们需要的是以字符串整个字段进行排序，因此该字段应要包含一项：  `not_analyzed` 标识字符串。 但是我们仍需要 `analyzed` 字段，这样才能以全文进行查询。一个简单的方案是在 `_source` 中搞两个相同字段，一个`analyzed` 用于搜索，一个 `not_analyzed` 用于排序。但这样一个字符串就存了两次，会造成空间浪费。同时，以全文 `analyzed` 字段排序会消耗大量的内存。

因此，通过 `fields` 参数解决。

多字段映射

```
"tweet": { 
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": { 
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
```

`tweet` 主字段与之前的一样: 是一个 `analyzed` 全文字段。


新的 `tweet.raw` 子字段是 `not_analyzed`。

现在，只要我们重新索引了我们的数据，使用 `tweet` 字段用于搜索，`tweet.raw` 字段用于排序：

```
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    },
    "sort": "tweet.raw"
}
```

### 相关性

评分的计算方式取决于查询类型，不同的查询语句用于不同的目的：

`fuzzy` 查询会计算与关键词的拼写相似程度

`terms` 查询会计算找到的内容与关键词组成部分匹配的百分比

但是通常我们说的 相关性 是我们用来计算全文本字段的值相对于全文本检索词相似程度的算法。

Elasticsearch 的相似度算法被定义为检索词频率/反向文档频率， *TF/IDF* 。

#### 理解评分标准

Elasticsearch 在 每个查询语句中都有一个 explain 参数，将 `explain` 设为 `true` 就可以得到更详细的信息。

输出 `explain` 结果代价是十分昂贵的，它只能用作调试工具 。千万不要用于生产环境。

```
{
  "query": {
    "multi_match": {
      "query": "pod11 spine2",
      "fields": [
        "neName",
        "name"
      ]
    }
  },
  "explain": "true"
}
```

`explain` 返回了该文档来自于哪个节点哪个分片的信息，这对我们是有帮助的。因为**词频率和文档频率是在每个分片中计算出来的，而不是每个索引中**：

```
	"_shard": "[fan_fi_sink_es_resource_fan_1638118815423][4]",
	"_node": "Rpj0WpLmQryW9D_KO6XaTA",
```

然后它提供了 `_explanation` 。每个入口都包含一个 `description` 、 `value` 、 `details` 字段，它分别告诉你计算的类型、计算结果和任何我们需要的计算细节。

以下可以看到分别计算了 `tf` `idf` 以及 `boost`权重，再求两个词的 ` tf-idf` 分数加和。

```

  "_explanation": {
    "value": 2.8566213,
    "description": "max of:",
    "details": [
      {
        "value": 2.8566213,
        "description": "sum of:",
        "details": [
          {
            "value": 1.1996454,
            "description": "weight(neName:pod11 in 1) [PerFieldSimilarity], result of:",
            "details": [
              {
                "value": 1.1996454,
                "description": "score(freq=1.0), computed as boost * idf * tf from:",
                "details": [
                  {
                    "value": 2.2,
                    "description": "boost",
                    "details": []
                  },
                  {
                    "value": 1.1856236,
                    "description": "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:",
                    "details": [
                      {
                        "value": 5,
                        "description": "n, number of documents containing term",
                        "details": []
                      },
                      {
                        "value": 17,
                        "description": "N, total number of documents with field",
                        "details": []
                      }
                    ]
                  },
                  {
                    "value": 0.45992112,
                    "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:",
                    "details": [
                      {
                        "value": 1,
                        "description": "freq, occurrences of term within document",
                        "details": []
                      },
                      {
                        "value": 1.2,
                        "description": "k1, term saturation parameter",
                        "details": []
                      },
                      {
                        "value": 0.75,
                        "description": "b, length normalization parameter",
                        "details": []
                      },
                      {
                        "value": 2,
                        "description": "dl, length of field",
                        "details": []
                      },
                      {
                        "value": 2.0588236,
                        "description": "avgdl, average length of field",
                        "details": []
                      }
                    ]
                  }
                ]
              }
            ]
          },
          {
            "value": 1.656976,
            "description": "weight(neName:spine2 in 1) [PerFieldSimilarity], result of:",
            "details": [
              {
                "value": 1.656976,
                "description": "score(freq=1.0), computed as boost * idf * tf from:",
                "details": [
                  {
                    "value": 2.2,
                    "description": "boost",
                    "details": []
                  },
                  {
                    "value": 1.6376088,
                    "description": "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:",
                    "details": [
                      {
                        "value": 3,
                        "description": "n, number of documents containing term",
                        "details": []
                      },
                      {
                        "value": 17,
                        "description": "N, total number of documents with field",
                        "details": []
                      }
                    ]
                  },
                  {
                    "value": 0.45992112,
                    "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:",
                    "details": [
                      {
                        "value": 1,
                        "description": "freq, occurrences of term within document",
                        "details": []
                      },
                      {
                        "value": 1.2,
                        "description": "k1, term saturation parameter",
                        "details": []
                      },
                      {
                        "value": 0.75,
                        "description": "b, length normalization parameter",
                        "details": []
                      },
                      {
                        "value": 2,
                        "description": "dl, length of field",
                        "details": []
                      },
                      {
                        "value": 2.0588236,
                        "description": "avgdl, average length of field",
                        "details": []
                      }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
  
```

#### 理解文档是如何被匹配到的

当 `explain` 选项加到某一文档上时， `explain` api 会帮助你理解为何这个文档会被匹配，更重要的是，一个文档为何没有被匹配。



## 分布式检索

### 搜索选项

有几个 查询参数可以影响搜索过程。

#### 偏好

偏好这个参数 `preference` 允许 用来控制由哪些分片或节点来处理搜索请求。 它接受像 `_primary`, `_primary_first`, `_local`, `_only_node:xyz`, `_prefer_node:xyz`, 和 `_shards:2,3` 这样的值, 这些值在 [search preference](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-preference.html) 文档页面被详细解释。

但是最有用的值是某些随机字符串，它可以避免 *bouncing results* 问题。

**Bouncing Results**

想象一下有两个文档有同样值的时间戳字段，搜索结果用 `timestamp` 字段来排序。 由于搜索请求是在所有有效的分片副本间轮询的，那就有可能发生主分片处理请求时，这两个文档是一种顺序， 而副本分片处理请求时又是另一种顺序。

这就是所谓的 *bouncing results* 问题: 每次用户刷新页面，搜索结果表现是不同的顺序。 让同一个用户始终使用同一个分片，这样可以避免这种问题， 可以设置 `preference` 参数为一个特定的任意值比如用户会话ID来解决。

简言之，两个分片数据可能不一致，轮循搜索，可能出现排序结果不同的情况。详见 [相关性不一致](# 相关性（score）不一致)。



## 分片原理

倒排索引相比特定词项出现过的文档列表，会包含更多其它信息。它会保存每一个词项出现过的文档总数， 在对应的文档中一个具体词项出现的总次数，词项在文档中的顺序，每个文档的长度，所有文档的平均长度，等等。

```
Term  | Doc 1 | Doc 2 | Doc 3 | ...
------------------------------------
brown |   X   |       |  X    | ...
fox   |   X   |   X   |  X    | ...
quick |   X   |   X   |       | ...
the   |   X   |       |  X    | ...
```

为了能够实现预期功能，倒排索引需要知道集合中的 *所有* 文档，这是需要认识到的关键问题。

早期的全文检索会为整个文档集合建立一个很大的倒排索引并将其写入到磁盘。 一旦新的索引就绪，旧的就会被其替换，这样最近的变化便可以被检索到。

### 不变性

倒排索引被写入磁盘后是 *不可改变* 的:它永远不会修改。 不变性有重要的价值：

- 不需要锁。如果你从来不更新索引，你就不需要担心多进程同时修改数据的问题。
- 一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。
- 其它缓存(像filter缓存)，在索引的生命周期内始终有效。它们不需要在每次数据改变时被重建，因为数据不会变化。
- 写入单个大的倒排索引允许数据被压缩，减少磁盘 I/O 和 需要被缓存到内存的索引的使用量。

当然，一个不变的索引也有不好的地方。主要事实是它是不可变的! 你不能修改它。如果你需要让一个新的文档 可被搜索，你需要重建整个索引。这要么对一个索引所能包含的数据量造成了很大的限制，要么对索引可被更新的频率造成了很大的限制。

**索引与分片的比较**

被混淆的概念是，一个 *Lucene 索引* 我们在 Elasticsearch 称作 *分片* 。 一个 Elasticsearch *索引* 是分片的集合。 当 Elasticsearch 在索引中搜索的时候， 他发送查询到每一个属于索引的分片(Lucene 索引)，然后像 [*执行分布式检索*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-search.html) 提到的那样，合并每个分片的结果到一个全局的结果集。



## 全文搜索

### 匹配查询

匹配查询 `match` 是个 *核心* 查询。无论需要查询什么字段， `match` 查询都应该会是首选的查询方式。它是一个高级 *全文查询* ，这表示它既能处理全文字段，又能处理精确字段。

这就是说， `match` 查询主要的应用场景就是进行全文搜索，我们以下面一个简单例子来说明全文搜索是如何工作的：

```
# 索引数据
POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "The quick brown fox" }
{ "index": { "_id": 2 }}
{ "title": "The quick brown fox jumps over the lazy dog" }
{ "index": { "_id": 3 }}
{ "title": "The quick brown fox jumps over the quick dog" }
{ "index": { "_id": 4 }}
{ "title": "Brown fox brown dog" }
```

### 单个词查询

```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "QUICK!"
        }
    }
}
```

Elasticsearch 执行上面这个 `match` 查询的步骤是：

1. *检查字段类型* 。

   标题 `title` 字段是一个 `string` 类型（ `analyzed` ）已分析的全文字段，这意味着查询字符串本身也应该被分析。

2. *分析查询字符串* 。

   将查询的字符串 `QUICK!` 传入标准分析器中，输出的结果是单个项 `quick` 。因为只有一个单词项，所以 **`match` 查询执行的是单个底层 `term` 查询。**

3. *查找匹配文档* 。

   用 `term` 查询在倒排索引中查找 `quick` 然后获取一组包含该项的文档，本例的结果是文档：1、2 和 3 。

4. *为每个文档评分* 。

   用 `term` 查询计算每个文档相关度评分 `_score` ，这是种将词频（term frequency，即词 `quick` 在相关文档的 `title` 字段中出现的频率）和反向文档频率（inverse document frequency，即词 `quick` 在所有文档的 `title` 字段中出现的频率），以及字段的长度（即字段越短相关度越高）相结合的计算方式。

```
# 结果
# 文档 1 最相关，因为它的 title 字段更短，即 quick 占据内容的一大部分。
# 文档 3 比 文档 2 更具相关性，因为在文档 3 中 quick 出现了两次。

"hits": [
 {
    "_id":      "1",
    "_score":   0.5, 
    "_source": {
       "title": "The quick brown fox"
    }
 },
 {
    "_id":      "3",
    "_score":   0.44194174, 
    "_source": {
       "title": "The quick brown fox jumps over the quick dog"
    }
 },
 {
    "_id":      "2",
    "_score":   0.3125, 
    "_source": {
       "title": "The quick brown fox jumps over the lazy dog"
    }
 }
]
```

### 多词查询

```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "BROWN DOG!"
        }
    }
}

# 结果
# 文档 4 最相关，因为它包含词 "brown" 两次以及 "dog" 一次。
# 文档 2、3 同时包含 brown 和 dog 各一次，而且它们 title 字段的长度相同，所以具有相同的评分。
# 文档 1 也能匹配，尽管它只有 brown 没有 dog 。

{
  "hits": [
     {
        "_id":      "4",
        "_score":   0.73185337, 
        "_source": {
           "title": "Brown fox brown dog"
        }
     },
     {
        "_id":      "2",
        "_score":   0.47486103, 
        "_source": {
           "title": "The quick brown fox jumps over the lazy dog"
        }
     },
     {
        "_id":      "3",
        "_score":   0.47486103, 
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "1",
        "_score":   0.11914785, 
        "_source": {
           "title": "The quick brown fox"
        }
     }
  ]
}
```

**因为 `match` 查询必须查找两个词（ `["brown","dog"]` ），它在内部实际上先执行两次 `term` 查询，然后将两次查询的结果合并作为最终结果输出。**为了做到这点，它将两个 `term` 查询包入一个 `bool` 查询中。

以上示例告诉我们一个重要信息：即任何文档只要 `title` 字段里包含 指定词项中的至少**一个词 就能匹配，被匹配的词项越多，文档就越相关**。

### 多词AND查询

匹配 `brown AND dog` 的所有文档。即所有词都要满足。

```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": {      
                "query":    "BROWN DOG!",
                "operator": "and"
            }
        }
    }
}
```

### 多词部分匹配查询

在 *所有* 与 *任意* 间二选一有点过于非黑即白。如果用户给定 5 个查询词项，想查找只包含其中 4 个的文档，该如何处理？

`match` 查询支持 `minimum_should_match` 最小匹配参数，这让我们可以指定必须匹配的词项数用来表示一个文档是否相关。我们可以将其设置为某个具体数字，更常用的做法是将其设置为一个百分数，因为我们无法控制用户搜索时输入的单词数量：

```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query":                "quick brown dog",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

组合查询至少匹配两个 `should` ：

```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2 
    }
  }
}
```

### 控制权重

`boost` 参数

```sense
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {  
                    "content": {
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [
                { "match": {
                    "content": {
                        "query": "Elasticsearch",
                        "boost": 3 
                    }
                }},
                { "match": {
                    "content": {
                        "query": "Lucene",
                        "boost": 2 
                    }
                }}
            ]
        }
    }
}
```

>`boost` 参数被用来提升一个语句的相对权重（ `boost` 值大于 `1` ）或降低相对权重（ `boost` 值处于 `0` 到 `1` 之间），但是这种提升或降低并不是线性的，换句话说，如果一个 `boost` 值为 `2` ，并不能获得两倍的评分 `_score` 。
>
>相反，新的评分 `_score` 会在应用权重提升之后被 *归一化* ，每种类型的查询都有自己的归一算法。简单的说，更高的 `boost` 值为我们带来更高的评分 `_score` 。
>
>如果不基于 TF/IDF 要实现自己的评分模型，我们就需要对权重提升的过程能有更多控制，可以使用 [`function_score` 查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/function-score-query.html)操纵一个文档的权重提升方式而跳过归一化这一步骤。



## 相关性（score）不一致

用户会时不时的抱怨无法按相关度排序并提供简短的重现步骤：

> 用户索引了一些文档，运行一个简单的查询，然后发现明显低相关度的结果出现在高相关度结果之上。

为了理解为什么会这样，可以设想，我们在两个主分片上创建了索引和总共 10 个文档，其中 6 个文档有单词 `foo` 。可能是分片 1 有其中 3 个 `foo` 文档，而分片 2 有其中另外 3 个文档，换句话说，所有文档是均匀分布存储的。

Elasticsearch 默认使用的相似度算法是 *词频/逆向文档频率* （TF/IDF） 。词频是计算某个词在当前被查询文档里某个字段中出现的频率，出现的频率越高，文档越相关。 *逆向文档频率* 将*某个词在索引内所有文档出现的百分数* 考虑在内，出现的频率越高，它的权重就越低。

但是由于性能原因， Elasticsearch 不会计算索引内所有文档的 IDF 。相反，每个分片会根据 *该分片* 内的所有文档计算一个本地 IDF 。

因为文档是均匀分布存储的，两个分片的 IDF 是相同的。相反，设想如果有 5 个 `foo` 文档存于分片 1 ，而第 6 个文档存于分片 2 ，在这种场景下， `foo` 在一个分片里非常普通（所以不那么重要），但是在另一个分片里非常出现很少（所以会显得更重要）。这些 IDF 之间的差异会导致不正确的结果。

在实际应用中，这并不是一个问题，本地和全局的 IDF 的差异会随着索引里文档数的增多渐渐消失，在真实世界的数据量下，局部的 IDF 会被迅速均化，所以上述问题并不是相关度被破坏所导致的，而是由于**数据太少**。

为了测试，我们可以通过两种方式解决这个问题。第一种是只在主分片上创建索引，如果只有一个分片，那么本地的 IDF *就是* 全局的 IDF。

第二个方式就是在搜索请求后添加 `?search_type=dfs_query_then_fetch` ， `dfs` 是指 *分布式频率搜索（Distributed Frequency Search）* ， 它告诉 Elasticsearch ，先分别获得每个分片本地的 IDF ，然后根据结果再计算整个索引的全局 IDF 。

> TIP：不要在生产环境上使用 `dfs_query_then_fetch` 。完全没有必要。只要有足够的数据就能保证词频是均匀分布的。没有理由给每个查询额外加上 DFS 这步。



## 多字段搜索

### 多字符串查询

区别评分权重

```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }},
        { "bool":  {
          "should": [
            { "match": { "translator": "Constance Garnett" }},
            { "match": { "translator": "Louise Maude"      }}
          ]
        }}
      ]
    }
  }
}
```

为什么将条件语句放入另一个独立的 `bool` 查询中呢？所有的四个 `match` 查询都是 `should` 语句，所以为什么不将 translator 语句与其他如 title 、 author 这样的语句放在同一层呢？

答案在于评分的计算方式。 `bool` 查询运行每个 `match` 查询，再把评分加在一起，然后将结果与所有匹配的语句数量相乘，最后除以所有的语句数量。处于同一层的每条语句具有相同的权重。在前面这个例子中，包含 translator 语句的 `bool` 查询，只占总评分的三分之一。如果将 translator 语句与 title 和 author 两条语句放入同一层，那么 title 和 author 语句只贡献四分之一评分。

也可用 `boost` 参数。

### 最佳字段

假设有个网站允许用户搜索博客的内容，以下面两篇博客内容文档为例：

```
PUT /my_index/my_type/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}

PUT /my_index/my_type/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}
```

用户输入词组 “Brown fox” 然后点击搜索按钮。事先，我们并不知道用户的搜索项是会在 `title` 还是在 `body` 字段中被找到，但是，用户很有可能是想搜索相关的词组。用肉眼判断，文档 2 的匹配度更高，因为它同时包括要查找的两个词：

现在运行以下 `bool` 查询：

```
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```

但是我们发现查询的结果是文档 1 的评分更高： 

```
{
  "hits": [
     {
        "_id":      "1",
        "_score":   0.14809652,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     },
     {
        "_id":      "2",
        "_score":   0.09256032,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     }
  ]
}
```

为了理解导致这样的原因，需要回想一下 `bool` 是如何计算评分的：

1. 它会执行 `should` 语句中的两个查询。
2. 加和两个查询的评分。
3. 乘以匹配语句的总数。
4. 除以所有语句总数（这里为：2）。

文档 1 的两个字段都包含 `brown` 这个词，所以两个 `match` 语句都能成功匹配并且有一个评分。文档 2 的 `body` 字段同时包含 `brown` 和 `fox` 这两个词，但 `title` 字段没有包含任何词。这样， `body` 查询结果中的高分，加上 `title` 查询中的 0 分，然后乘以二分之一，就得到比文档 1 更低的整体评分。

在本例中， `title` 和 `body` 字段是相互竞争的关系，所以就需要找到单个 *最佳匹配* 的字段。

如果不是简单将每个字段的评分结果加在一起，而是将 *最佳匹配* 字段的评分作为查询的整体评分，结果会怎样？这样返回的结果可能是： *同时* 包含 `brown` 和 `fox` 的单个字段比反复出现相同词语的多个不同字段有更高的相关度。

#### dis_max 查询

`dis_max` 即分离 *最大化查询（Disjunction Max Query）* ，指的是： *将任何与任一查询匹配的文档作为结果返回，但只将最佳匹配的评分作为查询的评分结果返回* ：

```
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```

得到我们想要的结果为：

```
{
  "hits": [
     {
        "_id":      "2",
        "_score":   0.21509302,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id":      "1",
        "_score":   0.12713557,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     }
  ]
}
```

#### 最佳字段查询调优

但当没有一个文档完全匹配 `dis_max` 的，即如搜索 `quick pets` 时，两个文档都包含词 `quick` ，但是只有文档 2 包含词 `pets` ，两个文档中都不具有同时包含 *两个词* 的 *相同字段* 。

此时，`dis_max` 查询会对两个文档返回相同的得分。即 `dis_max` 查询只会简单地使用 *单个* 最佳匹配语句的评分 `_score` 作为整体评分。

但显然，文档2同时包含了 `quick` 和 `pet` 两个词，应该得到更高的分数。

**tie_breaker 参数**

可以通过指定 `tie_breaker` 这个参数将其他匹配语句的评分也考虑其中：

```
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ],
            "tie_breaker": 0.3
        }
    }
}

# 结果
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.14757764, 
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id": "1",
        "_score": 0.124275915, 
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     }
   ]
}
```

`tie_breaker` 参数提供了一种 `dis_max` 和 `bool` 之间的折中选择，它的评分方式如下：

1. 获得最佳匹配语句的评分 `_score` 。
2. 将其他匹配语句的评分结果与 `tie_breaker` 相乘。
3. 对以上评分求和并规范化。

有了 `tie_breaker` ，会考虑所有匹配语句，但最佳匹配语句依然占最终结果里的很大一部分。

>NOTE：最佳的精确值需要根据数据与查询调试得出，但是合理值应该与零接近（处于 `0.1 - 0.4` 之间），这样就不会颠覆 `dis_max` 最佳匹配性质的根本。

### 查询字段名称模糊匹配

字段名称可以用模糊匹配的方式给出：任何与模糊模式正则匹配的字段都会被包括在搜索条件中，例如可以使用以下方式同时匹配 `book_title` 、 `chapter_title` 和 `section_title` （书名、章名、节名）这三个字段：

```
{
    "multi_match": {
        "query":  "Quick brown fox",
        "fields": "*_title"
    }
}
```

### multi_match提升单个字段权重

可以使用 `^` 字符语法为单个字段提升权重，在字段名称的末尾添加 `^boost` ，其中 `boost` 是一个浮点数：

```
{
    "multi_match": {
        "query":  "Quick brown fox",
        "fields": [ "*_title", "chapter_title^2" ] 
    }
}
```

`chapter_title` 这个字段的 `boost` 值为 `2` ，而其他两个字段 `book_title` 和 `section_title` 字段的默认 boost 值为 `1` 。

### 多数字段搜索

全文搜索被称作是 *召回率（Recall）* 与 *精确率（Precision）* 的战场： *召回率* ——返回所有的相关文档； *精确率* ——不返回无关文档。目的是在结果的第一页中为用户呈现最为相关的文档。

为了提高召回率的效果，我们扩大搜索范围——不仅返回与用户搜索词精确匹配的文档，还会返回我们认为与查询相关的所有文档。如果一个用户搜索 “quick brown box” ，一个包含词语 `fast foxes` 的文档被认为是非常合理的返回结果。

如果包含词语 `fast foxes` 的文档是能找到的唯一相关文档，那么它会出现在结果列表的最上面，但是，如果有 100 个文档都出现了词语 `quick brown fox` ，那么这个包含词语 `fast foxes` 的文档当然会被认为是次相关的，它可能处于返回结果列表更下面的某个地方。当包含了很多潜在匹配之后，我们需要将最匹配的几个置于结果列表的顶部。

提高全文相关性精度的常用方式是为同一文本建立多种方式的索引，每种方式都提供了一个不同的相关度信号 *signal* 。主字段会以尽可能多的形式的去匹配尽可能多的文档。举个例子，我们可以进行以下操作：

- 使用词干提取来索引 `jumps` 、 `jumping` 和 `jumped` 样的词，将 `jump` 作为它们的词根形式。这样即使用户搜索 `jumped` ，也还是能找到包含 `jumping` 的匹配的文档。
- 将同义词包括其中，如 `jump` 、 `leap` 和 `hop` 。
- 移除变音或口音词：如 `ésta` 、 `está` 和 `esta` 都会以无变音形式 `esta` 来索引。

尽管如此，如果我们有两个文档，其中一个包含词 `jumped` ，另一个包含词 `jumping` ，用户很可能期望前者能排的更高，因为它正好与输入的搜索条件一致。

**多字段映射**

首先要做的事情就是对我们的字段索引两次：一次使用词干模式以及一次非词干模式。为了做到这点，采用 *multifields* 来实现：

```sense
DELETE /my_index
PUT /my_index
{
    "settings": { "number_of_shards": 1 }, 
    "mappings": {
        "my_type": {
            "properties": {
                "title": { 
                    "type":     "string",
                    "analyzer": "english",
                    "fields": {
                        "std":   { 
                            "type":     "string",
                            "analyzer": "standard"
                        }
                    }
                }
            }
        }
    }
}

title 字段使用 english 英语分析器来提取词干。
title.std 字段使用 standard 标准分析器，所以没有词干提取。
```

索引文档

```sense
PUT /my_index/my_type/1
{ "title": "My rabbit jumps" }

PUT /my_index/my_type/2
{ "title": "Jumping jack rabbits" }
```

如果用一个简单 `match` 查询 `title` 标题字段是否包含 `jumping rabbits` ，因为用了 `english` 分析器，这个查询是在查找以 `jump` 和 `rabbit` 这两个被提取词的文档。两个文档的 `title` 字段都同时包括这两个词，所以两个文档得到的评分也相同。

如果只是查询 `title.std` 字段，那么只有文档 2 是匹配的。

如果同时查询两个字段，然后使用 `bool` 查询将评分结果 *合并* ，那么两个文档都是匹配的（ `title` 字段的作用），而且文档 2 的相关度评分更高（ `title.std` 字段的作用）：

```
GET /my_index/_search
{
   "query": {
        "multi_match": {
            "query":  "jumping rabbits",
            "type":   "most_fields", 
            "fields": [ "title", "title.std" ]
        }
    }
}
```

### 跨字段搜索

依次查询每个字段并将每个字段的匹配评分结果相加：

```js
{
  "query": {
    "bool": {
      "should": [
        { "match": { "street":    "Poland Street W1V" }},
        { "match": { "city":      "Poland Street W1V" }},
        { "match": { "country":   "Poland Street W1V" }},
        { "match": { "postcode":  "Poland Street W1V" }}
      ]
    }
  }
}
```

使用 `most_fields` 

```js
{
  "query": {
    "multi_match": {
      "query":       "Poland Street W1V",
      "type":        "most_fields",
      "fields":      [ "street", "city", "country", "postcode" ]
    }
  }
}
```

`most_fields` 查询的执行方式：Elasticsearch 为每个字段生成独立的 `match` 查询，再用 `bool` 查询将他们包起来。

**most_fields 方式的问题**

用 `most_fields` 这种方式搜索也存在某些问题：

- 它是为多数字段匹配 *任意* 词设计的，而不是在 *所有字段* 中找到最匹配的。

  - 生成 `explanation` 解释：

    ```
    (street:poland   street:street   street:w1v)
    (city:poland     city:street     city:w1v)
    (country:poland  country:street  country:w1v)
    (postcode:poland postcode:street postcode:w1v)
    ```

    可以发现，*两个* 字段都与 `poland` 匹配的文档要比一个字段同时匹配 `poland` 与 `street` 文档的评分高。

- 它不能使用 `operator` 或 `minimum_should_match` 参数来降低次相关结果造成的长尾效应。

- 词频对于每个字段是不一样的，而且它们之间的相互影响会导致不好的排序结果。

增加一个字段可解决上述问题，如查询姓名时：

```
{
    "first_name":  "Peter",
    "last_name":   "Smith",
    "full_name":   "Peter Smith"
}
```

当查询 `full_name` 字段时：

- 具有更多匹配词的文档会比只有一个重复匹配词的文档更重要。
- `minimum_should_match` 和 `operator` 参数会像期望那样工作。
- 姓和名的逆向文档频率被合并，所以 Smith 到底是作为姓还是作为名出现，都会变得无关紧要。

这么做当然是可行的，但我们并不太喜欢存储冗余数据。取而代之的是 Elasticsearch 可以提供两个解决方案——一个在索引时，而另一个是在搜索时——随后会讨论它们。

### 自定义_all字段

在 [all-field](https://www.elastic.co/guide/cn/elasticsearch/guide/current/root-object.html#all-field) 字段中，我们解释过 `_all` 字段的索引方式是将所有其他字段的值作为一个大字符串索引的。然而这么做并不十分灵活，为了灵活我们可以给人名添加一个自定义 `_all` 字段，再为地址添加另一个 `_all` 字段。

Elasticsearch 在字段映射中为我们提供 `copy_to` 参数来实现这个功能：

```sense
PUT /my_index
{
    "mappings": {
        "person": {
            "properties": {
                "first_name": {
                    "type":     "string",
                    "copy_to":  "full_name",
                    "fields": {
                        "raw": {
                            "type": "string",
                            "index": "not_analyzed"
                        }
                },
                "last_name": {
                    "type":     "string",
                    "copy_to":  "full_name" 
                },
                "full_name": {
                    "type":     "string"
                }
            }
        }
    }
}
```

`first_name` 和 `last_name` 字段中的值会被复制到 `full_name` 字段。

但 `copy_to` 对 `multi-field` 无效，即不能放到 `fields` 字段里。`copy_to` 是针对“主”字段，而不是多字段的。

### cross-fields 跨字段查询

自定义 `_all` 的方式是一个好的解决方案，只需在索引文档前为其设置好映射。不过， Elasticsearch 还在搜索时提供了相应的解决方案：使用 `cross_fields` 类型进行 `multi_match` 查询。

 `cross_fields` 使用词中心式（term-centric）的查询方式，这与 `best_fields` 和 `most_fields` 使用字段中心式（field-centric）的查询方式非常不同，它将所有字段当成一个大字段，并在 *每个字段* 中查找 *每个词* 。

为了说明字段中心式（field-centric）与词中心式（term-centric）这两种查询方式的不同，先看看以下字段中心式的 `most_fields` 查询的 `explanation` 解释：

```sense
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "most_fields",
            "operator":    "and", 
            "fields":      [ "first_name", "last_name" ]
        }
    }
}
```

对于匹配的文档， `peter` 和 `smith` 都必须同时出现在相同字段中，要么是 `first_name` 字段，要么 `last_name` 字段：

```
(+first_name:peter +first_name:smith)
(+last_name:peter  +last_name:smith)
```

*词中心式* 会使用以下逻辑：

```
+(first_name:peter last_name:peter)
+(first_name:smith last_name:smith)
```

换句话说，词 `peter` 和 `smith` 都必须出现，但是可以出现在任意字段中。

`cross_fields` 类型首先分析查询字符串并生成一个词列表，然后它从所有字段中依次搜索每个词。这种不同的搜索方式很自然的解决了 [字段中心式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/field-centric.html) 查询三个问题中的二个。剩下的问题是逆向文档频率不同。

幸运的是 `cross_fields` 类型也能解决这个问题，通过 `validate-query` 可以看到：

```sense
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields", 
            "operator":    "and",
            "fields":      [ "first_name", "last_name" ]
        }
    }
}
```

它通过 *混合* 不同字段逆向索引文档频率的方式解决了词频的问题：

```
+blended("peter", fields: [first_name, last_name])
+blended("smith", fields: [first_name, last_name])
```

换句话说，它会同时在 `first_name` 和 `last_name` 两个字段中查找 `smith` 的 IDF ，然后用两者的最小值作为两个字段的 IDF 。结果实际上就是 `smith` 会被认为既是个平常的姓，也是平常的名。

>NOTE: 为了让 `cross_fields` 查询以最优方式工作，所有的字段都须使用相同的分析器，具有相同分析器的字段会被分组在一起作为混合字段使用。
>
>如果包括了不同分析链的字段，它们会以 `best_fields` 的相同方式被加入到查询结果中。例如：我们将 `title` 字段加到之前的查询中（假设它们使用的是不同的分析器）， explanation 的解释结果如下：
>
>```
>(+title:peter +title:smith)
>(
>  +blended("peter", fields: [first_name, last_name])
>  +blended("smith", fields: [first_name, last_name])
>)
>```
>
>当在使用 `minimum_should_match` 和 `operator` 参数时，这点尤为重要。

采用 `cross_fields` 查询与 自定义 `_all` 字段 相比，其中一个优势就是它可以在搜索时为单个字段提升权重。可使用 `^` 符号语法来实现。

自定义单字段查询是否能够优于多字段查询，取决于在多字段查询与单字段自定义 `_all` 之间代价的权衡，即哪种解决方案会带来更大的性能优化就选择哪一种。

在 `multi_match` 查询中避免使用 `not_analyzed` 字段，因为 `not_analyzed` 的字段索引只有一个原始文本，如将 `title` 字段设置成 `not_analyzed` ：

```sense
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields",
            "fields":      [ "title", "first_name", "last_name" ]
        }
    }
}
```

显然，你不能在 `first_name` 的索引中找到 `peter smith`，也不能在 `title` 的索引中找到 `peter`。



## 近似匹配

使用 TF/IDF 的标准全文检索将文档或者文档中的字段作一大袋的词语处理。 `match` 查询可以告知我们这大袋子中是否包含查询的词条，但却无法告知词语之间的关系。即它不能确定这两个词是否只来自于一种语境，甚至都不能确定是否来自于同一个段落。

理解分词之间的关系是一个复杂的难题，我们也无法通过换一种查询方式去解决。但我们至少可以通过出现在彼此附近或者仅仅是彼此相邻的分词来判断一些似乎相关的分词。

这就是短语匹配或者近似匹配的所属领域。

### 短语匹配

**什么是短语**

一个被认定为和短语 `quick brown fox` 匹配的文档，必须满足以下这些要求：

- `quick` 、 `brown` 和 `fox` 需要全部出现在域中。
- `brown` 的位置应该比 `quick` 的位置大 `1` 。
- `fox` 的位置应该比 `quick` 的位置大 `2` 。

```sense
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": "quick brown fox"
        }
    }
}

或

"match": {
    "title": {
        "query": "quick brown fox",
        "type":  "phrase"
    }
}
```

类似 `match` 查询， `match_phrase` 查询首先将查询字符串解析成一个词项列表，然后对这些词项进行搜索，但只保留那些包含 *全部* 搜索词项，且 *位置* 与搜索词项相同的文档。

### 短语混合匹配

精确短语匹配 或许是过于严格了。也许我们想要包含 “quick brown fox” 的文档也能够匹配 “quick fox,” ， 尽管情形不完全相同。

我们能够通过使用 `slop` 参数将灵活度引入短语匹配中：

```sense
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": {
            	"query": "quick fox",
            	"slop":  1
            }
        }
    }
}
```

`slop` 参数告诉 `match_phrase` 查询词条相隔多远时仍然能将文档视为匹配 。 

尽管在使用了 `slop` 短语匹配中所有的单词都需要出现， 但是这些单词也不必为了匹配而按相同的序列排列。 有了足够大的 `slop` 值， 单词就能按照任意顺序排列了。

为了使查询 `fox quick` 匹配我们的文档， 我们需要 `slop` 的值为 `3`:

```
            Pos 1         Pos 2         Pos 3
-----------------------------------------------
Doc:        quick         brown         fox
-----------------------------------------------
Query:      fox           quick
Slop 1:     fox|quick  ↵  
Slop 2:     quick      ↳  fox
Slop 3:     quick                 ↳     fox
```

 注意 `fox` 和 `quick` 在这步中占据同样的位置。 因此将 `fox quick` 转换顺序成 `quick fox` 需要两步， 或者值为 `2` 的 `slop` 。

同时，`slop` 邻近查询也将查询词条的邻近度考虑到最终相关度 `_score` 中，两个词距离越近分数越高。

### 多值字段

对多值字段使用短语匹配时会发生奇怪的事。 想象一下你索引这个文档:

```sense
PUT /my_index/groups/1
{
    "names": [ "John Abraham", "Lincoln Smith"]
}
```

然后运行一个对 `Abraham Lincoln` 的短语查询:

```sense
GET /my_index/groups/_search
{
    "query": {
        "match_phrase": {
            "names": "Abraham Lincoln"
        }
    }
}
```

令人惊讶的是， 即使 `Abraham` 和 `Lincoln` 在 `names` 数组里属于两个不同的人名， 我们的文档也匹配了查询。 这一切的原因在Elasticsearch数组的索引方式。

在分析 `John Abraham` 的时候， 产生了如下信息：

- Position 1: `john`
- Position 2: `abraham`

然后在分析 `Lincoln Smith` 的时候， 产生了：

- Position 3: `lincoln`
- Position 4: `smith`

换句话说， Elasticsearch对以上数组分析生成了与分析单个字符串 `John Abraham Lincoln Smith` 一样几乎完全相同的语汇单元。 我们的查询示例寻找相邻的 `lincoln` 和 `abraham` ， 而且这两个词条确实存在，并且它们俩正好相邻， 所以这个查询匹配了。

使用 `position_increment_gap` 的解决方案避免以上问题。

```sense
DELETE /my_index/groups/ 

PUT /my_index/_mapping/groups 
{
    "properties": {
        "names": {
            "type":                "string",
            "position_increment_gap": 100
        }
    }
}
```

`position_increment_gap` 设置告诉 Elasticsearch 应该为数组中每个新元素增加当前词条 `position` 的指定值。 所以现在当我们再索引 names 数组时，会产生如下的结果：

- Position 1: `john`
- Position 2: `abraham`
- Position 103: `lincoln`
- Position 104: `smith`

现在我们的短语查询可能无法匹配该文档因为 `abraham` 和 `lincoln` 之间的距离为 100 。 为了匹配这个文档你必须添加值为 100 的 `slop` 。

### 使用邻近度提高相关性

虽然邻近查询很有用， 但是所有词条都出现在文档的要求过于严格了。 如果七个词条中有六个匹配， 那么这个文档对用户而言就已经足够相关了， 但是 `match_phrase` 查询可能会将它排除在外。

相比将使用邻近匹配作为绝对要求，我们可以将它作为许多潜在查询中的一个，对每个文档的最终分值做出贡献。

我们可以将一个简单的 `match` 查询作为一个 `must` 子句。 这个查询将决定哪些文档需要被包含到结果集中。 我们可以用 `minimum_should_match` 参数去除长尾。 然后我们可以以 `should` 子句的形式添加更多特定查询。 每一个匹配成功的都会增加匹配文档的相关度。

```sense
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { 
          "title": {
            "query":                "quick brown fox",
            "minimum_should_match": "30%"
          }
        }
      },
      "should": {
        "match_phrase": { 
          "title": {
            "query": "quick brown fox",
            "slop":  50
          }
        }
      }
    }
  }
}
```

### 性能优化

短语查询和邻近查询都比简单的 `query` 查询代价更高。一个 `match` 查询仅仅是看词条是否存在于倒排索引中，而一个 `match_phrase` 查询是必须计算并比较多个可能重复词项的位置。

[Lucene nightly benchmarks](http://people.apache.org/~mikemccand/lucenebench/) 表明一个简单的 `term` 查询比一个短语查询大约快 10 倍，比邻近查询(有 `slop` 的短语 查询)大约快 20 倍。当然，这个代价指的是在搜索时而不是索引时。

通常，短语查询的额外成本并不像这些数字所暗示的那么吓人。事实上，性能上的差距只是证明一个简单的 `term` 查询有多快。标准全文数据的短语查询通常在几毫秒内完成，因此实际上都是完全可用，即使是在一个繁忙的集群上。

如何限制短语查询和邻近近查询的性能消耗呢？一种有用的方法是减少需要通过短语查询检查的文档总数。

**结果集重新评分**

在先前的章节中，我们讨论了使用邻近查询来调整相关度，而不是使用它将文档从结果列表中添加或者排除。一个查询可能会匹配成千上万的结果，但我们的用户很可能只对结果的前几页感兴趣。

一个简单的 `match` 查询已经通过排序把包含所有含有搜索词条的文档放在结果列表的前面了。事实上，我们只想对这些 *顶部文档* 重新排序，来给同时匹配了短语查询的文档一个额外的相关度升级。

`search` API 通过 *重新评分* 明确支持该功能。重新评分阶段支持一个代价更高的评分算法—比如 `phrase` 查询—只是为了从每个分片中获得前 `K` 个结果。 然后会根据它们的最新评分 重新排序。

该请求如下所示：

```sense
GET /my_index/my_type/_search
{
    "query": {
        "match": {  
            "title": {
                "query":                "quick brown fox",
                "minimum_should_match": "30%"
            }
        }
    },
    "rescore": {
        "window_size": 50, 
        "query": {         
            "rescore_query": {
                "match_phrase": {
                    "title": {
                        "query": "quick brown fox",
                        "slop":  50
                    }
                }
            }
        }
    }
}
```

`match` 查询决定哪些文档将包含在最终结果集中，并通过 TF/IDF 排序。

`window_size` 是每一分片进行重新评分的顶部文档数量。

### 寻找相关词

短语查询和邻近查询都很好用，但仍有一个缺点。它们过于严格了：为了匹配短语查询，所有词项都必须存在，即使使用了 `slop` 。

用 `slop` 得到的单词顺序的灵活性也需要付出代价，因为失去了单词对之间的联系。即使可以识别 `sue` 、 `alligator` 和 `ate` 相邻出现的文档，但无法分辨是 *Sue ate* 还是 *alligator ate* 。

当单词相互结合使用的时候，表达的含义比单独使用更丰富。两个子句 *I’m not happy I’m working* 和 *I’m happy I’m not working* 包含相同 的单词，也拥有相同的邻近度，但含义截然不同。

如果索引单词对而不是索引独立的单词，就能对这些单词的上下文尽可能多的保留。

对句子 `Sue ate the alligator` ，不仅要将每一个单词（或者 *unigram* ）作为词项索引

```
["sue", "ate", "the", "alligator"]
```

也要将每个单词 *以及它的邻近词* 作为单个词项索引：

```
["sue ate", "ate the", "the alligator"]
```

这些单词对（或者 *bigrams* ）被称为 *shingles* 。

**生成 Shingles**

Shingles 需要在索引时作为分析过程的一部分被创建。我们可以将 unigrams 和 bigrams 都索引到单个字段中， 但将它们分开保存在能被独立查询的字段会更清晰。unigrams 字段将构成我们搜索的基础部分，而 bigrams 字段用来提高相关度。

首先，我们需要在创建分析器时使用 `shingle` 语汇单元过滤器：

```sense
DELETE /my_index

PUT /my_index
{
    "settings": {
        "number_of_shards": 1,  
        "analysis": {
            "filter": {
                "my_shingle_filter": {
                    "type":             "shingle",
                    "min_shingle_size": 2, 
                    "max_shingle_size": 2, 
                    "output_unigrams":  false   
                }
            },
            "analyzer": {
                "my_shingle_analyzer": {
                    "type":             "custom",
                    "tokenizer":        "standard",
                    "filter": [
                        "lowercase",
                        "my_shingle_filter" 
                    ]
                }
            }
        }
    }
}
```


默认最小/最大的 shingle 大小是 2 ，所以实际上不需要设置。

`shingle` 语汇单元过滤器默认输出 unigrams ，但是我们想让 unigrams 和 bigrams 分开。

`my_shingle_analyzer` 使用我们常规的 `my_shingles_filter` 语汇单元过滤器。

**多字段**

```js
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "title": {
                "type": "string",
                "fields": {
                    "shingles": {
                        "type":     "string",
                        "analyzer": "my_shingle_analyzer"
                    }
                }
            }
        }
    }
}
```

通过这个映射， JSON 文档中的 `title` 字段将会被以 unigrams (`title`)和 bigrams (`title.shingles`)被索引，这意味着可以独立地查询这些字段。

索引文档

```js
POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "Sue ate the alligator" }
{ "index": { "_id": 2 }}
{ "title": "The alligator ate Sue" }
{ "index": { "_id": 3 }}
{ "title": "Sue never goes anywhere without her alligator skin purse" }
```

**搜索 Shingles**

为了理解添加 `shingles` 字段的好处，让我们首先来看 `The hungry alligator ate Sue` 进行简单 `match` 查询的结果：

这个查询返回了所有的三个文档， 但是注意文档 1 和 2 有相同的相关度评分因为他们包含了相同的单词。

现在在查询里添加 `shingles` 字段。不要忘了在 `shingles` 字段上的匹配是充当一种信号—为了提高相关度评分—所以我们仍然需要将基本 `title` 字段包含到查询中：

```js
GET /my_index/my_type/_search
{
   "query": {
      "bool": {
         "must": {
            "match": {
               "title": "the hungry alligator ate sue"
            }
         },
         "should": {
            "match": {
               "title.shingles": "the hungry alligator ate sue"
            }
         }
      }
   }
}
```

仍然匹配到了所有的 3 个文档， 但是文档 2 现在排到了第一名因为它匹配了 shingled 词项 `ate sue`.

```js
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.4883322,
        "_source": {
           "title": "The alligator ate Sue"
        }
     },
     {
        "_id": "1",
        "_score": 0.13422975,
        "_source": {
           "title": "Sue ate the alligator"
        }
     },
     {
        "_id": "3",
        "_score": 0.014119488,
        "_source": {
           "title": "Sue never goes anywhere without her alligator skin purse"
        }
     }
  ]
}
```

即使查询包含的单词 `hungry` 没有在任何文档中出现，我们仍然使用单词邻近度返回了最相关的文档。

**Performance性能**

shingles 不仅比短语查询更灵活，而且性能也更好。 shingles 查询跟一个简单的 `match` 查询一样高效，而不用每次搜索花费短语查询的代价。只是在索引期间因为更多词项需要被索引会付出一些小的代价， 这也意味着有 shingles 的字段会占用更多的磁盘空间。 然而，大多数应用写入一次而读取多次，所以在索引期间优化我们的查询速度是有意义的。

这是一个在 Elasticsearch 里会经常碰到的话题：不需要任何前期进行过多的设置，就能够在搜索的时候有很好的效果。 一旦更清晰的理解了自己的需求，就能在索引时通过正确的为你的数据建模获得更好结果和性能。



## 部分匹配

敏锐的读者会注意，目前为止本书介绍的所有查询都是针对整个词的操作。为了能匹配，只能查找倒排索引中存在的词，最小的单元为单个词。

但如果想匹配部分而不是全部的词该怎么办？ *部分匹配* 允许用户指定查找词的一部分并找出所有包含这部分片段的词。

与想象的不太一样，对词进行部分匹配的需求在全文搜索引擎领域并不常见。

只在某些情况下部分匹配会比较有用，常见的应用如下：

- 匹配邮编、产品序列号或其他 `not_analyzed` 未分析值，这些值可以是以某个特定前缀开始，也可以是与某种模式匹配的，甚至可以是与某个正则式相匹配的。
- *输入即搜索（search-as-you-type）* ——在用户键入搜索词过程的同时就呈现最可能的结果。
- 匹配如德语或荷兰语这样有长组合词的语言，如： *Weltgesundheitsorganisation* （世界卫生组织，英文 World Health Organization）。

### 结构化数据

使用美国目前使用的邮编形式（United Kingdom postcodes 标准）来说明如何用部分匹配查询结构化数据。这种邮编形式有很好的结构定义。例如，邮编 `W1V 3DG` 可以分解成如下形式：

- `W1V` ：这是邮编的外部，它定义了邮件的区域和行政区：
  - `W` 代表区域（ 1 或 2 个字母）
  - `1V` 代表行政区（ 1 或 2 个数字，可能跟着一个字符）
- `3DG` ：内部定义了街道或建筑：
  - `3` 代表街区区块（ 1 个数字）
  - `DG` 代表单元（ 2 个字母）

假设将邮编作为 `not_analyzed` 的精确值字段索引，所以可以为其创建索引，如下：

```sense
PUT /my_index
{
    "mappings": {
        "address": {
            "properties": {
                "postcode": {
                    "type":  "string",
                    "index": "not_analyzed"
                }
            }
        }
    }
}
```

然后索引一些邮编：

```sense
PUT /my_index/address/1
{ "postcode": "W1V 3DG" }

PUT /my_index/address/2
{ "postcode": "W2F 8HW" }

PUT /my_index/address/3
{ "postcode": "W1F 7HW" }

PUT /my_index/address/4
{ "postcode": "WC1N 1LZ" }

PUT /my_index/address/5
{ "postcode": "SW5 0BE" }
```

### prefix 前缀查询

为了找到所有以 `W1` 开始的邮编，可以使用简单的 `prefix` 查询：

```sense
GET /my_index/address/_search
{
    "query": {
        "prefix": {
            "postcode": "W1"
        }
    }
}
```

> 默认状态下， `prefix` 查询不做相关度评分计算，它只是将所有匹配的文档返回，并为每条结果赋予评分值 `1` 。它的行为更像是过滤器而不是查询。 `prefix` 查询和 `prefix` 过滤器这两者实际的区别就是过滤器是可以被缓存的，而查询不行。

 `prefix` 查询的工作方式，倒排索引包含了一个有序的唯一词列表（本例是邮编）。对于每个词，倒排索引都会将包含词的文档 ID 列入 *倒排表（postings list）* 。与示例对应的倒排索引是：

```
Term:          Doc IDs:
-------------------------
"SW5 0BE"    |  5
"W1F 7HW"    |  3
"W1V 3DG"    |  1
"W2F 8HW"    |  2
"WC1N 1LZ"   |  4
-------------------------
```

为了支持前缀匹配，查询会做以下事情：

1. 扫描词列表并查找到第一个以 `W1` 开始的词。
2. 搜集关联的文档 ID 。
3. 移动到下一个词。
4. 如果这个词也是以 `W1` 开头，查询跳回到第二步再重复执行，直到下一个词不以 `W1` 为止。

这对于小的例子当然可以正常工作，但是如果倒排索引中有数以百万的邮编都是以 `W1` 开头时，前缀查询则需要访问每个词然后计算结果！

前缀越短所需访问的词越多。如果我们要以 `W` 作为前缀而不是 `W1` ，那么就可能需要做千万次的匹配。

>可以使用较长的前缀来限制这种影响，减少需要访问的量。

### 通配符与正则表达式查询

与 `prefix` 前缀查询的特性类似， `wildcard` 通配符查询也是一种底层基于词的查询，与前缀查询不同的是它允许指定匹配的正则式。它使用标准的 shell 通配符查询： `?` 匹配任意字符， `*` 匹配 0 或多个字符。

这个查询会匹配包含 `W1F 7HW` 和 `W2F 8HW` 的文档：

```sense
GET /my_index/address/_search
{
    "query": {
        "wildcard": {
            "postcode": "W?F*HW" 
        }
    }
}
```

`?` 匹配 `1` 和 `2` ， `*` 与空格及 `7` 和 `8` 匹配。

如果想匹配只以 `W` 开始并跟随一个数字的所有邮编， `regexp` 正则式查询允许写出这样更复杂的模式：

```sense
GET /my_index/address/_search
{
    "query": {
        "regexp": {
            "postcode": "W[0-9].+" 
        }
    }
}
```

这个正则表达式要求词必须以 `W` 开头，紧跟 0 至 9 之间的任何一个数字，然后接一或多个其他字符。

`wildcard` 和 `regexp` 查询的工作方式与 `prefix` 查询完全一样，它们也需要扫描倒排索引中的词列表才能找到所有匹配的词，然后依次获取每个词相关的文档 ID ，与 `prefix` 查询的唯一不同是：它们能支持更为复杂的匹配模式。

这也意味着需要同样注意前缀查询存在性能问题，对有很多唯一词的字段执行这些查询可能会消耗非常多的资源，所以要避免使用左通配这样的模式匹配（如： `*foo` 或 `.*foo` 这样的正则式）。

数据在索引时的预处理有助于提高前缀匹配的效率，而通配符和正则表达式查询只能在查询时完成，尽管这些查询有其应用场景，但使用仍需谨慎。

> `prefix` 、 `wildcard` 和 `regexp` 查询是基于词操作的，如果用它们来查询 `analyzed` 字段，它们会检查字段里面的每个词，而不是将字段作为整体来处理。

### 即时搜索

*即时搜索（instant search）* 或 *输入即搜索（search-as-you-type）*，例如，如果用户输入 `johnnie walker bl` ，我们希望在它们完成输入搜索条件前就能得到：Johnnie Walker Black Label 和 Johnnie Walker Blue Label 。

```sense
{
    "match_phrase_prefix" : {
        "brand" : "johnnie walker bl"
    }
}
```

这种查询的行为与 `match_phrase` 查询一致，不同的是它将查询字符串的最后一个词作为前缀使用，换句话说，可以将之前的例子看成如下这样：

- `johnnie`
- 跟着 `walker`
- 跟着以 `bl` 开始的词

如果通过 `validate-query` API 运行这个查询查询，explanation 的解释结果为：

```
"johnnie walker bl*"
```

与 `match_phrase` 一样，它也可以接受 `slop` 参数（参照 [slop](https://www.elastic.co/guide/cn/elasticsearch/guide/current/slop.html) ）让相对词序位置不那么严格：

```sense
{
    "match_phrase_prefix" : {
        "brand" : {
            "query": "walker johnnie bl", 
            "slop":  10
        }
    }
}
```

尽管词语的顺序不正确，查询仍然能匹配，因为我们为它设置了足够高的 `slop` 值使匹配时的词序有更大的灵活性。

但是只有查询字符串的最后一个词才能当作前缀使用。

因为 `prefix` 查询存在严重的资源消耗问题，短语查询的这种方式也同样如此。前缀 `a` 可能会匹配成千上万的词，这不仅会消耗很多系统资源，而且结果的用处也不大。

可以通过设置 `max_expansions` 参数来限制前缀扩展的影响，一个合理的值是可能是 50 ：

```sense
{
    "match_phrase_prefix" : {
        "brand" : {
            "query":          "johnnie walker bl",
            "max_expansions": 50
        }
    }
}
```

参数 `max_expansions` 控制着可以与前缀匹配的词的数量，它会先查找第一个与前缀 `bl` 匹配的词，然后依次查找搜集与之匹配的词（按字母顺序），直到没有更多可匹配的词或当数量超过 `max_expansions` 时结束。

### 索引时优化

到目前为止，所有谈论过的解决方案都是在 *查询时（query time）* 实现的。这样做并不需要特殊的映射或特殊的索引模式，只是简单使用已经索引的数据。

查询时的灵活性通常会以牺牲搜索性能为代价，有时候将这些消耗从查询过程中转移到别的地方是有意义的。在实时 web 应用中， 100 毫秒可能是一个难以忍受的巨大延迟。

可以通过在索引时处理数据提高搜索的灵活性以及提升系统性能。为此仍然需要付出应有的代价：增加的索引空间与变慢的索引能力，但这与每次查询都需要付出代价不同，索引时的代价只用付出一次。

用户会感谢我们。

### 索引时输入即搜索

准备索引

第一步需要配置一个自定义的 `edge_ngram` token 过滤器，称为 `autocomplete_filter` ：

```js
{
    "filter": {
        "autocomplete_filter": {
            "type":     "edge_ngram",
            "min_gram": 1,
            "max_gram": 20
        }
    }
}
```

这个配置的意思是：对于这个 token 过滤器接收的任意词项，过滤器会为之生成一个最小固定值为 1 ，最大为 20 的 n-gram 。

然后会在一个自定义分析器 `autocomplete` 中使用上面这个 token 过滤器：

```js
{
    "analyzer": {
        "autocomplete": {
            "type":      "custom",
            "tokenizer": "standard",
            "filter": [
                "lowercase",
                "autocomplete_filter" 
            ]
        }
    }
}
```

自定义的 edge-ngram token 过滤器。

这个分析器使用 `standard` 分词器将字符串拆分为独立的词，并且将它们都变成小写形式，然后为每个词生成一个边界 n-gram，这要感谢 `autocomplete_filter` 起的作用。

创建索引、实例化 token 过滤器和分析器的完整示例如下：

```sense
PUT /my_index
{
    "settings": {
        "number_of_shards": 1, 
        "analysis": {
            "filter": {
                "autocomplete_filter": { 
                    "type":     "edge_ngram",
                    "min_gram": 1,
                    "max_gram": 20
                }
            },
            "analyzer": {
                "autocomplete": {
                    "type":      "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "autocomplete_filter" 
                    ]
                }
            }
        }
    }
}
```

将这个分析器应用到具体字段：

```sense
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "name": {
                "type":     "string",
                "analyzer": "autocomplete"
            }
        }
    }
}
```

相同的 `autocomplete` 分析器同时被应用于索引时和搜索时，这在大多数情况下是正确的，只有在少数场景下才需要改变这种行为。

我们需要保证倒排索引表中包含边界 n-grams 的每个词，但是我们只想匹配用户输入的完整词组（ `brown` 和 `fo` ），可以通过在索引时使用 `autocomplete` 分析器，并在搜索时使用 `standard` 标准分析器来实现这种想法，只要改变查询使用的搜索分析器 `analyzer` 参数即可：

```sense
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "name": {
                "query":    "brown fo",
                "analyzer": "standard" 
            }
        }
    }
}
```

覆盖 `name` 字段 `analyzer` 的设置。

换种方式，我们可以在映射中，为 `name` 字段分别指定 `index_analyzer` 和 `search_analyzer` 。因为我们只想改变 `search_analyzer` ，这里只要更新现有的映射而不用对数据重新创建索引：

```sense
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "properties": {
            "name": {
                "type":            "string",
                "index_analyzer":  "autocomplete", 
                "search_analyzer": "standard" 
            }
        }
    }
}
```

>**补全提示（Completion Suggester）**
>
>使用边界 n-grams 进行输入即搜索（search-as-you-type）的查询设置简单、灵活且快速，但有时候它并不够快，特别是当试图立刻获得反馈时，延迟的问题就会凸显，很多时候不搜索才是最快的搜索方式。
>
>Elasticsearch 里的 [completion suggester](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-suggesters-completion.html) 采用与上面完全不同的方式，需要为搜索条件生成一个所有可能完成的词列表，然后将它们置入一个 *有限状态机（finite state transducer）* 内，这是个经优化的图结构。为了搜索建议提示，Elasticsearch 从图的开始处顺着匹配路径一个字符一个字符地进行匹配，一旦它处于用户输入的末尾，Elasticsearch 就会查找所有可能结束的当前路径，然后生成一个建议列表。
>
>本数据结构存于内存中，能使前缀查找非常快，比任何一种基于词的查询都要快很多，这对名字或品牌的自动补全非常适用，因为这些词通常是以普通顺序组织的：用 “Johnny Rotten” 而不是 “Rotten Johnny” 。
>
>当词序不是那么容易被预见时，边界 n-grams 比 Completion Suggester 更合适。

**边界 n-grams 与邮编**

边界 n-gram 的方式可以用来查询结构化的数据，如邮编（postcode）。当然 `postcode` 字段需要 `analyzed` 而不是 `not_analyzed` ，不过可以用 `keyword` 分词器来处理它，就好像他们是 `not_analyzed` 的一样。

`keyword` 分词器是一个非操作型分词器，这个分词器不做任何事情，它接收的任何字符串都会被原样发出，因此它可以用来处理 `not_analyzed` 的字段值，但这也需要其他的一些分析转换，如将字母转换成小写。

下面示例使用 `keyword` 分词器将邮编转换成 token 流，这样就能使用边界 n-gram token 过滤器：

```sense
{
    "analysis": {
        "filter": {
            "postcode_filter": {
                "type":     "edge_ngram",
                "min_gram": 1,
                "max_gram": 8
            }
        },
        "analyzer": {
            "postcode_index": { 
                "tokenizer": "keyword",
                "filter":    [ "postcode_filter" ]
            },
            "postcode_search": { 
                "tokenizer": "keyword"
            }
        }
    }
}
```



















