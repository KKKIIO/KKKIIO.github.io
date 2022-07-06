---
layout: post
title: "几行 Flink SQL 实现物化视图"
date: 2022-06-10 11:35:00 +0800
categories: engineering
tags: stream-processing database
---

![Materialized View](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/materialized-view-pattern-diagram.png)

几周前面试遇到一道题：在一个预约业务里，展示顾客复购的情况，是典型的在几张业务表上聚合信息的需求。用 SQL 可以很方便表达出我们需要的数据，但每次查询都要扫全表性能很差，这时可以使用[物化视图(Materialized View)](https://en.wikipedia.org/wiki/Materialized_view)，即预计算结果到一张表或 ElasticSearch 索引，就能缩短读路径，提高查询性能。物化视图有多种实现方式，各有明显优缺点，本文介绍用 Flink SQL 实现物化视图这种方案。

## 流处理+数据库

物化视图有两种实现方式：

- 存储服务原生实现：跟用 SQL 创建 View 一样，很方便，但功能一般不全。像 PostgreSQL 提供的 [Materialized Views](https://www.postgresql.org/docs/current/rules-materializedviews.html)，要用户用`REFRESH`更新，也就是不能自动增量更新。
- 外部程序同步数据：开发外部程序监听数据存储的变化([Change Data Capture(CDC)](https://en.wikipedia.org/wiki/Change_data_capture))，经过一些计算再把结果写进存储服务，例如写程序读 MySQL Binlog 增量更新结果。好处是灵活不受限制，但坏在需要开发工作。

这两种方式都有明显优缺点。恰巧面试后隔天我看一个[新闻](https://www.infoq.cn/article/OIFS2PtAZlMsgBbsW6eO)提到流数据库，突然想到可以用流处理框架 [Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/) 的 Table API 实现 Materialized View，这方案行得通的话我们就可以同时拥有两个优点：极少工作量的 SQL + 自动增量更新。

我很快写了用 Flink SQL 解决这道面试题的[Demo](https://github.com/KKKIIO/materialized_view_flink)，途中遇到一些部署难和文档不全的问题，所幸 Flink SQL 的简洁强大到达了我的预期。末尾我还写了 Flink DataStream 用算子实现 Materialized View 的方案作为对比，感兴趣可以看一下。

## 用 Flink SQL 实现物化视图

### 面试题

首先我们有三张已经存在的业务表，分别是顾客表、预约表、顾客预约习惯表：

```sql
CREATE TABLE `customer_tab` (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(256) NOT NULL,
    last_name VARCHAR(256) NOT NULL
);
CREATE TABLE `order_tab` (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    order_time BIGINT NOT NULL,
    create_time BIGINT NOT NULL
);
CREATE TABLE `customer_preference_tab` (
    customer_id BIGINT PRIMARY KEY,
    frequency INT NOT NULL COMMENT 'days'
);
```

我们需要展示每个顾客的预约单总数，最后的预约时间，以及猜测下次预约的时间，所有字段可过滤，需分页展示。

方便起见我们把结果放到表`customer_reorder_tab`：

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

### Flink SQL 读取表数据

[MySQL CDC Connector](https://ververica.github.io/flink-cdc-connectors/master/content/connectors/mysql-cdc.html) 能让我们把 MySQL 表引入 Flink 作为数据来源(Source)，是现成的解决方案，美中不足的是文档不够详细。

我们先映射`customer_tab`、`order_tab`、`customer_preference_tab` MySQL 表到 Flink 里作为 Source：

```java
TableEnvironment env = TableEnvironment.create(EnvironmentSettings.newInstance().inStreamingMode().build());

env.getConfig().getConfiguration().setString("execution.checkpointing.interval", "3s");

val cdcOptSQL = String.format(
        "WITH ('connector'='mysql-cdc', 'hostname'='%s', 'port'='%d','username'='%s', 'password'='%s', 'database-name'='%s', 'table-name'='%%s', 'server-id'='%%d')",
        mysqlHost, mysqlPort, mysqlUser, mysqlPassword, mysqlDb);
env.executeSql(
        "CREATE TEMPORARY TABLE customer_tab (id BIGINT, first_name STRING, last_name STRING, PRIMARY KEY(id) NOT ENFORCED) "
                + String.format(cdcOptSQL, "customer_tab", 5401));
env.executeSql(
        "CREATE TEMPORARY TABLE order_tab (id BIGINT, customer_id BIGINT, order_time BIGINT, create_time BIGINT, PRIMARY KEY(id) NOT ENFORCED) "
                + String.format(cdcOptSQL, "order_tab", 5402));
env.executeSql(
        "CREATE TEMPORARY TABLE customer_preference_tab (customer_id BIGINT, frequency INT, PRIMARY KEY(customer_id) NOT ENFORCED) "
                + String.format(cdcOptSQL, "customer_preference_tab", 5403));
```

除了连接配置和列类型，表定义跟 MySQL 几乎是一样的。

### Flink SQL 写入表数据

[JDBC SQL Connector](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/connectors/table/jdbc/) 能让我们把 MySQL 表作为 Flink 的输出(Sink)。

我们把 `customer_reorder_tab` 表作为 Sink

```java
env.executeSql(
        "CREATE TEMPORARY TABLE customer_reorder_tab (customer_id BIGINT, first_name STRING, last_name STRING, order_count INT, last_order_time BIGINT, expected_next_order_time BIGINT, PRIMARY KEY(customer_id) NOT ENFORCED) "
                + String.format(
                        "WITH ('connector'='jdbc', 'url'='jdbc:mysql://%s:%d/%s', 'table-name' = 'customer_reorder_tab', 'username' = '%s', 'password' = '%s')",
                        mysqlHost, mysqlPort, mysqlDb, mysqlUser, mysqlPassword));
```

### 用 Flink SQL 表达聚合计算

我们用 `JOIN` 连接三个表，用 `customer_id` 做分组(`GROUP BY`)聚合出结果，并更新(`INSERT INTO`)到`customer_reorder_tab`表

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

## Flink SQL 的工程问题

Flink SQL 这么简洁，这个解决方案可以在生产环境上用吗？

可以，但**门槛**有些高。

### 繁重的部署

![Flink Architecture](https://nightlies.apache.org/flink/flink-docs-master/fig/deployment_overview.svg)

Flink 一般作为集群部署，写一个计算程序需要部署上图这么多组件（即便出了[Application Mode](https://flink.apache.org/news/2020/07/14/application-mode.html)可以向 Kubernetes 提交单应用集群简化部署，组件还是一个没少），一般人都会犹豫，害怕一个问题变成两个问题。

相比，如果你能用 CLI 连接流数据库，直接输入 SQL 就能得到一个 Materialized View，是不是更让人心动呢？

### 高复杂度

用一个 Go 程序，把 Binlog 位置和计算结果都放到 MySQL，这样简单的方案即使出现问题，你都能很快地定位到并修复它。

而像 Flink 这样一个追求通用、高可用、强一致性的一个庞大的框架（跟所有的 Java 框架一样），你很难对它有掌握感，Java 库泛滥的 Runtime Dispatch 也是我本次看源码解决问题时的一个头疼的点。

## 附：用 Flink DataStream 实现物化视图

Flink 跟 MapReduce 、 Apache Storm 那段大数据时期的项目一样，一开始就提供了编程接口 [DataStream API](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/learn-flink/datastream_api/)。我们用 DataStream API 重新实现一下物化视图，对比一下其他方案，还可以看到 Flink 里的一些流处理概念。

### Flink MySQL Source

依旧先引入 Source

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

`MySqlSource` 可以让我们把过于灵活的`SourceRecord`转换成我们期望的结构`Change`，我们根据表做转换，并且为了方便不考虑删除事件：

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

### 单独处理`customer_tab`以减少聚合运算

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

### 分组聚合`order_tab`和`customer_preference_tab`

接下来我们将`order_tab`和`customer_preference_tab`的变更根据`customer_id`分组，每遇到一个变更就更新一下`customer_reorder_tab`：

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

### 分组聚合需要状态(State)

`customer_reorder_tab`新的值，可以对同个`customer_id`的`Change`做 _reduce_ 得出：

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

拓扑图变得很简单，因为 Flink 会合并 Trivial 的步骤：

![topology](/assets/image/flink-ds-topo.png)

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

### DataStream 比 SQL 更繁琐

Flink DataStream 明显难用了很多，它需要你处理具体的（增删改）事件，虽然没有 `JOIN` ，但我们还是要显示地管理每个顾客的状态`reorderState`，而 Flink SQL 让用户不太需要关注这些实现细节。

Flink DataStream 理论上可以做的事情比 Flink SQL 多，但代码多了，可能出错的地方也会变多，例如我一开始没有考虑 Order 是否是新增事件就递增了`state.orderCount`。

### DataStream 比 Homemade Program 更标准

基本上自研程序也是在做流处理：监听变化事件、写入计算结果，实现思路是差不多的。不同点在于高可用和强一致的方案：

- Flink 可以定时 Checkpoint 各个步骤的 State ，增强每个步骤的模块化，坏处是要提供额外的存储(`EmbeddedRocksDBStateBackend`)，而且 Flink 只（能）保证内部状态的一致性，外部结果只能保证最终一致性，例如重启 Flink Job 可能会导致 MySQL 数据“回退”几秒。
- 自研程序要自己实现 **事务性的存取** Binlog 位置和计算中间结果等数据，好处是可以实现端到端(end-to-end)的一致性，例如使用 MySQL 事务存取。
