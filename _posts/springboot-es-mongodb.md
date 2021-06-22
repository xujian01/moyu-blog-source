---
title: SpringBoot中使用ES和MongoDB常用API
date: 2020-12-20 10:33:29
tags: SpringBoot,ES,MongoDB
---
> 本文主要介绍一些ES和MongoDB的API使用，请不要纠结代码上下文。
# 1、调用ES接口
> 使用RestHighLevelClient


```xml
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>7.6.0</version>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-client</artifactId>
    <version>7.6.0</version>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.6.0</version>
</dependency>
```


```yaml
spring:
    elasticsearch:
        rest:
            uris: http://localhost:9200
            username: contentdev
            password: contentdevbeta123
```


## 1、条件查询+分页+排序

```java
@Autowired
private RestHighLevelClient client;
```

```java
//创建查询请求，“gushici”代表es的集合
SearchRequest searchRequest = new SearchRequest("gushici");
//构建查询条件
        SearchSourceBuilder searchSourceBuilder = buildSearchQuery(guShiCiListParam);
//应用查询条件，固定写法
        searchRequest.source(searchSourceBuilder);
        try {
//调用接口进行查询，RequestOptions.DEFAULT为请求选项，一般固定写成这个
            SearchResponse searchResponse = client.search(searchRequest,RequestOptions.DEFAULT);
            if (searchResponse.status() == RestStatus.OK) {
//处理es查询结果
                handleSearchResponse(result, searchResponse);
            }
        }
//构建查询条件
private SearchSourceBuilder buildSearchQuery(GuShiCiListParam guShiCiListParam) {
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
//构建布尔查询
        BoolQueryBuilder boolQueryBuilder = buildBoolQueryBuilder(guShiCiListParam);
//应用布尔查询，固定写法
        searchSourceBuilder.query(boolQueryBuilder);
        if (guShiCiListParam.getPageNo() != null && guShiCiListParam.getPageSize() != null) {
//设置起始偏移量
            searchSourceBuilder.from((guShiCiListParam.getPageNo() - 1)* guShiCiListParam.getPageSize());
//设置终止偏移量
            searchSourceBuilder.size(guShiCiListParam.getPageSize());
        }
        if (guShiCiListParam.getSortRegx() != null) {
            List<Map<String,String>> sortRegx = guShiCiListParam.getSortRegx();
            for (Map<String,String> map : sortRegx) {
                String sortField = map.get("sortField");
                if (StringUtils.isEmpty(sortField)) {
                    continue;
                }
                String sortType = map.get("sortType");
//设置排序，new FieldSortBuilder(sortField)设置排序字段，.order设置排序类型，可以调用多次sort方法，表示多个字段排序
                searchSourceBuilder.sort(new FieldSortBuilder(sortField).order("DESC".equals(sortType) ? SortOrder.DESC : SortOrder.ASC));
            }
        }
        return searchSourceBuilder;
    }
 
//构建布尔查询
private BoolQueryBuilder buildBoolQueryBuilder(GuShiCiListParam guShiCiListParam) {
        BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
        if (guShiCiListParam.getVersionId() != null) {
//filter相当于mysql中的“and”的意思，QueryBuilders.termQuery构建一个term查询，表示某个字段等于某个值，filter可以调用多次，实现xxx and xxx and xxx的效果
            boolQueryBuilder.filter(QueryBuilders.termQuery("versionId", guShiCiListParam.getVersionId()));
        }
        if (guShiCiListParam.getCopyrightRisk() != null) {
            boolQueryBuilder.filter(QueryBuilders.termQuery("copyrightRisk", guShiCiListParam.getCopyrightRisk()));
        }
        if (guShiCiListParam.getOffline() != null) {
            boolQueryBuilder.filter(QueryBuilders.termQuery("offline", guShiCiListParam.getOffline()));
        }
        return boolQueryBuilder;
    }
 
//处理es查询结果
private void handleSearchResponse(List<GuShiCiSummaryDTO> result, SearchResponse searchResponse) {
//获取命中的结果
        SearchHits hits = searchResponse.getHits();
        SearchHit[] searchHits = hits.getHits();
        for (SearchHit hit : searchHits) {
//获取每个结果中的字段数据
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
//获取高亮字段信息
            Map<String,HighlightField> highlightFieldMap = hit.getHighlightFields();
            GuShiCiSummaryDTO shiCiSummaryDTO = new GuShiCiSummaryDTO();
//获取指定字段的值
            shiCiSummaryDTO.setId((String) sourceAsMap.get("guShiCiId"));
            String author = (String) sourceAsMap.get("author");
//获取指定字段的高亮信息
            HighlightField highlightField = highlightFieldMap.get("author");
//获取加了高亮之后的字段内容，如本来author字段的值为“李白”，高亮以后是“<em>李</em>白”，需要用高亮以后的信息替换原始的信息
            StringBuilder c = new StringBuilder();
            if (highlightField != null) {
                for (Text text : highlightField.getFragments()) {
                    c.append(text);
                }
            }
            if (c.length() > 0) {
                author = c.toString();
            }
            shiCiSummaryDTO.setAuthor(author);
    }
```

