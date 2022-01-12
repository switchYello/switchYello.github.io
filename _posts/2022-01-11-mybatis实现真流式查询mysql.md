---
layout:     post
title:      "mybatis实现真流式查询mysql"
subtitle:   ""
date:       2022-01-11
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- mysql
- mybatis
- mysql流式查询技巧
- 技巧
typora-root-url: ..
---


# mybatis实现mysql流式查询的原理

## 背景
如果使用mybatis查询mysql，查询结果集非常大时，可能发生oom，我们可以使用mysql的流式查询，按需逐行查询每条数据，实现低内存占用。
下面分析其原理。

## jdbc驱动如何支持流式查询的

### jdbc驱动的基本原理
       jdbc驱动与mysql之间建立tcp连接，将sql语句发送到mysql服务端后，并从mysql服务端读取数据，并封装成resultSet的形式返回。

### resultSet
我们使用resultSet时，调用resultSet的next方法是否存在下一行数据，resultSet内部包装了ResultsetRows对象，他将请求委托给ResultsetRows对象来处理。resultSet本身只是一个包装类，本质还是被委托到ResultsetRows对象。

### resultRows
ResultsetRows 有两个实现类，ResultsetRowsStreaming和ResultsetRowsStatic。
ResultsetRowsStreaming发送完请求后，不会读取响应数据，他会等到调用next方法时才会从mysql读取。
ResultsetRowsStatic会使用while循环读取所有的数据存储到list中。

        下面是创建resultSet对象的过程过程如下，其中有一个核心逻辑是判断参数中的streamResults字段，决定是生成哪一种类型的ResultsetRows。

```java
//  com.mysql.cj.protocol.a.TextResultsetReader

    @Override
    public Resultset read(int maxRows, boolean streamResults, NativePacketPayload resultPacket, ColumnDefinition metadata,
            ProtocolEntityFactory<Resultset, NativePacketPayload> resultSetFactory) throws IOException {

        Resultset rs = null;
        //try {
        long columnCount = resultPacket.readInteger(IntegerDataType.INT_LENENC);

        if (columnCount > 0) {
            // Build a result set with rows.

            // Read in the column information
            ColumnDefinition cdef = this.protocol.read(ColumnDefinition.class, new ColumnDefinitionFactory(columnCount, metadata));

            // There is no EOF packet after fields when CLIENT_DEPRECATE_EOF is set
            if (!this.protocol.getServerSession().isEOFDeprecated()) {
                this.protocol.skipPacket();
                //this.protocol.readServerStatusForResultSets(this.protocol.readPacket(this.protocol.getReusablePacket()), true);
            }

            ResultsetRows rows = null;

            if (!streamResults) {
                TextRowFactory trf = new TextRowFactory(this.protocol, cdef, resultSetFactory.getResultSetConcurrency(), false);
                ArrayList<ResultsetRow> rowList = new ArrayList<>();

                ResultsetRow row = this.protocol.read(ResultsetRow.class, trf);
                while (row != null) {
                    if ((maxRows == -1) || (rowList.size() < maxRows)) {
                        rowList.add(row);
                    }
                    row = this.protocol.read(ResultsetRow.class, trf);
                }

                rows = new ResultsetRowsStatic(rowList, cdef);

            } else {
                rows = new ResultsetRowsStreaming<>(this.protocol, cdef, false, resultSetFactory);
                this.protocol.setStreamingData(rows);
            }

            /*
             * Build ResultSet from ResultsetRows
             */
            rs = resultSetFactory.createFromProtocolEntity(rows);

        } else {
            // check for file request
            if (columnCount == NativePacketPayload.NULL_LENGTH) {
                String charEncoding = this.protocol.getPropertySet().getStringProperty(PropertyKey.characterEncoding).getValue();
                String fileName = resultPacket.readString(StringSelfDataType.STRING_TERM, this.protocol.doesPlatformDbCharsetMatches() ? charEncoding : null);
                resultPacket = this.protocol.sendFileToServer(fileName);
            }

            /*
             * Build ResultSet with no ResultsetRows
             */

            // read and parse OK packet
            OkPacket ok = this.protocol.readServerStatusForResultSets(resultPacket, false); // oldStatus set in sendCommand()

            rs = resultSetFactory.createFromProtocolEntity(ok);
        }
        return rs;

        //} catch (IOException ioEx) {
        //    throw SQLError.createCommunicationsException(this.protocol.getConnection(), this.protocol.getPacketSentTimeHolder().getLastPacketSentTime(),
        //            this.protocol.getPacketReceivedTimeHolder().getLastPacketReceivedTime(), ioEx, this.protocol.getExceptionInterceptor());
        //}
    }
```



