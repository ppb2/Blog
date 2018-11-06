### Spring Data Module如何确定使用那个库


在应用程序中使用唯一的Spring Data模块会使事情变得简单，因为定义范围内的所有存储库接口都绑定到Spring Data模块。有时，应用程序需要使用多个Spring Data模块。在这种情况下，存储库定义必须区分持久性技术。当它在类路径上检测到多个存储库工厂时，Spring Data进入严格的存储库配置模式。严格配置使用存储库或域类的详细信息来确定存储库定义的Spring Data模块绑定：

1. 如果存储库定义扩展了特定于模块的存储库，那么它是特定Spring Data模块的有效候选者。

2. 如果使用特定于模块的类型注释对域类进行注释，则它是特定Spring Data模块的有效候选者。Spring Data模块接受第三方注释（例如JPA @Entity）或提供自己的注释（例如@DocumentSpring Data MongoDB和Spring Data Elasticsearch）。

以下示例显示了使用特定于模块的接口的存储库（在本例中为ES）：

```java
@Repository
public interface CandidateRepository extends ElasticsearchRepository<Candidate, String> {
}

@NoRepositoryBean
public interface ElasticsearchRepository<T, ID extends Serializable> extends ElasticsearchCrudRepository<T, ID> {

	<S extends T> S index(S entity);

	Iterable<T> search(QueryBuilder query);

	Page<T> search(QueryBuilder query, Pageable pageable);

	Page<T> search(SearchQuery searchQuery);

	Page<T> searchSimilar(T entity, String[] fields, Pageable pageable);

	void refresh();

	Class<T> getEntityClass();
}
```

- 未指定特定的存储库
```java
interface PersonRepository extends Repository<Person, Long> {
 …
}

@Entity
class Person {
  …
}

interface UserRepository extends Repository<User, Long> {
 …
}

@Document
class User {
  …
}
```
如果未指定则可通过Repository引入的实体Person上的注解@Entity（jpa），@Document（es）


### 如何脱离Spring使用这些存储库

ElasticsearchRepositoryFactory 
- Spring Data模块提供了特定RepositoryFactory于持久性技术的特性，此类的作用用于生成代理Repository
- ElasticsearchRepositoryFactory继承自RepositoryFactorySupport

