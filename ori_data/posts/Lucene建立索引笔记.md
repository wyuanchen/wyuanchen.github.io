---
title: Lucene建立索引笔记
date: 2017-01-28 11:03:52
categories: 后台
tags:
- 全文检索
- Lucene
---
## 前言
之前在项目中需要用到全文检索，根据搜索关键字来返回满足条件的商品，同时需要满足一定的商品类别和商城代码，刚好学下lucene来初步简单实现下这个需求。 

<!-- more -->

## 连接Mysql数据库和导出数据
首先我自己先封装一个很简单的数据库操作工具类
```java
public class DBUtil {

    private String url;
    private String user;
    private String password;
    private Connection conn;

    public DBUtil(String url,String user,String password) throws ClassNotFoundException {
        this.url=url;
        this.user=user;
        this.password=password;
        Class.forName("com.mysql.jdbc.Driver");
    }

    /**
     * 返回一个数据库连接
     * @return
     */
    public Connection getConnection(){
        try {
            if(conn==null)
                conn= DriverManager.getConnection(url,user,password);
            return conn;
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     *关闭数据库连接
     */
    public void closeConnection(){
        try {
            if (conn != null) {
                conn.close();
            }
        }catch (SQLException e){
            e.printStackTrace();
        } finally {
          conn=null;
        }
    }
}
```
## 创建Docement
首先我们得先建立一个Docement，然后往Document加入Field,最后将Document写入索引
下面我把创建索引抽象成一个接口
```java
public interface DocumentCreator {
    Document createDocument(ResultSet resultSet) throws Exception;
}
```
然后是一个具体的实现
```java
public class SearchGoodDocumentCreator implements DocumentCreator {
    @Override
    public Document createDocument(ResultSet rs) throws Exception {
        Document doc=new Document();
        doc.add(new StoredField("id",rs.getString("id")));
        doc.add(new TextField("title",rs.getString("title"), Field.Store.YES));
        doc.add(new StringField("type",rs.getString("type"), Field.Store.YES));
        doc.add(new StringField("mall",rs.getString("mall"), Field.Store.YES));
        doc.add(new DoubleField("rank_score",rs.getDouble("rank_score"), Field.Store.YES));
        doc.add(new DoubleDocValuesField("rank_score",rs.getDouble("rank_score")));
        doc.add(new StoredField("urls",rs.getString("urls")));
        doc.add(new StoredField("pic_urls",rs.getString("pic_urls")));
        doc.add(new StoredField("comment_num",rs.getString("comment_num")));

        doc.add(new SortedDocValuesField("id",new BytesRef(rs.getString("id"))));

        return doc;
    }
}
```
对于Field的选择有以下几种常用的情况   

- `StringField`: 该Field不会被分词，所以在建立完索引后想要搜索它则必须完全符合才行，因为我在搜索商品的时候`type`字段和`mall`字段都是明确的，所以我使用`StringField`，并且考虑到到时候搜索返回的Document结果也依然能够拿到当时存储的`type`和`mall`，所以我在创建该字段的时候还需声明`Field.Store.YES`。
- `TextField`：它与`StringField`的区别是它会先经过Analyzer进行分词后再建立索引，因为我是通过商品名字的关键字来搜索的，所以在创建`title`的Field时用TextField。如果需要存储原来还未经过分词的原子段以便在搜索得到的结果中能够获取原子段的话，就加多`Field.Store.YES`。
- `DoubleField`，`IntField`，`LongField`等：只索引不存储，在这里。如果需要存储原来还未经过分词的原子段以便在搜索得到的结果中能够获取原子段的话，就加多`Field.Store.YES`。
- `DoubleDocValuesField`：用于对`double`字段排序，在这里我需要根据商品评分来对已经检索得到的商品集合进行排序。并且我还需要在搜到的商品中仍然能保存着`rankScore`字段，所以我还对`rankScore`用了`DoubleField`。
- `StoredField`： 只存储不建立索引。因为我不需要通过搜索`urls`,`pic_urls`和`comment_num`来获得商品，所以只需要存储就行了。


