---
layout:     post
title:      后端实现 IOS 和 Android 的消息推送
subtitle:   基于 Spring boot 框架实现 IOS 和 Android 的消息推送
date:       2017-10-11
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Spring boot
    - IOS
    - Android
    - 消息推送
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrew.405go.cn/)

# 前言
现在主流的手机系统被 ios 和 android 两大巨头占领，所以本文主要是实现 ios 和 android 系统的消息推送。而在中国， android 系统的消息推送主要分为华为推送和小米的推送。

当然，以上所说的，都是指免费的推送功能。如果你们的公司足够大，不差钱的话，当然是可以用更强大的推送插件的，比如极光推送等等。

# 前期准备

1. 搭建一个轻量级的 [Spring boot](https://www.tianmaying.com/tutorial/spring-boot-overview) 项目。

2. 搭建一个 nexus 私服。 nexus 已经有稳定的 docker 镜像，推荐使用 [nexus3](https://hub.docker.com/r/sonatype/nexus3/)，以及相应安装教程可参照[这篇文章](https://zhuanlan.zhihu.com/p/30188482?group_id=903588272652132352)。


# 小米推送
1. 注册小米的[开发者账号](https://dev.mi.com/console/)，并且记住 secretKey 和 packageName。

2. 下载小米推送的SDK，将MiPush_SDK_Sever_1_2.jar 和 json-simple-1.1.1.jar 推到nexus私服中。

3. 在pom文件中添加小米推送的依赖。

4. 推送代码

```
/**
 * 个推，根据regID，发送消息到指定设备上，不重试。
 *
 * @param regId 即手机设备号 deviceToken
 * @param title
 * @param content
 * @return
 */
public void sendMessage(String regId, String title, String content) {
    if(isProduction){
        Constants.useOfficial();
    }else{
        Constants.useSandbox();
    }

    Sender sender = new Sender(secretKey);
    Message message = new Message.Builder()
        .title(title)
        .description(content)
        .payload(content)
        .restrictedPackageName(packageName)
        .notifyType(1)
        .notifyId(generateShortUuid())
        .extra(Constants.EXTRA_PARAM_NOTIFY_EFFECT,Constants.NOTIFY_LAUNCHER_ACTIVITY)//点击通知直接打开app
        .build();

        try {
            Result result = sender.send(message, regId, 0); //根据regID，发送消息到指定设备上，不重试。
            logger.debug("[single_send_regId:" + regId + ",result" + result.getErrorCode().getDescription() + "]");
        } catch (IOException ex) {
            ex.printStackTrace();
        } catch (ParseException ex) {
            ex.printStackTrace();
        }
}

/**
 * 广播
 * @param title
 * @param content
 */
public void sendBroadcastAll(String title, String content) {
    if(isProduction){
        Constants.useOfficial();
    }else{
        Constants.useSandbox();
    }

    Sender sender = new Sender(secretKey);
    Message message = new Message.Builder()
        .title(title)
        .description(content)
        .payload(content)
        .restrictedPackageName(packageName)
        .notifyType(1)     // 使用默认提示音提示
        .notifyId(generateShortUuid())
        .extra(Constants.EXTRA_PARAM_NOTIFY_EFFECT, Constants.NOTIFY_LAUNCHER_ACTIVITY)//点击通知直接打开app
        .build();

    try {
            Result result = sender.broadcastAll(message, 0);
            logger.debug("[sendBroadcastAll_result" + result.getErrorCode().getDescription() + "]");
        } catch (IOException ex) {
            ex.printStackTrace();
        } catch (ParseException ex) {
            ex.printStackTrace();
        }
}
```

# 华为推送
1. 注册华为的[开发者账号](http://developer.huawei.com/consumer/cn/)，并且记住appId 和 appKey。

2. 下载华为推送的SDK，将client-adapter-sdk-java-oauth2-json-0.3.12.jar 推到nexus私服中。

3. 下载华为推送的密钥文件 mykeystore.jks 和 mykeystorebj.jks（在华为的SDK里面的resource目录下面，默认密码123456）。

4. 将两个密钥文件放到Spring boot的resources的huawei文件夹下面

5. 在pom文件中添加华为推送的依赖。

6. 推送代码

```
private NSPClient initOAuth2Client() {
        NSPClient client = null;
        try {
            OAuth2Client oauth2Client = new OAuth2Client();
            oauth2Client.initKeyStoreStream(
                HuaWeiPushAPI.class.getResource("/huawei/mykeystorebj.jks").openStream(), password);
            AccessToken accessToken =
                oauth2Client.getAccessToken("client_credentials", appId, appKey);
            client = new NSPClient(accessToken.getAccess_token());
            client.initHttpConnections(30, 50);//设置每个路由的连接数和最大连接数
            client.initKeyStoreStream(
                SomeClass.class.getResource("/huawei/mykeystorebj.jks").openStream(), "123456");
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return client;
    }

    /**
     * 通知栏消息接口.
     */
    public void sendMessage(Integer pushType, String devideTokens, String title, String content) {
        //必选 1：指定用户，必须指定tokens字段 2：所有人，无需指定tokens，tags，exclude_tags  3：一群人，必须指定tags或者exclude_tags字段
        if (pushType == 2) {
            devideTokens = "";
        }

        //构造请求
        HashMap<String, Object> hashMap = new HashMap<String, Object>();
        hashMap.put("push_type", pushType);
        hashMap.put("tokens", devideTokens);

        HuaweiPushInfoBean pushBean = new HuaweiPushInfoBean();
        pushBean.setNotificationTitle(title);
        pushBean.setNotificationContent(content);
        pushBean.setDoings(1);
        pushBean.setUrl("www.test.com");
        //消息内容，必选
        String android = JSON.toJSONString(pushBean);
        hashMap.put("android", android);
        long currentTime = System.currentTimeMillis();
        SimpleDateFormat dataFormat1 = new SimpleDateFormat("yyyy-MM-dd");
        SimpleDateFormat dataFormat2 = new SimpleDateFormat("HH:mm:ss");
        //消息发送时间，可选
        String day = dataFormat1.format(currentTime);
        String time = dataFormat2.format(currentTime);
        String sendTime = day + "T" + time + "+08:00";
        hashMap.put("send_time", sendTime);
        //消息过期时间，可选
        //timestamp格式ISO 8601：2013-06-03T17:30:08+08:00
        day = dataFormat1.format(currentTime + 12 * 60 * 60 * 1000);
        time = dataFormat2.format(currentTime + 12 * 60 * 60 * 1000);
        String expireTime = day + "T" + time + "+08:00";
        hashMap.put("expire_time", expireTime);

        //设置http超时时间
        NSPClient client = this.initOAuth2Client();
        client.setTimeout(10000, 15000);
        //接口调用
        String rsp = null;
        try {
            rsp = client.call("openpush.openapi.notification_send", hashMap, String.class);
            logger.debug("[pushType:" + pushType + ",tokens" + devideTokens + ",result:" + rsp + "]");
        } catch (NSPException ex) {
            ex.printStackTrace();
        }
    }
```

# 苹果推送
1. 获取推送的密钥，沙箱和生产环境的P12文件，可参照[这篇文章](http://ask.dcloud.net.cn/article/152)。

2. 将两个P12文件放到 Spring boot 的 resources 的 apple 文件夹下面。

3. 这里我们使用了 [notnoop](https://github.com/notnoop/java-apns) 的推送 SDK。

4. 在pom文件中添加notnoop依赖，版本号是1.0.0.Beta6。

5. 推送代码

```
private ApnsService initApnsService() {
        ApnsService service = null;
        String certName = isProduction ? "production_certificate.p12" : "sandbox_certificate.p12";
        try {
            service = APNS.newService()
                .withCert(SomeClass.class.getResource("/apple/" + certName).openStream(), password)
                .withAppleDestination(isProduction)
                .build();
        } catch (IOException ex) {
            ex.printStackTrace();
        }
        return service;
    }

    public void sendMessageByDeviceToken(String deviceToken, String title, String content) {
        ApnsService service = this.initApnsService();
        String payload = APNS.newPayload()
            .alertTitle(title)
            .alertBody(content)
            .build();
        service.push(deviceToken, payload);
    }

    public void sendMessageByDeviceTokens(Collection<String> deviceTokens, String title,
        String content) {
        ApnsService service = this.initApnsService();
        String payload = APNS.newPayload()
            .alertTitle(title)
            .alertBody(content)
            .build();
        service.push(deviceTokens, payload);
    }
```

# resources目录
![](http://oxy6ml8al.bkt.clouddn.com/push-resources.jpg)



# 如何将这三种推送串联起来？
Spring boot项目只提供两个接口，个推和群推：

```
@RequestMapping(value = "/sendSingleMessage", method = RequestMethod.POST)
@ResponseBody
public void sendSingleMessage(@RequestBody RequestBean requestBean) {
    pushAPI.sendSingleMessage(requestBean);
}

@RequestMapping(value = "/sendBroadcast", method = RequestMethod.POST)
@ResponseBody
public void sendBroadcast(@RequestBody RequestBean requestBean) {
    pushAPI.sendBroadcast(requestBean);
}
```

我们看一下RequestBean里面的属性：
```
private Long userId;
private String title;
private String content;
```

当用户登录系统后，生成标识用户的令牌userToken，将userId作为Key，userToken作为Value存到redis中。并且将userToken作为hashKey，用户登录时的设备号（deviceToken）和设备类型（deviceModel：IOS、HUAWEI、XIAOMI）作为hashValue也存到redis中。

此时，当一个推送请求过来时，我们可以通过userId，获取到userToken，即获取到了deviceToken和deviceModel。根据deviceModel调用不同的推送接口即可。

推送中间件的代码如下：

```
public void sendSingleMessage(RequestBean requestBean) {
        Long userId = requestBean.getUserId();
        if (userId == null) {
            return;
        }
        String userToken = redisUtilService.vGet(RedisKey.getAppUserIdToken(userId));
        if (StringUtils.isEmpty(userToken)) {
            return;
        }
        String result = redisUtilService.hGet(RedisKey.FAS_SESSION, userToken);
        if (!StringUtils.isEmpty(result)) {
            SessionBean sessionBean = JSON.parseObject(result, SessionBean.class);
            String deviceModel = sessionBean.getDeviceModel();
            String deviceToken = sessionBean.getDeviceToken();
            if (StringUtils.isEmpty(deviceModel) || StringUtils.isEmpty(deviceToken)) {
                return;
            }
            String title = requestBean.getTitle();
            String content = requestBean.getContent();
            if (DeviceModel.IOS.equalsIgnoreCase(deviceModel)) {
                iosPushAPI.sendMessageByDeviceToken(deviceToken, title, content);
            } else if (DeviceModel.HUAWEI.equalsIgnoreCase(deviceModel)) {
                huaWeiPushAPI.sendMessage(1, deviceToken, title, content);
            } else {
                xiaoMiPushAPI.sendMessage(deviceToken, title, content);
            }
        }

    }

    public void sendBroadcast(RequestBean requestBean) {
        String title = requestBean.getTitle();
        String content = requestBean.getContent();
        huaWeiPushAPI.sendMessage(2, null, title, content);

        xiaoMiPushAPI.sendBroadcastAll(title, content);

        Map<String, String> map = redisUtilService.hGetAll(RedisKey.FAS_SESSION);
        if (map != null) {
            Collection<String> deviceTokens = new HashSet<String>();
            for (String value : map.values()) {
                SessionBean sessionBean = JSON.parseObject(value, SessionBean.class);
                if (DeviceModel.IOS.equalsIgnoreCase(sessionBean.getDeviceModel())) {
                    deviceTokens.add(sessionBean.getDeviceToken());
                }
            }
            iosPushAPI.sendMessageByDeviceTokens(deviceTokens, title, content);
        }
    }
```

# 总结
1. 部署的时候，我们可以增加一个特定的环境变量，通过这个变量在不同环境（测试和生产）的值，来区分消息推送使用沙箱模式还是生产模式。

2. 华为和小米的推送有很多种方式，可以参照官方文档选择适合自己app场景的方式。

3. 在程序中，一定要做好日志的记录，当推送失败后，便于查找问题的根源。

4. Spring boot 的推送项目一定要做好安全策略。
    - Http header 中增加 Authorization 认证。
    - 用户 token 的生成使用 jsonwebtoken 包下面的 JwtBuilder。 Jwt 使用的是 HS256 的加密方式。
    - 用户密码的存储推荐使用 PBKDF2 的加密方式。