```java
public class ElasticsearchRepositoryFactory extends RepositoryFactorySupport {

	private final ElasticsearchOperations elasticsearchOperations;
	private final ElasticsearchEntityInformationCreator entityInformationCreator;

	public ElasticsearchRepositoryFactory(ElasticsearchOperations elasticsearchOperations) {

		Assert.notNull(elasticsearchOperations, "ElasticsearchOperations must not be null!");

		this.elasticsearchOperations = elasticsearchOperations;
		this.entityInformationCreator = new ElasticsearchEntityInformationCreatorImpl(
				elasticsearchOperations.getElasticsearchConverter().getMappingContext());
	}

	@Override
	public <T, ID> ElasticsearchEntityInformation<T, ID> getEntityInformation(Class<T> domainClass) {
		return entityInformationCreator.getEntityInformation(domainClass);
	}

	@Override
	@SuppressWarnings({ "rawtypes", "unchecked" })
	protected Object getTargetRepository(RepositoryInformation metadata) {
		return getTargetRepositoryViaReflection(metadata, getEntityInformation(metadata.getDomainType()),
				elasticsearchOperations);
	}

	@Override
	protected Class<?> getRepositoryBaseClass(RepositoryMetadata metadata) {
		if (isQueryDslRepository(metadata.getRepositoryInterface())) {
			throw new IllegalArgumentException("QueryDsl Support has not been implemented yet.");
		}
		if (Integer.class.isAssignableFrom(metadata.getIdType()) || Long.class.isAssignableFrom(metadata.getIdType())
				|| Double.class.isAssignableFrom(metadata.getIdType())) {
			return NumberKeyedRepository.class;
		} else if (metadata.getIdType() == String.class) {
			return SimpleElasticsearchRepository.class;
		} else if (metadata.getIdType() == UUID.class) {
			return UUIDElasticsearchRepository.class;
		} else {
			throw new IllegalArgumentException("Unsupported ID type " + metadata.getIdType());
		}
	}

	private static boolean isQueryDslRepository(Class<?> repositoryInterface) {
		return QUERY_DSL_PRESENT && QuerydslPredicateExecutor.class.isAssignableFrom(repositoryInterface);
	}

	@Override
	protected Optional<QueryLookupStrategy> getQueryLookupStrategy(Key key,
			QueryMethodEvaluationContextProvider evaluationContextProvider) {
		return Optional.of(new ElasticsearchQueryLookupStrategy());
	}

	private class ElasticsearchQueryLookupStrategy implements QueryLookupStrategy {

		/*
		 * (non-Javadoc)
		 * @see org.springframework.data.repository.query.QueryLookupStrategy#resolveQuery(java.lang.reflect.Method, org.springframework.data.repository.core.RepositoryMetadata, org.springframework.data.projection.ProjectionFactory, org.springframework.data.repository.core.NamedQueries)
		 */
		@Override
		public RepositoryQuery resolveQuery(Method method, RepositoryMetadata metadata, ProjectionFactory factory,
				NamedQueries namedQueries) {

			ElasticsearchQueryMethod queryMethod = new ElasticsearchQueryMethod(method, metadata, factory);
			String namedQueryName = queryMethod.getNamedQueryName();

			if (namedQueries.hasQuery(namedQueryName)) {
				String namedQuery = namedQueries.getQuery(namedQueryName);
				return new ElasticsearchStringQuery(queryMethod, elasticsearchOperations, namedQuery);
			} else if (queryMethod.hasAnnotatedQuery()) {
				return new ElasticsearchStringQuery(queryMethod, elasticsearchOperations, queryMethod.getAnnotatedQuery());
			}
			return new ElasticsearchPartQuery(queryMethod, elasticsearchOperations);
		}
	}
}
```

脱离Spring容器使用

```java
ElasticsearchRepositoryFactory elasticFactory = new ElasticsearchRepositoryFactory(elasticsearchTemplate);
T proxyRepository = (T) esFactory.getRepository(Repository.class);
```


### Spring Data 如何知道通过ElasticsearchRepositoryFactory来获取Repository的实现类代理

在Spring Elasticsearch data源码中有这个配置文件spring.factories

```
org.springframework.data.repository.core.support.RepositoryFactorySupport=org.springframework.data.elasticsearch.repository.support.ElasticsearchRepositoryFactory

```
这个配置文件起到的作用。
Spring Factories实现原理（自己了解），spring-core包里定义了SpringFactoriesLoader类，这个类实现了检索META-INF/spring.factories文件，并获取指定接口的配置的功能。
在这个类中定义了两个对外的方法，在此处定义的不会与其他包下的factories冲突。

Spring通过这里知道该使用ElasticsearchRepositoryFactory来创建Repository。

### 动态索引的实现

#### 跳出Spring限制

- 因为在项目中默认的使用的是单例ElasticsearchRepositoryFactory来创建，而client，ElasticsearchTemplate都是单例，而indexName早在@Document中早已绑定
```java
@Document(indexName = "XXX'}")
```
- 而在创建中通过跟踪源码发现indexName 是通过ElasticsearchTemplate的ElasticsearchPersistentEntity对象中获取的.

