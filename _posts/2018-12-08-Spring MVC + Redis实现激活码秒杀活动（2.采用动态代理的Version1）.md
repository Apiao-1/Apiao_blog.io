---
layout: post
title: Spring MVC + Redis实现激活码秒杀活动（2.采用动态代理的Version1）
categories: blog
date: 2018-12-08
tags: [技术类]
description: "网易实习的第二个项目，激活码秒杀活动后台记录"
header-img: "img/article3.jpg"
---

# Spring MVC + Redis实现激活码秒杀活动（2.采用动态代理的Version1）

#### 流程图：
![在这里插入图片描述](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/seckill/seckill_flowChat.png)

第一个版本的QPS非常低，大概只有几百，原因是加锁的粒度太大 ，对整个抢码逻辑进行了加锁，其次动态代理增加了复杂度，会消耗一部分性能（虽然内存级的影响很小）
此处动态代理实现参照了：https://blog.csdn.net/u010359884/article/details/50310387?utm_source=blogxgwz0 ,在此不过多赘述

## 核心代码(第一个版本很多不规范的写法，大家看看思想就好)

##### 抢码逻辑

```java
@Service 
public class SecKillImpl implements SeckillInterface {
    private Logger logger = Logger.getLogger(getClass().getName());
    @Resource
    private CommonRedisDao commonRedisDao;
    @Resource
    private SecondKillService secondKillService;
    @Resource
    private GenerateCode generateCode;

    @Override
    public String secKill(String username, String lockItem, int rank) {
        int leftCodeWhite = Integer.parseInt(commonRedisDao.get("leftCodeWhite").toString());
        int leftCodeNormal = Integer.parseInt(commonRedisDao.get("leftCodeNormal").toString());
        int outCode = Integer.parseInt(commonRedisDao.get("outCode").toString());
        if (secondKillService.checkCode(username) == null) {
            if (rank == 1 && leftCodeWhite > 0) { //白名单用户秒杀
                secondKillService.addSuccess(rank);
                //根据rank生成不同类型的激活码
                String code = generateCode.generateCode(20);
                if (code == null) {
                    logger.info("本场激活码已抢完，剩余激活码数： " + leftCodeWhite);
                    return null;
                }
                commonRedisDao.put(username, code);
                //long status = secondKillService.addAwardedObj(username, code, rank);
                //logger.info("是否已插入数据库id：" + status);
                leftCodeWhite -= 1;
                outCode += 1;
                commonRedisDao.put("leftCodeWhite", leftCodeWhite);
                commonRedisDao.put("outCode", outCode);
                logger.info("用户" + username + "抢码成功，激活码为:" + code + " 本场白名单剩余激活码数： " + leftCodeWhite);
                return code;
            } else if (rank == 0 && leftCodeNormal > 0) {
                secondKillService.addSuccess(rank);
                //根据rank生成不同类型的激活码
                String code = generateCode.generateCode(20);
                if (code == null) {
                    logger.info("本场激活码已抢完，剩余激活码数： " + leftCodeNormal);
                    return null;
                }
                commonRedisDao.put(username, code);
                //long status = secondKillService.addAwardedObj(username, code, rank);
                //logger.info("是否已插入数据库id：" + status);
                leftCodeNormal -= 1;
                outCode += 1;
                commonRedisDao.put("leftCodeNormal", leftCodeNormal);
                commonRedisDao.put("outCode", outCode);
                logger.info("用户" + username + "抢码成功，激活码为:" + code + " 本场普通名单码库剩余激活码数： " + leftCodeNormal);
                return code;
            } else {
                logger.info("本场激活码已抢完，剩余激活码数：0");
                return "Empty";
            }
        } else {
            logger.error("用户" + username + "已有激活码");
            return "Repeat";
        }
    }

}
```
其实抢码的逻辑很简单，请求进来后的处理逻辑就如流程图所画的，因为用了代理加锁的方式此处其实已经是串行处理了，代理的好处是可以统一在莫一类的执行前进行加锁操作，逻辑上解耦。

加锁代码（真见鬼了controller侵入了大量service代码）

