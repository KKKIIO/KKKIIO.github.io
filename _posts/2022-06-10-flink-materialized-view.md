---
layout: post
title: "用Flink解面试题"
date: 2022-06-10 11:35:00 +0800
categories: stream-processing database
---

几周前面试遇到一道题：在一个预约业务里，展示顾客复购的情况。首先我们有顾客表、预约表、顾客预约习惯表：

```sql
CREATE TABLE `customer_tab` (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(256) NOT NULL,
    last_name VARCHAR(256) NOT NULL
);
```

```sql
CREATE TABLE `order_tab` (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    order_time BIGINT NOT NULL,
    create_time BIGINT NOT NULL
);
```

```sql
CREATE TABLE `customer_preference_tab` (
    customer_id BIGINT PRIMARY KEY,
    frequency INT NOT NULL COMMENT 'days'
);
```

要展示

| 列                     | 解释               |
| ---------------------- | ------------------ |
| customer_id            |                    |
| first_name             |                    |
| last_name              |                    |
| order_count            | 下单数             |
| last_order_time        | 最新的预约时间     |
| expect_next_order_time | 预期下次预约的时间 |

- 所有字段要能排序
- `first_name`、`last_name`要能模糊搜索/过滤
- 要支持过滤出**未像往常一样预定**的顾客(`expect_next_order_time < now()`)
- 支持分页

## Materialized View

用 SQL 可以很方便表达出我们需要的数据，但性能很差。如果能把数据聚合到同一张表或 ES 索引，就能缩短读路径，得到合理的查询性能。