ElasticsearchTemplate
```java
@Override
public <T> boolean putMapping(Class<T> clazz, Object mapping) {
		return putMapping(getPersistentEntityFor(clazz).getIndexName(), getPersistentEntityFor(clazz).getIndexType(),
				mapping);
	}
@Override
public <T> Map getMapping(Class<T> clazz) {
		return getMapping(getPersistentEntityFor(clazz).getIndexName(), getPersistentEntityFor(clazz).getIndexType());
	}
 public <T> T queryForObject(GetQuery query, Class<T> clazz, GetResultMapper mapper) {
		ElasticsearchPersistentEntity<T> persistentEntity = getPersistentEntityFor(clazz);
		GetRequest request = new GetRequest(persistentEntity.getIndexName(), persistentEntity.getIndexType(),
				query.getId());
		GetResponse response;
		try {
			response = client.get(request);
			T entity = mapper.mapResult(response, clazz);
			return entity;
		} catch (IOException e) {
			throw new ElasticsearchException("Error while getting for request: " + request.toString(), e);
		}
	}   

```
ElasticsearchPersistentEntity 以SimpleElasticsearchPersistentEntity为例

```java
	private String indexName;
	private String indexType;
	private boolean useServerConfiguration;
	private short shards;
	private short replicas;
	private String refreshInterval;
	private String indexStoreType;
	private String parentType;
@Override
public String getIndexName() {
		Expression expression = parser.parseExpression(indexName, ParserContext.TEMPLATE_EXPRESSION);
		return expression.getValue(context, String.class);
}
```
这里会根据indexName获取索引名，当然这里也支持表达式的方式（#{indexNameGenerator.getName('ats_candidate')}），所以通过Expression来解析。如果我们直接修改indexName，则就达到了我们的目的。


#### 排除多线程并发影响

- 由上可见ElasticsearchTemplate成了我们突破的关键。
- 使得我们的ElasticsearchTemplate作用域不再是单例（@Scope("prototype")）
- 反射技术修改ElasticsearchPersistentEntity的indexName
- 共用同一个client实例
- 使得代码改动最小

配置类

```java
@Configuration
@Profile("dev")
public class ElasticSearchConfig {

    @Bean
    @Scope("prototype")
    public ElasticsearchTemplate elasticsearchTemplate() {
        return new ElasticsearchTemplate(client());
    }

    @Bean
    public Client client() {
        Settings.Builder settingsBuilder =
                Settings.builder()
                        .put("cluster.name", "ATS-CLUSTER")
                        .put("client.transport.sniff", false);
        Settings settings = settingsBuilder.build();
        TransportClient client = new PreBuiltTransportClient(settings);
        try {
            client.addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("10.150.27.149"), 9300));
            client.addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("10.150.27.15"), 9300));
        } catch (UnknownHostException e) {
            e.printStackTrace();
        }
        return client;
    }
}

```
### 构建动态索引的ElasticSearchTemplate

- 通过上面的分析我们通过反射的方法获取并修改默认的index

```java
@Component
public class DynamicIndexElasticsearchTemplate implements ApplicationContextAware {
    private static final Logger logger = LoggerFactory.getLogger(DynamicIndexElasticsearchTemplate.class);
    private ApplicationContext applicationContext;

    protected ElasticsearchTemplate getElasticsearchTemplate() {
        return applicationContext.getBean(ElasticsearchTemplate.class);
    }

    public ElasticsearchTemplate getElasticsearchTemplate(String indexPrefix, Class cls) {
        ElasticsearchTemplate esTemplate = getElasticsearchTemplate();
        setIndex(indexPrefix, cls, esTemplate);
        return esTemplate;
    }

    protected void setIndex(String indexPrefix, Class cls, ElasticsearchTemplate elasticsearchTemplate) {
        ElasticsearchPersistentEntity entity = elasticsearchTemplate.getPersistentEntityFor(cls);
        try {
            Field field = SimpleElasticsearchPersistentEntity.class.getDeclaredField("indexName");
            field.setAccessible(true);
            String indexDefault = field.get(entity).toString();
            if (!StringUtils.isEmpty(indexPrefix)) {
                field.set(entity, indexPrefix + "_" + indexDefault);
            }
        } catch (IllegalAccessException e) {
            logger.error("can not access field: ", e);
        } catch (NoSuchFieldException e) {
            logger.error("no such field: ", e);
        }
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}

```
#### 调用方式

