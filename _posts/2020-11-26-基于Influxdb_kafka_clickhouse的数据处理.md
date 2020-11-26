---
title: 基于Influxdb_Kafa_Clickhouse的数据处理
author: heinigai
date: 2020-11-26 15:10:00 +0800
categories: [大数据]
tags: [Influx, Kafa, Clickhouse]
---
## clickhouse

1. 执行sql文件创建基础表

    ```shell
    clickhouse-client --user slawrite --ask-password --multiquery < /root/init_db_pro.sql
    fQHa53hQs03g1cfc0x
    ```

2. 创建物化视图

    ```sql
    --- 物化视图 P95 P99 avg err_rate 按月
    create MATERIALIZED VIEW sla_month ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(create_time) ORDER BY (create_time, srv_name) as select t1.create_time as create_time, t1.service_name as srv_name, t1.name as name, t1.P95 as P95, t1.P99 as P99, t1.avg as avg, t1.all_count as all_count, t2.err_count as err_count, divide(t2.err_count, t1.all_count) as err_rate from (select create_time, service_name, name, quantileTiming(0.95)(intDivOrZero(total_time, total_count)) as P95, quantileTiming(0.99)(intDivOrZero(total_time, total_count)) as P99, intDivOrZero(sum(total_time), sum(total_count)) as avg, sum(total_count) as all_count from (select toStartOfInterval(time, INTERVAL 1 month) as create_time, time, service_name, name, total_count, total_time from klooksla.kafka_data where measurement = 'service_stat' and time >= '2020-11-11 00:00:00' order by time) group by service_name, name, create_time order by create_time) as t1 left join (select toStartOfInterval(time, INTERVAL 1 month) as create_time, service_name, name, sum(total_count) as err_count from klooksla.kafka_data where measurement = 'service_error_stat' and time >= '2020-11-11 00:00:00' group by service_name, name, create_time) as t2 on (t1.service_name = t2.service_name and t1.name=t2.name and t1.create_time = t2.create_time) order by create_time

    insert into sla_month select t1.create_time as create_time, t1.service_name as srv_name, t1.name as name, t1.P95 as P95, t1.P99 as P99, t1.avg as avg, t1.all_count as all_count, t2.err_count as err_count, divide(t2.err_count, t1.all_count) as err_rate from (select create_time, service_name, name, quantileTiming(0.95)(intDivOrZero(total_time, total_count)) as P95, quantileTiming(0.99)(intDivOrZero(total_time, total_count)) as P99, intDivOrZero(sum(total_time), sum(total_count)) as avg, sum(total_count) as all_count from (select toStartOfInterval(time, INTERVAL 1 month) as create_time, time, service_name, name, total_count, total_time from klooksla.kafka_data where measurement = 'service_stat' and time < '2020-11-11 00:00:00' order by time) group by service_name, name, create_time order by create_time) as t1 left join (select toStartOfInterval(time, INTERVAL 1 month) as create_time, service_name, name, sum(total_count) as err_count from klooksla.kafka_data where measurement = 'service_error_stat' and time < '2020-11-11 00:00:00' group by service_name, name, create_time) as t2 on (t1.service_name = t2.service_name and t1.name=t2.name and t1.create_time = t2.create_time) order by create_time
    --- 物化视图 P95 P99 avg err_rate 按天
    create MATERIALIZED VIEW sla_day ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(create_time) ORDER BY (create_time, srv_name) as select t1.create_time as create_time, t1.service_name as srv_name, t1.name as name, t1.P95 as P95, t1.P99 as P99, t1.avg as avg, t1.all_count as all_count, t2.err_count as err_count, divide(t2.err_count, t1.all_count) as err_rate from (select create_time, service_name, name, quantileTiming(0.95)(intDivOrZero(total_time, total_count)) as P95, quantileTiming(0.99)(intDivOrZero(total_time, total_count)) as P99, intDivOrZero(sum(total_time), sum(total_count)) as avg, sum(total_count) as all_count from (select toStartOfInterval(time, INTERVAL 1 day) as create_time, time, service_name, name, total_count, total_time from klooksla.kafka_data where measurement = 'service_stat' and time >= '2020-11-11 00:00:00' order by time) group by service_name, name, create_time order by create_time) as t1 left join (select toStartOfInterval(time, INTERVAL 1 day) as create_time, service_name, name, sum(total_count) as err_count from klooksla.kafka_data where measurement = 'service_error_stat' and time >= '2020-11-11 00:00:00' group by service_name, name, create_time) as t2 on (t1.service_name = t2.service_name and t1.name=t2.name and t1.create_time = t2.create_time) order by create_time

    insert into sla_day select t1.create_time as create_time, t1.service_name as srv_name, t1.name as name, t1.P95 as P95, t1.P99 as P99, t1.avg as avg, t1.all_count as all_count, t2.err_count as err_count, divide(t2.err_count, t1.all_count) as err_rate from (select create_time, service_name, name, quantileTiming(0.95)(intDivOrZero(total_time, total_count)) as P95, quantileTiming(0.99)(intDivOrZero(total_time, total_count)) as P99, intDivOrZero(sum(total_time), sum(total_count)) as avg, sum(total_count) as all_count from (select toStartOfInterval(time, INTERVAL 1 day) as create_time, time, service_name, name, total_count, total_time from klooksla.kafka_data where measurement = 'service_stat' and time < '2020-11-11 00:00:00' order by time) group by service_name, name, create_time order by create_time) as t1 left join (select toStartOfInterval(time, INTERVAL 1 day) as create_time, service_name, name, sum(total_count) as err_count from klooksla.kafka_data where measurement = 'service_error_stat' and time < '2020-11-11 00:00:00' group by service_name, name, create_time) as t2 on (t1.service_name = t2.service_name and t1.name=t2.name and t1.create_time = t2.create_time) order by create_time

    --- 服务按分钟聚合response_code数据
    create MATERIALIZED VIEW response_code_minute_test ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMMDD(create_time) ORDER BY (create_time, srv_name) as select t1.create_time as create_time, t1.service_name as srv_name, t1.all_count as all_count, t2.all_count as err_count, divide(t2.all_count, t1.all_count) as err_rate from (select toStartOfInterval(time, INTERVAL 1 minute) as create_time, service_name, sum(total_count) as all_count from kafka_data where measurement = 'srv_rsp_code' and time >= '2020-11-24 01:55:00' group by service_name, create_time) as t1 left join (select toStartOfInterval(time, INTERVAL 1 minute) as create_time, service_name, sum(total_count) as all_count from kafka_data where measurement = 'srv_rsp_code' and response_code >= 400 and time >= '2020-11-24 01:55:00' group by service_name, create_time) as t2 on (t1.create_time = t2.create_time and t1.service_name = t2.service_name) order by create_time

    insert into response_code_minute_test select t1.create_time as create_time, t1.service_name as srv_name, t1.all_count as all_count, t2.all_count as err_count, divide(t2.all_count, t1.all_count) as err_rate from (select toStartOfInterval(time, INTERVAL 1 minute) as create_time, service_name, sum(total_count) as all_count from kafka_data where measurement = 'srv_rsp_code' and time < '2020-11-24 01:55:00' group by service_name, create_time) as t1 left join (select toStartOfInterval(time, INTERVAL 1 minute) as create_time, service_name, sum(total_count) as all_count from kafka_data where measurement = 'srv_rsp_code' and response_code >= 400 and time < '2020-11-24 01:55:00' group by service_name, create_time) as t2 on (t1.create_time = t2.create_time and t1.service_name = t2.service_name) order by create_time
    --- 服务可用性 按天
    create MATERIALIZED VIEW srv_valid_day ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMMDD(day_date) ORDER BY (day_date, srv_name) as select t1.day_date as day_date, t1.srv_name as srv_name, divide(t2.all_minutes, t1.all_minutes) as valid_rate from (select toStartOfInterval(create_time, INTERVAL 1 day) as day_date, srv_name, count() as all_minutes from response_code_minute_test where create_time >= '2020-11-24 01:55:00' group by srv_name, day_date) as t1 left join (select toStartOfInterval(create_time, INTERVAL 1 day) as day_date, srv_name, count() as all_minutes from response_code_minute_test where create_time >= '2020-11-24 01:55:00' and err_rate < 0.1 group by srv_name, day_date) as t2 on (t1.day_date = t2.day_date and t1.srv_name = t2.srv_name) order by day_date

    insert into srv_valid_day select t1.day_date as day_date, t1.srv_name as srv_name, divide(t2.all_minutes, t1.all_minutes) as valid_rate from (select toStartOfInterval(create_time, INTERVAL 1 day) as day_date, srv_name, count() as all_minutes from response_code_minute_test where create_time < '2020-11-24 01:55:00' group by srv_name, day_date) as t1 left join (select toStartOfInterval(create_time, INTERVAL 1 day) as day_date, srv_name, count() as all_minutes from response_code_minute_test where create_time < '2020-11-24 01:55:00' and err_rate < 0.1 group by srv_name, day_date) as t2 on (t1.day_date = t2.day_date and t1.srv_name = t2.srv_name) order by day_date

    --- 服务可用性 按月
    create MATERIALIZED VIEW srv_valid_month ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(month_date) ORDER BY (month_date, srv_name) as select t1.month_date as month_date, t1.srv_name as srv_name, divide(t2.all_minutes, t1.all_minutes) as valid_rate from (select toStartOfInterval(create_time, INTERVAL 1 month) as month_date, srv_name, count() as all_minutes from response_code_minute_test where create_time >= '2020-11-24 01:56:00' group by srv_name, month_date) as t1 left join (select toStartOfInterval(create_time, INTERVAL 1 month) as month_date, srv_name, count() as all_minutes from response_code_minute_test where create_time >= '2020-11-24 01:56:00' and err_rate < 0.1 group by srv_name, month_date) as t2 on (t1.month_date = t2.month_date and t1.srv_name = t2.srv_name) order by month_date

    insert into srv_valid_month select t1.month_date as month_date, t1.srv_name as srv_name, divide(t2.all_minutes, t1.all_minutes) as valid_rate from (select toStartOfInterval(create_time, INTERVAL 1 month) as month_date, srv_name, count() as all_minutes from response_code_minute_test where create_time < '2020-11-24 01:56:00' group by srv_name, month_date) as t1 left join (select toStartOfInterval(create_time, INTERVAL 1 month) as month_date, srv_name, count() as all_minutes from response_code_minute_test where create_time < '2020-11-24 01:56:00' and err_rate < 0.1 group by srv_name, month_date) as t2 on (t1.month_date = t2.month_date and t1.srv_name = t2.srv_name) order by month_date
    ```

