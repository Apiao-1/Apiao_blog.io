---
layout: post
title: Spring MVC + Redis实现激活码秒杀活动（3.通过lpop操作实现并发控制的Version2）
categories: blog
date: 2018-11-11
tags: [技术类]
description: "网易实习的第二个项目，激活码秒杀活动后台记录"
header-img: "img/article3.jpg"
---

# Spring MVC + Redis实现激活码秒杀活动（3.通过lpop操作实现并发控制的Version2）

![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_flowChat.png)

在这个版本中，并发控制直接通过redis的lpop操作去实现，具体如下：

#### 核心代码

```java
 public SeckillResponse seckill(String urs) {
        SeckillResponse seckillResponse = new SeckillResponse();
        if (!isLegalTime()) {
            seckillResponse.setStatus(TIME_ILLEGAL);
            return seckillResponse;
        }
        //先在本实例的内存中查重,未找到再去redis查重
        if (winUserMap != null) {
            String rank = winUserMap.get(urs);
            if (!StringUtils.isEmpty(rank)) {
                codeService.put((PARTICIPATE + rank), urs);
                seckillResponse.setStatus(DUPLICATED_CODE);
                return seckillResponse;
            }
        }
        //在redis中找
        if (codeService.isExists(WIN_CODE_NORMAL_USER, urs)) {
            codeService.put((PARTICIPATE_NORMAL_USER), urs);
            seckillResponse.setStatus(DUPLICATED_CODE);
            return seckillResponse;
        } else if (codeService.isExists(WIN_CODE_WHITE_USER, urs)) {
            seckillResponse.setStatus(DUPLICATED_CODE);
            return seckillResponse;
        }
        if (codeService.isWhiteUser(urs)) {
            codeService.put((PARTICIPATE_WHITE_USER), urs);
            return seckill(urs, WHITE_USER, seckillResponse);
        } else if (codeService.isBlackUser(urs)) {
            codeService.put((PARTICIPATE_BLACK_USER), urs);
            seckillResponse.setStatus(DUPLICATED_CODE);
            seckillResponse.setStatus(RANK_BLACK_PARTICIPATED);
            return seckillResponse;
        } else {
            codeService.put((PARTICIPATE_NORMAL_USER), urs);
            return seckill(urs, NORMAL_USER, seckillResponse);
        }
    }

    /**
     * seckill
     */
    private SeckillResponse seckill(String urs, String userRank, SeckillResponse seckillResponse) {
        String code = codeService.getCode(CODE_LIST + userRank);
        if (code != null) {
            if (codeService.setCodeWinner(WIN_CODE + userRank, urs, code) > 0) {
                winUserMap.put(urs, userRank);
                seckillResponse.setStatus(GET_CODE);
                seckillResponse.setCode(code);
                return seckillResponse;
            } else {
                codeService.put(CODE_LIST + userRank, code);
                seckillResponse.setStatus(DUPLICATED_CODE);
                return seckillResponse;
            }
        }
        seckillResponse.setStatus(FAIL_CODE);
        return seckillResponse;
    }

    /**
     * check current time and start time
     */
    private boolean isLegalTime() {
        String startTimeStr = jedisDao.get(CURRENT_FIELD);
        String durableSecondStr = jedisDao.get(DURABLE_TIME);
        if (StringUtils.isEmpty(startTimeStr) || StringUtils.isEmpty(durableSecondStr)) {
            return false;
        }
        long startTime = Long.valueOf(startTimeStr);
        long durableSecond =  Long.valueOf(durableSecondStr) * SECOND_TRANS;
        long currentTime = System.currentTimeMillis();
        return (startTime <= currentTime && currentTime <= startTime + durableSecond);
    }
```

可以看见较Version1，这里的优化包括：