```java
 @Autowired
private DynamicIndexElasticsearchTemplate dynamicIndexElasticsearchTemplate;
 @Test
public void searchByElasticsearchTemplate() {
      
    NativeSearchQueryBuilder nativeSearchQueryBuilder = new NativeSearchQueryBuilder();
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    boolQuery.must(QueryBuilders.termsQuery("id", "123"));
    SearchQuery searchQuery = nativeSearchQueryBuilder.withFilter(boolQuery).build();
    List<XXX> XXXList = dynamicIndexElasticsearchTemplate
            .getElasticsearchTemplate("ut", XXX.class)
            .queryForList(searchQuery, XXX.class);
    assertEquals(1, XXXList.size());
}
```

### 构建动态索引Repository

#### 问题发现

- 如果要修改使得原有的Repository可用，仅仅只修改*ElasticsearchTemplate*的*ElasticsearchPersistentEntity*是不能完全解决问题的，因为我们发现这个只在sava的时候起作用

AbstractElasticsearchRepository.class

```java
public abstract class AbstractElasticsearchRepository<T, ID extends Serializable> implements ElasticsearchRepository<T, ID> {
    static final Logger LOGGER = LoggerFactory.getLogger(AbstractElasticsearchRepository.class);
    protected ElasticsearchOperations elasticsearchOperations;
    protected Class<T> entityClass;
    protected ElasticsearchEntityInformation<T, ID> entityInformation;

    public AbstractElasticsearchRepository() {
    }

    public AbstractElasticsearchRepository(ElasticsearchOperations elasticsearchOperations) {
        Assert.notNull(elasticsearchOperations, "ElasticsearchOperations must not be null!");
        this.setElasticsearchOperations(elasticsearchOperations);
    }

    public AbstractElasticsearchRepository(ElasticsearchEntityInformation<T, ID> metadata, ElasticsearchOperations elasticsearchOperations) {
        this(elasticsearchOperations);
        Assert.notNull(metadata, "ElasticsearchEntityInformation must not be null!");
        this.entityInformation = metadata;
        this.setEntityClass(this.entityInformation.getJavaType());

        try {
            if (this.createIndexAndMapping()) {
                this.createIndex();
                this.putMapping();
            }
        } catch (ElasticsearchException var4) {
            LOGGER.error("failed to load elasticsearch nodes : " + var4.getDetailedMessage());
        }

    }
	public <S extends T> S save(S entity) {
        Assert.notNull(entity, "Cannot save 'null' entity.");
        this.elasticsearchOperations.index(this.createIndexQuery(entity));
        this.elasticsearchOperations.refresh(this.entityInformation.getIndexName());
        return entity;
    }

    public <S extends T> List<S> save(List<S> entities) {
        Assert.notNull(entities, "Cannot insert 'null' as a List.");
        Assert.notEmpty(entities, "Cannot insert empty List.");
        List<IndexQuery> queries = new ArrayList();
        Iterator var3 = entities.iterator();

        while(var3.hasNext()) {
            S s = var3.next();
            queries.add(this.createIndexQuery(s));
        }

        this.elasticsearchOperations.bulkIndex(queries);
        this.elasticsearchOperations.refresh(this.entityInformation.getIndexName());
        return entities;
    }
	public void deleteById(ID id) {
        Assert.notNull(id, "Cannot delete entity with id 'null'.");
        this.elasticsearchOperations.delete(this.entityInformation.getIndexName(), this.entityInformation.getType(), this.stringIdRepresentation(id));
        this.elasticsearchOperations.refresh(this.entityInformation.getIndexName());
    }

    public void delete(T entity) {
        Assert.notNull(entity, "Cannot delete 'null' entity.");
        this.deleteById(this.extractIdFromBean(entity));
        this.elasticsearchOperations.refresh(this.entityInformation.getIndexName());
    }

    public void deleteAll(Iterable<? extends T> entities) {
        Assert.notNull(entities, "Cannot delete 'null' list.");
        Iterator var2 = entities.iterator();

        while(var2.hasNext()) {
            T entity = var2.next();
            this.delete(entity);
        }

    }

    public void deleteAll() {
        DeleteQuery deleteQuery = new DeleteQuery();
        deleteQuery.setQuery(QueryBuilders.matchAllQuery());
        this.elasticsearchOperations.delete(deleteQuery, this.getEntityClass());
        this.elasticsearchOperations.refresh(this.entityInformation.getIndexName());
    }


```
- 如上所示，经分析发现sava，delete等方法都是从如下方式获取*indexName*,indexName 并非是从*ElasticsearchTemplate*的*ElasticsearchPersistentEntity*中获取的。虽然最终调用的是*ElasticsearchTemplate*的方法