3. 常用命令

    ```sql
    --- 删除表分区
    ALTER TABLE table_name DROP PARTITION partition_expr
    ALTER TABLE db_name.table_name DROP PARTITION '20200601'
    --- 删除表 若表数据大于设置的值默认50G则禁止删除，可创建文件解决
    touch '/data/ck/clickhouse/flags/force_drop_table'
    chmod 666 '/data/ck/clickhouse/flags/force_drop_table'
    --- clickhouse客户端登录
    clickhouse-client --user slawrite --ask-password --multiquery < /root/init_db_pro.sql
    clickhouse-client --user slawrite --ask-password
    ```

---

## logstash

1. 执行命令`java -version`检查是否安装java8
2. 如果没有则安装,安装命令`yum install java-1.8.0-openjdk -y`
3. 执行以下命令安装 logstash

    ```shell
    cd /tmp
    wget https://artifacts.elastic.co/downloads/logstash/logstash-7.9.1.rpm
    rpm -ivh /tmp/logstash-7.9.1.rpm
    ```

4. 编辑配置文件 `vim /etc/logstash/conf.d/logstash_pro.conf`

5. 以systemctl 管理 logstash 服务并设置开机自启动

    ```shell
    systemctl start logstash
    systemctl enable logstash.service
    ```

---