PostgreSQL 的[Materialized Views](https://www.postgresql.org/docs/current/rules-materializedviews.html)更新性能不够，传统做法是用[Change Data Capture](https://en.wikipedia.org/wiki/Change_data_capture)(CDC)，例如写程序读 MySQL Binlog 增量更新结果。

隔天看一个[新闻](https://www.infoq.cn/article/OIFS2PtAZlMsgBbsW6eO)提到流数据库，突然想到可以用 [Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/) SQL 实现 Materialized View，我对方案能有多简单感到很兴奋。

准备好的环境已放在 [GitHub](https://github.com/KKKIIO/materialized_view_flink) 上。

## 环境准备

### MySQL

首先准备一个 MySQL 服务，开启 Binlog

```yaml
# docker-compose.yml
version: "2.1"
services:
  mysql:
    image: mysql:5.7
    command: --server-id=1 --log-bin=/var/lib/mysql/mysql-bin.log --binlog_do_db=demo --binlog_format=row --expire_logs_days=1
    environment:
      MYSQL_DATABASE: "demo"
      MYSQL_ROOT_PASSWORD: "123456"
    volumes:
      - ./init-sql:/docker-entrypoint-initdb.d
    ports:
      - "3306:3306"
```

在 init-sql 里创建之前提到的三张表，并增加结果表`customer_reorder_tab`：

```sql
CREATE TABLE `customer_reorder_tab` (
    customer_id BIGINT PRIMARY KEY,
    first_name VARCHAR(256) NOT NULL DEFAULT '',
    last_name VARCHAR(256) NOT NULL DEFAULT '',
    order_count INT NOT NULL DEFAULT '0',
    last_order_time BIGINT NOT NULL DEFAULT '0',
    expected_next_order_time BIGINT NOT NULL DEFAULT '0'
);
```

也可以把结果同步到 ElasticSearch，这里不演示。

### Flink

启动 [Flink v1.13.6](https://nightlies.apache.org/flink/flink-docs-release-1.13//docs/try-flink/local_installation/) 集群（不同 Flink 版本与库有兼容问题），将 [MySQL CDC Connector](https://ververica.github.io/flink-cdc-connectors/master/content/connectors/mysql-cdc.html) 加入 Java 程序依赖。

```xml
<dependency>
    <groupId>com.ververica</groupId>
    <artifactId>flink-connector-mysql-cdc</artifactId>
    <version>2.2.1</version>
</dependency>
```

MySQL CDC Connector 能让我们把 MySQL 表引入 Flink 作为数据来源(Source)，它是[CDC Connectors for Apache Flink](https://ververica.github.io/flink-cdc-connectors/master/index.html)项目的一部分，这个项目里还有其他场景的[Demo](https://ververica.github.io/flink-cdc-connectors/master/content/quickstart/index.html)。

我们还需要依赖 [JDBC SQL Connector](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/table/jdbc/)，它能让我们把 MySQL 表作为 Flink 的输出(Sink)。

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-jdbc_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
</dependency>
```

## 编写 Flink SQL

我们先映射`customer_tab`、`order_tab`、`customer_preference_tab` MySQL 表到 Flink 里作为 Source：

```java
TableEnvironment env = TableEnvironment.create(EnvironmentSettings.newInstance().inStreamingMode().build());

env.getConfig().getConfiguration().setString("execution.checkpointing.interval", "3s");

String cdcOptSQL = String.format(
        "WITH ('connector'='mysql-cdc', 'hostname'='%s', 'port'='%d','username'='%s', 'password'='%s', 'database-name'='%s', 'table-name'='%%s')",
        mysqlHost, mysqlPort, mysqlUser, mysqlPassword, mysqlDb);
env.executeSql(
        "CREATE TEMPORARY TABLE customer_tab (id BIGINT, first_name STRING, last_name STRING, PRIMARY KEY(id) NOT ENFORCED) "
                + String.format(cdcOptSQL, "customer_tab"));
env.executeSql(
        "CREATE TEMPORARY TABLE order_tab (id BIGINT, customer_id BIGINT, order_time BIGINT, create_time BIGINT, PRIMARY KEY(id) NOT ENFORCED) "
                + String.format(cdcOptSQL, "order_tab"));
env.executeSql(
        "CREATE TEMPORARY TABLE customer_preference_tab (customer_id BIGINT, frequency INT, PRIMARY KEY(customer_id) NOT ENFORCED) "
                + String.format(cdcOptSQL, "customer_preference_tab"));
```

`customer_reorder_tab` 表作为 Sink

```java
env.executeSql(
        "CREATE TEMPORARY TABLE customer_reorder_tab (customer_id BIGINT, first_name STRING, last_name STRING, order_count INT, last_order_time BIGINT, expected_next_order_time BIGINT, PRIMARY KEY(customer_id) NOT ENFORCED) "
                + String.format(
                        "WITH ('connector'='jdbc', 'url'='jdbc:mysql://%s:%d/%s', 'table-name' = 'customer_reorder_tab', 'username' = '%s', 'password' = '%s')",
                        mysqlHost, mysqlPort, mysqlDb, mysqlUser, mysqlPassword));
```

用 SQL 聚合出结果，并更新到`customer_reorder_tab`表

```java
env.executeSql(
        "INSERT INTO customer_reorder_tab \n" +
                "SELECT c.id, FIRST_VALUE(c.first_name), FIRST_VALUE(c.last_name), CAST(COUNT(o.id) AS INT)\n" +
                ", IFNULL(MAX(o.order_time),0)\n" +
                ", IFNULL(MAX(o.order_time) + NULLIF(FIRST_VALUE(cp.frequency),0) * 24 * 3600000, 0) \n" +
                "FROM customer_tab AS c \n" +
                "LEFT OUTER JOIN order_tab AS o ON c.id = o.customer_id \n" +
                "LEFT OUTER JOIN customer_preference_tab AS cp ON c.id = cp.customer_id \n" +
                "GROUP BY c.id");
```

完成！看一下测试结果：

```
...
Day 34
new customer count: 85, new order count: 320
after waiting 3s, mismatch count: 0
Day 35
new customer count: 1, new order count: 276
after waiting 3s, mismatch count: 0
Day 36
new customer count: 84, new order count: 338
after waiting 3s, mismatch count: 0
Day 37
new customer count: 62, new order count: 370
after waiting 3s, mismatch count: 0
Day 38
new customer count: 40, new order count: 285
after waiting 3s, mismatch count: 0
...
```

没问题，再看看拓扑图：

![topology](/assets/image/flink-sql-topo.png)

也很直观，容易对应到 SQL 的 Join。

结束了？

## 太多魔法

Flink 跟 MapReduce 、 Apache Storm 那段大数据时期的项目一样，一开始就提供了编程接口 [DataStream API](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/learn-flink/datastream_api/)。我们用 DataStream API 重新实现一下，靠近一些看 Flink。

### Flink DataStream

依旧先引入 `MySqlSource`

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(3000);

val mySqlSource = MySqlSource.<Change>builder()
        .hostname(mysqlHost)
        .port(mysqlPort)
        .databaseList(mysqlDb)
        .tableList("demo.customer_tab", "demo.order_tab", "demo.customer_preference_tab")
        .username(mysqlUser)
        .password(mysqlPassword)
        .deserializer(new ChangeDeserializer())
        .build();
val src = env.fromSource(mySqlSource, WatermarkStrategy.noWatermarks(), "MySQL Customer Source");
```

方便起见我们不考虑删除事件：

```java
public class ChangeDeserializer implements DebeziumDeserializationSchema<Change> {
    // ...

    @Override
    public void deserialize(SourceRecord record, Collector<Change> out) throws Exception {
        val payload = ((Struct) record.value());
        val table = payload.getStruct("source").getString("table");
        val newValue = payload.getStruct("after");
        if (newValue == null) {
            return;
        }
        boolean create = false;
        switch (payload.getString("op")) {
            case "c":
            case "r": // snapshot read
                create = true;
                break;
        }
        val changeBuilder = Change.builder().create(create);
        try {
            switch (table) {
                case "customer_tab":
                    changeBuilder.customer(new Customer(newValue.getInt64("id"), newValue.getString("first_name"),
                            newValue.getString("last_name")));
                    break;
                case "order_tab":
                    changeBuilder.order(new Order(newValue.getInt64("id"), newValue.getInt64("customer_id"),
                            newValue.getInt64("order_time"), newValue.getInt64("create_time")));
                    break;
                case "customer_preference_tab":
                    changeBuilder.customerPreference(new CustomerPreference(newValue.getInt64("customer_id"),
                            newValue.getInt32("frequency")));
                    break;
                default:
                    throw new IllegalArgumentException(String.format("Unknown table %s, payload: %s", table, payload));

            }
        } catch (DataException e) {
            throw new DataException(String.format("Failed to deserialize payload: %s", payload), e);
        }
        out.collect(changeBuilder.build());
    }
}
```

这次我想处理得更简单些，因为`customer_tab`的数据可以单独同步到`customer_reorder_tab`，而`customer_reorder_tab`其他需要计算的字段并不需要`customer_tab`的参与，`order_tab`和`customer_preference_tab`都有`customer_id`，可以自己聚合。

我们把`customer_tab`的变更事件单独拎出来：

```java
val customerTag = new OutputTag<Customer>("customer") { };
val mainStream = src.process(new ProcessFunction<Change, Change>() {
    private static final long serialVersionUID = 1L;

    @Override
    public void processElement(Change value, ProcessFunction<Change, Change>.Context ctx,
            Collector<Change> out)
            throws Exception {
        if (value.getCustomer() != null) {
            ctx.output(customerTag, value.getCustomer());
        } else {
            out.collect(value);
        }
    }
});
```

遇到新增和修改就同步到`customer_reorder_tab`：

```java
val jdbcConnOpts = new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
        .withUrl(String.format("jdbc:mysql://%s:%d/%s?useServerPrepStmts=false&rewriteBatchedStatements=true",
                mysqlHost, mysqlPort, mysqlDb))
        .withDriverName("com.mysql.cj.jdbc.Driver")
        .withUsername(mysqlUser)
        .withPassword(mysqlPassword)
        .build();
mainStream.getSideOutput(customerTag).addSink(JdbcSink.sink(
        "INSERT INTO customer_reorder_tab (customer_id, first_name, last_name) VALUES (?, ?, ?) ON DUPLICATE KEY UPDATE first_name=VALUES(first_name), last_name=VALUES(last_name)",
                (statement, customer) -> {
                    statement.setLong(1, customer.getId());
                    statement.setString(2, customer.getFirstName());
                    statement.setString(3, customer.getLastName());
                },
        JdbcExecutionOptions.builder().withBatchIntervalMs(200).build(), jdbcConnOpts))
        .name("MySQL Customer Sink");
```

然后我们将`order_tab`和`customer_preference_tab`的变更根据`customer_id`分组，每遇到一个变更就更新一下`customer_reorder_tab`：

```java
mainStream.keyBy(c -> {
    if (c.getOrder() != null) {
        return c.getOrder().getCustomerId();
    } else if (c.getCustomerPreference() != null) {
        return c.getCustomerPreference().getCustomerId();
    } else {
        throw new IllegalArgumentException(String.format("Unknown change type: %s", c));
    }
}).map(new ReorderCalc()).name("Calculate reorder info").addSink(
        JdbcSink.sink(
                "INSERT INTO customer_reorder_tab (customer_id, order_count, last_order_time, expected_next_order_time) VALUES (?, ?, ?, ?) ON DUPLICATE KEY UPDATE order_count=VALUES(order_count), last_order_time=VALUES(last_order_time), expected_next_order_time=VALUES(expected_next_order_time)",
                (statement, reorder) -> {
                    statement.setLong(1, reorder.getCustomerId());
                    statement.setInt(2, reorder.getOrderCount());
                    statement.setLong(3, reorder.getLastOrderTime());
                    statement.setLong(4, reorder.getExpectedNextOrderTime());
                },
                JdbcExecutionOptions.builder().withBatchIntervalMs(200).build(), jdbcConnOpts))
        .name("MySQL ReorderInfo Sink");
```

`customer_reorder_tab`新的值，是对同个`customer_id`的变更做 reduce 得出的：

- `order_count`：遇到新订单就递增
- `last_order_time`：遇到订单变更就取更大的值
- `expected_next_order_time`：遇到顾客习惯变更就更新`frequency`，并将`frequency`与`last_order_time`相加

```java
public class ReorderCalc extends RichMapFunction<Change, ReorderInfo> {
    // ...
    private transient AggregatingState<Change, ReorderState> reorderState;

    @Override
    public void open(Configuration parameters) throws Exception {
        AggregatingStateDescriptor<Change, ReorderState, ReorderState> descriptor = new AggregatingStateDescriptor<>(
                "reorder",
                new AggregateFunction<Change, ReorderState, ReorderState>() {
                    @Override
                    public ReorderState createAccumulator() {
                        return new ReorderState(0, 0, 0);
                    }

                    @Override
                    public ReorderState add(Change value, ReorderState accumulator) {
                        if (value.getOrder() != null) {
                            return new ReorderState(
                                    accumulator.getOrderCount() + (value.isCreate() ? 1 : 0),
                                    Math.max(accumulator.getLastOrderTime(), value.getOrder().getOrderTime()),
                                    accumulator.getFrequency());
                        } else {
                            return new ReorderState(
                                    accumulator.getOrderCount(),
                                    accumulator.getLastOrderTime(),
                                    value.getCustomerPreference().getFrequency());
                        }
                    }
                    //...
                }, TypeInformation.of(ReorderState.class));
        this.reorderState = this.getRuntimeContext().getAggregatingState(descriptor);
    }

    @Override
    public ReorderInfo map(Change value) throws Exception {
        this.reorderState.add(value);
        val state = this.reorderState.get();
        long customerId;
        if (value.getOrder() != null) {
            customerId = value.getOrder().getCustomerId();
        } else {
            customerId = value.getCustomerPreference().getCustomerId();
        }
        long expectedNextOrderTime = 0;
        if (state.getFrequency() > 0 && state.getLastOrderTime() > 0) {
            expectedNextOrderTime = state.getLastOrderTime() + state.getFrequency() * DAY_IN_MILLIS;
        }
        return new ReorderInfo(customerId, state.getOrderCount(), state.getLastOrderTime(), expectedNextOrderTime);
    }
}
```

代码变多了，可能出错的地方也变多了。看一下拓扑图：

![topology](/assets/image/flink-ds-topo.png)

虽然没有 Join 了，但实际上我们还是在 Flink 保存了每个顾客的状态`reorderState`。

测试也过了：

```
...
Day 13
new customer count: 77, new order count: 160
after waiting 3s, mismatch count: 0
Day 14
new customer count: 31, new order count: 132
after waiting 3s, mismatch count: 0
Day 15
new customer count: 30, new order count: 140
after waiting 3s, mismatch count: 0
Day 16
new customer count: 56, new order count: 164
after waiting 3s, mismatch count: 0
Day 17
new customer count: 35, new order count: 166
after waiting 3s, mismatch count: 0
...
```

Flink DataStream 明显难用了很多，它需要你处理具体的（增删改）事件，显示地管理状态，而 Flink SQL 让用户不太需要接触这些概念和实现细节。

## 门槛

Flink SQL 这个解决方案可以在生产环境上用吗？可以，但门槛有些高。

### 部署

![Flink Architecture](https://nightlies.apache.org/flink/flink-docs-master/fig/deployment_overview.svg)

Flink 一般作为集群部署，写一个计算程序需要部署上图这么多组件（即便出了[Application Mode](https://flink.apache.org/news/2020/07/14/application-mode.html)可以向 Kubernetes 提交单应用集群简化部署，组件还是一个没少），一般人都会犹豫，害怕一个问题变成两个问题；相比，如果你能用 CLI 连接流数据库，直接输入 SQL 就能得到一个 Materialized View，是不是更让人心动呢？

### 复杂度

用一个 Go 程序，把 Binlog 位置和计算结果都放到 MySQL，这样简单的方案即使出现问题，你都能很快地定位到并修复它；而像 Flink 这样一个追求通用、高可用、强一致性的一个庞大的框架（跟所有的 Java 框架一样），你很难对它有掌握感。