```java
this.entityInformation.getIndexName()
```

### ElasticsearchEntityInformation

#### 分析

- *ElasticsearchEntityInformation* 的实现类 *MappingElasticsearchEntityInformation*

```java
public class MappingElasticsearchEntityInformation<T, ID> extends PersistentEntityInformation<T, ID> implements ElasticsearchEntityInformation<T, ID> {
    private final ElasticsearchPersistentEntity<T> entityMetadata;
    private final String indexName;
    private final String type;
}
```
- 修改final定义的indexName即可。final无法修改？ 反射解决！

#### 如何反射处理？

- *ElasticsearchRepositoryFactory* 创建Repository的代理工厂类。*ElasticsearchEntityInformation* 由 *ElasticsearchEntityInformationCreatorImpl*创建

```java
public class ElasticsearchRepositoryFactory extends RepositoryFactorySupport {
    private final ElasticsearchOperations elasticsearchOperations;
    private final ElasticsearchEntityInformationCreator entityInformationCreator;

    public ElasticsearchRepositoryFactory(ElasticsearchOperations elasticsearchOperations) {
        Assert.notNull(elasticsearchOperations, "ElasticsearchOperations must not be null!");
        this.elasticsearchOperations = elasticsearchOperations;
        this.entityInformationCreator = new ElasticsearchEntityInformationCreatorImpl(elasticsearchOperations.getElasticsearchConverter().getMappingContext());
    }

    public <T, ID> ElasticsearchEntityInformation<T, ID> getEntityInformation(Class<T> domainClass) {
        return this.entityInformationCreator.getEntityInformation(domainClass);
    }

    protected Object getTargetRepository(RepositoryInformation metadata) {
        return this.getTargetRepositoryViaReflection(metadata, new Object[]{this.getEntityInformation(metadata.getDomainType()), this.elasticsearchOperations});
    }
}
```
- *getTargetRepository* 此方法最终得到代理Repository。

```java
T proxyRepository = (T) esFactory.getRepository(Repository.class);

```
- *getEntityInformation* 为代理Repository创建 *MappingElasticsearchEntityInformation* 其中就有从解析@Document注解获取的。需要动态的修改此indexName。

#### 自定义ElasticsearchRepositoryFactory

- 自定义的目的为了覆盖*getEntityInformation* 并且通过反射修改indexname。（此处仅在原有的index加上前缀，可以根据需求自己定义）