## kafka

### 安装java环境

1. 执行命令`java -version`检查是否安装java8
2. 如果没有则安装,安装命令`yum install java-1.8.0-openjdk -y`
3. 编辑文件配置java环境变量`vim /etc/bashrc`

    ```shell
    export JRE_HOME=/usr/lib/jvm/jre
    export JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk
    PATH=$PATH:$JRE_HOME:$JAVA_HOME
    ```

4. 确保~/.bashrc文件引用了/etc/bashrc文件，执行命令`source /etc/bashrc`使配置生效

### 安装kafka

1. 创建kafka用户

    ```shell
    # 该-m标志确保将为用户创建主目录。此主目录/home/kafka充当我们工作区目录
    useradd kafka -m
    ```

2. 设置kafka用户密码，klook101,命令 `passwd kafka`

3. 将kafka用户添加到该wheel组中,以便它具有安装Kafka依赖项所需的特权 `usermod -aG wheel kafka`

4. 切换到kafka用户 `su -l kafka`

5. 执行以下命令下载安装kafka

    ```shell
    cd
    wget http://apache.osuosl.org/kafka/2.6.0/kafka_2.13-2.6.0.tgz
    tar -zxvf kafka_2.13-2.6.0.tgz
    mv kafka_2.13-2.6.0/* .
    rmdir kafka_2.13-2.6.0
    ```