## 2、统计符合条件的数据总数

```java
//构建查询数量请求
CountRequest countRequest = new CountRequest(matrixGuShiCiCollection);
//应用查询条件
        countRequest.query(buildBoolQueryBuilder(guShiCiListParam));
        try {
//调用count接口，固定写法
            CountResponse countResponse = client.count(countRequest,RequestOptions.DEFAULT);
            if (countResponse.status() == RestStatus.OK) {
                return countResponse.getCount();
            }
        }
```

## 3、分词搜索+分页+排序+高亮

```java
List<GuShiCiSummaryDTO> result = new ArrayList<>();
        SearchRequest searchRequest = new SearchRequest(matrixGuShiCiCollection);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
//构建MultiMatch分词查询，"title","author","gen","body"是要分词查询的字段
        MultiMatchQueryBuilder multiMatchQueryBuilder = new MultiMatchQueryBuilder(searchAnalysedParam.getKeyword().trim(),"title","author","gen","body");
//设置每个字段查询的权重
        multiMatchQueryBuilder.field("title",100).field("author",10).field("gen",10)
                .field("body",2).analyzer("ik_smart");
        searchSourceBuilder.query(multiMatchQueryBuilder);
//配置高亮选项，field("title")是需要高亮的字段，preTags()和postTags()用来设置高亮的首尾标签，fragmentSize设置高亮结果每个片段的字符串长度，numOfFragments设置将高亮结果分为多少个片段
        HighlightBuilder highlightBuilder = new HighlightBuilder().field("title").field("author").field("gen")
                .field("body").preTags(searchAnalysedParam.getPreTag()).postTags(searchAnalysedParam.getPostTag()).fragmentSize(800000).numOfFragments(1);
//应用高亮设置
        searchSourceBuilder.highlighter(highlightBuilder);
//设置分页
        if (searchAnalysedParam.getPageNo() != null && searchAnalysedParam.getPageSize() != null) {
            searchSourceBuilder.from((searchAnalysedParam.getPageNo() - 1) * searchAnalysedParam.getPageSize());
            searchSourceBuilder.size(searchAnalysedParam.getPageSize());
        }
//设置排序
        searchSourceBuilder.sort(new FieldSortBuilder("createTime").order(SortOrder.DESC));
        searchRequest.source(searchSourceBuilder);
        try {
            SearchResponse searchResponse = client.search(searchRequest,RequestOptions.DEFAULT);
            if (searchResponse.status() == RestStatus.OK) {
                handleSearchResponse(result, searchResponse);
            }
        }
 
 
        SearchHits hits = searchResponse.getHits();
        SearchHit[] searchHits = hits.getHits();
        for (SearchHit hit : searchHits) {
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            Map<String,HighlightField> highlightFieldMap = hit.getHighlightFields();
            GuShiCiSummaryDTO shiCiSummaryDTO = new GuShiCiSummaryDTO();
            shiCiSummaryDTO.setId((String) sourceAsMap.get("guShiCiId"));
            String author = (String) sourceAsMap.get("author");
            HighlightField highlightField = highlightFieldMap.get("author");
            StringBuilder c = new StringBuilder();
            if (highlightField != null) {
                for (Text text : highlightField.getFragments()) {
                    c.append(text);
                }
            }
            if (c.length() > 0) {
                author = c.toString();
            }
        }
```

## 4、模糊搜索（不分词）+分页+排序+高亮（不分词，手动处理高亮）