```java
public class CustomElasticsearchRepositoryFactory extends RepositoryFactorySupport {
    private static final Logger logger = LoggerFactory.getLogger(CustomElasticsearchRepositoryFactory.class);
    private final ElasticsearchOperations elasticsearchOperations;
    private final ElasticsearchEntityInformationCreator entityInformationCreator;
    private String indexPrefix;

    public CustomElasticsearchRepositoryFactory(ElasticsearchOperations elasticsearchOperations, String indexPrefix) {
        Assert.notNull(elasticsearchOperations, "ElasticsearchOperations must not be null!");
        this.elasticsearchOperations = elasticsearchOperations;
        this.indexPrefix = indexPrefix;
        this.entityInformationCreator = new ElasticsearchEntityInformationCreatorImpl(elasticsearchOperations.getElasticsearchConverter().getMappingContext());
    }

    private void setEntityInformationIndexName(MappingElasticsearchEntityInformation entityInformation) {
        try {
            Field field = MappingElasticsearchEntityInformation.class.getDeclaredField("indexName");
            field.setAccessible(true);
            String indexDefault = field.get(entityInformation).toString();
            if (!StringUtils.isEmpty(this.indexPrefix)) {
                field.set(entityInformation, this.indexPrefix + "_" + indexDefault);
            }
        } catch (IllegalAccessException e) {
            logger.error("can not access field: ", e);
        } catch (NoSuchFieldException e) {
            logger.error("no such field: ", e);
        }
    }

    public <T, ID> ElasticsearchEntityInformation<T, ID> getEntityInformation(Class<T> domainClass) {
        ElasticsearchEntityInformation entityInformation = this.entityInformationCreator.getEntityInformation(domainClass);
        if (!StringUtils.isEmpty(this.indexPrefix)) {
            setEntityInformationIndexName((MappingElasticsearchEntityInformation) entityInformation);
        }
        return entityInformation;

    }

    protected Object getTargetRepository(RepositoryInformation metadata) {
        return this.getTargetRepositoryViaReflection(metadata, new Object[]{this.getEntityInformation(metadata.getDomainType()), this.elasticsearchOperations});
    }

    protected Class<?> getRepositoryBaseClass(RepositoryMetadata metadata) {
        if (isQueryDslRepository(metadata.getRepositoryInterface())) {
            throw new IllegalArgumentException("QueryDsl Support has not been implemented yet.");
        } else if (!Integer.class.isAssignableFrom(metadata.getIdType()) && !Long.class.isAssignableFrom(metadata.getIdType()) && !Double.class.isAssignableFrom(metadata.getIdType())) {
            if (metadata.getIdType() == String.class) {
                return SimpleElasticsearchRepository.class;
            } else if (metadata.getIdType() == UUID.class) {
                return UUIDElasticsearchRepository.class;
            } else {
                throw new IllegalArgumentException("Unsupported ID type " + metadata.getIdType());
            }
        } else {
            return NumberKeyedRepository.class;
        }
    }

    private static boolean isQueryDslRepository(Class<?> repositoryInterface) {
        return QuerydslUtils.QUERY_DSL_PRESENT && QuerydslPredicateExecutor.class.isAssignableFrom(repositoryInterface);
    }

    private class ElasticsearchQueryLookupStrategy implements QueryLookupStrategy {
        private ElasticsearchQueryLookupStrategy() {
        }

        public RepositoryQuery resolveQuery(Method method, RepositoryMetadata metadata, ProjectionFactory factory, NamedQueries namedQueries) {
            ElasticsearchQueryMethod queryMethod = new ElasticsearchQueryMethod(method, metadata, factory);
            String namedQueryName = queryMethod.getNamedQueryName();
            if (namedQueries.hasQuery(namedQueryName)) {
                String namedQuery = namedQueries.getQuery(namedQueryName);
                return new ElasticsearchStringQuery(queryMethod, CustomElasticsearchRepositoryFactory.this.elasticsearchOperations, namedQuery);
            } else {
                return (RepositoryQuery) (queryMethod.hasAnnotatedQuery() ? new ElasticsearchStringQuery(queryMethod, CustomElasticsearchRepositoryFactory.this.elasticsearchOperations, queryMethod.getAnnotatedQuery()) : new ElasticsearchPartQuery(queryMethod, CustomElasticsearchRepositoryFactory.this.elasticsearchOperations));
            }
        }
    }

    protected Optional<QueryLookupStrategy> getQueryLookupStrategy(QueryLookupStrategy.Key key, EvaluationContextProvider evaluationContextProvider) {
        return Optional.of(new CustomElasticsearchRepositoryFactory.ElasticsearchQueryLookupStrategy());
    }
}
```
#### 定义ESDynamicIndexRepository