6. 切换到root用户设置system service配置文件

    ```shell
    vi /etc/systemd/system/zookeeper.service
    vi /etc/systemd/system/kafka.service
    # 设置日志路径 log.dirs=/var/log/kafka-logs
    vi /home/kafka/config/server.properties
    mkdir -p /var/log/kafka-logs
    chown kafka:kafka -R /var/log/kafka-logs
    ```

7. 以systemctl 管理进程并设置开机自启

    ```shell
    systemctl start zookeeper.service
    systemctl start kafka.service
    systemctl status zookeeper
    systemctl status kafka
    systemctl enable zookeeper.service
    systemctl enable kafka.service
    ```

### kafka-connect

1. 将libs目录下所有jar拷贝到/home/kafka/libs下

    ```shell
    cd /home/kafka/libs
    cp /tmp/libs.tar.gz .
    chown kafka:kafka -R libs.tar.gz
    su -l kafka
    tar -zxvf libs.tar.gz
    rm -rf libs.tar.gz
    cp /tmp/kafka-connect-influxdb-1.0.0.jar .
    rm -f kafka-connect-influxdb-1.2.1.jar
    ```

2. 切换到root用户设置system service配置文件并进行管理

    ```shell
    vi /etc/systemd/system/kafka-connect-alone.service
    systemctl start kafka-connect.service
    systemctl status kafka-connect
    systemctl enable kafka-connect.service
    ```

3. 检查连接器

    ```shell
    # 列举连接器插件
    curl http://localhost:8083/connector-plugins
    # 删除连接器 删除后不再写消息到指定topic
    curl -X DELETE http://localhost:8083/connectors/InfluxDBSourceStatConnector
    curl -X DELETE http://localhost:8083/connectors/InfluxDBSourceErrorConnector
    curl -X DELETE http://localhost:8083/connectors/InfluxDBSourceRespCodeConnector
    ```