决定是否启用流式的开关是使用下面的方法判断的。

其中有一个重要的判断参数是fetchSize == Integer.MIN_VALUE。
也就是说我们要设置statement的fetchSize为Integer.MIN_VALUE才能启用流式查询。

```java
    protected boolean createStreamingResultSet() {
        return ((this.query.getResultType() == Type.FORWARD_ONLY) && (this.resultSetConcurrency == java.sql.ResultSet.CONCUR_READ_ONLY)
                && (this.query.getResultFetchSize() == Integer.MIN_VALUE));
    }
```







## mybatis如何处理流式查询的

mybatis查询mysql后会获取到resultSet对象，其会对resultSet中的数据进行逐行next()处理。
如果resultHandler为null，则创建一个默认的resultHandler，否则使用我们提供的resultHandler。
默认的resultHandler源码如下，他会将结果存储到list中作为结果返回，如果使用我们提供的resultHandler，则返回的是空集合。

```java
// org.apache.ibatis.executor.resultset.DefaultResultSetHandler#handleResultSet

  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }
```



可以看到默认的resultHandler是将结果聚集到list中。

```java
public class DefaultResultHandler implements ResultHandler<Object> {

  private final List<Object> list;

  public DefaultResultHandler() {
    list = new ArrayList<>();
  }

  @SuppressWarnings("unchecked")
  public DefaultResultHandler(ObjectFactory objectFactory) {
    list = objectFactory.create(List.class);
  }

  @Override
  public void handleResult(ResultContext<?> context) {
    list.add(context.getResultObject());
  }

  public List<Object> getResultList() {
    return list;
  }

}
```



处理resultSet的逻辑就是调用resultSet的next方法获取下一条，将每一条都解析成对象后回调resultHandler。

```java
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }	
```



## mybatis实现流式读取的正确的写法

1. mapper接口这样写，提供第二个参数，即resultHandler,因为提供resultHandler后，mybatis就不会有返回值了，这里直接返回void

```java
interface CustomSettleInChannelRecordMapper {

    void query(@Param("request") SettleRequest request, ResultHandler<SettleRecord> handler);

}
```



2. xml文件应该这样写，一定要写 fetchSize="-2147483648"，不然到达jdbc那里返回的并不是流式的结果集。

```xml
    <select id="query" 
            fetchSize="-2147483648"
            resultMap="bastResultMap">
        select
        id,currency,pay_amount,status,payment_status
        from settle_record
        <where>
            <if test="request.reconCompleteTimeEnd != null">
                and txn_complete_time &lt;= #{request.reconCompleteTimeEnd}
            </if>
            <if test="request.bizType != null">
                and biz_type = #{request.bizType}
            </if>
        </where>
    </select>

```



## 总结
mybatis本身支持在mapper里面提供resultHandler用来流式处理每个结果。其实它内部本身就是创建一个默认的resultHandler将结果汇集后返回给我们的。

mybatis本身属于框架上层服务，他只负责从jdbc出取得resultSet，将resultSet每一行读取出来包装成流式的形式。
而resultSet本身是不是流式的，mybatis无法决定。
resultSet是真流式还是伪流式，需要判断一个重要的参数fetchSize，
要想实现真流式一定要将fetchSize设置为Integer.MIN_VALUE 底层才是真流式，否则只是mybatis包装的伪流式而已。


验证的话可以debug查看resultSet内部的resultRows是什么类型的，如果是ResultsetRowsStreaming则是真流式，如果是ResultsetRowsStatic则是假流式。
