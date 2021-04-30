---
layout: post
title: mybatisPlus使用问题定位
date: 2021-04-30
categories: blog
tags: []
description: Java是最好的语言。
---
## 简述
有个项目按照之前[Serverless订单应用部署](https://freeeeeedom.github.io/blog/2020/10/24/serverless-order/ "Serverless订单应用部署")的方式进行的服务部署，测试阶段并未发生问题，但是到了线上后发现会产生数据库连接超时，本以为是个既然报错了就很好改了，没想到这只是我的一厢情愿……

## 错误
通过云函数日志，很快，很快啊，就找到了错误，具体错误如下:
```
Caused by: com.mysql.cj.exceptions.CJCommunicationsException: The last packet successfully received from the server was 36,630,008 milliseconds ago. The last packet sent successfully to the server was 36,630,011 milliseconds ago. is longer than the server configured value of 'wait_timeout'. You should consider either expiring and/or testing connection validity before use in your application, increasing the server configured values for client timeouts, or using the Connector/J connection property 'autoReconnect=true' to avoid this problem.
```
从上面错误很容易看到原因，似乎是连接等待时长过长了，比数据库配置的wait_timeout这个时间长，并且也提供了解决方法：在数据库连接后边加上参数autoReconnect=true。

## 第一次尝试
看起来这个问题挺简单的，直接配个连接参数就完事了嘛，于是我下载了源码，给数据库连接加上了一堆配置
```
db_url=jdbc:mysql://ip:port/vwork?autoReconnect=true&autoReconnectForPools=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
```
顺便把连接改为了环境变量配置，然后进行了部署，一切行云流水，没有丝毫停顿。
自信如我部署后直接都没看就去干别的了，三四个小时侯回来再看日志时候，发现了意思不对劲。

## 错误依旧
查看了失败日志，发现还是有错误。
```
Caused by: com.mysql.cj.exceptions.CJCommunicationsException: The last packet successfully received from the server was 36,630,008 milliseconds ago. The last packet sent successfully to the server was 29,245,233 milliseconds ago. is longer than the server configured value of 'wait_timeout'. You should consider either expiring and/or testing connection validity before use in your application, increasing the server configured values for client timeouts, or using the Connector/J connection property 'autoReconnect=true' to avoid this problem.
```
除了时间数字不一样，其他都是一毛一样。没道理啊，上面说这么改也没问题啊，后续我加上了日志，再次确认函数中的数据库连接已经是读取的环境变量并且也加上了参数，此时的我感觉到了丝丝寒意。
此时只能开始google了起来，看看其他人是怎么碰上这问题的，一查发现原因五花八门，不过大都是因为参数没配，或者说druid的池管理用的不太对劲，于是我也只能从这方面下手。

## 代码审查
首先排除了参数没配置的情况，那么只能是池用的有毛病，那么先看看池怎么创建的。
```
    public static final int MAX_ACTIVE_SIZE = 10;
    public static DruidDataSource dataSourcePool = new DruidDataSource();
    private static void initialDruidDataSource(String url, String user, String pwd) {
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        dataSourcePool.setUrl(url);
        dataSourcePool.setUsername(user);
        dataSourcePool.setPassword(pwd);
        dataSourcePool.setMaxActive(MAX_ACTIVE_SIZE);
        dataSourcePool.setInitialSize(1);
        dataSourcePool.setMinIdle(1);
        dataSourcePool.setMaxWait(10000);
        dataSourcePool.setTimeBetweenEvictionRunsMillis(60000);
        dataSourcePool.setMinEvictableIdleTimeMillis(30000);
        dataSourcePool.setMaxEvictableIdleTimeMillis(180000);
        dataSourcePool.setPhyTimeoutMillis(15000);
        dataSourcePool.setValidationQuery("select 1");
        dataSourcePool.setTestWhileIdle(true);
        dataSourcePool.setTestOnBorrow(false);
        dataSourcePool.setTestOnReturn(false);
        dataSourcePool.setPoolPreparedStatements(true);
        dataSourcePool.setMaxOpenPreparedStatements(20);
        dataSourcePool.setUseGlobalDataSourceStat(true);
        dataSourcePool.setKeepAlive(true);
        dataSourcePool.setRemoveAbandoned(true);
        dataSourcePool.setRemoveAbandonedTimeout(180);
        System.out.println("initialDruidDataSource end....");
    }
```
除了参数多了点，似乎没啥不对劲，继续往下看。
```
    public static SqlSessionFactory factory;
    public static SqlSession session;
	private static void initialMybatisPlus() {
        System.out.println("initialMybatisPlus start....");
        TransactionFactory transactionFactory = new JdbcTransactionFactory();
        Environment environment = new Environment("prod", transactionFactory, dataSourcePool);
        MybatisConfiguration config = new MybatisConfiguration();
        config.setEnvironment(environment);
        config.addMappers(Mapper1.class);
        config.addMappers(Mapper2.class);
        config.setMapUnderscoreToCamelCase(true);
        factory = new MybatisSqlSessionFactoryBuilder().build(config);
        System.out.println("initialMybatisPlus end....");
        session = factory.openSession(true);
    }

    //在service中重载getBaseMapper方法
    @Override
    public Mapper1 getBaseMapper() {
        return ScfRouterSingle.session.getMapper(Mapper1.class);
    }
```
此时就发现不对劲了， 这个SqlSession看起来，似乎，有些，少？(只有一个session)
并且session似乎没有close?
这么算起来一个session一直跑一直跑，估计支持那么久也该超时了，而且那些druid的调优参数都没啥用了。

## 改错
发现问题后，那就好修改了，只有一个session那就改成每次创建新的session，而且不确定云函数情况下的托管好不好使，直接自己写个list对session进行管控。
修改后如下：
```
    public static final int MAX_ACTIVE_SIZE = 10;
    public static DruidDataSource dataSourcePool = new DruidDataSource();
    public static SqlSessionFactory factory;
    public static volatile List<SqlSession> sessionList = Collections.synchronizedList(new ArrayList<>(MAX_ACTIVE_SIZE * 2));
    private static SqlSessionFactory initialMybatisPlus(DruidDataSource dataSourcePool) {
        System.out.println("initialMybatisPlus start....");
        TransactionFactory transactionFactory = new JdbcTransactionFactory();
        Environment environment = new Environment("prod", transactionFactory, dataSourcePool);
        MybatisConfiguration config = new MybatisConfiguration();
        config.setEnvironment(environment);
        config.addMapper(Mapper1.class);
        config.addMapper(Mapper2.class);
        config.setMapUnderscoreToCamelCase(true);
        System.out.println("initialMybatisPlus end....");
        return new MybatisSqlSessionFactoryBuilder().build(config);
    }

    public static SqlSession getSession() {
        SqlSession sqlSession = SqlSessionManager.newInstance(factory);
        sessionList.add(sqlSession);
        return sqlSession;
    }

    private static void releaseSession() {
        for (SqlSession sqlSession : sessionList) {
            try {
                sqlSession.close();
                System.out.println("释放之前的SqlSession");
            } catch (SqlSessionException e) {
                System.out.println("不需要释放");
            }
        }
        sessionList.clear();
        System.out.println("============================SqlSession已释放,sessionListSize=" + sessionList.size());
    }

    public Object routePath(Map<String, Object> param) throws Exception {
        try {
             ……
             ……
        } finally {
            releaseSession();
        }
    }
    //在service中重载getBaseMapper方法
    @Override
    public Mapper1 getBaseMapper() {
        return ScfRouterSingle.getSession().getMapper(Mapper1.class);
    }
```

这样一来，每一次service要执行sql都会调用getSession()获取一个新的session，并且每次函数执行完后还会调用releaseSession()进行session统一释放，虽然session的复用几乎没有，但是这样一来不用再担心那个超时报错了。

## 第二次尝试
再次打包部署，行云流水，一气呵成。</br>
打开监控，等待，</br>
十分钟，刷新，</br>
二十分钟，刷新，</br>
三十分钟，刷新，</br>
……</br>
等待一小时后，再也没有出现过这个错，纠错成功！</br>

## 总结
此次问题，主要是因为开发者对mybatis和sqlsession不够了解导致，所有数据库请求都用了同一个session，然后就超时了，并且也巧妙的避过了druid的调优。

算不得大问题，但是如果任其下去，问题也不小，毕竟没有人愿意改持续了几个月小概率调用失败的问题。

在修改过程中因为不够细致，也导致了多花了改了小半天盯日志(摸鱼摸了小半天)

不过值得庆幸的是这个问题发现比较早，在上线初期发现这个问题，并且进行了修改。

仅写日志记录此次问题修改。