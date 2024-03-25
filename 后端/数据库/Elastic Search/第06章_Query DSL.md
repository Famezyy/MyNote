# 第06章_Query DSL

## 1.基础概念

**查询**指的是有明确的搜索条件边界。比如，年龄 15~25 岁，颜色 = 红色，价格 < 3000，这里的 15、25、红色、3000 都是条件边界。

**检索**即全文检索，无搜索条件边界，召回结果取决于相关性，其相关性计算无明确边界性条件，如同义词、谐音、别名、错别字、混淆词、网络热梗等均可成为其相关性判断依据。

<img src="img/第06章_Query DSL/image-20240321164749593.png" alt="image-20240321164749593" style="zoom: 50%;" />

在 ES 中使用 `_score` 来表示**相关度**。相关度评分用于对搜索结果排序，评分越高则认为其结果和搜索的预期值相关度越高，即越符合搜索预期值。在 5.x 之前相关度评分默认使用 TF/IDF 算法计算而来，5.x 之后默认为 BM25。**如果指定了排序字段则不会计算 `_socre`**；如果没有指定排序字段则按照评分高低排序。

## 2.条件查询

### 2.1 过滤源数据字段

第 03 章中介绍过了查询源数据时有两种 API 可以使用：

```json
get my_index/_doc/1?_source=false
get my_index/_source/1
```

但是这两种方式无法更进一步的过滤出想要的字段数据。如果想要过滤数据可以有两种方式，并且支持使用通配符：

- **在 `mapping` 中定义过滤规则**

  这种方式不推荐，因为一旦创建则不可变。

  ```json
  PUT product
  {
      "mappings": {
          "_source": {
              // 源数据结果中返回哪些 field
              "includes": [
                  "name",
                  "price"
              ],
              // 源数据结果中不要返回哪些 field，会覆盖 includes 中重复项
              "excludes": [
                  "desc",
                  "tags"
              ]
          }
      }
  }
  ```

  此时查询源数据只会显示 `name` 和 `price` 的信息：

  ```json
  PUT product/_doc/1
  {
    "name": "product1",
    "desc": "hi"
  }
  --
  GET product/_source/1
  ```

  > **注意**
  >
  > 使用 `excludes` 时不返回的 field 不代表不能通过该字段进行检索，因为索引已经被创建了。

- **在查询中过滤**

  - 不查看源数据，仅查看元字段

    ```json
    get product/_search
    {
        {
            "_source": false,
            "query": {
            	...
            } 
    	}
    }
    ```

  - 只看 `obj.` 的所有内嵌字段

    ```json
    get product/_search
    {
        {
            "_source": "obj.*",
            "query": {
            	...
            } 
    	}
    }
    ```

  - 查看 `obj` 或者 `obj2` 的所有内嵌字段

    ```json
    get product/_search
    {
        {
            "_source": [ "obj1.*", "obj2.*" ],
            "query": {
            	...
            } 
    	}
    }
    ```

  - 查看以 `obj.` 或者 `obj2.` 开头，并且不以 `.desc` 为后缀的字段

    ```json
    get product/_search
    {
        {
            "_source": {
                "includes": [
                    "obj1.*",
                    "obj2.*"
                ],
                "excludes": [
                    "*.desc"
                ]
            },
            "query": {
            	...
            } 
    	}
    }
    ```

> **补充**
>
> 我们也可以禁用 `_source` 来节省存储开销：
>
> ```json
> PUT product/_mapping
> {
>     "_source": {
>         "enabled": false
>     }
> }
> ```
>
> 但是会带来以下缺点：
>
> - 不支持 update、update_by_query 和 reindex API
> - 不支持高亮
> - 不支持 reindex、更改 mapping 分析器和版本升级
> - 通过查看索引时使用的原始文档来调试查询或聚合的功能
> - 将来有可能自动修复索引损坏
>
> 如果只是为了节省磁盘，压缩索引比禁用 `_source` 更好。

### 2.2 分页

默认情况下，查询只返回前 10 个匹配命中。

分页查找可以使用以下两个关键字：

- `from`：从低几个文档开始返回，需要为非负数，默认为 ：0
- `size`：定义要返回的命中数，默认值为：10

