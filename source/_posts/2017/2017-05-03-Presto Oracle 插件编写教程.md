---
title: Presto Oracle 插件编写教程
categories:
  - bigdata
tags:
  - presto
abbrlink: a7895520
date: 2017-05-03 23:29:53
updated: 2020-01-04 13:08:41
---

Presto 是一个分布式的 SQL 查询引擎，非常适合用于 OLAP 场景。官方也许因为版权原因没有提供 oracle 的插件，oracle 在实际场景中还是使用的非常多的，有必要介绍些插件开发的流程。如果读者只是部署，不做开发，可以 clone 我托管在 GitHub 的[Presto](https://github.com/itren/presto-with-oracle) 来进行编译、部署。

<!--more-->

## 搭建开发环境

关于如何搭建开发环境，presto 的 github 首页已经给出教程，这里不再赘述。但是要注意**presto 在 windows 平台下会编译失败**，而且对源码开发之前必须要先编译 presto。这里推荐使用 IntelliJIDEA 作为开发的 IDE， 如果你已经将 presto 导入到 IDE 中，并且成功运行“PrestoServer”，那么你已经成功一半了，其实在 github 上面有人已经托管了[presto-oracle](https://github.com/marcelopaesrech/presto-oracle)这个插件，但是这个插件只能满足简单的查询，无法通过 presto 向 oracle 中插入数据。而且它这个不是集成到 presto 的源码中的，无法对插件进行调试。

## 新建 module

官方已经编写了 MySQL 插件，我们可以按照这个模板来开发。我们在 Presto 的根目录下新建 module，该 module 的 pom 信息如下：

```
    <?xml version="1.0"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>

        <parent>
            <groupId>com.facebook.presto</groupId>
            <artifactId>presto-root</artifactId>
            <version>0.157.2-SNAPSHOT</version>
        </parent>

        <artifactId>presto-oracle</artifactId>
        <description>Presto - Oracle Connector</description>
        <packaging>presto-plugin</packaging>

        <properties>
            <!--check license-->
            <air.main.basedir>${project.parent.basedir}</air.main.basedir>
        </properties>

        <dependencies>
            <dependency>
                <groupId>com.facebook.presto</groupId>
                <artifactId>presto-base-jdbc</artifactId>
            </dependency>

            <dependency>
                <groupId>io.airlift</groupId>
                <artifactId>configuration</artifactId>
            </dependency>

            <dependency>
                <groupId>com.google.guava</groupId>
                <artifactId>guava</artifactId>
            </dependency>

            <dependency>
                <groupId>com.google.inject</groupId>
                <artifactId>guice</artifactId>
            </dependency>

            <dependency>
                <groupId>javax.validation</groupId>
                <artifactId>validation-api</artifactId>
            </dependency>

            <dependency>
                <groupId>com.oracle</groupId>
                <artifactId>ojdbc6</artifactId>
            </dependency>

            <dependency>
                <groupId>javax.inject</groupId>
                <artifactId>javax.inject</artifactId>
            </dependency>

            <!-- Presto SPI -->
            <dependency>
                <groupId>com.facebook.presto</groupId>
                <artifactId>presto-spi</artifactId>
                <scope>provided</scope>
            </dependency>

            <dependency>
                <groupId>io.airlift</groupId>
                <artifactId>slice</artifactId>
                <scope>provided</scope>
            </dependency>

            <dependency>
                <groupId>io.airlift</groupId>
                <artifactId>units</artifactId>
                <scope>provided</scope>
            </dependency>

            <dependency>
                <groupId>com.fasterxml.jackson.core</groupId>
                <artifactId>jackson-annotations</artifactId>
                <scope>provided</scope>
            </dependency>

            <!-- for testing -->
            <dependency>
                <groupId>org.testng</groupId>
                <artifactId>testng</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>io.airlift</groupId>
                <artifactId>testing</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>io.airlift</groupId>
                <artifactId>json</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>com.facebook.presto</groupId>
                <artifactId>presto-main</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>com.facebook.presto</groupId>
                <artifactId>presto-tpch</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>io.airlift.tpch</groupId>
                <artifactId>tpch</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>com.facebook.presto</groupId>
                <artifactId>presto-tests</artifactId>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>io.airlift</groupId>
                <artifactId>testing-mysql-server</artifactId>
                <scope>test</scope>
            </dependency>
        </dependencies>

        <build>

            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-dependency-plugin</artifactId>
                    <version>2.10</version>
                    <configuration>
                        <!--disable check dependency-->
                        <skip>true</skip>
                        <failOnWarning>${air.check.fail-dependency}</failOnWarning>
                        <ignoreNonCompile>true</ignoreNonCompile>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-checkstyle-plugin</artifactId>
                    <executions>
                        <execution>
                            <phase>validate</phase>
                            <goals>
                                <goal>check</goal>
                            </goals>
                            <configuration>
                                <!--disable check code style-->
                                <skip>true</skip>
                            </configuration>
                        </execution>
                    </executions>

                </plugin>

            </plugins>
        </build>

    </project>
```

这里需要说明下的是 Maven 的公共库查找不到 Oracle JDBC 的依赖，所以需要用户自行下载 jar 包并安装到本地 Maven 库中。另外 Presto 有很严格的代码规范以及依赖检查，如果代码或者依赖不通过检查是无法编译成功的。而且公共 Maven 库中无法找到 Oracle JDBC 的依赖，所以依赖检查肯定不能通过。所以我在 pom 文件中禁用了代码规范检查插件，还有依赖检查插件。

### 1. 禁用依赖检查

```
        <plugin>
                      <groupId>org.apache.maven.plugins</groupId>
                      <artifactId>maven-dependency-plugin</artifactId>
                      <version>2.10</version>
                      <configuration>
                          <!--disable check dependency-->
                          <skip>true</skip>
                          <failOnWarning>${air.check.fail-dependency}</failOnWarning>
                          <ignoreNonCompile>true</ignoreNonCompile>
                      </configuration>
                  </plugin>
```

### 2. 禁用代码规范检查

```
        <plugin>
                      <groupId>org.apache.maven.plugins</groupId>
                      <artifactId>maven-checkstyle-plugin</artifactId>
                      <executions>
                          <execution>
                              <phase>validate</phase>
                              <goals>
                                  <goal>check</goal>
                              </goals>
                              <configuration>
                                  <!--disable check code style-->
                                  <skip>true</skip>
                              </configuration>
                          </execution>
                      </executions>
                  </plugin>
```

### 3. 在 presto-root 下添加 ojdbc 的依赖信息

```
        <dependency>
                      <groupId>com.oracle</groupId>
                      <artifactId>ojdbc6</artifactId>
                      <version>11.2.0.4.0-atlassian-hosted</version>
                  </dependency>
```

## 集成 module 到 Presto

因为我们是基于源码开发的，为了将 presto-oracle 集成到 Presto 中进行测试以及打包发布还需如下配置：

### 1.修改config.properties 配置文件 

config.properties 文件在“presto/presto-main/etc”路径下，在plugin.bundles 下添加“../presto-oracle/pom.xml”。 

![](https://site.itgrocery.cn/2017/media/15781152880509.jpg)

**只有添加了 presto-oracle 的 pom 信息 presto 在 IDE 中调试时再回加载 presto-oracle 插件，否则无效**，上述配置只是用于开发环境，正式环境下无需配置。

### 2. 修改 presto.xml 配置文件 presto.xml 文件在“presto/presto-server/src/main/provisio”路径下，添加如下配置信息：

```
        <artifactSet to="plugin/oracle">
              <artifact id="${project.groupId}:presto-oracle:zip:${project.version}">
                  <unpack />
              </artifact>
          </artifactSet>
```

![](https://site.itgrocery.cn/2017/media/15781153734779.jpg)

上述配置的作用是在 presto 编译时可以将我们的 presto-oracle 插件添加到 plugin 目录下。

![](https://site.itgrocery.cn/2017/media/15781153846079.jpg)

## 编写代码

插件的代码就四个类，一点都不复杂，但是需要说明的是这些代码必须包含 license 信息，因为 presto 配置了证书的检查插件，如果代码中不包含license 编译时会报错。这个不像代码检查那么麻烦，代码检查有一点点不规范就会报错，这个证书检查只要在自己新建的类中添加 license 就即可通过。

### 1.OraclePlugin.java

```
        /*
        * Licensed under the Apache License, Version 2.0 (the "License");
        * you may not use this file except in compliance with the License.
        * You may obtain a copy of the License at
        *
        *     http://www.apache.org/licenses/LICENSE-2.0
        *
        * Unless required by applicable law or agreed to in writing, software
        * distributed under the License is distributed on an "AS IS" BASIS,
        * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        * See the License for the specific language governing permissions and
        * limitations under the License.
        */
        package com.facebook.presto.plugin.oracle;

        import com.facebook.presto.plugin.jdbc.JdbcPlugin;

        public class OraclePlugin
              extends JdbcPlugin
        {
          public OraclePlugin()
          {
              super("oracle", new OracleClientModule());
          }
        }
```

上面构造函数中传入的"oracle" 应该是用于后面 catalog 中 name 的配置项，这个猜测没有验证，先这样配置。

### 2.OracleConfig.java

```
        /*
        * Licensed under the Apache License, Version 2.0 (the "License");
        * you may not use this file except in compliance with the License.
        * You may obtain a copy of the License at
        *
        *     http://www.apache.org/licenses/LICENSE-2.0
        *
        * Unless required by applicable law or agreed to in writing, software
        * distributed under the License is distributed on an "AS IS" BASIS,
        * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        * See the License for the specific language governing permissions and
        * limitations under the License.
        */
        package com.facebook.presto.plugin.oracle;

        import io.airlift.configuration.Config;

        public class OracleConfig {
          private String user;
          private String password;
          private String url;

          public String getUser() {
              return user;
          }

          @Config("oracle.user")
          public OracleConfig setUser(String user) {
              this.user = user;
              return this;
          }

          public String getPassword() {
              return password;
          }

          @Config("oracle.password")
          public OracleConfig setPassword(String password) {
              this.password = password;
              return this;
          }

          public String getUrl() {
              return url;
          }

          @Config("oracle.password")
          public OracleConfig setUrl(String url) {
              this.url = url;
              return this;
          }
        }
```

上述代码中的注解千万不要省略掉，否则 presto 加载 catalog 时无法查找到这些属性。
    
### 3.OracleClientModule.java

```
          /*
           * Licensed under the Apache License, Version 2.0 (the "License");
           * you may not use this file except in compliance with the License.
           * You may obtain a copy of the License at
           *
           *     http://www.apache.org/licenses/LICENSE-2.0
           *
           * Unless required by applicable law or agreed to in writing, software
           * distributed under the License is distributed on an "AS IS" BASIS,
           * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
           * See the License for the specific language governing permissions and
           * limitations under the License.
           */
          package com.facebook.presto.plugin.oracle;

          import com.facebook.presto.plugin.jdbc.BaseJdbcConfig;
          import com.facebook.presto.plugin.jdbc.JdbcClient;
          import com.google.inject.Binder;
          import com.google.inject.Module;
          import com.google.inject.Scopes;

          import static io.airlift.configuration.ConfigBinder.configBinder;

          public class OracleClientModule
                  implements Module
          {
              @Override
              public void configure(Binder binder)
              {
                  binder.bind(JdbcClient.class).to(OracleClient.class).in(Scopes.SINGLETON);
                  configBinder(binder).bindConfig(BaseJdbcConfig.class);
                  configBinder(binder).bindConfig(OracleConfig.class);
              }
          }
          
```

### 4.OracleClient.java

```
        /*
        * Licensed under the Apache License, Version 2.0 (the "License");
        * you may not use this file except in compliance with the License.
        * You may obtain a copy of the License at
        *
        *     http://www.apache.org/licenses/LICENSE-2.0
        *
        * Unless required by applicable law or agreed to in writing, software
        * distributed under the License is distributed on an "AS IS" BASIS,
        * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        * See the License for the specific language governing permissions and
        * limitations under the License.
        */
        package com.facebook.presto.plugin.oracle;

        import com.facebook.presto.plugin.jdbc.*;
        import com.facebook.presto.spi.*;
        import com.facebook.presto.spi.type.Type;
        import com.google.common.base.Joiner;
        import com.google.common.base.Throwables;
        import com.google.common.collect.ImmutableList;
        import com.google.common.collect.ImmutableSet;
        import oracle.jdbc.OracleDriver;

        import javax.annotation.Nullable;
        import javax.inject.Inject;

        import java.sql.Connection;
        import java.sql.DatabaseMetaData;
        import java.sql.ResultSet;
        import java.sql.SQLException;
        import java.util.ArrayList;
        import java.util.List;
        import java.util.Set;
        import java.util.UUID;

        import static com.facebook.presto.plugin.jdbc.JdbcErrorCode.JDBC_ERROR;
        import static com.facebook.presto.spi.StandardErrorCode.NOT_FOUND;
        import static com.facebook.presto.spi.StandardErrorCode.NOT_SUPPORTED;
        import static com.google.common.collect.Iterables.getOnlyElement;
        import static com.google.common.collect.Maps.fromProperties;
        import static java.util.Locale.ENGLISH;

        public class OracleClient
              extends BaseJdbcClient
        {
          @Inject
          public OracleClient(JdbcConnectorId connectorId, BaseJdbcConfig config, OracleConfig oracleConfig)
                  throws SQLException
          {
              super(connectorId, config, "", new OracleDriver());
          }

          @Override
          public Set<String> getSchemaNames() {
              try (Connection connection = driver.connect(connectionUrl,
                      connectionProperties);
                   ResultSet resultSet = connection.getMetaData().getSchemas()) {
                  ImmutableSet.Builder<String> schemaNames = ImmutableSet.builder();
                  while (resultSet.next()) {
                      String schemaName = resultSet.getString(1).toLowerCase();
                      schemaNames.add(schemaName);
                  }
                  return schemaNames.build();
              } catch (SQLException e) {
                  throw Throwables.propagate(e);
              }
          }

          @Override
          protected ResultSet getTables(Connection connection, String schemaName, String tableName) throws SQLException {
              return connection.getMetaData().getTables(null, schemaName, tableName,
                      new String[] { "TABLE", "SYNONYM" });
          }

          @Nullable
          @Override
          public JdbcTableHandle getTableHandle(SchemaTableName schemaTableName) {
              try (Connection connection = driver.connect(connectionUrl,
                      connectionProperties)) {
                  DatabaseMetaData metadata = connection.getMetaData();
                  String jdbcSchemaName = schemaTableName.getSchemaName();
                  String jdbcTableName = schemaTableName.getTableName();
                  if (metadata.storesUpperCaseIdentifiers()) {
                      jdbcSchemaName = jdbcSchemaName.toUpperCase();
                      jdbcTableName = jdbcTableName.toUpperCase();
                  }
                  try (ResultSet resultSet = getTables(connection, jdbcSchemaName,
                          jdbcTableName)) {
                      List<JdbcTableHandle> tableHandles = new ArrayList<>();
                      while (resultSet.next()) {
                          tableHandles.add(new JdbcTableHandle(connectorId,
                                  schemaTableName, resultSet.getString("TABLE_CAT"),
                                  resultSet.getString("TABLE_SCHEM"), resultSet
                                  .getString("TABLE_NAME")));
                      }
                      if (tableHandles.isEmpty()) {
                          return null;
                      }
                      if (tableHandles.size() > 1) {
                          throw new PrestoException(NOT_SUPPORTED,
                                  "Multiple tables matched: " + schemaTableName);
                      }
                      return getOnlyElement(tableHandles);
                  }
              } catch (SQLException e) {
                  throw Throwables.propagate(e);
              }
          }

          @Override
          public List<JdbcColumnHandle> getColumns(JdbcTableHandle tableHandle) {
              try (Connection connection = driver.connect(connectionUrl,
                      connectionProperties)) {

                  ( (oracle.jdbc.driver.OracleConnection)connection ).setIncludeSynonyms(true);
                  DatabaseMetaData metadata = connection.getMetaData();
                  String schemaName = tableHandle.getSchemaName().toUpperCase();
                  String tableName = tableHandle.getTableName().toUpperCase();
                  try (ResultSet resultSet = metadata.getColumns(null, schemaName,
                          tableName, null)) {
                      List<JdbcColumnHandle> columns = new ArrayList<>();
                      boolean found = false;
                      while (resultSet.next()) {
                          found = true;
                          Type columnType = toPrestoType(resultSet
                                  .getInt("DATA_TYPE"), resultSet.getInt("COLUMN_SIZE"));
                          if (columnType != null) {
                              String columnName = resultSet.getString("COLUMN_NAME");
                              columns.add(new JdbcColumnHandle(connectorId,
                                      columnName, columnType));
                          }
                      }
                      if (!found) {
                          throw new TableNotFoundException(
                                  tableHandle.getSchemaTableName());
                      }
                      if (columns.isEmpty()) {
                          throw new PrestoException(NOT_SUPPORTED,
                                  "Table has no supported column types: "
                                          + tableHandle.getSchemaTableName());
                      }
                      return ImmutableList.copyOf(columns);
                  }
              } catch (SQLException e) {
                  throw Throwables.propagate(e);
              }
          }

          @Override
          public List<SchemaTableName> getTableNames(@Nullable String schema) {
              try (Connection connection = driver.connect(connectionUrl,
                      connectionProperties)) {
                  DatabaseMetaData metadata = connection.getMetaData();
                  if (metadata.storesUpperCaseIdentifiers() && (schema != null)) {
                      schema = schema.toUpperCase();
                  }
                  try (ResultSet resultSet = getTables(connection, schema, null)) {
                      ImmutableList.Builder<SchemaTableName> list = ImmutableList
                              .builder();
                      while (resultSet.next()) {
                          list.add(getSchemaTableName(resultSet));
                      }
                      return list.build();
                  }
              } catch (SQLException e) {
                  throw Throwables.propagate(e);
              }
          }

          @Override
          protected SchemaTableName getSchemaTableName(ResultSet resultSet) throws SQLException {
              String tableSchema = resultSet.getString("TABLE_SCHEM");
              String tableName = resultSet.getString("TABLE_NAME");
              if (tableSchema != null) {
                  tableSchema = tableSchema.toLowerCase();
              }
              if (tableName != null) {
                  tableName = tableName.toLowerCase();
              }
              return new SchemaTableName(tableSchema, tableName);
          }

          @Override
          public void commitCreateTable(JdbcOutputTableHandle handle) {
              StringBuilder sql = new StringBuilder()
                      .append("ALTER TABLE ")
                      .append(quoted(handle.getCatalogName(), handle.getSchemaName(), handle.getTemporaryTableName()))
                      .append(" RENAME TO ")
                      //new table name needn't to be with catalog and schema
                      .append(handle.getTableName());

              try (Connection connection = getConnection(handle)) {
                  execute(connection, sql.toString());
              }
              catch (SQLException e) {
                  throw new PrestoException(JDBC_ERROR, e);
              }
          }

        }
        
```

上述代码中覆写了很多方法，主要是不同的数据库规则不一样，需要我们一一适配。之前提到过 github 中已经有人托管了[presto-oracle](https://github.com/marcelopaesrech/presto-oracle) 的插件，但是这个插件没有适配好。例如我又覆写了**commitCreateTable** 这个方法，主要是因为 oracle 中修改表名时新表名不需要再添加 schema ，否则会报错。

### 5.BaseJdbcClient.java 

“BaseJdbcClient”是“OracleClient”的基类，它里面有一个方法涉及到创建临时表的方法，oracle 中表名有长度限制（30以内），所以我对表名进行了字符串的截取操作。

```
        private JdbcOutputTableHandle beginWriteTable(ConnectorTableMetadata tableMetadata)
         {
             SchemaTableName schemaTableName = tableMetadata.getTable();
             String schema = schemaTableName.getSchemaName();
             String table = schemaTableName.getTableName();

             if (!getSchemaNames().contains(schema)) {
                 throw new PrestoException(NOT_FOUND, "Schema not found: " + schema);
             }

             try (Connection connection = driver.connect(connectionUrl, connectionProperties)) {
                 boolean uppercase = connection.getMetaData().storesUpperCaseIdentifiers();
                 if (uppercase) {
                     schema = schema.toUpperCase(ENGLISH);
                     table = table.toUpperCase(ENGLISH);
                 }
                 String catalog = connection.getCatalog();

                 String temporaryName = "tmp_presto_" + UUID.randomUUID().toString().replace("-", "");
                 temporaryName = temporaryName.substring(0, 29);
                 StringBuilder sql = new StringBuilder()
                         .append("CREATE TABLE ")
                         .append(quoted(catalog, schema, temporaryName))
                         .append(" (");
                 ImmutableList.Builder<String> columnNames = ImmutableList.builder();
                 ImmutableList.Builder<Type> columnTypes = ImmutableList.builder();
                 ImmutableList.Builder<String> columnList = ImmutableList.builder();
                 for (ColumnMetadata column : tableMetadata.getColumns()) {
                     String columnName = column.getName();
                     if (uppercase) {
                         columnName = columnName.toUpperCase(ENGLISH);
                     }
                     columnNames.add(columnName);
                     columnTypes.add(column.getType());
                     columnList.add(new StringBuilder()
                             .append(quoted(columnName))
                             .append(" ")
                             .append(toSqlType(column.getType()))
                             .toString());
                 }
                 Joiner.on(", ").appendTo(sql, columnList.build());
                 sql.append(")");

                 execute(connection, sql.toString());

                 return new JdbcOutputTableHandle(
                         connectorId,
                         catalog,
                         schema,
                         table,
                         columnNames.build(),
                         columnTypes.build(),
                         temporaryName,
                         connectionUrl,
                         fromProperties(connectionProperties));
             }
             catch (SQLException e) {
                 throw new PrestoException(JDBC_ERROR, e);
             }
         }
```

## 编译打包

如果上述操作无误的话，重新编译 presto。编译成功之后会有 tar.gz 和 rpm 两种安装包。

###  1.tar.gz 文件

tar.gz 文件路径：presto/presto-srver/target 

![](https://site.itgrocery.cn/2017/media/15781154142633.jpg)

### 2.rpm 文件路径：presto/presto-server-rpm/target

![](https://site.itgrocery.cn/2017/media/15781154252940.jpg)


如果上述操作出现问题，可以参照我托管的 [Presto](https://github.com/itren/presto-with-oracle)，也可以留言与我共同探讨 。