1. 在内存中维护一个已抢码成功的用户map，减少了部分检查用户是否有码而访问redis的操作
2. 事先把黑白名单导入到内存，判断用户黑白名单也直接从内存读取
3. 事先生成激活码存在redis中，用户抢码时直接lpop获取（多用户并发控制），获取成功后用setnx存入（单用户并发控制），存入失败说明是单用户并发抢码，由于redis不支持事物的回滚，所以我们手动进行回滚，把lpop的激活码再用lpush放回队列

此时的流程图如下：
![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_flowChat2.png)
其中的getCode操作和setCodeWinner操作如下：

```java
    /**
     * get code according to userRank
     * @param rank userRank {@link SeckillSpecialUser#userRank}
     */
    public String getCode(String rank) {
        if (StringUtils.isEmpty(rank)) {
            return null;
        }
        return jedisDao.lpop(rank);
    }
    
    /**
     * setCodeWinner
     */
    public Long setCodeWinner(final String key, final String field, final String value) {
        if (StringUtils.isEmpty(key) || StringUtils.isEmpty(field) || StringUtils.isEmpty(value)) {
            return 0L;
        }
        return jedisDao.hsetnx(key, field, value);
    }
```
这个版本的QPS如下：
线上环境的机器峰值约在3600左右，当时前辈做的对照的QPS 20w抢20w在2600,20w抢2w在3000
![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_QPS.png)

其实这个版本还有可以优化的地方见下文，最后的性能超过了3000的qps

#### timer
这里的timer也改成了JDK的timer，再也不用直接去上下文拿实例了，直接注入依赖即可
isEnd 和isBegin是用来控制场次开始结束的两个标记
```java
    public void initTimer() {
        if (timer != null) {
            timer.cancel();
        }
        timer = new Timer();
        timer.schedule(new TimerTask() {
            public void run() {
                if (jedisDao.lock(REDIS_PREFIX, LOCK, EXPIRE) > 0) {
                    nextField = getNextField();
                    isEnd = Boolean.parseBoolean(jedisDao.get(REDIS_PREFIX + IS_END));
                    isBegin = Boolean.parseBoolean(jedisDao.get(REDIS_PREFIX + IS_BEGIN));
                    if (nextField == null) {
                        persistService.persistCodeOwner();
                        LOGGER.info("All fields has been finished, timer end");
                        isEnd = false;
                        isBegin = false;
                        timer.cancel();
                    }
                    if (isBegin && isBeginTime()) {
                        redisService.initRedis(nextField);
                        isEnd = true;
                        isBegin = false;
                    }
                    if (isEnd && isEndTime()) {
                        persistService.persistToDb(nextField);
                        nextField = getNextField();
                        isEnd = false;
                        isBegin = true;
                    }
                    redisService.updateFlag(String.valueOf(isBegin), String.valueOf(isEnd));
                }
            }
        }, DELAY, INTERVAL);//scan per 10 seconds
    }
```

注意Timer在执行定时任务时只会创建一个线程，所以如果存在多个任务，且任务时间过长，超过了两个任务的间隔时间，会发生一些缺陷

<br>

##### 这个版本在第二次代码review时还是被找出了不少问题，梳理如下：

1. redis应该在每场结束后就把抢到激活码的用户写回mysql，不应该最后再写。
> 这里的设计把redis当成了持久不应该最后再写。这里的设计把redis当成了持久层，所有的抢到码的用户都先存储在redis中，几天的活动结束后才写回redis不妥当，
2. 此处的单用户并发的setnx考虑是否可以换到前面，先setnx用户再抢码， 这样乍一看要两次操作redis，先写一次用户，再抢到码了写激活码，但实际上可以有两种方法去优化：
- 第二次写回的时候可以在内存攒到一定数量再统一写会redis，我当时问前辈这样写不是牺牲了可靠性吗，万一服务宕机重启，这部分未写入redis的抢码数据不就丢了吗？答：你可以事先从redis拿出一批码，比如100个，那最后写回也是每100个写一次，即使这部分信息丢了，你只需要找到这发出来的100个码将其标示为可用就可以了，其实哪个用户对哪个激活码不是必要的这样乍一看要两次操作redis，先写一次用户，再抢到码了写激活码，但实际上可以有两种方法去优化，1.第二次写回的时候可以在内存攒到一定数量再统一写会redis，我当时问前辈这样写不是牺牲了可靠性吗，万一服务宕机重启，这部分未写入redis的抢码数据不就丢了吗？答：你可以事先从redis拿出一批码，比如100个，那最后写回也是每100个写一次，即使这部分信息丢了，你只需要找到这发出来的100个码将其标示为可用就可以了，其实哪个用户对哪个激活码不是必要的
>事先拿100个码到每个实例的内存里，引申出的另一个弊端是由于加了负载均衡，用户可能访问这一台机器的时候告知抢码失败，已经没有码了，但他再抢一次可能又能抢到码（这时候请求被转发到了另一台有码的机器）  ————这一点弊端需要根据实际情况取舍