```java
SearchRequest searchRequest = new SearchRequest(matrixGuShiCiCollection);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
//构建布尔查询
        BoolQueryBuilder boolQueryBuilder = buildSearchSplitKeyWordBuilder(keyword);
//应用查询条件
        searchSourceBuilder.query(boolQueryBuilder);
//设置分页信息
        if (pageNo != null && pageSize != null) {
            searchSourceBuilder.from((pageNo - 1)*pageSize);
            searchSourceBuilder.size(pageSize);
        }
//设置排序信息
        searchSourceBuilder.sort(new FieldSortBuilder("createTime").order(SortOrder.DESC));
//应用查询
        searchRequest.source(searchSourceBuilder);
        try {
//调用查询接口
            SearchResponse searchResponse = client.search(searchRequest,RequestOptions.DEFAULT);
            if (searchResponse.status() == RestStatus.OK) {
//处理查询结果
                handleSearchResponse(result, searchResponse);
            }
            if (StringUtils.isNotEmpty(preTag) && StringUtils.isNotEmpty(postTag)) {
//处理查询结果高亮
                handleHighLightRegex(result,keyword,preTag,postTag);
            }
        }
 
//构建布尔查询
private BoolQueryBuilder buildSearchSplitKeyWordBuilder(String keyword) {
        BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
        String[] keywords = keyword.split(" ");
        for (String key:keywords){
            if (StringUtils.isEmpty(key)) {
                continue;
            }
//should相当于mysql中的“OR”，QueryBuilders.regexpQuery表示使用正则匹配，类似于mysql中的like，boost()设置权重，多次调用should，实现xxx OR xxx OR xxx OR xxx
            boolQueryBuilder.should(QueryBuilders.regexpQuery("title.raw",".*"+key+".*").boost(100));
            boolQueryBuilder.should(QueryBuilders.regexpQuery("author.raw",".*"+key+".*").boost(10));
            boolQueryBuilder.should(QueryBuilders.regexpQuery("gen.raw",".*"+key+".*").boost(10));
            boolQueryBuilder.should(QueryBuilders.regexpQuery("body.raw",".*"+key+".*").boost(2));
        }
        return boolQueryBuilder;
    }
 
//处理查询结果高亮
private void handleHighLightRegex(List<GuShiCiSummaryDTO> result,String keyword,String preTag,String postTag){
        String replacement;
        String[] keywords = keyword.split(" ");
        preTag = preTag.replaceAll("\"","'");
        postTag = postTag.replaceAll("\"","'");
        for (GuShiCiSummaryDTO guShiCiSummaryDTO:result){
            for (String kw : keywords) {
                replacement = preTag + kw + postTag;
                if (CommonUtils.stringIsNotEmpty(guShiCiSummaryDTO.getTitle())){
                    guShiCiSummaryDTO.setTitle(guShiCiSummaryDTO.getTitle().replaceAll(kw,replacement));
                }
                if (CommonUtils.stringIsNotEmpty(guShiCiSummaryDTO.getAuthor())){
                    guShiCiSummaryDTO.setAuthor(guShiCiSummaryDTO.getAuthor().replaceAll(kw,replacement));
                }
                if (CommonUtils.stringIsNotEmpty(guShiCiSummaryDTO.getGen())){
                    guShiCiSummaryDTO.setGen(guShiCiSummaryDTO.getGen().replaceAll(kw,replacement));
                }
                if (CommonUtils.stringIsNotEmpty(guShiCiSummaryDTO.getBody())){
                    guShiCiSummaryDTO.setBody(guShiCiSummaryDTO.getBody().replaceAll(kw,replacement));
                }
            }
        }
    }
```

## 5、查询只返回数据在ES中的id

```java
SearchRequest searchRequest = new SearchRequest(matrixGuShiCiCollection);
        SearchSourceBuilder searchSourceBuilder = buildSearchQuery(guShiCiListParam);
//不获取_source字段信息，即不获取数据的所有字段信息，查询结果只包含ES的元数据，包括ES的_id字段
        searchSourceBuilder.fetchSource(false);
        searchRequest.source(searchSourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest,RequestOptions.DEFAULT);
```

## 6、根据条件修改字段值
```java
//根据条件修改请求
UpdateByQueryRequest updateByQueryRequest = new UpdateByQueryRequest(matrixGuShiCiCollection);
//构建查询条件
            BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
            boolQueryBuilder.filter(QueryBuilders.termQuery("title",title));
            if (b) {
                boolQueryBuilder.filter(QueryBuilders.termQuery("gradeTypeId",1));
            }
//设置查询条件
            updateByQueryRequest.setQuery(boolQueryBuilder);
//设置修改脚本，ctx为内置上下文，固定写法，_source为ES里的元字段，topRank为自定义的数据字段
            updateByQueryRequest.setScript(new Script("ctx._source.topRank = "+top));
//调用修改接口
            client.updateByQuery(updateByQueryRequest, RequestOptions.DEFAULT);
```

