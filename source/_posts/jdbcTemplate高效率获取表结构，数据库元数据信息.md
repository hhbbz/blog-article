---
title: jdbcTemplate高效率获取表结构，数据库元数据信息
date: 2019-03-31 11:44:12
categories: 
- 后端
tags:
- Spring
- Java
---

# 场景

在项目中需要通过表名来获取数据库中元数据相关信息，比如表名，字段名，长度等
使用spring自带的jdbcTemplate 可以通过SqlRowSetMetaData 可以获取到部分元数据，但是不能获取备注信息（comment中的内容）

# 最简单的解决方案

使用jdbcTemplate的queryForRowSet方法。

该方法很简单

```java
        SqlRowSet sqlRowSet ;
        try {
            sqlRowSet = this.jdbcTemplate.queryForRowSet(sql.toString());
            //获取字段列
            SqlRowSetMetaData rowSetMetaData = sqlRowSet.getMetaData();
            //获取列总数
            int columnCount = rowSetMetaData.getColumnCount();
            for (int i = 1; i <= columnCount; i++) {
                rowSetMetaData.getColumnName(i);
                rowSetMetaData.getColumnType(i);

            }
        } catch (Exception e) {
            e.printStackTrace();
        }
```

>但是此方案存在严重的效率问题，10W的结果集数据，37个字段，24核cpu，基本没30s都返回不过来，占用内存也极高。

进行debug跟进去看，看到jdbcTemplate调用jdbc返回ResultSet只用了10秒左右，之后就一直耗在extractData方法里。该方法是用默认的RowMapper，先取得MetaData然后根据这个去生成Map。

# 对比方法

1. 使用纯jdbc对比，手工码代码，直接调用Map的put方法逐个生成Map并填充数据。同样的sql，耗时8秒左右，其中根据ResultSet生成List<Map<String, Object>>的过程不超过10s。

2. 改用jdbcTemplate的public <T> List<T> query(String sql, RowMapper<T> rowMapper)方法，T填入Map<String, Object>，这样就跟queryForList方法返回值一样了，然后自己实现RowMapper，直接用Map的put方法填充数据。实验结果跟直接用纯jdbc效率相同。

3. 使用纯jdbc，自己写一个实体类，把resultSet里的数据循环填入对象放到List里。耗时也差不多8s，不过还是会省一点内存的。但是性能提升有限。

# 最优解决方案

只需要通过jdbcTemplate获取jdbc Connection即可获取全部信息。

代码示例如下：

```java
        StringBuilder sql = new StringBuilder();
        sql.append("SELECT *");
        sql.append(" FROM ").append(tableCode);
        ResultSet tabs;
        try {
            DatabaseMetaData dbMetaData = this.jdbcTemplate.getDataSource().getConnection().getMetaData();
            String[] types = {"TABLE"};
            tabs = dbMetaData.getTables(null, null, tableCode, types);
            while (tabs.next()) {
                String tableName = tabs.getString("TABLE_NAME");
                //表对应的schema
                String tableSchema = tabs.getString("TABLE_SCHEM");
                ResultSet resultSet = dbMetaData.getColumns(null, schema, tableName, null);
                while (resultSet.next()) {
                    String name = resultSet.getString("COLUMN_NAME");
                    String type = resultSet.getString("TYPE_NAME");
                    String colRemarks = resultSet.getString("REMARKS");
                    int size = resultSet.getInt("COLUMN_SIZE");

                    //业务逻辑

                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
```

>经过测试，同样的10w+数据，37个字段，加上我写的业务逻辑也只需要700ms的响应时间，效率可谓是大大提升。

我觉得该方法效率得到提升的原因在于，它没有对全表进行扫描，不关乎结果集的多少，只在乎表结构，所以可以极大的提高响应效率。
