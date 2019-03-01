---
layout: post
title: Spring MVC + Redis实现激活码秒杀活动（4.性能优化极致的Version3）
categories: blog
date: 2018-12-16
tags: [技术类]
description: "网易实习的第二个项目，激活码秒杀活动后台记录"
header-img: "img/article3.jpg"
---

# Spring MVC + Redis实现激活码秒杀活动（4.性能优化极致的Version3）

![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_flowChat.png)

在这个版本中，进一步优化统计接口，将redis中的计数用新建异步线程完成，避免和主线程竞争redis资源；同时在前台加上定时器，方便时间验证直接从实例内存里获取。

#### 核心代码 SeckillService
基于访问的统计（一次请求参与人数+1）
```java
@Service
public class SeckillService {
    private static final ExecutorService executorService = new ThreadPoolExecutor(1, 1,
            60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
    private boolean isRemainCode = true;

    @Resource
    private CodeService codeService;
    @Resource
    private SeckillRecordDao seckillRecordDao;
    @Resource
    private LoadRoundService loadRoundService;
    @Resource
    private SeckillCacheDao seckillCacheDao;

    public void setIsRemainCode(boolean flag) {
        this.isRemainCode = flag;
    }

    /**
     * seckill
     */
    public SeckillResponse seckill(String urs) {
        SeckillResponse seckillResponse = new SeckillResponse();
        //Object startTime = commonRedisDao.get(SeckillConsts.CURRENT_ROUND);
        Object startTime = loadRoundService.getRound();
        if (startTime == null || !isLegalTime(startTime.toString())) {
            seckillResponse.setStatus(SeckillBusinessProtocol.TIME_ILLEGAL);
            return seckillResponse;
        }
        //先在本实例的内存中查重,未找到再去redis查重
        String ursInCache = seckillCacheDao.get(urs);
        if (!StringUtils.isEmpty(ursInCache)) {
            executorService.execute(
                    new SeckillRecordTask(ursInCache, Long.valueOf(startTime.toString()), seckillRecordDao));
            seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
            return seckillResponse;
        }
        //在redis中找
        String status = codeService.hget(SeckillConsts.WIN_CODE, urs);
        if (!StringUtils.isEmpty(status)) {
            executorService.execute(
                    new SeckillRecordTask(status, Long.valueOf(startTime.toString()), seckillRecordDao));
            seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
            return seckillResponse;
        }
        //根据身份抢码
        if (codeService.isWhiteUser(urs)) {
            executorService.execute(new SeckillRecordTask(SeckillConsts.WHITE_USER,
                    Long.valueOf(startTime.toString()), seckillRecordDao));
            return seckill(urs, SeckillConsts.WHITE_USER, seckillResponse);
        } else if (codeService.isBlackUser(urs)) {
            executorService.execute(new SeckillRecordTask(SeckillConsts.BLACK_USER,
                    Long.valueOf(startTime.toString()), seckillRecordDao));
            seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
            seckillResponse.setStatus(SeckillBusinessProtocol.RANK_BLACK_PARTICIPATED);
            return seckillResponse;
        } else {
            executorService.execute(new SeckillRecordTask(SeckillConsts.NORMAL_USER,
                    Long.valueOf(startTime.toString()), seckillRecordDao));
            return seckill(urs, SeckillConsts.NORMAL_USER, seckillResponse);
        }
    }

    /**
     * seckill
     */
    private SeckillResponse seckill(String urs, String userRank, SeckillResponse seckillResponse) {
        if (isRemainCode) {
            String code = codeService.getCode(SeckillConsts.CODE_LIST + userRank);
            if (code != null) {
                if (codeService.setCodeWinner(SeckillConsts.WIN_CODE + userRank, urs, code)) {
                    seckillCacheDao.put(urs, userRank);
                    seckillResponse.setStatus(SeckillBusinessProtocol.GET_CODE);
                    seckillResponse.setCode(code);
                    return seckillResponse;
                } else {
                    codeService.put(SeckillConsts.CODE_LIST + userRank, code);
                    isRemainCode = true;
                    seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
                    return seckillResponse;
                }
            }
        }
        isRemainCode = false;
        seckillResponse.setStatus(SeckillBusinessProtocol.FAIL_CODE);
        return seckillResponse;
    }
```

