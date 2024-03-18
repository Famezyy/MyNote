# 第06章_Query DSL

## 1.基础概念

**查询**指的是有明确的搜索条件边界。比如，年龄 15~25 岁，颜色 = 红色，价格 < 3000，这里的 15、25、红色、3000 都是条件边界。

**检索**即全文检索，无搜索条件边界，召回结果取决于相关性，其相关性计算无明确边界性条件，如同义词、谐音、别名、错别字、混淆词、网络热梗等均可成为其相关性判断依据。

在 ES 中使用 `_score` 来表示**相关度**。相关度评分用于对搜索结果排序，评分越高则认为其结果和搜索的预期值相关度越高，即越符合搜索预期值。在 5.x 之前相关度评分默认使用 TF/IDF 算法计算而来，5.x 之后默认为 BM25。**如果指定了排序字段则不会计算 `_socre`**；如果没有指定排序字段则按照评分高低排序。

## 2.查询

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
  > 使用 `excludes` 时不返回的 field 不代表不能通过该字段进行检索，因为源数据不存在不代表索引不存在。

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

  - 只看以 `obj.` 开头的字段

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

  - 查看以 `obj.` 或者 `obj2.` 开头的字段

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