## 通过IndexWriter写入Document
主要的步骤：

1. 创建`Analyzer`
2. 创建`IndexWriterConfig`
3. 创建`Directory`
4. 创建`IndexWriter` 
5. 用`IndexWriter`把创建好的`Document`依次写入`Directory`

因为项目中商品的名字是中文，所以在建立索引和检索中都需要中文分词，所以使用了`IKAnalyzer`或者`SmartChineseAnalyzer`等支持中文分词的`Analyzer`。

```java
public class IndexBuilder {
    private String url;
    private String user;
    private String password;
    private String sql;
    private DBUtil dbUtil;
    private Analyzer analyzer=new SmartChineseAnalyzer();
    private String indexDirUrl ;
    private DocumentCreator documentCreator;

    public void setAnalyzer(Analyzer analyzer) {
        this.analyzer = analyzer;
    }

    public void setDbUtil(DBUtil dbUtil) {
        this.dbUtil = dbUtil;
    }

    public void setDocumentCreator(DocumentCreator documentCreator) {
        this.documentCreator = documentCreator;
    }

    public void setIndexDirUrl(String indexDirUrl) {
        this.indexDirUrl = indexDirUrl;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public void setSql(String sql) {
        this.sql = sql;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public void setUser(String user) {
        this.user = user;
    }

    public IndexBuilder(){}

    /**
     * 从数据库查询获取结果集
     * @return
     * @throws ClassNotFoundException
     * @throws SQLException
     */
    public ResultSet getResultSet() throws ClassNotFoundException, SQLException {
        dbUtil=new DBUtil(url,user,password);
        Connection conn=dbUtil.getConnection();
        Statement statement=conn.createStatement();
        return statement.executeQuery(sql);
    }

    /**
     * 结束工作
     */
    public void complete(){
        if(dbUtil!=null)
            dbUtil.closeConnection();
        dbUtil=null;
    }

    /**
     * 启动建立索引
     * @throws ClassNotFoundException
     * @throws SQLException
     * @throws IOException
     */
    public void start() throws Exception {
        System.out.println("luceneIndexBuilder start!");
        long startTime=System.currentTimeMillis();

        ResultSet rs=getResultSet();

        IndexWriterConfig indexWriterConfig = new IndexWriterConfig(analyzer);
        indexWriterConfig.setOpenMode(IndexWriterConfig.OpenMode.CREATE);
        Directory directory= FSDirectory.open(Paths.get(indexDirUrl));
        IndexWriter indexWriter=new IndexWriter(directory,indexWriterConfig);

        Document doc=null;
        while(rs.next()){
            doc=documentCreator.createDocument(rs);
            //加入Document
            indexWriter.addDocument(doc);
        }
        //记得调用close()才能确保索引真正写入
        indexWriter.close();
        complete();
        long endTime=System.currentTimeMillis();
        System.out.println("luceneIndexBuilder complete");

    }
}
```

## 测试
```java
    @Test
    public void testStart() throws Exception {
        System.out.println("---testStrat()----");
        String url="jdbc:mysql://123.12.123.12:3306/example_db?useUnicode=true&characterEncoding=utf8";
        String user="root";
        String password="123456";
        String sql="select id,title,type, mall,goods_rank.rank_score from goods;";
        String indexDirUrl = new String("./indexDir/");
        
        //SmartChineseAnalyzer和IKAnalyzer都支持中文分词
        Analyzer analyzer=new SmartChineseAnalyzer();
//        Analyzer analyzer=new IKAnalyzer();
        DocumentCreator documentCreator=new SearchGoodDocumentCreator();

        //配置indexBuilder
        IndexBuilder indexBuilder=new IndexBuilder();
        indexBuilder.setUrl(url);
        indexBuilder.setUser(user);
        indexBuilder.setPassword(password);
        indexBuilder.setSql(sql);
        indexBuilder.setAnalyzer(analyzer);
        indexBuilder.setIndexDirUrl(indexDirUrl);
        indexBuilder.setDocumentCreator(documentCreator);

        indexBuilder.start();
    }
```
