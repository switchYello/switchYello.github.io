---
layout:     post
title:      "mybatis-generator多个数据库存在相同的表"
subtitle:   ""
date:       2021-07-17
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- mybatis
- generator
- bug
typora-root-url: ..
---



# mybatis-generator多个数据库存在相同的表

​

## 问题现象		


使用mybatis时难免会使用其提供的mybatis-generator生成默认的xml和实体类，今天我发现如果本地安装一个mysql，里面创建两个不同的数据库，而他们有相同的表，生成的实体类中可能是随机的。
比如我有一个数据库 ‘a’ 里面有一张表 ‘user’，我现在在建一个库‘b’，里面也有一张表‘user’，这样我生成代码时，虽然jdbcUrl配置的是a库，但生成的User实体类可能是b库里面的。



## 源码解析
查看其源码，下面是核心方法
org.mybatis.generator.internal.db.DatabaseIntrospector#getColumns

```java
    private Map<ActualTableName, List<IntrospectedColumn>> getColumns(
            TableConfiguration tc) throws SQLException {
        String localCatalog;
        String localSchema;
        String localTableName;

        boolean delimitIdentifiers = tc.isDelimitIdentifiers()
                || stringContainsSpace(tc.getCatalog())
                || stringContainsSpace(tc.getSchema())
                || stringContainsSpace(tc.getTableName());

        if (delimitIdentifiers) {
            localCatalog = tc.getCatalog();
            localSchema = tc.getSchema();
            localTableName = tc.getTableName();
        } else if (databaseMetaData.storesLowerCaseIdentifiers()) {
            localCatalog = tc.getCatalog() == null ? null : tc.getCatalog()
                    .toLowerCase();
            localSchema = tc.getSchema() == null ? null : tc.getSchema()
                    .toLowerCase();
            localTableName = tc.getTableName().toLowerCase();
        } else if (databaseMetaData.storesUpperCaseIdentifiers()) {
            localCatalog = tc.getCatalog() == null ? null : tc.getCatalog()
                    .toUpperCase();
            localSchema = tc.getSchema() == null ? null : tc.getSchema()
                    .toUpperCase();
            localTableName = tc.getTableName().toUpperCase();
        } else {
            localCatalog = tc.getCatalog();
            localSchema = tc.getSchema();
            localTableName = tc.getTableName();
        }

        if (tc.isWildcardEscapingEnabled()) {
            String escapeString = databaseMetaData.getSearchStringEscape();

            if (localSchema != null) {
                localSchema = escapeName(localSchema, escapeString);
            }

            localTableName = escapeName(localTableName, escapeString);
        }

        Map<ActualTableName, List<IntrospectedColumn>> answer = new HashMap<>();

        if (logger.isDebugEnabled()) {
            String fullTableName = composeFullyQualifiedTableName(localCatalog, localSchema,
                            localTableName, '.');
            logger.debug(getString("Tracing.1", fullTableName)); //$NON-NLS-1$
        }

        ResultSet rs = databaseMetaData.getColumns(localCatalog, localSchema,
                localTableName, "%"); //$NON-NLS-1$

        boolean supportsIsAutoIncrement = false;
        boolean supportsIsGeneratedColumn = false;
        ResultSetMetaData rsmd = rs.getMetaData();
        int colCount = rsmd.getColumnCount();
        for (int i = 1; i <= colCount; i++) {
            if ("IS_AUTOINCREMENT".equals(rsmd.getColumnName(i))) { //$NON-NLS-1$
                supportsIsAutoIncrement = true;
            }
            if ("IS_GENERATEDCOLUMN".equals(rsmd.getColumnName(i))) { //$NON-NLS-1$
                supportsIsGeneratedColumn = true;
            }
        }

        while (rs.next()) {
            IntrospectedColumn introspectedColumn = ObjectFactory
                    .createIntrospectedColumn(context);

            introspectedColumn.setTableAlias(tc.getAlias());
            introspectedColumn.setJdbcType(rs.getInt("DATA_TYPE")); //$NON-NLS-1$
            introspectedColumn.setActualTypeName(rs.getString("TYPE_NAME")); //$NON-NLS-1$
            introspectedColumn.setLength(rs.getInt("COLUMN_SIZE")); //$NON-NLS-1$
            introspectedColumn.setActualColumnName(rs.getString("COLUMN_NAME")); //$NON-NLS-1$
            introspectedColumn
                    .setNullable(rs.getInt("NULLABLE") == DatabaseMetaData.columnNullable); //$NON-NLS-1$
            introspectedColumn.setScale(rs.getInt("DECIMAL_DIGITS")); //$NON-NLS-1$
            introspectedColumn.setRemarks(rs.getString("REMARKS")); //$NON-NLS-1$
            introspectedColumn.setDefaultValue(rs.getString("COLUMN_DEF")); //$NON-NLS-1$

            if (supportsIsAutoIncrement) {
                introspectedColumn.setAutoIncrement(
                        "YES".equals(rs.getString("IS_AUTOINCREMENT"))); //$NON-NLS-1$ //$NON-NLS-2$
            }

            if (supportsIsGeneratedColumn) {
                introspectedColumn.setGeneratedColumn(
                        "YES".equals(rs.getString("IS_GENERATEDCOLUMN"))); //$NON-NLS-1$ //$NON-NLS-2$
            }

            ActualTableName atn = new ActualTableName(
                    rs.getString("TABLE_CAT"), //$NON-NLS-1$
                    rs.getString("TABLE_SCHEM"), //$NON-NLS-1$
                    rs.getString("TABLE_NAME")); //$NON-NLS-1$

            List<IntrospectedColumn> columns = answer.computeIfAbsent(atn, k -> new ArrayList<>());

            columns.add(introspectedColumn);

            if (logger.isDebugEnabled()) {
                logger.debug(getString(
                        "Tracing.2", //$NON-NLS-1$
                        introspectedColumn.getActualColumnName(), Integer
                                .toString(introspectedColumn.getJdbcType()),
                        atn.toString()));
            }
        }

        closeResultSet(rs);

        if (answer.size() > 1
                && !stringContainsSQLWildcard(localSchema)
                && !stringContainsSQLWildcard(localTableName)) {
            // issue a warning if there is more than one table and
            // no wildcards were used
            ActualTableName inputAtn = new ActualTableName(tc.getCatalog(), tc
                    .getSchema(), tc.getTableName());

            StringBuilder sb = new StringBuilder();
            boolean comma = false;
            for (ActualTableName atn : answer.keySet()) {
                if (comma) {
                    sb.append(',');
                } else {
                    comma = true;
                }
                sb.append(atn.toString());
            }

            warnings.add(getString("Warning.25", //$NON-NLS-1$
                    inputAtn.toString(), sb.toString()));
        }

        return answer;
    }

```