SeckillRecordTask :
```java
public class SeckillRecordTask implements Runnable {
    private String rank;
    private long currentRound;
    private SeckillRecordDao seckillRecordDao;

    /**
     * SeckillRecordTask construct
     */
    public SeckillRecordTask(String rank, long currentRound, SeckillRecordDao seckillRecordDao) {
        this.rank = rank;
        this.currentRound = currentRound;
        this.seckillRecordDao = seckillRecordDao;
    }

    @Override
    public void run() {
        seckillRecordDao.increaseParticipate(this.rank, this.currentRound);
    }
}
```
看一下具体的任务创建的方法，由于用了new每次新建一个任务，所以这里**不能再通过Spring注入的方式完成依赖的注入**，因为用new了之后就不再归Spring管理，组件涉及到的所有的注入都会失效（直接报NPE）,两种解决方法，1.直接把任务设定成单例，这样也可以避免每次都新创建一个任务，增加GC的工作；2.将要注入的Dao，直接作为形参传入，我用了第二种实现方式，其实感觉语义会有点奇怪。

上述实现的逻辑还是基于访问次数的统计，后将逻辑改成按账号统计，故不再需要线程池，核心代码如下：
```java
@Service
public class SeckillService {
    private boolean isRemainCode = true;

    @Resource
    private CodeService codeService;
    @Resource
    private LoadRoundService loadRoundService;
    @Resource
    private SeckillCacheDao seckillCacheDao;

    public void setIsRemainCode(boolean flag) {
        this.isRemainCode = flag;
    }

    /**
     * seckill
     */
    public SeckillResponse seckill(String urs) {
        SeckillResponse seckillResponse = new SeckillResponse();
        Object startTime = loadRoundService.getRound();
        if (startTime == null || !isLegalTime(startTime.toString())) {
            seckillResponse.setStatus(SeckillBusinessProtocol.TIME_ILLEGAL);
            return seckillResponse;
        }
        //先在本实例的内存中查重,未找到再去redis查重
        String ursInCache = seckillCacheDao.get(urs);
        if (!StringUtils.isEmpty(ursInCache)) {
            seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
            return seckillResponse;
        }
        //在redis中找
        String ursInRedis = codeService.getUrsInRedis(SeckillConsts.WIN_CODE, urs);
        if (!StringUtils.isEmpty(ursInRedis)) {
            seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
            return seckillResponse;
        }
        //根据身份抢码
        if (codeService.isBlackUser(urs)) {
            codeService.setUrsFailure(SeckillConsts.FAIL_CODE_BLACK_USER, urs);
            seckillResponse.setStatus(SeckillBusinessProtocol.RANK_BLACK_PARTICIPATED);
            return seckillResponse;
        } else if (codeService.isWhiteUser(urs)) {
            return seckill(urs, SeckillConsts.WHITE_USER, seckillResponse);
        } else {
            return seckill(urs, SeckillConsts.NORMAL_USER, seckillResponse);
        }
    }

    /**
     * seckill
     */
    private SeckillResponse seckill(String urs, String userRank, SeckillResponse seckillResponse) {
        if (isRemainCode) {
            String code = codeService.getCode(SeckillConsts.CODE_LIST + userRank);
            if (code != null) {
                if (codeService.setUrsSuccess(SeckillConsts.WIN_CODE + userRank, urs, code) > 0) {
                    seckillCacheDao.put(urs, userRank);
                    seckillResponse.setStatus(SeckillBusinessProtocol.GET_CODE);
                    seckillResponse.setCode(code);
                    return seckillResponse;
                } else {
                    codeService.put(SeckillConsts.CODE_LIST + userRank, code);
                    isRemainCode = true;
                    seckillResponse.setStatus(SeckillBusinessProtocol.DUPLICATED_CODE);
                    return seckillResponse;
                }
            }
        }
        isRemainCode = false;
        codeService.setUrsFailure(SeckillConsts.FAIL_CODE + userRank, urs);
        seckillResponse.setStatus(SeckillBusinessProtocol.FAIL_CODE);
        return seckillResponse;
    }
```
#### 后台定时器