- 此类用于新的Repository继承，并修改index返回代理的Repository

```java
public class ESDynamicIndexRepository<T> {
    private static final Logger logger = LoggerFactory.getLogger(ESDynamicIndexRepository.class);
    @Autowired
    private DynamicIndexElasticsearchTemplate dynamicIndexElasticsearchTemplate;

    private CustomElasticsearchRepositoryFactory getFactory(String indexPrefix, ElasticsearchTemplate elasticsearchTemplate) {
        CustomElasticsearchRepositoryFactory elasticFactory = new CustomElasticsearchRepositoryFactory(elasticsearchTemplate, indexPrefix);
        return elasticFactory;
    }

    @SuppressWarnings("unchecked")
    private Class<T> resolveReturnedClassFromGenericType() {
        ParameterizedType parameterizedType = resolveReturnedClassFromGenericType(getClass());
        return (Class<T>) parameterizedType.getActualTypeArguments()[0];
    }

    private ParameterizedType resolveReturnedClassFromGenericType(Class<?> clazz) {
        Object genericSuperclass = clazz.getGenericSuperclass();
        if (genericSuperclass instanceof ParameterizedType) {
            return (ParameterizedType) genericSuperclass;
        }
        return resolveReturnedClassFromGenericType(clazz.getSuperclass());
    }

    /**
     * @param indexPrefix
     * @param cls         domain.class  interview.class
     * @return
     */
    public ElasticsearchTemplate getElasticsearchTemplate(String indexPrefix, Class cls) {
        ElasticsearchTemplate esTemplate = dynamicIndexElasticsearchTemplate.getElasticsearchTemplate();
        dynamicIndexElasticsearchTemplate.setIndex(indexPrefix, cls, esTemplate);
        return esTemplate;
    }

    private Class getClazz(Class proxy) {
        Type[] types = proxy.getGenericInterfaces();
        Type t1 = ((ParameterizedType) types[0]).getActualTypeArguments()[0];
        return (Class) t1;
    }


    public T proxyRepository() {
        return getProxyRepository(null);
    }

    public T getProxyRepository(String indexPrefix) {
        Class<T> proxy = resolveReturnedClassFromGenericType();
        if (proxy.getClass().isInstance(ElasticsearchRepository.class)) {
            ElasticsearchTemplate esTemplate = dynamicIndexElasticsearchTemplate.getElasticsearchTemplate();
            CustomElasticsearchRepositoryFactory esFactory = getFactory(indexPrefix, esTemplate);
            T proxyRepository = esFactory.getRepository(proxy);
            if (!StringUtils.isEmpty(indexPrefix)) {
                dynamicIndexElasticsearchTemplate.setIndex(indexPrefix, getClazz(proxy), esTemplate);
            }
            return proxyRepository;
        } else {
            throw new RuntimeException("do not support thie proxy class");
        }
    }

}
```

#### 原Repository，Spring Data ElasticSearch框架

```java
@Repository
public interface UserRepository extends ElasticsearchRepository<User, String> {
    User getUserByLoginName(String loginName);
    User getUserByEmail(String email);
    User getUserByTelephone(String telephone);
    List<User> getUserByDetailedInfoEmployeeId(String employeeId);
}
```

#### 在原Repository上制定能动态修改的indexName的Repository

```java
@Component
public class UserDynamicRepository extends ESDynamicIndexRepository<UserRepository> {
}
```

#### 用法

- 获得修改indexName后的原Repository，像原Repository一样取调用

```java
@Autowired
private UserDynamicRepository userDynamicRepository;

UserRepository UserRepository = userDynamicRepository.getProxyRepository("index_name")
UserRepository.save(user)


```




