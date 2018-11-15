### mget与 bulk

### mget

![mget换取文档](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/images/elas_0405.png)

mget请求批量取回文档。在多文档取回时，尽量使用这个接口，会减少网络IO开销。

#### 取回步骤


以下是使用单个 mget 请求取回多个文档所需的步骤顺序：

1. 客户端向 Node 1 发送 mget 请求。
2. Node 1 为每个分片构建多文档获取请求，然后并行转发这些请求到托管在每个所需的主分片或者副本分片的节点上。一旦收到所有答复， Node 1 构建响应并将其返回给客户端。

#### Http请求示例

请求体
```json
GET /website/blog/_mget
{
   "ids" : [ "2", "1" ]
}
```

响应体
```json
{
  "docs" : [
    {
      "_index" :   "website",
      "_type" :    "blog",
      "_id" :      "2",
      "_version" : 10,
      "found" :    true,
      "_source" : {
        "title":   "My first external blog entry",
        "text":    "This is a piece of cake..."
      }
    },
    {
      "_index" :   "website",
      "_type" :    "blog",
      "_id" :      "1",
      "found" :    false  
    }
  ]
}
```
在查询过程中只要mget执行完成结果都会返回200状态码，如果某个文档没有找到，会被标记 *"found" :    false*  

#### java调用

1. elasticsearch-data 
```java
 public Iterable<T> findAllById(Iterable<ID> ids) {
        Assert.notNull(ids, "ids can't be null.");
        SearchQuery query = (new NativeSearchQueryBuilder()).withIds(this.stringIdsRepresentation(ids)).build();
        return this.elasticsearchOperations.multiGet(query, this.getEntityClass());
    }
```
2. elasticSearchTemplate
```java
elasticsearchOperations.multiGet(query, this.getEntityClass())
```

### bulk

bulk API使得在单个API调用中执行许多索引/删除操作成为可能。这可以大大提高索引速度。
bulk API 允许在单个步骤中进行多次 create 、 index 、 update 或 delete 请求。 如果你需要索引一个数据流比如日志事件，它可以排队和索引数百或数千批次。

bulk 与其他的请求体格式稍有不同，如下所示：
```json

{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
```
这种格式类似一个有效的单行 JSON 文档 流 ，它通过换行符(\n)连接到一起。注意两个要点：

每行一定要以换行符(\n)结尾， 包括最后一行 。这些换行符被用作一个标记，可以有效分隔行。
这些行不能包含未转义的换行符，因为他们将会对解析造成干扰。

action/metadata 行指定 哪一个文档 做 什么操作 。

action 必须是以下选项之一:
>
> create
> 如果文档不存在，那么就创建它。详情请见 创建新文档。
>
> index
> 创建一个新文档或者替换一个现有的文档。详情请见 索引文档 和 更新整个文档。
>
> update
> 部分更新一个文档。详情请见 文档的部分更新。
>
> delete
> 删除一个文档。详情请见 删除文档。
>
> metadata 应该 指定被索引、创建、更新或者删除的文档的 _index 、 _type 和 _id 。

例如，一个 delete 请求看起来是这样的：

{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }}
request body 行由文档的 _source 本身组成--文档包含的字段和值。它是 index 和 create 操作所必需的，这是有道理的：你必须提供文档以索引。

它也是 update 操作所必需的，并且应该包含你传递给 update API 的相同请求体： doc 、 upsert 、 script 等等。 删除操作不需要 request body 行。

{ "create":  { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
如果不指定 _id ，将会自动生成一个 ID ：

{ "index": { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }


!["bulk"](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/images/elas_0406.png)

bulk API 按如下步骤顺序执行：

1. 客户端向 Node 1 发送 bulk 请求。
2. Node 1 为每个节点创建一个批量请求，并将这些请求并行转发到每个包含主分片的节点主机。
主分片一个接一个按顺序执行每个操作。当每个操作成功时，主分片并行转发新文档（或删除）到副本分片，然后执行下一个操作。 一旦所有的副本分片报告所有操作成功，该节点将向协调节点报告成功，协调节点将这些响应收集整理并返回给客户端。
3. bulk API 还可以在整个批量请求的最顶层使用 consistency 参数，以及在每个请求中的元数据中使用 routing 参数。

优势，合并请求转发，减少网络IO开销

#### Http请求

请求体：

```json
POST _bulk
{ "index" : { "_index" : "test", "_type" : "_doc", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_type" : "_doc", "_id" : "2" } }
{ "create" : { "_index" : "test", "_type" : "_doc", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_type" : "_doc", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

响应体:

```json
{
   "took": 30,
   "errors": false,
   "items": [
      {
         "index": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 0,
            "_primary_term": 1
         }
      },
      {
         "delete": {
            "_index": "test",
            "_type": "_doc",
            "_id": "2",
            "_version": 1,
            "result": "not_found",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 404,
            "_seq_no" : 1,
            "_primary_term" : 2
         }
      },
      {
         "create": {
            "_index": "test",
            "_type": "_doc",
            "_id": "3",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 2,
            "_primary_term" : 3
         }
      },
      {
         "update": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 2,
            "result": "updated",
            "_shards": {
                "total": 2,
                "successful": 1,
                "failed": 0
            },
            "status": 200,
            "_seq_no" : 3,
            "_primary_term" : 4
         }
      }
   ]
}
```

与mget类似bulk执行成功就会返回200状态码，无论单个文档是否更新都是如此。