
介绍两种读写分离方案
1. 基于`Spring`
2. 基于`MySQL`驱动

使用到的jar：

    mysql:mysql-connector-java:6.0.6
    org.springframework:spring-jdbc:4.3.12.RELEASE
    org.springframework:spring-context:4.3.12.RELEASE
    org.apache.tomcat:tomcat-jdbc:9.0.1
    junit:junit:4.12
    ch.qos.logback:logback-classic:1.2.3
    org.projectlombok:lombok:1.16.18
    
## 基于Spring
### AbstractRoutingDataSource
基于`Spring`的数据库读写分离，主要在`AbstractRoutingDataSource`类中实现。
涉及该类的以下属性：

    //以key-value的形式存放用于写和读的数据源
    private Map<Object, Object> targetDataSources;
    //默认数据源，master
	private Object defaultTargetDataSource;

    //调用afterPropertiesSet()方法根据targetDataSources和defaultTargetDataSource初始化这两个属性的值
	private Map<Object, DataSource> resolvedDataSources;
	private DataSource resolvedDefaultDataSource;


子类实现`protected abstract Object determineCurrentLookupKey();`方法来做数据源的切换。
### 示例程序
#### 数据源配置

    //将所使用数据源的key存储在线程本地变量中，默认使用master
    ThreadLocal<Object> threadLocal = ThreadLocal.withInitial(() -> "master");
    
    //实现protected Object determineCurrentLookupKey()方法
    AbstractRoutingDataSource dataSource = new AbstractRoutingDataSource() {
        @Override
        protected Object determineCurrentLookupKey() {
            Object o = threadLocal.get();
            log.info("db {} selected", o);
            return o;
        }
    
        //just for log
        //根据上面protected Object determineCurrentLookupKey()返回值决定使用那个数据源
        @Override
        protected javax.sql.DataSource determineTargetDataSource() {
            javax.sql.DataSource source = super.determineTargetDataSource();
            log.info("db {} selected.", source);
            return source;
        }
    };
    
    Map<Object, Object> targetDataSources = new HashMap<>();
    //配置master数据源
    {
        DataSource master = new DataSource();
        master.setUrl("jdbc:mysql://127.0.0.1:3306/dev?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&autoReconnect=true&useSSL=false");
        master.setUsername("root");
        master.setPassword("mysql");
        master.setInitialSize(20);
        master.setMaxActive(200);
        master.setMinIdle(20);
        master.setMaxIdle(20);
        master.setMaxWait(5000);
        master.setInitSQL("select 1");
        master.setValidationQuery("select 1");
        String name = Driver.class.getName();
        master.setDriverClassName(name);
    
        targetDataSources.put("master", master);
    
        dataSource.setDefaultTargetDataSource(master);
    }
    //配置slave数据源
    {
        DataSource slave = new DataSource();
        slave.setUrl("jdbc:mysql://127.0.0.1:3307/dev?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&autoReconnect=true&useSSL=false");
        slave.setUsername("root");
        slave.setPassword("mysql");
        slave.setInitialSize(20);
        slave.setMaxActive(200);
        slave.setMinIdle(20);
        slave.setMaxIdle(20);
        slave.setMaxWait(5000);
        slave.setInitSQL("select 1");
        slave.setValidationQuery("select 1");
        String name = Driver.class.getName();
        slave.setDriverClassName(name);
    
        targetDataSources.put("slave", slave);
    
        dataSource.setTargetDataSources(targetDataSources);
    }
    dataSource.afterPropertiesSet();

#### 读写分离

1. 读read，使用slave库
        
        threadLocal.set("slave");
        Connection connection = dataSource.getConnection();
        PreparedStatement preparedStatement = connection.prepareStatement("select * from test where id = ?;");
        preparedStatement.setInt(1, 1);
        boolean execute = preparedStatement.execute();
        ResultSet resultSet = preparedStatement.getResultSet();
        resultSet.first();
        ResultSetMetaData metaData = resultSet.getMetaData();
        int columnCount = metaData.getColumnCount();
        for (int i = 0; i < columnCount; i++) {
            String columnName = metaData.getColumnName(i + 1);
            int columnType = metaData.getColumnType(i + 1);
            Object object = resultSet.getObject(i + 1);
            log.info(String.format("column:%-15s, type:%-5s, value:%s", columnName, columnType, object));
        }
        resultSet.close();
        connection.close();
        threadLocal.remove();
2. 写write，使用master
   
        threadLocal.set("master");
        Connection connection = dataSource.getConnection();
        connection.setAutoCommit(false);
        PreparedStatement preparedStatement = connection.prepareStatement("INSERT INTO test(test) VALUES (?);");
        preparedStatement.setString(1, "test master");
        boolean execute = preparedStatement.execute();
        Statement statement = connection.createStatement();
        statement.execute("SELECT last_insert_id();");
        ResultSet resultSet = statement.getResultSet();
        resultSet.first();
        log.info(String.format("id:%-15s", resultSet.getInt(1)));
        connection.commit();
        connection.close();
        threadLocal.remove();

