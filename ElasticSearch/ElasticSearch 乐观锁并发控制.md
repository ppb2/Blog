### Spring ElasticSearch data 乐观锁并发控制


#### 乐观锁与悲观锁

悲观锁(Pessimistic Lock), 顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。

乐观锁(Optimistic Lock), 顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库如果提供类似于write_condition机制的其实都是提供的乐观锁。

两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果经常产生冲突，上层应用会不断的进行retry，这样反倒是降低了性能，所以这种情况下用悲观锁就比较合适。


#### 并发问题的产生

我们知道Elasticsearch document维护者一个版本号，每次更新版本号都会增加。在我们使用elasticsearch时会存在这样一个问题，如果A用户在修改document x，version为1，B用户同时也在修改document x，version为1，此时AB用户同时都想修改version为1的document x。但是es的机制在并发情况下。B修改之后，A再修改但是A修改的不是原来的document，而把B修改的结果给覆盖了。

#### 乐观并发解决方案

为什么使用乐观并发方案？ 通过前面乐观锁的介绍，了解到乐观锁，认为每次我们拿数据的时候都认为别人不会做修改。再更新的时候加锁。因为es时一个读多修改少。而乐观锁还能带来大的吞吐量的优势。所以我们在修改一个document的时候先获取这个document，然后通过document的version来实现乐观并发控制。

#### Spring Data Elasticsearch 如何实现？

#####  @Version注解

org.springframework.data.annotation.Version;划分要用作版本字段的属性，以实现对实体的乐观锁定

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.ANNOTATION_TYPE})
public @interface Version {
}
```
##### 在实体中设置version字段

```java
Document(indexName = "xxxx", createIndex = false)
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Entity extends BaseModel {
    private String id;
    private String entityName;
    @Version
    private Long version;
    
}
```
1. 执行修改前先查询从而获取version
2. 执行修改时设置修改的值，同时将version设置为version+1
伪代码

```java
Optional<Entity> entityOptional = entityRepository.findById("xxx");
Entity entity = entityOptional.get();
entity.setEntityName("xxx");
....
entity.setVersion(entity.getVersion() + 1);
entityRepository.save(entity)
```

##### 版本冲突会发生什么？

```java
version conflict, current version [1] is higher or equal to the one provided [1]
```

#### 总结

到此乐观锁的实现就结束了，elasticsearch同样也存在全局锁，文档锁，树锁之类的，感兴趣可以继续研究。
 
 


