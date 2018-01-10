---
title: Lucene检索笔记
date: 2017-01-28 11:04:45
categories: 后台
tags:
- 全文检索
- Lucene
---
## 把Document映射为Object类  

```java
public interface Doc2ObjectMapper {
    /**
     * 将多个Document映射成一个对象
     * @param documents
     * @return
     */
    Object mapDocumentsToObject(List<Document> documents);


    /**
     * 将单个Document映射成一个对象
     * @param document
     * @return
     */
    Object mapDocumentToObject(Document document);
}
```

<!-- more -->

## 普通检索   

```java
public class SearchHelper {

    private Analyzer analyzer;
    private String indexDirUrl;
    private Directory directory;
    private IndexReader reader;
    private IndexSearcher indexSearcher;


    public SearchHelper(String indexDirUrl,Analyzer analyzer){
        this.indexDirUrl=indexDirUrl;
        this.analyzer=analyzer;
        try {
            init();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public SearchHelper(String indexDirUrl){
        this(indexDirUrl, new SmartChineseAnalyzer());
    }


    private void init() throws IOException {
        directory=FSDirectory.open(Paths.get(indexDirUrl));
        reader= DirectoryReader.open(directory);
        indexSearcher=new IndexSearcher(reader);
    }

    /**
     * 查询并且返回经过映射后的对象List
     * @param query
     * @param offset
     * @param topN
     * @return
     * @throws IOException
     */
    public List<Object> search(Query query,int offset,int topN,Sort sort,Doc2ObjectMapper doc2ObjectMapper) throws IOException {
        TopDocs topDocs=null;
        ScoreDoc after=null;
        if(offset>0){
            TopDocs docsBefore=indexSearcher.search(query,offset,sort);
            ScoreDoc[] scoreDocs=docsBefore.scoreDocs;
            if(scoreDocs.length>0)
                after=scoreDocs[scoreDocs.length-1];
        }
        topDocs=indexSearcher.searchAfter(after,query,topN,sort);
        return creatObjectList(topDocs.scoreDocs,doc2ObjectMapper);
    }

    /**
     * 没有Sort的search
     * @param query
     * @param offset
     * @param topN
     * @return
     * @throws IOException
     */
    public List<Object> search(Query query,int offset,int topN,Doc2ObjectMapper doc2ObjectMapper) throws IOException {
        TopDocs topDocs=null;
        ScoreDoc after=null;
        if(offset>0){
            TopDocs docsBefore=indexSearcher.search(query,offset);
            ScoreDoc[] scoreDocs=docsBefore.scoreDocs;
            if(scoreDocs.length>0)
                after=scoreDocs[scoreDocs.length-1];
        }
        topDocs=indexSearcher.searchAfter(after,query,topN);

        return creatObjectList(topDocs.scoreDocs,doc2ObjectMapper);
    }



    /**
     * 获取查询到的总数量
     * @param query
     * @return
     * @throws IOException
     */
    public int getSum(Query query) throws IOException {
        return indexSearcher.search(query,1).totalHits;
    }

    private List<Object> creatObjectList(ScoreDoc[] scoreDocs,Doc2ObjectMapper doc2ObjectMapper) throws IOException {
        List<Object> result=new LinkedList<Object>();
        for(ScoreDoc scoreDoc:scoreDocs){
            result.add(doc2ObjectMapper.mapDocumentToObject(indexSearcher.doc(scoreDoc.doc)));
        }
        return result;
    }

}
```

## 基于Group by的检索  

