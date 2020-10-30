---
layout: post
title: Serverless订单应用部署
date: 2020-10-24
categories: blog
tags: []
description: Java是最好的语言。
---
## 前言
要完整代码twitter私信，没图，github太卡了，估计也加载不粗来。
## 简述
为响应国家“十四五”计划的环保计划，特地的研究了一下传说中的**无服务器部署(Serverless)**方案(省服务器😄)，于是便有了这次尝试。

## 语言选择
# JAVA天下第一
### c/c++/c#/node/python/go/php/vb这些也不错

## 框架选择
### JAVA的单体应用还能选什么呢？只能是Springboot啊

## 部署准备
注册个腾讯云账号
开通以下产品权限
- 云函数
- API网关
- 对象存储
   
财力允许的话还可以购买数据库服务（因为年少轻狂打折时我购买了这俩很长很长时间)
- mysql数据库
- redis数据库
![](http://freeeeeedom.github.io/img/img1.png)

## 部署方案

订单应用来说的话，必然是提供restful的接口，所以在统一VPC内采用了云函数+API网关的模式提供接口，于是就有了以下方案：
1. 应用主体部署在云函数
2. 使用API网关作为函数入口
3. 页面则是使用了对象存储部署
4. 数据库方面则使用了同一vpc下的云数据库(财力有限只尝试了mysql、redis，理论上其他应该都可行)

## 尝试部署
要让JAVA工程部署到云函数上，首先的了解什么是云函数(以下摘自微信开放文档)
```
云函数即在云端（服务器端）运行的函数。在物理设计上，一个云函数可由多个文件组成，占用一定量的 CPU 内存等计算资源；各云函数完全独立；可分别部署在不同的地区。开发者无需购买、搭建服务器，只需编写函数代码并部署到云端即可在小程序端调用，同时云函数之间也可互相调用。
```
查了一些资料后了解到，云函数其实就是将业务拆分成函数粒度又部署在云上，那么就写了个简单的demo部署到云函数上，并且配上了API网关尝试调用。
```java
 /**
 * 纯javascf快速开发部署(不走springboot)
 *
 * @author Freeeeeedom
 * @date 2020/10/24 10:31
 */
public class Scf {
    /**
     * log Object
     */
    private static Logger log = LoggerFactory.getLogger(Scf.class);
    private static DruidDataSource dataSource1 = new DruidDataSource();

    static {
        //此处加载或修改数据源 多数据源配置多个
        dataSource1.setUsername("Freeeeeedom");
        dataSource1.setUrl("jdbc:mysql://Freeeeeedom?autoReconnectForPools=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai");
        dataSource1.setPassword("Freeeeeedom");
        dataSource1.setMinIdle(1);
        dataSource1.setMaxActive(5);
        dataSource1.setMaxWait(10000);
        dataSource1.setValidationQuery("SELECT 1 from dual");
        log.info("数据源加载ok~");
    }

    /**
     * 纯scf入口参数
     *
     * @param insertParam 入参
     * @return java.lang.Object 执行结果
     * @author Freeeeeedom
     * @date 2020/10/24 10:31
     */
    public Object pure(Map<String, Object> insertParam) {
        log.info("param:{}", gson.toJson(insertParam);
        Gson gson = new GsonBuilder().disableHtmlEscaping().create();
        try {
            Class.forName("com.mysql.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            log.error("内部处理异常", e);
        }
        Map response = new HashMap();
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        jdbcTemplate.setDataSource(dataSource1);
        Map order = jdbcTemplate.queryForMap("select order_id,create_time from `order` limit 1");
        log.info(order.toString());
        return buildResponse(gson, gson.toJson(order), response);
    }

    private Object buildResponse(Gson gson, String json, Map response) {
        Map<String, String> headers = new HashMap(1);
        headers.put("Content-Type", "application/json");
        response.put("statusCode", HttpStatus.OK.value());
        response.put("headers", headers);
        response.put("body", json);
        return gson.toJson(response);
    }
}
```
只需要打包好代码，然后将入口函数设置为scf.Scf::pure就实现了接收数据，然后从数据库查询了第一个订单的id和创建时间并且返回的能力(代码写的烂别在意)。
![](http://freeeeeedom.github.io/img/scfrun.png)
每一次通过API网关触发云函数都会触发pure这个方法(调用者>调用API网关>云函数-->pure)，但经测试发现static的数据源初始化并不会被重复加载，这也奠定了springboot可部署基础。
其中通过log打印API网关带来的参数，直接将其复制为json，然后通过main函数模拟调用，这样就实现了本地模拟serverless部署后的调用。
```
log.info("param:{}", gson.toJson(insertParam);
```

有了这些基础，那么只需要有一个入口类模拟springboot启动的加载，然后再映射一下API网关过来入口参数，即可实现springboot再云函数上部署（其实就是上面SCF类的超级plus版本）。
```
代码还没清理干净有空再补
代码还没清理干净有空再补
代码还没清理干净有空再补
```
### Api网关配置
不知道咋描述直接上图吧，这里的路径参数对应springboot里的mapping路径
![](http://freeeeeedom.github.io/img/apiconfig.png)
![](http://freeeeeedom.github.io/img/apiconfigdetail.png)

## 细节
### 本地调试
有了上面那些demo后，可得知我们模拟云端部署运行已经不是问题。
那么怎么在本地调试呢？答案很简单，直接启动springboot然后调正常就完事了。
没错，就是直接用原生的springboot玩法即可。
把springboot部署到云函数其实就是**外挂了一个springboot的启动类**（设计模式上叫适配器模式？(+_+)?

### 功能
完整的springboot，能用springboot做的都能实现，我只是编写了一些小功能验证这个应用。
- [x] 与本地服务器数据库连接
- [x] 云数据库连接
- [x] vpc数据库连接
- [x] 外部接口调用(发短信验证码)
- [x] 实现简单的订单流(crud)
- [x] 实现简单的登录能力
- [x] 实现简单的数据验证能力

整个项目功能简单但代码却不少。

### 安全
首先 "serverless"、"腾讯"、"云服务" 这几个词就足以代表安全了，但为了功能完整性我还是尝试加了点东西。
在这个系统中，我选择了header中加签名的方式验证数据，原因是啥，操作简单，有效呗。
加密手段和方案暂且不说，就从流程上来看，是很方便的:
1. 从API网关调用参数中获取到header，body
2. 验证数据有效性
3. 请求转入业务模块
4. 验证数据有效性
5. 参数进入功能模块
6. 验证数据有效性
7. ………………
∞

其实只有123步骤是最有效的，后面的45678如果你想的话……
更不用说API网关本身提供的鉴权功能了。
![](http://freeeeeedom.github.io/img/apiconfigaccess.png)
### 性能
内存的话对于订单系统来说单次请求加上JVM也才300mb，而云函数单个函数执行内存能拉到3GB，哪怕有点量的分布式计算应该问题也不大。
![](http://freeeeeedom.github.io/img/3gb.png)
并发的话云函数上的预置并发上限200个，订单系统嘛，QPS1000?10000?100000? ezpz了，再怎么也比自家机柜服务器强几百几千个量级了。
![](http://freeeeeedom.github.io/img/predeploy.png)
内存算力不够服务器扩容？不存在的。

### 页面
生成个VUE项目，改改链接调调页面，然后上传到存储桶上，一键打开CDN ~(￣▽￣)~*完美

## 事后
察觉到了到了科技的进步，时代的发展，serverless的强大。

## 最后
代码写的烂（勿念）