## MySQL驱动
[MySQL multi host connection 文档](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-multi-host-connections.html)

MySQL 6.0.6版本的驱动有两个：`com.mysql.jdbc.Driver`和`com.mysql.cj.jdbc.Driver`，
其共同父类为`com.mysql.cj.jdbc.NonRegisteringDriver`，建议使用`com.mysql.cj.jdbc.Driver`
(在5.1.42版本的驱动中，则需要使用`com.mysql.jdbc.ReplicationDriver`)，
主从读写分离是在该类的`public java.sql.Connection connect(String url, Properties info)`方法中实现的。
它根据配置的connection url解析出不同的连接类型，如下：
    
    //单机
    SINGLE_CONNECTION("jdbc:mysql:", HostsCardinality.SINGLE), //
    //failover协议，客户端故障转移，多机，首先连接primary,若与primary-host简历连接异常，将依次尝试与secondary-host建立连接直到成功。
    //读写都只发生在一台机器
    //url格式 jdbc:mysql://[primary-host]:[port],[secondary-host]:[port],.../[database]?[property=<value>]&[property=<value>]  
    FAILOVER_CONNECTION("jdbc:mysql:", HostsCardinality.MULTIPLE), //
    //基于FAILOVER，适用于master-master双向同步模式
    //url格式 jdbc:mysql:loadbalance://[host]:[port],[host]:[port],...[/database]?[property=<value>]&[property=<value>]  
    LOADBALANCE_CONNECTION("jdbc:mysql:loadbalance:", HostsCardinality.ONE_OR_MORE), //
    //基于FAILOVER与LOADBALANCE，适用于mastet-slave主从复制模式
    //url格式 jdbc:mysql:replication://[master-host]:[port],[slave-host]:[port],.../database?[property=<value>]  
    REPLICATION_CONNECTION("jdbc:mysql:replication:", HostsCardinality.ONE_OR_MORE), //
    //test
    XDEVAPI_SESSION("mysqlx:", HostsCardinality.ONE_OR_MORE);
    
然后，根据不同的连接类型返回不同的连接类型：

    com.mysql.cj.api.jdbc.JdbcConnection
    com.mysql.cj.api.jdbc.ha.LoadBalancedConnection
    com.mysql.cj.api.jdbc.ha.ReplicationConnection
    
### LoadBalancedConnection
`LoadBalancedConnection`是一个逻辑连接(`LoadBalancedConnectionProxy`)，其内部持有一个`Map<String, ConnectionImpl> liveConnections`属性用于存放到每一个主机的物理连接。
MySQL提供的负载策略有3种，分别为
1. com.mysql.cj.jdbc.ha.RandomBalanceStrategy
2. com.mysql.cj.jdbc.ha.BestResponseTimeBalanceStrategy
3. com.mysql.cj.jdbc.ha.SequentialBalanceStrategy

默认会使用第一种，即random，策略选择源码如下：(`见public LoadBalancedConnectionProxy(LoadbalanceConnectionUrl connectionUrl)`)

    String strategy = props.getProperty(PropertyDefinitions.PNAME_loadBalanceStrategy, "random");
    try {
        switch (strategy) {
            case "random":
                this.balancer = new RandomBalanceStrategy();
                break;
            case "bestResponseTime":
                this.balancer = new BestResponseTimeBalanceStrategy();
                break;
            default:
                this.balancer = (BalanceStrategy) Class.forName(strategy).newInstance();
        }
    } catch (Throwable t) {
        throw SQLError.createSQLException(Messages.getString("InvalidLoadBalanceStrategy", new Object[] { strategy }), SQLError.SQL_STATE_ILLEGAL_ARGUMENT,
                t, null);
    }

### ReplicationConnection
`slaves`的负载均衡与`LoadBalancedConnection`一致，由`public synchronized void setReadOnly(boolean readOnly)`方法来做`master-slave`的切换，
具体方法实现看`com.mysql.cj.jdbc.ha.ReplicationMySQLConnection.setReadOnly(boolean readOnlyFlag)`。
### 示例程序
#### 数据源配置

    DataSource dataSource = new DataSource();
    dataSource.setUrl("jdbc:mysql:replication://127.0.0.1:3306,127.0.0.1:3307/dev?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&autoReconnect=true&useSSL=false");
    dataSource.setUsername("root");
    dataSource.setPassword("mysql");
    dataSource.setInitialSize(20);
    dataSource.setMaxActive(200);
    dataSource.setMinIdle(20);
    dataSource.setMaxIdle(20);
    dataSource.setMaxWait(5000);
    dataSource.setInitSQL("select 1");
    dataSource.setValidationQuery("select 1");
    String name = Driver.class.getName();
    dataSource.setDriverClassName(name);
    