- 将setnx放置最前面检查用户是否重复抢码时就加上，加失败了就说明该用户已参与过抢码，弊端也很明显，这就限制了单场抢码的用户次数，即使抢失败了也不能再抢一次（和需求有出入）

3. redis加锁时setnx和expire要一起加，https://blog.csdn.net/d1562901685/article/details/54881862?utm_source=blogxgwz2 ，即这篇文章提到的问题，事实上不需要用setnx+getset，新的jedis中的set方法已支持同时设置和两个参数）
![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_RedisSet.png)
4. 加上即时显示码剩余数量的进度条，额外的需求，显得抢码活动更真实（所以程序猿还是要培养自己的产品意识，做的时候就可以考虑到这些可能要加的但没在交互稿体现出来的需求）
5. 用户查询激活码时，一定是从持久化层（数据库）给用户的，不能返回不可靠的未被持久化的数据
6. 题外话，git push时，确认自己的个人信息是否已设置，当时没注意，100+的commit的作者都是unknown，被骂惨
7. hashmap线程不安全，用CurrentHashMap
8. 内存中也会维护一份已抢码用户的信息，从分层的角度来讲这应该也要写在Dao层（MVC里只有Dao是有状态的，Service不应有状态，状态通俗来讲就是数据）
9. 在数据库记录抢码用户列表时，应记录批次和抢码的时间
10. sql拼写时，拼接语句前后的空格别忘了
11.  因为我额外加了conut的功能，就被Q了这个点，详见大表count的优化https://blog.csdn.net/vip_hitwu/article/details/77066688 ，一种优化方法是如文中提到的加辅助索引，第二种以空间换时间再维护一张统计信息的数据表，在空闲时进行统计，之后查询直接select操作即可
12. 数据库中该用unique的不能少
13. 如果判断条件有<和> ,那就要考虑=应该怎么办
14. **一定要考虑活动中服务器随时可能宕机重启，这时候不能影响其他正常运行的机器或者数据（比如启动自动清空数据是禁止的）**
15. 定时器要手动去catch 异常，否则出现异常后定时器会自动停止（一般只要catch runtime exception即可）
16. 如果是项目专用的redis操作，命名不能明很全局的类似redisService，建议前面加业务逻辑，如SeckillRedisService
17. 所有consts常量的引用要用类名+变量名去操作，不能直接import这个consts，可读性差，反例如
![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_WrongImport.png)

___

#### 2019.2.1更新
今天刚看完策略模式，如上的版本可以进一步用策略模式+工厂模式（反射）+枚举优化代替 if..else，更为优雅，此前大量的if...else 的弊端有以下：
1. 一旦分支多太多，逻辑复杂，会导致代码十分冗长，增加阅读难度
2. 违背了开闭原则。如果需要增加或减少分支，需要改动if…elseif，增大因代码改动而出错的风险
3. 具体算法执行策略与判断逻辑耦合

详情见：https://blog.csdn.net/u012557814/article/details/81671928
等年后把策略模式都学完了再把代码重构一把