```java
	@RequestMapping(value = "/secondKill", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    @ResponseBody
    public String secondKill(@RequestParam("urs") String username) {
        if (!secondKillService.checkTime()) {
            return BusinessResp.builder(39).setMsg("不在秒杀时间内").build();
        }
        int rank = 0;
        try {
            if (secondKillService.checkIdentity(username) < 0) { //黑名单用户
                secondKillService.addParticipate(-1);
                logger.error("黑名单用户不能秒杀激活码： " + username);
                return BusinessResp.builder(41).setMsg("黑名单用户不能秒杀激活码").build();
            } else {
                logger.info("白名单用户:  " + username);
                rank = 1;
                secondKillService.addParticipate(1); //白名单用户进行秒杀操作
            }
        } catch (EmptyResultDataAccessException e1) {
            logger.info("普通用户： " + username);
            secondKillService.addParticipate(0); //普通用户进行秒杀操作
        }
        try {
            //实现动态代理
            InvocationHandler handler = new CacheLockInterceptor(secKillImpl);
            SeckillInterface proxy = (SeckillInterface) Proxy.newProxyInstance(handler.getClass().getClassLoader(),
                    secKillImpl.getClass().getInterfaces(), handler);
            String code = proxy.secKill(username, "leftCode", rank);
            if (code.equals("Empty")) {
                return BusinessResp.builder(38).setMsg("本场激活码已发完").build();
            } else if (code.equals("Repeat")) {
                return BusinessResp.builder(40).setMsg("用户已有激活码").build();
            } else {
                return BusinessResp.builder(21).setMsg("秒杀成功").setPayload(code).build();
            }
        } catch (CacheLockException m) {
            logger.error(username + "秒杀失败请重试");
            return BusinessResp.builder(42).setMsg("秒杀失败请重试").build();
        }
    }
```

加锁（完整的流程见链接：https://blog.csdn.net/u010359884/article/details/50310387?utm_source=blogxgwz0）

```java
    @Override //获取代理对象注解的方法method和参数args
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        CacheLock cacheLock = method.getAnnotation(CacheLock.class);
        //没有cacheLock注解即不加锁，pass
        if (cacheLock == null) {
            System.out.println("No cacheLock annotation");
            return method.invoke(proxied, args);
        }
        //获得方法中参数的注解
        Annotation[][] annotations = method.getParameterAnnotations();
        //根据参数注解获得加锁的参数
        Object lockedObject = getLockedObject(annotations, args);
        String objectValue = lockedObject.toString();
        //对加锁的参数上锁
        RedisLock lock = new RedisLock(cacheLock.lockedPrefix(), objectValue);
        boolean result = lock.lock(cacheLock.timeOut(), cacheLock.expireTime());
        if (!result) { //取锁失败
            ERROR_COUNT += 1;
            throw new CacheLockException("get lock fail,秒杀失败请重试");
        }
        try {
            //执行方法
            return method.invoke(proxied, args);
        } finally {
            //System.out.println("Intecepor 释放锁");
            lock.unlock();//释放锁
        }
    }
```
其实这里用代理加锁也可以，只不过加锁的粒度可以更小，只需要锁目前已发放码的数量就可以了，不用把整个流程都锁上。可以用双重检查机制实现只锁发放码数量



#### 定时器