#### 读写分离

1. 读read，从库

        Connection connection = dataSource.getConnection();
        connection.setReadOnly(true);//设置连接只读true，切换到slave
        PreparedStatement preparedStatement = connection.prepareStatement("select * from test where id = ?;");
        preparedStatement.setInt(1, 1);
        boolean execute = preparedStatement.execute();
        ResultSet resultSet = preparedStatement.getResultSet();
        resultSet.first();
        ResultSetMetaData metaData = resultSet.getMetaData();
        int columnCount = metaData.getColumnCount();
        for (int i = 0; i < columnCount; i++) {
            String columnName = metaData.getColumnName(i + 1);
            int columnType = metaData.getColumnType(i + 1);
            Object object = resultSet.getObject(i + 1);
            log.info(String.format("column:%-15s, type:%-5s, value:%s", columnName, columnType, object));
        }
        resultSet.close();
        connection.close();
2. 读read，主库

        Connection connection = dataSource.getConnection();
        connection.setReadOnly(false);//设置连接只读为false，切换到master
        PreparedStatement preparedStatement = connection.prepareStatement("select * from test where id = ?;");
        preparedStatement.setInt(1, 1);
        boolean execute = preparedStatement.execute();
        ResultSet resultSet = preparedStatement.getResultSet();
        resultSet.first();
        ResultSetMetaData metaData = resultSet.getMetaData();
        int columnCount = metaData.getColumnCount();
        for (int i = 0; i < columnCount; i++) {
            String columnName = metaData.getColumnName(i + 1);
            int columnType = metaData.getColumnType(i + 1);
            Object object = resultSet.getObject(i + 1);
            log.info(String.format("column:%-15s, type:%-5s, value:%s", columnName, columnType, object));
        }
        resultSet.close();
        connection.close();
3. 写write，主库

        Connection connection = dataSource.getConnection();
        connection.setReadOnly(false);//设置连接只读为false，切换到master
        connection.setAutoCommit(false);
        PreparedStatement preparedStatement = connection.prepareStatement("INSERT INTO test(test) VALUES (?);");
        preparedStatement.setString(1, "test master");
        boolean execute = preparedStatement.execute();
        Statement statement = connection.createStatement();
        statement.execute("SELECT last_insert_id();");
        ResultSet resultSet = statement.getResultSet();
        resultSet.first();
        log.info(String.format("id:%-15s", resultSet.getInt(1)));
        connection.commit();
        connection.close();
        
## 结束

两种实现的对比：
1. Spring方式可扩展，适用于多种数据库，而MySQL驱动方式只适用于MySQL
2. MySQL驱动方式必须主从库用户名、密码一致，而Spring更灵活
3. Spring需要更多的配置，而MySQL驱动方式配置简单

## 主从搭建
### MySQL安装

        brew install mysql;
        
### 编辑配置文件

* master my.cnf

        [mysqld]
        bind-address = 127.0.0.1
        port = 3306 
        basedir = /usr/local/opt/mysql
        datadir = /usr/local/var/mysql
        socket = /tmp/mysql.sock3306
        server-id=1	#服务器的标识  要和slave不一样
        log-bin=mysql-bin	#二进制文件存放路径 (说明开启了二进制日志,关键点)
* slave my.cnf

        [mysqld]
        bind-address = 127.0.0.1
        port = 3307 
        basedir = /usr/local/opt/mysql
        datadir = /usr/local/var/mysql3307
        socket = /tmp/mysql.sock3307
        server-id=2		 #服务器的标识  要和master不一样
        binlog-do-db=dev	#是你需要复制的数据库名称，如果有多个就用逗号“,”分开;或者分行写

### 初始化数据库
 
        mysqld --defaults-file=/path/to/my.cnf  —initialize #这个会在控制台打印出生成的随机密码，切记要记下来 
        mysqld --defaults-file=/path/to/my.cnf —initialize-insecure #无密码
  
### 配置主从同步

* 登录master

        create user 'slave'@'127.0.0.1' identified by 'slave';
        GRANT REPLICATION SLAVE ON *.* TO 'slave'@'127.0.0.1’;
        show master status;
* 登录slave

        stop slave;#停止同步
        reset slave;
        change master to 
        master_host='127.0.0.1',
        master_user='slave',#主库上新建的用户名
        master_password='slave',##对应的密码
        master_log_file='mysql-bin.000005',#主库show master status结果中复制
        master_log_pos=1483;#主库show master status结果中复制
        start slave user='slave' password='slave' ;#开始同步
        show slave status;
* 完成