这里换了种timer的实现，直接用Spring的timer：
```java
    @Scheduled(cron = "0/5 * * * * ?")
    public void initTimer() {
        try {
            if (!StringUtils.isEmpty(jedisDao.lock(
                    SeckillConsts.REDIS_PREFIX, SeckillConsts.LOCK, SeckillConsts.EXPIRE))) {
                nextRound = seckillConfigDao.getNextRound();
                boolean isBegin = isBeginTime();
                if (nextRound != null && isBegin) {
                    seckillRedisService.initRedis(nextRound);
                    currentRound = nextRound;
                } else if (!isBegin && currentRound != null && isEndTime()) {
                    persistService.persistToDb(currentRound);
                    currentRound = null;
                }
                jedisDao.unlock(SeckillConsts.REDIS_PREFIX);
            }
        } catch (RuntimeException e) {
            //
        }
    }
```
可以看见上面的代码比之前简介了很多，一是直接用Spring管理定时器，省去了我们自己管理的操作，二是将之前的两个标志位（isEnd 和isBegin，其实之前一直只需要一个标志就好了，二者是互斥的），通过redis加分布式锁的方式优化掉了，语义更简洁

#### 前台定时器
在这里最简单粗暴的操作就是直接复制后台的定时器，我比较作的又换了种方法实现：即通过后台跳转前台的url实现定时的操作

```java
    @Value(value = "${seckill.basePath}")
    private String basePath;
    @Value(value = "${seckill.url}")
    private String url;
    
    private void frontendFoward() { //这一步封装在定时器的initRedis中，也就是每场活动前都会调用
        for (int i = 0;i < REPEAT;i++) {
            String response = HttpClient4Utils.httpPost(basePath + url, null); //这里直接用了现成的工具类，变量从.properties读取
            if (StringUtils.isEmpty(response)) {
                LOGGER.error("Frontend forward failed, url: {}", basePath + url);
            }
        }
    }
```
用完发现其实还是直接用定时器更好，这样写有几个地方要考虑：

1. 分布式环境下，前台做了负载均衡则这边要慎重使用，因为url跳转一次代表一个请求发出，只能被一台前台 接收，固此处我重复将该请求发了REPEAT次，确保每台实例都收到请求（不推荐）
2. 需考虑网络阻塞导致失败的情况
3. 需在前台Controller中做好权限验证

#### 小结
总结一下做的几个优化：

1. 黑白用户名单可以事先导入内存，节省确认用户身份时间
2. 生成的激活码也可以在活动开始前生成并导入redis
3. 已抢码用户在每场活动结束后，同步至本实例内存中，以加快查重
4. 统计功能用异步线程实现，避免争抢redis资源
5. 新增isRemainCode标签，用于在激活码已抢完时直接返回
6. 前台获取时间直接从内存中读取（由前台定时器直接控制该变量的改变）
7. 更换redis的连接池配置（后来用了三种不同的连接池方式测试，见 [源码解析：探究JedisPool与CommonRedis的性能差异](https://mp.csdn.net/mdeditor/84939744#)）
8. redis优化，直接用原生jedis速度最快，关闭了testOnBorrow/testOnReturn
9. JVM调优：java -Xmx3550m -Xms3550m -Xmn2g *-Xss128k*（128K可能会报StackOverflow）

最坏情况下（用户抢码成功）：检查时间（内存级）+ 在内存中确认是否重复（内存）+ 内存中没有继续在redis中查重（redis）+ 确认用户身份（内存） + 检查是否有码（内存） + lpop激活码（redis） + 信息写回redis（写回redis） = 3次redis操作 + 若干内存级操作 + 0 次db操作

最坏情况下（用户抢码失败）：2次redis操作（即少了lpop激活码一次访问redis） + 若干内存级操作 + 0 次db操作

这个版本的QPS已成功突破3000,线上服务器的峰值QPS测试在3600左右，可以说性能已经达到了目前机器的上限
![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_QPS2.png)

其实还要优化的话，可以从批处理的角度去考虑，即一次导出100个码到内存中，成功/失败抢码的写回也是100个才写一次，这个做法牺牲了部分的可靠性但节省的是成百倍的中间件性能，根据需求考虑是否使用。
>这部分的思考见上一篇系列博文[《Spring MVC + Redis实现激活码秒杀活动（3.通过lpop操作实现并发控制的Version2）](http://zc-apiao.world/blog/2018/12/12/Spring-MVC-+-Redis%E5%AE%9E%E7%8E%B0%E6%BF%80%E6%B4%BB%E7%A0%81%E7%A7%92%E6%9D%80%E6%B4%BB%E5%8A%A8-3.%E9%80%9A%E8%BF%87lpop%E6%93%8D%E4%BD%9C%E5%AE%9E%E7%8E%B0%E5%B9%B6%E5%8F%91%E6%8E%A7%E5%88%B6%E7%9A%84Version2/)》