4. 启动连接器插件写数据

    ```shell
    # influxdb usercenterdb.service_stat 数据
    echo '{"name": "InfluxDBSourceStatConnector", "config": {"connector.class": "io.confluent.influxdb.source.InfluxdbSourceConnector", "tasks.max": "1", "topic.prefix": "influx_", "influxdb.url": "http://10.15.29.164:8086", "influxdb.db": "usercenterdb", "query":"select time, service_name, \"name\", sys_fail_count, time_lt_10ms, time_lt_50ms, time_lt_200ms, time_lt_1000ms, time_gt_1000ms, total_count, total_time from service_stat where time >= 1605688860000000000", "mode": "timestamp", "batch.size": 500, "timestamp.delay.interval.ms": 3600000, "value.converter": "org.apache.kafka.connect.json.JsonConverter", "value.converter.schemas.enable": "false"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"
    echo '{"name": "InfluxDBSourceStatConnector", "config": {"connector.class": "io.confluent.influxdb.source.InfluxdbSourceConnector", "tasks.max": "1", "topic.prefix": "influx_", "influxdb.url": "http://10.15.29.164:8086", "influxdb.db": "usercenterdb", "query":"select time, service_name, \"name\", sys_fail_count, time_lt_10ms, time_lt_50ms, time_lt_200ms, time_lt_1000ms, time_gt_1000ms, total_count, total_time from service_stat where time >= 1605484740000000000 and time  < 1605484800000000000", "mode": "bulk", "batch.size": 10000, "timestamp.delay.interval.ms": 3600000, "value.converter": "org.apache.kafka.connect.json.JsonConverter", "value.converter.schemas.enable": "false"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"
    # influxdb usercenterdb.service_error_stat 数据
    echo '{"name": "InfluxDBSourceErrorConnector", "config": {"connector.class": "io.confluent.influxdb.source.InfluxdbSourceConnector", "tasks.max": "1", "topic.prefix": "influx_", "influxdb.url": "http://10.15.29.164:8086", "influxdb.db": "usercenterdb", "query":"select time, error_code, service_name, \"name\", total_count from service_error_stat where time > 1605024000000000000 fill(0)", "mode": "timestamp", "timestamp.delay.interval.ms": 3600000, "value.converter": "org.apache.kafka.connect.json.JsonConverter", "value.converter.schemas.enable": "false"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"
    # influxdb two_weeks_only.srv_rsp_code 数据
    echo '{"name": "InfluxDBSourceRespCodeConnector", "config": {"connector.class": "io.confluent.influxdb.source.InfluxdbSourceConnector", "tasks.max": "1", "topic.prefix": "influx_", "influxdb.url": "http://10.15.29.164:8086", "influxdb.db": "usercenterdb", "query":"select time, service_name, response_code, \"count\" as total_count from two_weeks_only.srv_rsp_code where time > 1605600600000000000 fill(0)", "mode": "timestamp", "timestamp.delay.interval.ms": 3600000, "value.converter": "org.apache.kafka.connect.json.JsonConverter", "value.converter.schemas.enable": "false"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"
    ```

5. kafka常用命令

    ```shell
    bin/kafka-console-consumer.sh --topic=influx-connect-offsets --from-beginning --bootstrap-server b-1.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-2.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-3.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092 --property print.key=true --property key.separator=--

    bin/kafka-console-consumer.sh --topic=influx_usercenterdb --from-beginning --bootstrap-server b-1.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-2.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-3.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092

    bin/kafka-console-consumer.sh --topic=influx_usercenterdb --offset latest --partition 0 --bootstrap-server b-1.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-2.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-3.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092

    bin/kafka-topics.sh --topic=influx_usercenterdb --describe --bootstrap-server b-1.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-2.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-3.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092


    influx_clickhouse
    influx_usercenterdb

    # 删除并清空topic
    tail

    bin/kafka-topics.sh --delete --topic influx_clickhouse --bootstrap-server b-1.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-2.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092,b-3.prod-log-cluster.1tt1gt.c4.kafka.ap-southeast-1.amazonaws.com:9092

    echo '{"name": "InfluxDBSourceStatConnector", "config": {"connector.class": "io.confluent.influxdb.source.InfluxdbSourceConnector", "tasks.max": "1", "poll.interval.ms": 5000, "topic.prefix": "influx_", "influxdb.url": "http://10.15.29.164:8086", "influxdb.db": "usercenterdb", "query":"select time, service_name, \"name\", sys_fail_count, time_lt_10ms, time_lt_50ms, time_lt_200ms, time_lt_1000ms, time_gt_1000ms, total_count, total_time from service_stat where time > 1605176100000000000 fill(0)", "mode": "timestamp", "timestamp.delay.interval.ms": 3600000, "value.converter": "org.apache.kafka.connect.json.JsonConverter", "value.converter.schemas.enable": "false"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"

    ExecStart=/home/kafka/bin/connect-standalone.sh /home/kafka/config/connect-standalone.properties /home/kafka/config/connect-influxdb-source-stat.properties /home/kafka/config/connect-influxdb-source-error.properties /home/kafka/config/connect-influxdb-source-resp.properties

    ExecStart=/home/kafka/bin/connect-standalone.sh /home/kafka/config/connect-standalone.properties /home/kafka/config/connect-influxdb-source-stat.properties /home/kafka/config/connect-influxdb-source-error.properties /home/kafka/config/connect-influxdb-source-resp.properties
    ```