## 7、查询只返回指定字段

```java
SearchRequest searchRequest = new SearchRequest(matrixGuShiCiCollection);
        SearchSourceBuilder searchSourceBuilder = buildSearchQuery(guShiCiListParam);
//fetchSource第一个参数表示想返回的字段，第二个参数表示不想返回的字段
        searchSourceBuilder.fetchSource(new String[] {"title","body},null);
        searchRequest.source(searchSourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest,RequestOptions.DEFAULT);
```


# 2、调用MongoDB接口
> 使用MongoTemplate


```xml
<dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-data-mongodb</artifactId>
     <version>2.1.4.RELEASE</version>
</dependency>
```


```yaml
spring:
    data:
        mongodb:
            uri: mongodb://root:1234@localhost:3717/matrix?replicaSet=mgset-33473615
```


```java
//声明mongodb集合名称
@Document("gushici")
public class GuShiCi {
//声明主键
    @Id
    private String id;
    /**
     * 文学体裁，1：诗歌，2：词
     */
    private String type;
    /**
     * 学段
     */
    //声明对应的mongodb字段
    @Field("grade_type_id")
    private Integer gradeTypeId;
    /**
     * 创建时间
     */
    @Field("create_time")
    private Date createTime;
    ...
}
@Document("bibeimingju")
public class BiBeiMingJu {
    @Id
    private String id;
    private String content;
    private String title;
    private String author;
    ...
}
```



## 1、插入数据

```java
//保存单条数据，第二个参数为集合名称
mongoTemplate.save(guShiCi,COLLECTION_BI_BEI_MING_JU);
mongoTemplate.insert(biBeiMingJuList,COLLECTION_BI_BEI_MING_JU);
```

## 2、更新数据
            
```java
Criteria criteria = Criteria.where("title").is(title);
            Query query = new Query(criteria);
//          Query query = new Query();
//          query.addCriteria(Criteria.where("title").is(title));
            Update update = new Update().set("topRank",top);
//更新满足条件的第一条数据
            mongoTemplate.updateFirst(query,update,GuShiCi.class);
//更新满足条件的所有数据
            mongoTemplate.updateMulti(query,update,GuShiCi.class);
```

## 3、根据id查询数据

```java
GuShiCi guShiCi = mongoTemplate.findById(guShiCiId, GuShiCi.class);
```

## 4、分页排序条件查询
        
```java
Criteria criteria = new Criteria();
        if (guShiCiListParam.getGradeId() != null) {
            criteria.and("gradeId").is(guShiCiListParam.getGradeId());
        }
        if (guShiCiListParam.getVersionId() != null) {
            criteria.and("versionId").is(guShiCiListParam.getVersionId());
        }
        if (guShiCiListParam.getCopyrightRisk() != null) {
            criteria.and("copyrightRisk").is(guShiCiListParam.getCopyrightRisk());
        }
        if (guShiCiListParam.getOffline() != null) {
            criteria.and("offline").is(guShiCiListParam.getOffline());
        }
        Query query = new Query(criteria);
        if (guShiCiListParam.getPageNo() != null && guShiCiListParam.getPageSize() != null) {
//分页
            query.with(PageRequest.of(guShiCiListParam.getPageNo() - 1, guShiCiListParam.getPageSize()));
        }
        if (guShiCiListParam.getSortRegx() != null) {
            List<Map<String,String>> sortRegx = guShiCiListParam.getSortRegx();
            for (Map<String,String> map : sortRegx) {
                String sortField = map.get("sortField");
                if (StringUtils.isEmpty(sortField)) {
                    continue;
                }
                String sortType = map.get("sortType");
//排序
                query.with(Sort.by("DESC".equals(sortType) ? Sort.Direction.DESC : Sort.Direction.ASC, sortField));
            }
        }
        List<GuShiCi> guShiCiList = mongoTemplate.find(query,GuShiCi.class);
```

## 5、统计数量

```java
Query query = new Query(Criteria.where("gradeTypeId").is(gradeTypeId));
return mongoTemplate.count(query,BiBeiMingJu.class);
```