# MySQL 快速安装

要了解MySQL的各项特性以及各种问题，最好的办法，还是猜想->实验->验证

1. 安装docker及docker-compose, 请直接百度安装方法.
1. 建立docker-compose的脚本，命名为`dc-mysql-starter.yml`

    ```yml
    version: '3.7'

    services:
        mysqlstarter:
            image: 'mysql:5.7.29'
            ports:
                - '3800:3306'
            environment:
                - "MYSQL_ROOT_PASSWORD=root"
            volumes:
                - './config:/etc/mysql/conf.d/'
    ```

    其中:
    - 注意image中固定版本号，这是一个好习惯，防止以后mysql升级后，实现无法重复
    - 映射主机端口3800，防止与已安装的本地mysql默认端口3306的潜在冲突
    - 实验性质的密码直接与用户名相同，方便记忆
    - 同目录下建立config文件夹，里面放置数据库的配置文件`my.cnf`

        ```
        [mysqld]
        port = 3306
        datadir=/var/lib/mysql
        server-id=10001
        log-bin=mysql-bin
        relay-log=relay-log
        log-slave-updates=1
        gtid-mode=on
        enforce-gtid-consistency=on
        #slave-skip-errors=ddl_exist_errors
        slave-skip-errors=all
        binlog_format=row
        #binlog-ignore-db=test
        #binlog-ignore-db=information_schema
        #replicate-ignore-db=test
        #replicate-ignore-db=information_schema
        auto-increment-increment=2
        auto-increment-offset=1
        expire_logs_days=10
        #max_binlog_size = 100M
        explicit_defaults_for_timestamp=1
        ```

1. 启动环境 `docker-compose -f dc-mysql-starter.yml up`
