title: SpringData ES 关于字段名和索引中的列名字不一致导致的查询问题
date: 2017-05-24 00:33:33
tags:
- java
- elasticsearch
categories: java

----------------

最近工作中使用了[Spring Data Elasticsearch](https://github.com/spring-projects/spring-data-elasticsearch)。发生它存在一个问题：

**Document对应的POJO的属性跟es里面文档的字段名字不一样，这样Repository里面编写自定义的查询方法就会查询不出结果。**

比如有个Person类，它有2个属性goodFace和goodAt。这2个属性在es的索引里对应的字段表为good_face和good_at：

```java
@Document(replicas = 1, shards = 1, type = "person", indexName = "person")
@Getter
@Setter
@JsonNaming(PropertyNamingStrategy.SnakeCaseStrategy.class)
public class Person {
    @Id
    private String id;
    private String name;
    private boolean goodFace;
    private String goodAt;
}
```

Repository中的自定义查询：

```java
@Repository
public interface PersonRepository extends ElasticsearchRepository<Person, String> {
    List<Person> findByGoodFace(boolean isGoodFace);
    List<Person> findByName(String name);
}
```

方法findByGoodFace是查询不出结果的，而findByName是ok的。

为什么findByGoodFace不行而findByName可以呢，来探究一下。

<!--more-->

Person类的name属性跟ES中的字段名是一模一样的，而goodFace字段在ES中的字段是good_face(因为我们使用了SnakeCaseStrategy策略)。

所以产生这个问题的原因在于ES中文档的字段名跟POJO中的字段名不统一造成的。

**但是我们使用PersonRepository的save方法保存文档的时候属性和字段是可以对上的。**

那为什么使用repository的save方法能对应上文档和字段，而自定义的find方法却不行呢？

ES是使用[jackson](https://github.com/FasterXML/jackson)来完成POJO到json的映射关系的。

在Person类上使用@JsonNaming注解完成POJO和json的映射，我们使用了SnakeCaseStrategy策略，这个策略会把属性从驼峰方式改成小写带下划线的方式。

比如goodAt属性映射的时候就会变成good_at，good_face变成good_face，name变成name。

Spring Data Elasticsearch把对ES的操作封装成了一个ElasticsearchOperations接口。比如queryForObject、queryForPage、count、queryForList方法。

ElasticsearchOperations接口目前有一个实现类ElasticsearchTemplate。

ElasticsearchTemplate内部有个ResultsMapper属性，这个ResultsMapper目前只有一个实现类DefaultResultMapper，DefaultResultMapper内部使用DefaultEntityMapper完成映射。DefaultEntityMapper是个EntityMapper接口的实现类，它的定义如下：

```java
public interface EntityMapper {
  public String mapToString(Object object) throws IOException;
  public <T> T mapToObject(String source, Class<T> clazz) throws IOException;
}
```

方法很明白：对象到json字符串的转换和json字符串倒对象的转换。

DefaultEntityMapper内部使用jackson的ObjectMapper完成。


自定义的Repository继承自ElasticsearchRepository，最后会使用代理映射成SimpleElasticsearchRepository。

SimpleElasticsearchRepository内部有个属性ElasticsearchOperations用于完成与ES的交互。


我们看下SimpleElasticsearchRepository的save方法：

```java
@Override
public <S extends T> S save(S entity) {
  Assert.notNull(entity, "Cannot save 'null' entity.");
  // createIndexQuery方法会构造一个IndexQuery，然后调用ElasticsearchOperations的index方法
  elasticsearchOperations.index(createIndexQuery(entity));
  elasticsearchOperations.refresh(entityInformation.getIndexName());
  return entity;
}

// ElasticsearchTemplate的index方法
@Override
public String index(IndexQuery query) {
      // 调用prepareIndex方法构造一个IndexRequestBuilder
	String documentId = prepareIndex(query).execute().actionGet().getId();
	// 设置保存文档的id
	if (query.getObject() != null) {
		setPersistentEntityId(query.getObject(), documentId);
	}
	return documentId;
}

private IndexRequestBuilder prepareIndex(IndexQuery query) {
	try {
              // 从@Document注解中得到索引的名字
		String indexName = isBlank(query.getIndexName()) ? retrieveIndexNameFromPersistentEntity(query.getObject()
				.getClass())[0] : query.getIndexName();
              // 从@Document注解中得到索引的类型
		String type = isBlank(query.getType()) ? retrieveTypeFromPersistentEntity(query.getObject().getClass())[0]
				: query.getType();

		IndexRequestBuilder indexRequestBuilder = null;

		if (query.getObject() != null) { // save方法这里保存的object就是POJO
            // 得到id字段
			String id = isBlank(query.getId()) ? getPersistentEntityId(query.getObject()) : query.getId();
			if (id != null) { // 如果设置了id字段
				indexRequestBuilder = client.prepareIndex(indexName, type, id);
			} else { // 如果没有设置id字段
				indexRequestBuilder = client.prepareIndex(indexName, type);
			}
            // 使用ResultsMapper映射POJO到json字符串
			indexRequestBuilder.setSource(resultsMapper.getEntityMapper().mapToString(query.getObject()));
		} else if (query.getSource() != null) { // 如果自定义了source属性，直接赋值
			indexRequestBuilder = client.prepareIndex(indexName, type, query.getId()).setSource(query.getSource());
		} else { // 没有设置object属性或者source属性，抛出ElasticsearchException异常
			throw new ElasticsearchException("object or source is null, failed to index the document [id: " + query.getId() + "]");
		}
		if (query.getVersion() != null) { // 设置版本
			indexRequestBuilder.setVersion(query.getVersion());
			indexRequestBuilder.setVersionType(EXTERNAL);
		}

		if (query.getParentId() != null) { // 设置parentId
			indexRequestBuilder.setParent(query.getParentId());
		}

		return indexRequestBuilder;
	} catch (IOException e) {
		throw new ElasticsearchException("failed to index the document [id: " + query.getId() + "]", e);
	}
}
```

save方法使用ResultsMapper完成了POJO到json的转换，所以save方法保存成功对应的文档数据：

```java
indexRequestBuilder.setSource(resultsMapper.getEntityMapper().mapToString(query.getObject()));
```

自定义的findByGoodFace方法：

由于是Repository中的自定义方法，会被Spring Data通过代理进行构造，内部还是用了AOP，最终在QueryExecutorMethodInterceptor中并解析成ElasticsearchPartQuery这个RepositoryQuery接口的实现类，然后调用execute方法：

```java
@Override
public Object execute(Object[] parameters) {
  ParametersParameterAccessor accessor = new ParametersParameterAccessor(queryMethod.getParameters(), parameters);
  CriteriaQuery query = createQuery(accessor);
  if(tree.isDelete()) { // 如果是删除方法
    Object result = countOrGetDocumentsForDelete(query, accessor);
    elasticsearchOperations.delete(query, queryMethod.getEntityInformation().getJavaType());
    return result;
  } else if (queryMethod.isPageQuery()) { // 如果是分页查询
    query.setPageable(accessor.getPageable());
    return elasticsearchOperations.queryForPage(query, queryMethod.getEntityInformation().getJavaType());
  } else if (queryMethod.isStreamQuery()) { // 如果是流式查询
    Class<?> entityType = queryMethod.getEntityInformation().getJavaType();
    if (query.getPageable() == null) {
      query.setPageable(new PageRequest(0, 20));
    }

    return StreamUtils.createStreamFromIterator((CloseableIterator<Object>) elasticsearchOperations.stream(query, entityType));

  } else if (queryMethod.isCollectionQuery()) { // 如果是集合查询
    if (accessor.getPageable() == null) {
      int itemCount = (int) elasticsearchOperations.count(query, queryMethod.getEntityInformation().getJavaType());
      query.setPageable(new PageRequest(0, Math.max(1, itemCount)));
    } else {
        query.setPageable(accessor.getPageable());
      }
    return elasticsearchOperations.queryForList(query, queryMethod.getEntityInformation().getJavaType());
  } else if (tree.isCountProjection()) { // 如果是count查询
    return elasticsearchOperations.count(query, queryMethod.getEntityInformation().getJavaType());
  }
  // 单个查询
  return elasticsearchOperations.queryForObject(query, queryMethod.getEntityInformation().getJavaType());
}
```

findByGoodFace方法是个集合查询，最终会调用ElasticsearchOperations的queryForList方法：

```java
@Override
public <T> List<T> queryForList(CriteriaQuery query, Class<T> clazz) {
  // 调用queryForPage方法
  return queryForPage(query, clazz).getContent();
}

@Override
public <T> Page<T> queryForPage(CriteriaQuery criteriaQuery, Class<T> clazz) {
  // 查询解析器进行语法的解析
  QueryBuilder elasticsearchQuery = new CriteriaQueryProcessor().createQueryFromCriteria(criteriaQuery.getCriteria());
  QueryBuilder elasticsearchFilter = new CriteriaFilterProcessor().createFilterFromCriteria(criteriaQuery.getCriteria());
  SearchRequestBuilder searchRequestBuilder = prepareSearch(criteriaQuery, clazz);

  if (elasticsearchQuery != null) {
    searchRequestBuilder.setQuery(elasticsearchQuery);
  } else {
    searchRequestBuilder.setQuery(QueryBuilders.matchAllQuery());
  }

  if (criteriaQuery.getMinScore() > 0) {
    searchRequestBuilder.setMinScore(criteriaQuery.getMinScore());
  }

  if (elasticsearchFilter != null)
    searchRequestBuilder.setPostFilter(elasticsearchFilter);
  if (logger.isDebugEnabled()) {
    logger.debug("doSearch query:\n" + searchRequestBuilder.toString());
  }

  SearchResponse response = getSearchResponse(searchRequestBuilder
      .execute());
  // 最终的结果是用ResultsMapper进行映射
  return resultsMapper.mapResults(response, clazz, criteriaQuery.getPageable());
}
```

自定义的方法使用ElasticsearchQueryCreator去创建CriteriaQuery，内部做一些词法的分析，有了CriteriaQuery之后，使用CriteriaQueryProcessor基于Criteria构造了QueryBuilder，最后使用QueryBuilder去做rest请求得到es的查询结果。这些过程中是没有用到ResultsMapper，而只是用反射得到POJO的属性，只有在得到查询结果后才会用ResultsMapper去做映射。

如果出现了这种情况，解决方案目前有两种：

1.使用repository的search方法，参数可以是QueryBuilder或者SearchQuery

```java
personRepository.search(
        QueryBuilders.boolQuery()
                .must(QueryBuilders.termQuery("good_face", true))
)
```

2.使用@Query注解

```java
@Query("{\"bool\" : {\"must\" : {\"term\" : {\"good_face\" : \"?0\"}}}}")
List<Person> findByGoodFace(boolean isGoodFace);
```

暂时发现这两种解决方法，不知还有否更好的解决方案。