因为本次的抢码活动需要定时执行，要求在活动开始后不得重启服务器，所以我们需要在后台写一个定时器，去完成相关的初始化和持久化操作。
此处的实现用了原生的selvet，随着tomcat启动而启动，在spring初始化之前定时器就启动了所以直接注入会报NPE，只能直接从上下文里去拿要的变量
```java
public class ListenerService implements ServletContextListener {
    private Logger logger = Logger.getLogger(getClass().getName());
    private static Timer timer  =  new Timer(true);
    private InitRedis initRedis;
    private SecondKillConfigDao secondKillConfigDao;
    private UpToMysql upToMysql;

    @Override
    public void contextInitialized(ServletContextEvent  event)  {
        //在listener中注入不可以用注解模式,要采用以下方式
        final WebApplicationContext webApplicationContext =
                WebApplicationContextUtils.getWebApplicationContext(event.getServletContext());
        initRedis = webApplicationContext.getBean(InitRedis.class);
        secondKillConfigDao = webApplicationContext.getBean(SecondKillConfigDao.class);
        upToMysql = webApplicationContext.getBean(UpToMysql.class);
        event.getServletContext().log("定时器已启动");
        //得到执行任务的起始场次
        final Map<String, Object> configList = secondKillConfigDao.getConfigList().get(0);
        String beginDate = configList.get("beginDate").toString();
        int durableTime = Integer.parseInt(configList.get("durableSecond").toString()); //这里的date在数据库里还是以String去存储的，所以拿出来要进行转化，规范要用Long
        final String endDate = configList.get("endDate").toString();
        String []timeBegin = beginDate.split(" ");
        String []timeHms = timeBegin[1].split(":");
        //得到执行任务的起始时间
        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.HOUR_OF_DAY, Integer.parseInt(timeHms[0]));
        calendar.set(Calendar.MINUTE, Integer.parseInt(timeHms[1]));
        calendar.set(Calendar.SECOND, Integer.parseInt(timeHms[2]) + durableTime + 30);
        Date date = calendar.getTime();
        //如果第一次执行定时任务的时间小于当前的时间就加一天
        if (date.before(new Date())) {
            date = this.addDay(date, 1);
            event.getServletContext().log("第一场场次时间已过，从第二天开始，新的时间为：" + date);
        }
        initRedis.initRedisAtFirstStart();//在最开始先按数据库配置初始化一次redis，之后每次在秒杀活动结束2分钟之后运行
        event.getServletContext().log("第一场场次时间为：" + date + "，redis初始化已完成");
        TimedTask timedTask = new TimedTask(event.getServletContext(), event);
        timer.schedule(timedTask, date,12 * 60 * 60 * 1000);//指定第一次统计时间为第一场活动结束30秒后，之后每隔12小时统计一次
        event.getServletContext().log("已经添加任务调度表");
    }

    @Override
    public void contextDestroyed(ServletContextEvent event)  {
        timer.cancel();
        event.getServletContext().log("定时器销毁");
    }
```

### 不足之处：
##### 需求部分
- 此处config中只有一列数据，不能实现调节每场的配置，同时一列数据再去计算下一场会显得比较复杂，解决办法：直接换成多条配置即可，每场一条记录更灵活
- timer采用非常早期的写法，在Spring开始前完成，不推荐使用
- 最致命的问题：锁的粒度太大，qps低
- 统计出现争议，最后统一了应该按urs账号统计，重复抢码的不再计数
- 定时器未考虑分布式部署
##### 代码规范部分

- 考虑访问的速度（按速度排列）
1. 本机的内存
2. redis（网络io）
3. Mysql （网络io + 硬盘io）

- inputUserList.whiteUserList.contains(OBJ)，调用的是OBJ中的equals方法，不是whiteUserList中Object的equals方法
- 不同实例的同一个包内的变量是彼此独立的（即若后台实例初始化一变量，前台实例直接去读取该变量还是原来未初始化的结果）
- static final用来修饰成员变量和成员方法，可简单理解为“全局常量”！ 
对于变量，表示一旦给值就不可修改，并且通过类名可以访问。 
对于方法，表示不可覆盖，并且可以通过类名直接访问。声明为static的方法有以下几条限制： 1.它们仅能调用其他的static 方法。 2.它们只能访问static数据。 3.它们不能以任何方式引用this 或super
- Java变量的初始化顺序为：静态变量或静态语句块–>实例变量或初始化语句块–>构造方法–>@Autowired(注入)–>@PostConstruct
（ Constructor >> @Autowired >> @PostConstruct）
- 使用Redis记得加前缀，不一定只有一个人用，同时使用flushDB/flushAll的时候要考虑会不会把其他人的数据删了
- redis分布式锁1.setnx+expire防死锁（升级版：setnx+getset）；2.lpop原子操作
- @RestController = @Controller + @ResponseBody
- 全局变量是每个方法共同持有的，因此很容易产生线程问题，在多线程环境中建议多使用局部变量，局部变量每个线程都有自己的一份，不会产生线程问题
- 抢码的准确性：先写到数据库再返回给用户
- new 自己去实例化一个对象后，其中所有spring注入的标签都会失效（两者不能同时使用），错误示例：

<br>