```java
/**
 * 使用Group by进行搜索
 * Created by yuan on 1/8/17.
 */
public class GroupSearcherHelper {

    private Analyzer analyzer;
    private String indexDirUrl;
    private Directory directory;
    private IndexReader reader;
    private IndexSearcher indexSearcher;
    private double maxCacheRAMMB;
    private boolean isCacheScores=true;
    private boolean ifFillFields=true;

    public static final double DEFAULT_MAX_CACHE_RAM_MB=4.0;

    public GroupSearcherHelper(String indexDirUrl,Analyzer analyzer,double maxCacheRAMMB){
        this.indexDirUrl=indexDirUrl;
        this.analyzer=analyzer;
        this.maxCacheRAMMB=maxCacheRAMMB;
        try {
            init();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public GroupSearcherHelper(String indexDirUrl,Analyzer analyzer){
        this(indexDirUrl,analyzer,DEFAULT_MAX_CACHE_RAM_MB);
    }

    public GroupSearcherHelper(String indexDirUrl){
        this(indexDirUrl,new SmartChineseAnalyzer());
    }



    private void init() throws IOException {
        directory= FSDirectory.open(Paths.get(indexDirUrl));
        reader= DirectoryReader.open(directory);
        indexSearcher=new IndexSearcher(reader);
    }


    /**
     * 搜索返回文档分组
     * @param query
     * @param groupFieldName
     * @param groupSort
     * @param withinGroupSort
     * @param groupOffset
     * @param topNGroups
     * @return
     * @throws IOException
     */
    public List<List<Document>> searchDocument(Query query, String groupFieldName, Sort groupSort, Sort withinGroupSort, int groupOffset, int topNGroups) throws IOException {
        List<List<Document>> result=new LinkedList<List<Document>>();
        TopGroups<BytesRef> topGroupsResult=searchHelp(query,groupFieldName,groupSort,withinGroupSort,groupOffset,topNGroups);
        if(topGroupsResult==null)
            return result;
        GroupDocs<BytesRef>[] groupDocses=topGroupsResult.groups;
        for(GroupDocs<BytesRef> groupDocs:groupDocses){
            List<Document> subList=new LinkedList<Document>();
            for(ScoreDoc scoreDoc:groupDocs.scoreDocs){
                Document document=indexSearcher.doc(scoreDoc.doc);
                subList.add(document);
            }
            result.add(subList);
        }

        return result;
    }

    /**
     * 使用默认Sort的searchDocument
     * @param query
     * @param groupFieldName
     * @param groupOffset
     * @param topNGroups
     * @return
     * @throws IOException
     */
    public List<List<Document>> searchDocument(Query query, String groupFieldName, int groupOffset, int topNGroups) throws IOException {
        return searchDocument(query,groupFieldName,Sort.INDEXORDER,Sort.INDEXORDER,groupOffset,topNGroups);
    }

    /**
     * 分组搜索并且将每一组Document映射成一个对象并且返回所有对象组成的List
     * @param query
     * @param groupFieldName
     * @param groupSort
     * @param withinGroupSort
     * @param groupOffset
     * @param topNGroups
     * @param mapper
     * @return
     * @throws IOException
     */
    public List<Object> search(Query query, String groupFieldName, Sort groupSort, Sort withinGroupSort, int groupOffset, int topNGroups, Doc2ObjectMapper mapper) throws IOException {
        List<Object> result=new LinkedList<Object>();
        List<List<Document>> documentsList=searchDocument(query,groupFieldName,groupSort,withinGroupSort,groupOffset,topNGroups);
        if(documentsList.size()==0)
            return result;
        Object o=null;
        for(List<Document> documents:documentsList){
            o=mapper.mapDocumentsToObject(documents);
            result.add(o);
        }
        return result;
    }

    /**
     * 使用默认Sort的search
     * @param query
     * @param groupFieldName
     * @param groupOffset
     * @param topNGroups
     * @param mapper
     * @return
     * @throws IOException
     */
    public List<Object> search(Query query, String groupFieldName,  int groupOffset, int topNGroups, Doc2ObjectMapper mapper) throws IOException {
        return search(query,groupFieldName,Sort.INDEXORDER,Sort.INDEXORDER,groupOffset,topNGroups,mapper);
    }



    TopGroups<BytesRef> searchHelp(Query query, String groupFieldName, Sort groupSort, Sort withinGroupSort, int groupOffset, int topNGroups) throws IOException {
        TermFirstPassGroupingCollector c1=new TermFirstPassGroupingCollector(groupFieldName,groupSort,groupOffset+topNGroups);
        /**
         * 将TermFirstPassGroupingCollector包装成CachingCollector，为第一次查询加缓存，避免重复评分
         *  CachingCollector就是用来为结果收集器添加缓存功能的
         */
        CachingCollector cachingCollector=CachingCollector.create(c1,isCacheScores,maxCacheRAMMB);
        //开始第一次分组统计
        indexSearcher.search(query,cachingCollector);

        /**第一次查询返回的结果集TopGroups中只有分组域值以及每组总的评分，至于每个分组里有几条，分别哪些索引文档，则需要进行第二次查询获取*/
        Collection<SearchGroup<BytesRef>> topGroups=c1.getTopGroups(groupOffset,ifFillFields);
        if(topGroups==null){
            return null;
        }

        Collector secondPassCollector=null;
        // 是否获取每个分组内部每个索引的评分
        boolean ifGetScores=true;
        // 是否计算最大评分
        boolean ifGetMaxScores=true;
        int maxDocsPerGroup=10;
        // 如果需要对Lucene的score进行修正，则需要重载TermSecondPassGroupingCollector
        TermSecondPassGroupingCollector c2=new TermSecondPassGroupingCollector(groupFieldName,topGroups,
                groupSort,withinGroupSort,
                maxDocsPerGroup,ifGetScores,ifGetMaxScores,ifFillFields);

        secondPassCollector=c2;

        /**如果第一次查询已经加了缓存，则直接从缓存中取*/
        if(cachingCollector.isCached()){
            //第二次查询直接从缓存中取
            cachingCollector.replay(secondPassCollector);
        }else{
            // 开始第二次分组查询
            indexSearcher.search(query,secondPassCollector);
        }


        TopGroups<BytesRef> topGroupsResult=c2.getTopGroups(0);

        return topGroupsResult;
    }

    /**
     * 查询符合条件的分组总数量
     * @param query
     * @param groupFieldName
     * @return
     * @throws Exception
     */
    public int getGroupSum(Query query,String groupFieldName) throws Exception{
        TermFirstPassGroupingCollector c1=new TermFirstPassGroupingCollector(groupFieldName,Sort.INDEXORDER,1);
        TermAllGroupsCollector termAllGroupsCollector=new TermAllGroupsCollector(groupFieldName);
        Collector collector=  MultiCollector.wrap(c1,termAllGroupsCollector);
        indexSearcher.search(query,collector);

        return termAllGroupsCollector.getGroupCount();
    }

```