上面方法内容有点多，核心逻辑就是

1. 根据CataLog，Schema，TableName三个属性从databaseMetaData里面获取ResultSet。

    ResultSet rs = databaseMetaData.getColumns(localCatalog, localSchema,localTableName, "%");

2. 遍历上面的结果集，按照结果集返回值里面的 TABLE_CAT ， TABLE_SCHEM  ， TABLE_NAME 存入到answer里。

    ActualTableName atn = new ActualTableName(
                    rs.getString("TABLE_CAT"), //$NON-NLS-1$
                    rs.getString("TABLE_SCHEM"), //$NON-NLS-1$
                    rs.getString("TABLE_NAME")); //$NON-NLS-1$

3. 返回 answer对象。

    因为我们我先前没有指定catalog 和 scheme，这里的REsultSet里会获取a库和b库中的user表，即两张表都会返回。


## 解决办法

添加 <property name="nullCatalogMeansCurrent" value="true"/> 属性，不要从metadata中获取其他数据库的表

改动前
```xml
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="${jdbc.url}"
                        userId="${jdbc.username}"
                        password="${jdbc.password}">
        </jdbcConnection>

```


改动后
```xml
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="${jdbc.url}"
                        userId="${jdbc.username}"
                        password="${jdbc.password}">
            <property name="nullCatalogMeansCurrent" value="true"/>
        </jdbcConnection>
```

