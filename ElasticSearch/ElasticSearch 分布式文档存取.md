### 主分片与副分片的交互

我们可以发送请求到集群中的任一节点。 每个节点都有能力处理任意请求。 每个节点都知道集群中任一文档位置，所以可以直接将请求转发到需要的节点上。

例如写入一个文档。请求发送到任意一个节点node1，在node1节点上有所有的分片信息。此时的node1也称之为协调节点，当node1通过文档的id可计算出该索引下的主分片在什么位置（路由），然后将请求转发给主分片，主分片处理成功后，将请求转发到其副本分片中。
路由计算公式
```java
shard = hash(routing) % number_of_primary_shards
```
routing 是一个可变值，默认是文档的 _id ，也可以设置成一个自定义的值。 routing 通过 hash 函数生成一个数字，然后这个数字再除以 number_of_primary_shards （主分片的数量）后得到 余数 。这个分布在 0 到 number_of_primary_shards-1 之间的余数，就是我们所寻求的文档所在分片的位置。

这就解释了为什么我们要在创建索引的时候就确定好主分片的数量 并且永远不会改变这个数量：因为如果数量变化了，那么所有之前路由的值都会无效，文档也再也找不到了

#### 新建索引删除文档

  ![es写操作流程图](./elas_0402.png)

  以下是在主副分片和任何副本分片上面 成功新建，索引和删除文档所需要的步骤顺序：

1. 客户端向 Node 1 发送新建、索引或者删除请求。
2. 节点使用文档的 _id 确定文档属于分片 0 。请求会被转发到 Node 3`，因为分片 0 的主分片目前被分配在 `Node 3 上。
3. Node 3 在主分片上面执行请求。如果成功了，它将请求并行转发到 Node 1 和 Node 2 的副本分片上。一旦所有的副本分片都报告成功, Node 3 将向协调节点报告成功，协调节点向客户端报告成功。

在客户端收到成功响应时，文档变更已经在主分片和所有副本分片执行完成，变更是安全的。

一致性问题思考：
 
- 如果个别副本分片没有报告成功如何处理？

   - 主分片在写操作时会要求必须达到规定数量的活跃副本才会执行写操作，这样做防止了因网络故障而导致的数据不一致的问题
   - 默认的活跃数量为*int( (primary + number_of_replicas) / 2 ) + 1*。
   - 如果没有足够的副本数量es会等待直到超时

#### 查找文档

![es写操作流程图](./elas_0403.png)
以下是从主分片或者副本分片检索文档的步骤顺序：

1. 客户端向 Node 1 发送获取请求。

2. 节点使用文档的 _id 来确定文档属于分片 0 。分片 0 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 Node 2 。

3. Node 2 将文档返回给 Node 1 ，然后将文档返回给客户端。

在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡。

在文档被检索时，已经被索引的文档可能已经存在于主分片上但是还没有复制到副本分片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。 一旦索引请求成功返回给用户，文档在主分片和副本分片都是可用的。

### 局部文档更新

![es局部文档更新流程图](./elas_0404.png)

以下是部分更新一个文档的步骤：

1. 客户端向 Node 1 发送更新请求。
2. 它将请求转发到主分片所在的 Node 3 。
3. Node 3 从主分片检索文档，修改 _source 字段中的 JSON ，并且尝试重新索引主分片的文档。 如果文档已经被另一个进程修改，它会重试步骤 3 ，超过 retry_on_conflict 次后放弃。
如果 Node 3 成功地更新文档，它将新版本的文档并行转发到 Node 1 和 Node 2 上的副本分片，重新建立索引。 一旦所有副本分片都返回成功， Node 3 向协调节点也返回成功，协调节点向客户端返回成功。

注意：当主分片把更改转发到副本分片时， 它不会转发更新请求。 相反，它转发完整文档的新版本。请记住，这些更改将会异步转发到副本分片，并且不能保证它们以发送它们相同的顺序到达。 如果Elasticsearch仅转发更改请求，则可能以错误的顺序应用更改，导致得到损坏的文档。

#### 为什么不转发更新请求，而是转发完整的文档。
```json
{
  "name":"zhangsan",
  "sex":"male",
  "age":18
}
```

 假设A与B两个操作到达主分片,A 操作修改name为lisi、性别female，B操作修改age为20、性别male ,同时转发A请求到副本分片,然后转发请求B操作到副本分片。由于请求时异步的，我们假设A先到达B后到达（与主分片更新循序一直）

 - 主分片执行A操作，主分片最终文档结果如下。同时转发A请求到副本分片

 ```json
{
  "name":"lisi",
  "sex":"female",
  "age":18
}
```

 - 主分片执行B操作，主分片最终文档结果如下。同时转发B请求到副本分片

```json
{
  "name":"lisi",
  "sex":"male",
  "age":20
}
```
##### 转发请求方式处理 

- 若B操作先于A操作到达 *R0* 副本分片。

```json
{
  "name":"zhangsan",
  "sex":"male",
  "age":20
}

```
- 随后A到达副本分片此时在B的基础上进行更新操作，结果如下

```json
{
  "name":"lisi",
  "sex":"female",
  "age":18
}

```
这样也就造就了副本分片的顺序与主分片不一致，所以文档也就破坏了

- 若A操作按照主分片循序，先于B操作到达副本分片那么可以保证一致性。但是A，B操作时异步发送到副本分片的，无法保证循序，即使保证顺序也是牺牲性能，假设以保证循序的方式，B先到达，我必须等待A到达之后然后按照循序执行，这样牺牲了性能
- 通过版本号的方式来保证顺序？，这种方式，如果B先到达，那么必须等待A的到达然后执行，降低了系统的响应性能。
- 只要最终一致，每个文档都有版本号，如果以转发更新后的完整文档就不会出现以上情况。
    - A 先 B 后到达副本分片，都是A B更新之后的完整文档副本。那么二服务器只需要按照循序执行就行。
    - 如果B 先到达，副本分片执行B更新后的文档覆盖。此时A到达时。副本分片，发现B的文档版本号高于A操作的版本号，所以丢弃A的更新。但是由于是以整个文档更新的方式，所以无需担心主分片与副本分片不一致的问题

### 检索文档一致性问题

通过以上描述，我们知道主分片与副本分片都是可以进行检索的，那么当文档_id为xx的文档被更新时，同时我们在检索_id为xx的文档时会不会检索到还未更新的文档呢？理论上来讲是会出现这种情况的。
- 我们知道当文本更新时，协调节点会通过主分片进行转发到副本分片，如果副本分片更新失败的话，主分片得到的失败反馈到master节点，然后Master会在Meta中更新这个Index的InSyncAllocations配置，InSyncAllocations中维护着一个与主分片同步的列表，并将失败的副本分片从中移除。移除之后该副本分片将不再承担检索请求，换句话说就是es节点存在InSyncAllocations配置，读取请求会通过负载发送到各个节点，从而保证每一个节点都保持着与主分片数据一致，从而保证检索的数据一定是最新的。
- 在Meta更新到各个Node之前，用户可能还会读到这个Replica的数据，但是更新了Meta之后就不会了。所以这个方案并不是非常的严格，考虑到ES本身就是一个近实时系统，数据写入后需要refresh才可见，所以一般情况下，在短期内读到旧数据应该也是可接受的。