**基本语法**

```json
GET <index>/_search
{
  "from": 0,
  "size": 20
}
--
GET <index>/_search?from=0&size=20
```

- `from` + `size` 必须小于等于 10000，[参考](第03章_索引和文档操作.md#4.动态索引设置)

- `max_result_window`：可以解除 `from` + `size` 必须小于 10000 的限制

- `track_total_hits`：查询结果中的 `hits.total` 返回的也不是实际的命中数，可以设置该参数为 true 使 `hits.total` 返回实际命中数，但是会牺牲性能，[参考](https://www.elastic.org.cn/archives/esyuzhi)

  ```json
  GET my-index-000001/_search
  {
    "track_total_hits": true,
    "query": {
      "match" : {
        "user.id" : "elkbee"
      }
    }
  }
  ```

### 2.3 排序

**基本语法**

```json
GET <index>/_search
{
	"query": {
        "match": {
            "name": "xiaomi"
        }
    }
    "sort": [
    // 按照顺序依次排序
        {
            "<sort_field1>": {
                "order": "desc" // or asc
            }
        },
		{
            "<sort_field2>": {
                "order": "desc" // or asc
            }
        }
    ]
}
```

- `sort_field`：可以是 `_source` field，也可以是 `metadata` field

### 2.4 Url Query

**查询所有文档**

```json
GET /goods/_search 
```

**带参数查询**

```bash
# 查询 name 中包含 xiaomi 的文档
GET /goods/_search?q=name:xiaomi
```

**分页**

```json
GET /goods/_search?from=0&size=2&sort=price:asc 
```

**精准匹配**

```json
GET /goods/_search?q=date:2040-07-27
```

如果不指定参数名称，即 `_all` 搜索，相当于在索引的所有字段中检索，**会被进行分词**

```json
GET /goods/_search?q=2040-07-27
```

## 3.全文检索：match⭐

全文检索是一种搜索技术，用于在大量文本数据中查找包含用户输入的关键字的文档或记录。全文检索的核心是建立一个**倒排索引**（Inverted Index），即将每个词与包含这个词的文档列表进行关联。当用户输入关键词时，系统会根据这些关键词在倒排索引中查找相应的文档列表，并将匹配度高的文档排在前面返回给用户。

在全文检索中，文本数据通常被存储在数据库或文件系统中。在搜索之前，先将文本数据**分词**（Tokenization），将文本分解成一系列单词（**词项**），并将它们存储在倒排索引中。倒排索引是一种数据结构，用于存储每个单词所在的文档列表。它使得搜索引擎可以快速地找到包含用户输入的关键字的文档。因此全文检索要求字段类型必须是可分词的字段类型。

当用户输入一个或多个关键字进行搜索时，搜索引擎将查询倒排索引，找到包含这些关键字的文档，并将它们按照相关度排序后返回给用户。搜索引擎通常使用一些算法和技术来计算文档与关键字之间的相关度，例如 TF-IDF（词频-逆文档频率）算法、BM25（Okapi Best Matching 25）算法等。

<img src="img/第06章_Query DSL/2.jpg" alt="img" style="zoom:67%;" />

全文检索有许多优点：

- 可以非常快速地查找文本数据
- 可以处理大量的文本数据
- 可以处理复杂的查询，例如布尔查询、短语查询、模糊查询等
- 可以根据相关度对文档进行排序，提高搜索结果的质量

但是，全文检索也存在一些挑战和限制：

- 分词过程可能会存在歧义，例如“香蕉奶油”可能会被分解成“香蕉”和“奶油”，也可能被分解成“香蕉奶”和“油”
- 倒排索引可能会占用大量的存储空间
- 搜索结果可能会受到数据质量和相关度计算算法的影响
- 全文检索不适直接用于非文本数据，例如图像、音频、视频等

### 3.1 match

匹配包含某个词项的文档。

```json
GET <index>/_search
{
    "query": {
        "match": {
            "<field_name>": "<field_value>"
        }
    }
}
```

如果搜索词包含多个词项，那么文档**只要匹配其中任意一个词项**，就会被匹配。

例如对于下面的查询：

```json
GET product/_search
{
    "query": {
        "match": {
            "name": "小米手机"
        }
    }
}
```

假设 `小米手机` 被分成了 `小米` 和 `手机`，则只要匹配上其中一个词项就会被返回作为结果。

### 3.2 match_all

匹配所有结果的子句，并给定所有文档的 `_source` 为 `1.0`。

```json
GET <index>/_search
{
    "query": {
        "match_all": {}
    }
}
```

可以指定一个 `boost` 参数统一设置返回结果的分数：

```json
GET <index>/_search
{
    "query": {
        "match_all": {"boost" : 1.2}
    }
}
```

类似的，ES 还提供了 `match_none`。

### 3.3 match_phrase

`match_phrase` 查询是一种短语查询，它用于匹配包含指定短语的文档。与 `Match` 查询不同，`match_phrase` 查询只匹配包含短语的文档，而不会匹配单个词条。

```json
GET <index>/_search
{
    "query": {
        "match_phrase": {
            "<field_name>": "<field_value>"
        }
    }
}
```

**匹配规则**

- **会分词**：`match_phrase` 首先会将查询短语拆分为单个词项。例如，如果查询短语是 “elastic org cn”，则将其拆分为三个词项：

- **必须全部匹配且顺序必须相同**：被检索字段必须包含 `match_phrase`  短语中的所有词项，并且顺序必须是相同的。比如查询短语是`elastic org cn`，那么只有字段中的词项必须包含 "elastic"、"org" 和 "cn" 这三个短语，切顺序不能颠倒

- **可以设置 slop 距离**：默认情况下被检索字段包含的 `match_phrase` 中的词项之间不能有其他词项，即 `slop` 默认为 0，如果人为将 `slop` 设置为其他数值，则多个词项将允许有 `slop` 规定的距离

  ```json
  get product/_search
  {
      "query": {
          "match_phrase": {
              "text": {
                  "query": "there is apple",
                  "slop": 1
              }
          }
      }
  }
  ```

  > **注意**
  >
  > `slop` 参数可以设置为一个非负整数，默认值为 0，表示查询短语中的词项必须以查询短语中的相同顺序连续出现。当 `slop` 参数大于 0 时，查询短语中的词项可以以指定的最大间隔数任意顺序出现。例如，如果 `slop` 参数设置为 1，则查询短语中的所有的相邻词项间最多忽视一个单词。
  >
  > 使用 `slop` 参数时需要注意以下几点：
  >
  > - `slop` 参数只适用于 `match_phrase` 查询，不适用于 `match_phrase_prefix` 查询
  > - `slop` 参数的值越大，匹配的文档数就越多，同时查询的性能也会下降，相关度也可能会下降

### 3.4 multi_match

支持在查询时指定多个需要匹配的字段。

```json
GET <index>/_search
{
    "query": {
        "multi_match": {
            "query": "<query>",
            "fields": ["<field1>", "<field2>"]
        }
    }
}
```

## 4.精准查询：term

### 4.1 简介

精准查询也被称为术语级查询，术语指的其实就是词项，简单来说，术语就是在全文检索中的最小粒度文本，而术语级查询指的就是不可分词的查询类型。即有明确查询条件边界的查询。如：范围查询、词项查询等。`term` 查询一般用于如：Id、姓名、价格等**不需要分词**的字段。

```json
GET <index>/_search
{
    "query": {
        "term": {
            "<keyword_field_name>": "<field_value>"
        }
    }
}
```

**示例**

```json
GET product/_search
{
    "query": {
        "term": {
            "name.keyword": "this is a very long phrase"
        }
    }
}
```

如果这里使用的是本来的字段 `name` 则会查询 `name` 分词后的索引。

### 4.2 与match对比

和 `match` 相比，`term` 不会对搜索词分词，而且会**保留搜索词原有的所有属性**，如大小写、标点符号等。

|          | term query              | match query        | match_phrase       |
| :------- | :---------------------- | :----------------- | :----------------- |
| 字段类型 | 不分词（如：`keyword`） | 分词（如：`text`） | 分词（如：`text`） |
| 搜索场景 | 精确匹配                | 全文检索           | 全文检索           |

通常来说 `term` 会和 `keyword` 类型一起使用，其他不分词字段亦可。

|          | term     | keyword  |
| :------- | :------- | -------- |
| 参数类型 | 查询函数 | 字段类型 |
| 是否分词 | 否       | 否       |
| 作用范围 | 搜索词   | 源字段   |

**示例**

创建 product 索引并自动映射 desc 字段。

```json
put product/_doc/1
{
    "desc": "This is an apple"
}
```

查看

```json
// 搜索词不分词，查询的也是不分词的 desc，匹配
get product/_search
{
  "query": {
    "term": {
      "desc.keyword": "This is an apple"
      
    }
  }
}
--
// 搜索词不分词，查询的是分词后的 desc，不匹配
get product/_search
{
  "query": {
    "term": {
      "desc": "This is an apple"
      
    }
  }
}
--
// 搜索词分词，查询的是分词后的 desc，匹配
get product/_search
{
  "query": {
    "match": {
      "desc": "this is an apple"
      
    }
  }
}
--
// 搜索词分词，查询的不分词的 desc，不匹配
get product/_search
{
  "query": {
    "match": {
      "desc.keyword": "this is an apple"
      
    }
  }
}
--
// 虽然搜索词分词，查询的不分词的 desc，但是由于是完美匹配，即当搜索词和文档源字段中值完全一致时，可以匹配上
get product/_search
{
  "query": {
    "match": {
      "desc.keyword": "This is an apple"
      
    }
  }
}
```

### 4.3 匹配多个值

匹配多个词项时可以使用 `terms`，匹配任意一个则返回结果：

```json
GET <index>/_search
{
    "query": {
        "terms": {
            "<field_name>": [ "<value1>", "<value2>" ]
        }
    }
}
```

默认情况下 `term` 只能查找 `_source` 下的字段，此外还可以针对文档元字段 `_id` 进行查询，此时需要使用 `ids` 查找：

```json
GET goods/_search
{
    "query": {
        "ids": {
            "values": [1,4,7]
        }
    }
}
```

## 5.范围查询：range

```json
GET <index>/_search
{
    "query": {
        "range": {
            "age": {
                "gte": 10,
                "lte": 20,
                "boost": 2.0
            }
        }
    }
}
```

- `gt`：大于
- `lte`：大于等于
- `lt`：小于
- `let`：小于等于

默认返回的所有文档的 `_score` 为 `1.0`，可以通过 `boost` 指定返回的分数。

对于日期类型的数据可以使用 `now` 来表示现在，同时支持使用数学符号来计算：

```json
get test/_search
{
    "query": {
        "range": {
            "date": {
               	// 大于等于 4 年前的今天
                "gte": "now-4y",
                // 小于等于今天
                "lte": "now"
            }
        }
    }
}
```

还可以使用 `time_zone` 来修改文档数据的时区：

```json
get test/_search
{
    "query": {
        "range": {
            "date": {
                // 表示文档数据要加上 8 小时
                "time_zone": "+08:00", 
                "gte": "2020-03-25T08:00:00",
                "lte": "now"
            }
        }
    }
}
```

## 6.固定分数查询

使用 `constant_score` 可以不计算分数进行查询：

```json
get test/_search
{
    "query": {
        "constant_score": {
            "filter": {
                "term": {
                    "FIELD": "VALUE"
                }
            },
            // 指定返回的分数
            "boost": 1.2
        }
    }
}
```

## 7.布尔查询：boolean

### 7.1 简介

布尔查询可以组合多个查询条件，采用 `more_matches_is_better` 的机制，因此满足 `must` 和 `should` 子句的文档将会合并起来计算分值，多用于多条件组合查询。

**基本语法**

```json
GET _search
{
    "query": {
        "bool": {
            // 必须符合每条件
            "filter": [
                {条件一},
                {条件二}
            ],
            // 必须符合每条件
            "must": [ 		
                {条件一},
                {条件二}
            ], 
            // 必须不满足每个条件
            "must_not": [
                {条件一},
                {条件二}
            ], 
            // 可以满足其中若干条件或全部不满足
            "should": [ 
                {条件一},
                {条件二}
            ] 
        }
    }
}
```

**测试数据**

```json
PUT /goods_en/_doc/1
{
    "name" : "HuaRen 2060 Super Phone ShaNiu",
    "desc" :  "From the future of technology,zhenren moshi zongxiang kuaile",
    "brand":	"HuaRen",
    "price" :  99999,
    "lv":"ceiling",
    "type":"phone",
    "createtime":"2050-10-01T08:00:00Z",
    "tags": [ "future", "chuanyue","zhenrenmoshi","shoujimoshi" ]
}
PUT /goods_en/_doc/2
{
    "name" : "HuaWei Mate 9000 Phone",
    "desc" :  "zhichi weixing tongxun,xinhao qiang, gaoduan daqi shangdangci",
    "brand":	"HuaWei",
    "price" :  29999,
    "lv":"qijianji",
    "type":"phone",
    "createtime":"2050-05-21T08:00:00Z",
    "tags": [ "xinhao", "shangwu","timian","xuhang"]
}
PUT /goods_en/_doc/3
{
    "name" : "iphone 120 Pro Max Ultra Phone",
    "desc" :  "sihua liuchuang bukadun,nianqingren de diyitai shouji hebi shi iphone",
    "brand":	"Apple",
    "price" :  12999,
    "lv":"gaoduan",
    "type":"phone",
    "createtime":"2050-06-20",
    "tags": [ "IOS", "sihua", "liuchang" ]
}
PUT /goods_en/_doc/4
{
    "name" : "XiaoMi 110 Pro Ultra",
    "desc" :  "erji zhong de huangmenji",
    "brand":	"Xiaomi",
    "price" :  6999,
    "lv":"gaoduan",
    "type":"erji",
    "createtime":"2050-06-23",
    "tags": [ "fashao", "nfc","miui" ]
}
PUT /goods_en/_doc/5
{
    "name" : "hongmi k100",
    "desc" :  "xingjiabi qijian, dijia gaopei",
    "brand":	"Xiaomi",
    "price" :  1999,
    "type":"phone",
    "lv":"qianyuanji",
    "createtime":"2050-07-20",
    "tags": [ "xingjiabi","xuhang", "shishang","miui" ]
}

```

### 7.2 查询子句

#### 1.must

用于计算相关度得分，多个条件必须同时满足。

```json
// 条件1: brand：xiaomi
// AND
// 条件2: price：> 5000
GET goods_en/_search
{
    "query": {
        "bool": {
            "must": [
                {
                    "match": {
                        "brand": "Xiaomi"
                    }
                },
                {
                    "range": {
                        "price": {
                            "gt": 5000
                        }
                    }
                }
            ]
        }
    }
}
```

#### 2.filter

和 `must` 作用相同，返回的文档必须同时满足字句的条件。但是不像 `must` 会计算评分，使用 `filter` 查询的评分将被忽略，只是根据过滤标准来排除或包含文档，并且结果会被缓存。

```json
// 条件1: type：phone
// AND
// 条件2: price：< 20000
GET goods_en/_search
{
    "_source": false, 
    "query": {
        "bool": {
            "filter": [
                {
                    "term": {
                        "type.keyword": "phone"
                    }
                },
                {
                    "range": {
                        "price": {
                            "lt": 20000
                        }
                    }
                }
            ]
        }
    }
}
```

#### 3.should

如果满足这些语句中的任意语句，将增加 `_score`，否则无任何影响。它们主要用于修正每个文档的相关性得分。

```json
// 条件1: name: Huawei Xiaomi 
// OR
// 条件2: band: xiaomi huawei

GET goods_en/_search
{
    "_source": false, 
    "query": {
        "bool": {
            "should": [
                {
                    "match": {
                        "name": "xiaomi huawei"
                    }
                },
                {
                    "terms": {
                        "brand.keyword": [
                            "Xiaomi",
                            "HuaWei"
                        ]
                    }
                }
            ]
        }
    }
}
```

ES 提供了 `minimum_should_match` 参数用来指定 `should` 返回的文档必须匹配的条件的数量或百分比，如果 `bool` 查询包含至少一个 `should` 子句，且没有 `must` 或 `filter` 子句，则默认值为 1。

```json
// 条件1: name中包含 "phone"
// 条件2: type 等于 "phone"

GET goods_en/_search
{
    "_source": false,
    "query": {
        "bool": {
            "should": [
                {
                    "term": {
                        "type.keyword": "phone"
                    }
                },
                {
                    "match": {
                        "name": "phone"
                    }
                }
            ],
            "minimum_should_match": 2 // 设置为 2 则此时需要至少满足 2 个条件，相当于 must
        }
    }
}
```

但是如果 `bool` 查询中同级子句中出现了 `must` 或者 `filter` 子句（`must_not` 不影响），则 `minimum_should_match` 的默认值将变为 0，即相当于不考虑 `should` 的条件。此时需要显式指定 `minimum_should_match` 为 1。

```json
// ( must 或者 filter ) 和 should 组合
// 条件1: 价格小于 20000
// 条件2: name 中包含 "phone" 或者 type 等于 "phone"
GET goods_en/_search
{
    "_source": false,
    "query": {
        "bool": {
            "filter": [
                {
                    "range": {
                        "price": {
                            "lte": "20000"
                        }
                    }
                }
            ],
            "should": [
                {
                    "term": {
                        "type.keyword": "phone"
                    }
                },
                {
                    "match": {
                        "name": "phone"
                    }
                }
            ],
            // 需要显示指定
            "minimum_should_match": 1
        }
    }
}
```

#### 4.must_not

文档必须不匹配这些条件才能被包含进来，并且不计算相关度评分。

```json
// 条件1: 不要价格小于等于 10000 的
// 条件2: 不要 Apple 品牌的
GET goods_en/_search
{
    "query": {
        "bool": {
            "must_not": [
                {
                    "range": {
                        "price": {
                            "lte": 10000
                        }
                    }
                },
                {
                    "term": {
                        "brand.keyword": {
                            "value": "Apple"
                        }
                    }
                }
            ]
        }
    }
}
```

#### 5.组合

当多个子句同时出现时，多个子句之间的逻辑关系为 `AND`，即需要同时满足。如，当同时出现 `must [case1, case2]` 和 `must_not [case3, case4]` 时，其语义为必须同时满足 case1 和 case2，且必须同时不满足 case3、case4。

当 `filter` 和 `must` 一起使用时，`filter` 发挥的作用是对数据的过滤，这个过程不计算评分，而 `must` 则会对数据进行评分。经过了 `filter` 的过滤之后，需要评分的文档数量将降低，从而提升了查询效率。

```json
// filter 和 must 组合
// 条件一：价格小于 5000
// 条件二：要求是手机
GET goods_en/_search
{
    "_source": false, 
    "query": {
        "bool": {
            "filter": [
                {
                    "range": {
                        "price": {
                            "gte": "5000"
                        }
                    }
                }
            ],
            "must": [
                {
                    "match": {
                        "name": "phone"
                    }
                }
            ]
        }
    }
}
```

#### 6.子查询嵌套

**示例**

查询条件如下：

- name 中不包含 iphone

- 并且 name 中包含 "phone" 且价格价格大于 20000

  或者  type 等于 "phone" 且 Brand = HuaWei or Apple

```json
GET goods_en/_search
{
    "_source": false,
    "query": {
        "bool": {
            // name 中不包含 iphone
            "must_not": [
                {
                    "match": {
                        "name": "iphone"
                    }
                }
            ],
            // name 中包含 "phone" 且价格价格大于 20000
            // 或者  type 等于 "phone" 且 Brand = HuaWei or Apple
            "should": [
                {
                    "bool": {
                        "must": [
                            {
                                "match": {
                                    "name": "phone"
                                }
                            },
                            {
                                "range": {
                                    "price": {
                                        "gte": 20000
                                    }
                                }
                            }
                        ]
                    }
                },
                {
                    "bool": {
                        "must": [
                            {
                                "term": {
                                    "type.keyword": {
                                        "value": "phone"
                                    }
                                }
                            },
                            {
                                "terms": {
                                    "brand.keyword": [
                                        "HuaWei",
                                        "Apple"
                                    ]
                                }
                            }
                        ]
                    }
                }
            ]
        }
    }
}
```

## 
