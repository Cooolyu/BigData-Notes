# Hbase Java API 的基本使用

<nav>
<a href="#一简述">一、简述</a><br/>
<a href="#二Java-API-1x-基本使用">二、Java API 1.x 基本使用</a><br/>
<a href="#三Java-API-2x-基本使用">三、Java API 2.x 基本使用</a><br/>
<a href="#四正确连接Hbase">四、正确连接Hbase</a><br/>
</nav>



## 一、简述

截至到目前（2019年4月），HBase 主要有1.x 和 2.x 两个主要的版本，两个版本的Java API的接口和方法有所不同的，1.x 中某些方法在2.x中被标识为`@deprecated `过时，所以下面关于API的样例，我会分别给出1.x和2.x两个版本。完整的代码见本仓库：

>+ [Java API 1.x Examples](https://github.com/heibaiying/BigData-Notes/tree/master/code/Hbase/hbase-java-api-1.x)
>
>+ [Java API 2.x Examples](https://github.com/heibaiying/BigData-Notes/tree/master/code/Hbase/hbase-java-api-2.x)

同时在实际使用中，客户端的版本必须与服务端保持一致，如果用2.x版本的客户端代码去连接1.x版本的服务端，是会抛出`NoSuchColumnFamilyException`等异常的。

## 二、Java API 1.x 基本使用

#### 2.1 新建Maven工程，导入项目依赖

要使用Java API 操作HBase,仅需要引入`hbase-client`。这里我服务端的HBase版本为`hbase-1.2.0-cdh5.15.2`，对应的`Hbase client` 选取 1.2.0  版本 

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>1.2.0</version>
</dependency>
```

#### 2.2 API 基本使用

这里列举了常用的增删改查操作

```java
public class HBaseUtils {

    private static Connection connection;

    static {
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
        // 如果是集群 则主机名用逗号分隔
        configuration.set("hbase.zookeeper.quorum", "hadoop001");
        try {
            connection = ConnectionFactory.createConnection(configuration);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建HBase表
     *
     * @param tableName      表名
     * @param columnFamilies 列族的数组
     */
    public static boolean createTable(String tableName, List<String> columnFamilies) {
        try {
            HBaseAdmin admin = (HBaseAdmin) connection.getAdmin();
            if (admin.tableExists(tableName)) {
                return false;
            }
            HTableDescriptor tableDescriptor = new HTableDescriptor(TableName.valueOf(tableName));
            columnFamilies.forEach(columnFamily -> {
                HColumnDescriptor columnDescriptor = new HColumnDescriptor(columnFamily);
                columnDescriptor.setMaxVersions(1);
                tableDescriptor.addFamily(columnDescriptor);
            });
            admin.createTable(tableDescriptor);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 删除hBase表
     *
     * @param tableName 表名
     */
    public static boolean deleteTable(String tableName) {
        try {
            HBaseAdmin admin = (HBaseAdmin) connection.getAdmin();
            // 删除表前需要先禁用表
            admin.disableTable(tableName);
            admin.deleteTable(tableName);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 插入数据
     *
     * @param tableName        表名
     * @param rowKey           唯一标识
     * @param columnFamilyName 列族名
     * @param qualifier        列标识
     * @param value            数据
     */
    public static boolean putRow(String tableName, String rowKey, String columnFamilyName, String qualifier,
                                 String value) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Put put = new Put(Bytes.toBytes(rowKey));
            put.addColumn(Bytes.toBytes(columnFamilyName), Bytes.toBytes(qualifier), Bytes.toBytes(value));
            table.put(put);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 插入数据
     *
     * @param tableName        表名
     * @param rowKey           唯一标识
     * @param columnFamilyName 列族名
     * @param pairList         列标识和值的集合
     */
    public static boolean putRow(String tableName, String rowKey, String columnFamilyName, List<Pair<String, String>> pairList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Put put = new Put(Bytes.toBytes(rowKey));
            pairList.forEach(pair -> put.addColumn(Bytes.toBytes(columnFamilyName), Bytes.toBytes(pair.getKey()), Bytes.toBytes(pair.getValue())));
            table.put(put);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 根据rowKey获取指定行的数据
     *
     * @param tableName 表名
     * @param rowKey    唯一标识
     */
    public static Result getRow(String tableName, String rowKey) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Get get = new Get(Bytes.toBytes(rowKey));
            return table.get(get);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 获取指定行指定列(cell)的最新版本的数据
     *
     * @param tableName    表名
     * @param rowKey       唯一标识
     * @param columnFamily 列族
     * @param qualifier    列标识
     */
    public static String getCell(String tableName, String rowKey, String columnFamily, String qualifier) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Get get = new Get(Bytes.toBytes(rowKey));
            if (!get.isCheckExistenceOnly()) {
                get.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(qualifier));
                Result result = table.get(get);
                byte[] resultValue = result.getValue(Bytes.toBytes(columnFamily), Bytes.toBytes(qualifier));
                return Bytes.toString(resultValue);
            } else {
                return null;
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 检索全表
     *
     * @param tableName 表名
     */
    public static ResultScanner getScanner(String tableName) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 检索表中指定数据
     *
     * @param tableName  表名
     * @param filterList 过滤器
     */

    public static ResultScanner getScanner(String tableName, FilterList filterList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            scan.setFilter(filterList);
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 检索表中指定数据
     *
     * @param tableName   表名
     * @param startRowKey 起始RowKey
     * @param endRowKey   终止RowKey
     * @param filterList  过滤器
     */

    public static ResultScanner getScanner(String tableName, String startRowKey, String endRowKey,
                                           FilterList filterList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            scan.setStartRow(Bytes.toBytes(startRowKey));
            scan.setStopRow(Bytes.toBytes(endRowKey));
            scan.setFilter(filterList);
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 删除指定行记录
     *
     * @param tableName 表名
     * @param rowKey    唯一标识
     */
    public static boolean deleteRow(String tableName, String rowKey) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Delete delete = new Delete(Bytes.toBytes(rowKey));
            table.delete(delete);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 删除指定行的指定列
     *
     * @param tableName  表名
     * @param rowKey     唯一标识
     * @param familyName 列族
     * @param qualifier  列标识
     */
    public static boolean deleteColumn(String tableName, String rowKey, String familyName,
                                          String qualifier) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Delete delete = new Delete(Bytes.toBytes(rowKey));
            delete.addColumn(Bytes.toBytes(familyName), Bytes.toBytes(qualifier));
            table.delete(delete);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

}
```

### 2.3 单元测试

以单元测试的方式对封装的API进行测试

```java
public class HBaseUtilsTest {

    private static final String TABLE_NAME = "class";
    private static final String TEACHER = "teacher";
    private static final String STUDENT = "student";

    @Test
    public void createTable() {
        // 新建表
        List<String> columnFamilies = Arrays.asList(TEACHER, STUDENT);
        boolean table = HBaseUtils.createTable(TABLE_NAME, columnFamilies);
        System.out.println("表创建结果:" + table);
    }

    @Test
    public void insertData() {
        List<Pair<String, String>> pairs1 = Arrays.asList(new Pair<>("name", "Tom"),
                new Pair<>("age", "22"),
                new Pair<>("gender", "1"));
        HBaseUtils.putRow(TABLE_NAME, "rowKey1", STUDENT, pairs1);

        List<Pair<String, String>> pairs2 = Arrays.asList(new Pair<>("name", "Jack"),
                new Pair<>("age", "33"),
                new Pair<>("gender", "2"));
        HBaseUtils.putRow(TABLE_NAME, "rowKey2", STUDENT, pairs2);

        List<Pair<String, String>> pairs3 = Arrays.asList(new Pair<>("name", "Mike"),
                new Pair<>("age", "44"),
                new Pair<>("gender", "1"));
        HBaseUtils.putRow(TABLE_NAME, "rowKey3", STUDENT, pairs3);
    }


    @Test
    public void getRow() {
        Result result = HBaseUtils.getRow(TABLE_NAME, "rowKey1");
        if (result != null) {
            System.out.println(Bytes
                    .toString(result.getValue(Bytes.toBytes(STUDENT), Bytes.toBytes("name"))));
        }

    }

    @Test
    public void getCell() {
        String cell = HBaseUtils.getCell(TABLE_NAME, "rowKey2", STUDENT, "age");
        System.out.println("cell age :" + cell);

    }

    @Test
    public void getScanner() {
        ResultScanner scanner = HBaseUtils.getScanner(TABLE_NAME);
        if (scanner != null) {
            scanner.forEach(result -> System.out.println(Bytes.toString(result.getRow()) + "->" + Bytes
                    .toString(result.getValue(Bytes.toBytes(STUDENT), Bytes.toBytes("name")))));
            scanner.close();
        }
    }


    @Test
    public void getScannerWithFilter() {
        FilterList filterList = new FilterList(FilterList.Operator.MUST_PASS_ALL);
        SingleColumnValueFilter nameFilter = new SingleColumnValueFilter(Bytes.toBytes(STUDENT),
                Bytes.toBytes("name"), CompareOperator.EQUAL, Bytes.toBytes("Jack"));
        filterList.addFilter(nameFilter);
        ResultScanner scanner = HBaseUtils.getScanner(TABLE_NAME, filterList);
        if (scanner != null) {
            scanner.forEach(result -> System.out.println(Bytes.toString(result.getRow()) + "->" + Bytes
                    .toString(result.getValue(Bytes.toBytes(STUDENT), Bytes.toBytes("name")))));
            scanner.close();
        }
    }

    @Test
    public void deleteColumn() {
        boolean b = HBaseUtils.deleteColumn(TABLE_NAME, "rowKey2", STUDENT, "age");
        System.out.println("删除结果: " + b);
    }

    @Test
    public void deleteRow() {
        boolean b = HBaseUtils.deleteRow(TABLE_NAME, "rowKey2");
        System.out.println("删除结果: " + b);
    }

    @Test
    public void deleteTable() {
        boolean b = HBaseUtils.deleteTable(TABLE_NAME);
        System.out.println("删除结果: " + b);
    }
}
```



## 三、Java API 2.x 基本使用

#### 3.1 新建Maven工程，导入项目依赖

这里选取的`HBase Client`的版本为最新的`2.1.4`

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>2.1.4</version>
</dependency>
```

#### 3.2 API 的基本使用

2.x 版本相比于1.x 废弃了一部分方法，关于废弃的方法在源码中都会指明新的替代方法，比如，在2.x中创建表时：`HTableDescriptor`和`HColumnDescriptor`等类都标识为废弃，且会在3.0.0版本移除，取而代之的是使用`TableDescriptorBuilder`和`ColumnFamilyDescriptorBuilder`来定义表和列族。在升级版本时，可以用源码中指明的新的替代方法来代替过期的方法。

<div align="center"> <img width="700px"  src="https://github.com/heibaiying/BigData-Notes/blob/master/pictures/deprecated.png"/> </div>



以下为HBase  2.x 版本Java API使用的完整示例：

```java
public class HBaseUtils {

    private static Connection connection;

    static {
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
        // 如果是集群 则主机名用逗号分隔
        configuration.set("hbase.zookeeper.quorum", "hadoop001");
        try {
            connection = ConnectionFactory.createConnection(configuration);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建HBase表
     *
     * @param tableName      表名
     * @param columnFamilies 列族的数组
     */
    public static boolean createTable(String tableName, List<String> columnFamilies) {
        try {
            HBaseAdmin admin = (HBaseAdmin) connection.getAdmin();
            if (admin.tableExists(TableName.valueOf(tableName))) {
                return false;
            }
            TableDescriptorBuilder tableDescriptor = TableDescriptorBuilder.newBuilder(TableName.valueOf(tableName));
            columnFamilies.forEach(columnFamily -> {
                ColumnFamilyDescriptorBuilder cfDescriptorBuilder = ColumnFamilyDescriptorBuilder.newBuilder(Bytes.toBytes(columnFamily));
                cfDescriptorBuilder.setMaxVersions(1);
                ColumnFamilyDescriptor familyDescriptor = cfDescriptorBuilder.build();
                tableDescriptor.setColumnFamily(familyDescriptor);
            });
            admin.createTable(tableDescriptor.build());
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 删除hBase表
     *
     * @param tableName 表名
     */
    public static boolean deleteTable(String tableName) {
        try {
            HBaseAdmin admin = (HBaseAdmin) connection.getAdmin();
            // 删除表前需要先禁用表
            admin.disableTable(TableName.valueOf(tableName));
            admin.deleteTable(TableName.valueOf(tableName));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 插入数据
     *
     * @param tableName        表名
     * @param rowKey           唯一标识
     * @param columnFamilyName 列族名
     * @param qualifier        列标识
     * @param value            数据
     */
    public static boolean putRow(String tableName, String rowKey, String columnFamilyName, String qualifier,
                                 String value) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Put put = new Put(Bytes.toBytes(rowKey));
            put.addColumn(Bytes.toBytes(columnFamilyName), Bytes.toBytes(qualifier), Bytes.toBytes(value));
            table.put(put);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 插入数据
     *
     * @param tableName        表名
     * @param rowKey           唯一标识
     * @param columnFamilyName 列族名
     * @param pairList         列标识和值的集合
     */
    public static boolean putRow(String tableName, String rowKey, String columnFamilyName, List<Pair<String, String>> pairList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Put put = new Put(Bytes.toBytes(rowKey));
            pairList.forEach(pair -> put.addColumn(Bytes.toBytes(columnFamilyName), Bytes.toBytes(pair.getKey()), Bytes.toBytes(pair.getValue())));
            table.put(put);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 根据rowKey获取指定行的数据
     *
     * @param tableName 表名
     * @param rowKey    唯一标识
     */
    public static Result getRow(String tableName, String rowKey) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Get get = new Get(Bytes.toBytes(rowKey));
            return table.get(get);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 获取指定行指定列(cell)的最新版本的数据
     *
     * @param tableName    表名
     * @param rowKey       唯一标识
     * @param columnFamily 列族
     * @param qualifier    列标识
     */
    public static String getCell(String tableName, String rowKey, String columnFamily, String qualifier) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Get get = new Get(Bytes.toBytes(rowKey));
            if (!get.isCheckExistenceOnly()) {
                get.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(qualifier));
                Result result = table.get(get);
                byte[] resultValue = result.getValue(Bytes.toBytes(columnFamily), Bytes.toBytes(qualifier));
                return Bytes.toString(resultValue);
            } else {
                return null;
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 检索全表
     *
     * @param tableName 表名
     */
    public static ResultScanner getScanner(String tableName) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 检索表中指定数据
     *
     * @param tableName  表名
     * @param filterList 过滤器
     */

    public static ResultScanner getScanner(String tableName, FilterList filterList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            scan.setFilter(filterList);
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 检索表中指定数据
     *
     * @param tableName   表名
     * @param startRowKey 起始RowKey
     * @param endRowKey   终止RowKey
     * @param filterList  过滤器
     */

    public static ResultScanner getScanner(String tableName, String startRowKey, String endRowKey,
                                           FilterList filterList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            scan.withStartRow(Bytes.toBytes(startRowKey));
            scan.withStopRow(Bytes.toBytes(endRowKey));
            scan.setFilter(filterList);
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 删除指定行记录
     *
     * @param tableName 表名
     * @param rowKey    唯一标识
     */
    public static boolean deleteRow(String tableName, String rowKey) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Delete delete = new Delete(Bytes.toBytes(rowKey));
            table.delete(delete);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }


    /**
     * 删除指定行指定列
     *
     * @param tableName  表名
     * @param rowKey     唯一标识
     * @param familyName 列族
     * @param qualifier  列标识
     */
    public static boolean deleteColumn(String tableName, String rowKey, String familyName,
                                          String qualifier) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Delete delete = new Delete(Bytes.toBytes(rowKey));
            delete.addColumn(Bytes.toBytes(familyName), Bytes.toBytes(qualifier));
            table.delete(delete);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

}
```



## 四、正确连接Hbase

在上面的代码中在类加载时候就初始化了Connection连接，并且之后的方法都是复用这个Connection，这时我们可能会考虑是否可以使用自定义连接池来获取更好的性能表现？实际上这是没有必要的。

首先官方对于`Connection Pooling`做了如下表述：

```properties
Connection Pooling For applications which require high-end multithreaded   
access (e.g., web-servers or  application servers  that may serve many   
application threads in a single JVM), you can pre-create a Connection,   
as shown in the following example:

对于高并发多线程访问的应用程序（例如，在单个JVM中存在的为多个线程服务的Web服务器或应用程序服务器），  
您只需要预先创建一个Connection。例子如下：
```

```java
// Create a connection to the cluster.
Configuration conf = HBaseConfiguration.create();
try (Connection connection = ConnectionFactory.createConnection(conf);
     Table table = connection.getTable(TableName.valueOf(tablename))) {
  // use table as needed, the table returned is lightweight
}
```

之所以能这样使用，这是因为Connection并不是一个简单的socket连接，[接口文档](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Connection.html)中对Connection的表述是：

```properties
A cluster connection encapsulating lower level individual connections to actual servers and a  
connection to zookeeper.  Connections are instantiated through the ConnectionFactory class.  
The lifecycle of the connection is managed by the caller,  who has to close() the connection   
to release the resources. 

Connection是一个集群连接，封装了与多台服务器（Matser/Region Server）的底层连接以及与zookeeper的连接。  
连接通过ConnectionFactory  类实例化。连接的生命周期由调用者管理，调用者必须使用close()关闭连接以释放资源。
```

之所以封装这些连接，是因为HBase客户端需要连接三个不同的服务角色：

+ Zookeeper：主要用于获得meta-region位置，集群Id、master等信息。
+ HBase Master：主要用于执行HBaseAdmin接口的一些操作，例如建表等。
+ HBase RegionServer：用于读、写数据。

<div align="center"> <img width="700px"  src="https://github.com/heibaiying/BigData-Notes/blob/master/pictures/hbase-arc.png"/> </div>

Connection对象和实际的socket连接之间的对应关系如下图：

<div align="center"> <img width="700px"   src="https://github.com/heibaiying/BigData-Notes/blob/master/pictures/hbase-connection.png"/> </div>

> 上面两张图片引用自博客：[连接HBase的正确姿势](https://yq.aliyun.com/articles/581702?spm=a2c4e.11157919.spm-cont-list.1.146c27aeFxoMsN%20%E8%BF%9E%E6%8E%A5HBase%E7%9A%84%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF)

在HBase客户端代码中，真正对应socket连接的是RpcConnection对象。HBase使用PoolMap这种数据结构来存储客户端到HBase服务器之间的连接。PoolMap封装了ConcurrentHashMap>的结构，key是ConnectionId(封装了服务器地址和用户ticket),value是一个RpcConnection对象的资源池。当HBase需要连接一个服务器时，首先会根据ConnectionId找到对应的连接池，然后从连接池中取出一个连接对象。

```java
@InterfaceAudience.Private
public class PoolMap<K, V> implements Map<K, V> {
  private PoolType poolType;

  private int poolMaxSize;

  private Map<K, Pool<V>> pools = new ConcurrentHashMap<>();

  public PoolMap(PoolType poolType) {
    this.poolType = poolType;
  }
  .....
```

HBase中提供了三种资源池的实现，分别是Reusable，RoundRobin和ThreadLocal。具体实现可以通hbase.client.ipc.pool.type配置项指定，默认为Reusable。连接池的大小也可以通过hbase.client.ipc.pool.size配置项指定，默认为1,即每个Server 1个连接。也可以通过修改配置实现：

```java
config.set("hbase.client.ipc.pool.type",...);
config.set("hbase.client.ipc.pool.size",...);
connection = ConnectionFactory.createConnection(config);
```

从以上的表述中，可以看出HBase中Connection类已经实现了对连接的管理功能，所以我们不需要自己在Connection之上再做额外的管理。

另外，Connection是线程安全的，但Table和Admin却不是线程安全的，因此正确的做法是一个进程共用一个Connection对象，而在不同的线程中使用单独的Table和Admin对象。Table和Admin的获取`getTable()`和`getAdmin()`都是轻量级的操作，所以不必担心性能的消耗，同时使用完成后建议显示的调用`close()`方法关闭它们。



## 参考资料

1. [连接HBase的正确姿势](https://yq.aliyun.com/articles/581702?spm=a2c4e.11157919.spm-cont-list.1.146c27aeFxoMsN%20%E8%BF%9E%E6%8E%A5HBase%E7%9A%84%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF)
2. [Apache HBase ™ Reference Guide](http://hbase.apache.org/book.htm)
