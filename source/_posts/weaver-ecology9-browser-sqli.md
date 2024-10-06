---
created: 2024-10-01T07:31:00+00:00
categories:
  - 技术研究
tags:
  - 漏洞分析
updated: 2023-03-04T00:00:00+00:00
date: 2023-03-03T00:00:00+00:00
slug: weaver-ecology9-browser-sqli
title: 泛微e-cology9 browser.jsp SQL注入漏洞分析
cover: /img/post/weaver-ecology9-browser-sqli/20230305235041800.png
id: 112906e1-7468-804b-9c44-e84c7588eb2e
---

## 引子

2023 年 02 月 23 日，微步发布了一个关于泛微 e-cology9 SQL 注入的漏洞通告。如下图所示，根据其说明，受影响的版本范围是<=10.55 版本。另外，他们还提到该漏洞无权限要求，并不是后台洞。

![](/img/post/weaver-ecology9-browser-sqli/20230225181503065.png)

## 补丁包对比

E-COLOGY 安全补丁下载网址如下：

https://www.weaver.com.cn/cs/securityDownload.html?src=cn

通过如下两个链接，下载该次漏洞的以及上一个版本的补丁包：

v10.55：<https://www.weaver.com.cn/cs/package/Ecology_security_20221014_v10.55.zip>

v10.56：<https://www.weaver.com.cn/cs/package/Ecology_security_20230213_v10.56.zip>

将两个补丁压缩包分别解压，然后使用 IDEA 工具对比差异。

![](/img/post/weaver-ecology9-browser-sqli/20230225204339203.png)

这里对比看了很久，但是却没有看出有价值的内容。

嗯？先了解下`web.xml`文件中的内容。

![](/img/post/weaver-ecology9-browser-sqli/20230301154357757.png)

在开头存在一个`SecurityFilter`的过滤器，`SecurityFilter`在初始化时会调用`weaver.security.filter.SecurityMain`中的`initFilterBean`方法初始化安全规则。而在`weaver.security.rules.ruleImp`包中的每个类差不多就是每次打的补丁，此包中的类将被重点关注。

所以，缩小范围去补丁包的`WEB-INF/myclasses/weaver/security/rules/ruleImp`目录寻找。并且我们还可以根据文件时间戳，将 2022 年 10 月前的补丁文件都排除在外，继续过滤一遍。

```shell
security/rules/ruleImp» stat -f %SB----%N *.class | grep -v "2021----" | grep -v -E '^[Jan|Mar|Apr|May|Jul|Aug|Sep].*2022' | wc -l
     125
```

这样还剩下 125 个补丁文件，然后通过[jadx](https://github.com/skylot/jadx)这款反编译工具一次性打开这些补丁文件，然后搜索`Xss(Validate failed`关键词，不断寻找，最终找到了一段 SQL 注入漏洞的补丁代码，下图所框疑似就是本次漏洞的位置。

![](/img/post/weaver-ecology9-browser-sqli/20230301163444206.png)

上图所示的补丁代码，所处如下位置。

```shell
$ Ecology_security_20230213_v10.56 » fd SecurityRuleForMobileBrowser
WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleForMobileBrowser.class
```

然而在 v10.55 补丁包中也同样发现该文件所在，并且是一摸一样的。

```shell
$ Ecology_security_20221014_v10.55 » fd SecurityRuleForMobileBrowser
WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleForMobileBrowser.class
```

```shell
$ patch » md5 Ecology_security_20230213_v10.56/WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleForMobileBrowser.class Ecology_security_20221014_v10.55/WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleForMobileBrowser.class | awk -F "=" '{print $2}'
 391d53bb28cffa1bf6974e21eac16b7d
 391d53bb28cffa1bf6974e21eac16b7d
```

查看这两个文件的时间戳，发现最后修改时间也是一致，都是 2022 年 12 月 8 日。

```shell
$ patch » stat -f %SB Ecology_security_20230213_v10.56/WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleForMobileBrowser.class Ecology_security_20221014_v10.55/WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleForMobileBrowser.class
Dec  8 19:55:26 2022
Dec  8 19:55:26 2022
```

但是在下载 v10.55 补丁包的时候，通过其下载链接，可以得知此版本补丁包的发布日期是 2022 年 10 月 14 日。是早于其中的`SecurityRuleForMobileBrowser.class`文件的最后修改时间的。

```url
https://www.weaver.com.cn/cs/package/Ecology_security_20221014_v10.55.zip
```

在通过微步关于这个漏洞的通告中的时间线中，大致可以猜测出来，他们在 2022 年 9 月，正值 HW 期间收到该漏洞，在 2022 月 12 月上报给监管单位，监管单位应该是收到了该漏洞就立马通知给厂商，在 2022 年 12 月 8 日厂商就已经开发出本次 SQL 注入漏洞的补丁代码，也就是`SecurityRuleForMobileBrowser.class`文件中的内容。但厂商在开发出该漏洞的补丁代码后并未立即发布 v10.56 补丁包，而是先将其更新至 v10.55 补丁包中了。

![](/img/post/weaver-ecology9-browser-sqli/20230225210416116.png)

这也就是为什么用 IDEA 对比 v10.55 和 v10.56 补丁包，却一无所获的原因。

那么既然如此，拿 v10.54 补丁包与 v10.56 补丁包对比呢？v10.54 补丁包下载地址如下：

```
https://www.weaver.com.cn/cs/package/Ecology_security_20220805_v10.54.zip
```

如下图所示，确实对比出了该文件存在于 V10.56 中，而不存在于 V10.54 中。

![](/img/post/weaver-ecology9-browser-sqli/20230225212954101.png)

## 确定漏洞位置

虽然通过如上的补丁包对比分析找到了一个疑似的漏洞路径，但是未必就能肯定这是真正的漏洞位置。

我们先来简单看看`/mobile/plugin/browser.jsp`的内容。

![](/img/post/weaver-ecology9-browser-sqli/20230225213535824.png)

参数很多，继续往下看，发现一个`isDis`参数及其判断语句。

```java
boolean isDis = "1".equals(Util.null2String(request.getParameter("isDis"))) ? true : false;
if (!isDis) {
	request.getRequestDispatcher("/mobile/plugin/dialog.jsp").forward(request, response);
	return;
}
```

如果`isDis`参数值不为`1`的话，则进入 if 条件语句之中，`RequestDispatcher`的作用是将请求分配给另一个资源，后面使用的是`forward`方法，此处就是将请求转发到`/mobile/plugin/dialog.jsp`处理。那么就看看`/mobile/plugin/dialog.jsp`的内容。

![](/img/post/weaver-ecology9-browser-sqli/20230225214851650.png)

发现开头的`HrmUserVarify.getUser`，该方法部分代码如下：

![](/img/post/weaver-ecology9-browser-sqli/20230306094845347.png)

可以看出此处有个登录判断，根据未登录的情况，肯定会返回`null`到`dialog.jsp`就直接返回空了。死路一条，弃之。

回到上面，现在可以确定的是该参数是必须需要存在的，且参数值还得必须为`1`。不妨构造一个请求发送看看。

```http
POST /mobile/plugin/browser.jsp HTTP/1.1
Host:
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

isDis=1
```

![](/img/post/weaver-ecology9-browser-sqli/20230308145329059.png)

与他们放出的测试图相比，是不是就对上了。那么也就能够确定漏洞的路径就是`/mobile/plugin/browser.jsp`。

在使用 BurpSuite Intruder 多个不同的目标时，发现除了 200 的状态码，还有很多 404 的，这种情况我们最后再说。

![](/img/post/weaver-ecology9-browser-sqli/20230301145931253.png)

## 补丁代码分析

通过前面两段内容的相互印证，可以确定本次 SQL 注入的漏洞补丁代码就是`SecurityRuleForMobileBrowser.class`文件的内容，完整补丁代码内容如下。

![](/img/post/weaver-ecology9-browser-sqli/20230305221730843.png)

对该段补丁代码作简单分析。从第5行的第一个if条件语句开始，判断如果`../`、`\`、十六进制的`00`都不存在于URI中，则进入第6行下一个if条件语句判断，如果`/mobile/`、`/plugin/`、`/browser.jsp`都存在于URI中，那么获取`keyword`参数值；接着进入到第9行try语句，首先对`keyword`参数值进行了一次URL解码，并判断其中是否有恶意SQL注入payload `'`，如果有的话，则进行拦截、拉黑IP，返回false；紧接着对`keyword`参数值进行二次URL解码，并将二次URL解码后的值与第一次URL解码的值将比较，如果不一致也会进行拦截、拉黑IP，返回false，此处判断是为了防止多层URL编码Bypass的，即第一次URL解码的结果必须是最终的的解码结果。

通过分析可以得知存在注入的参数就是`/mobile/plugin/browser.jsp`中的`keyword`参数。

## SQL 注入分析

现在继续关注`/mobile/plugin/browser.jsp`中的内容。刚刚说到对`isDis`的判断，继续往下看。

```java
String f_weaver_belongto_userid=Util.null2String(request.getParameter("f_weaver_belongto_userid"));//需要增加的代码
String f_weaver_belongto_usertype=Util.null2String(request.getParameter("f_weaver_belongto_usertype"));//需要增加的代码
User user  = HrmUserVarify.getUser(request, response, f_weaver_belongto_userid, f_weaver_belongto_usertype) ;//需要增加的代码

BrowserAction braction = new BrowserAction(user, browserTypeId, pageNo, pageSize);
```

跟进`HrmUserVarify.getUser`，代码如下，这里肯定会返回`null`。

![](/img/post/weaver-ecology9-browser-sqli/20230306161509379.png)

起初这个地方让我感到很迷惑，误以为这个鉴权会被用到，实际并不会用到，这里返回的`null`作为`browser.jsp`中的`user`变量的值，但是在`browser.jsp`中并未对`user`做检查。如果需要达到鉴权的效果，那么正确的写法应该是增加如下代码片段：

```java
if(user == null)  return ;
```

如下图所示，`SearchSubDept.jsp`正是采用的这种写法。

![](/img/post/weaver-ecology9-browser-sqli/20230301151835907.png)

继续往下，设置了很多参数值，但不过大部分参数都不是必须的。

![](/img/post/weaver-ecology9-browser-sqli/20230306162337947.png)

最后到`braction.getBrowserData()`方法，这个方法位于`classbean/weaver/mobile/webservices/common/BrowserAction.class`文件。

![](/img/post/weaver-ecology9-browser-sqli/20230306163219969.png)

最开始有对`browserTypeId`参数值进行判断，以及很多`list`开头的方法，接着还对`method`参数值进行判断，根据不同的值执行不同的`list`开头的方法，那么注入很大可能就存在某个`list`开头的方法之中。

先简单尝试注入一下，请求如下：

```http
POST /mobile/plugin/browser.jsp HTTP/1.1
Host:
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 54

isDis=1&browserTypeId=160&keyword=a%' union select 1,'
```

![](/img/post/weaver-ecology9-browser-sqli/20230306222810681.png)

可以发现一些与 SQL 注入相关的敏感关键词（`'`、`select`）均被全角化了。那尝试 URL 编码一下，一层 URL 编码的请求如下，服务器直接返回 500 了。

![](/img/post/weaver-ecology9-browser-sqli/20230306223130961.png)

双层 URL 编码试试，如下图所示，服务器端依旧做了两次 URL 解码，然后发现`'`，将其转换成全角字符了，但不过`select`关键词却没有被全角化。

![](/img/post/weaver-ecology9-browser-sqli/20230306223346940.png)

继续三层 URL 编码，通过下图可以发现，经过三层编码后的字符串被 URL 解码两次后顺利到达`BrowserAction.getBrowserData()`，而在每一个`list`开头的方法中都有一次 URL 解码操作，那么就能确保我们的 SQL 注入 payload 顺利传递到 SQL 查询语句中。

![](/img/post/weaver-ecology9-browser-sqli/20230306223646378.png)

下面依次对各个`list`开头的方法进行审计，最后发现当`browserTypeId`等于 269 时，执行的`listRemindType()`方法中存在一个有回显的 SQL 注入漏洞。

```java
public void listRemindType() {
    try {
        new MeetingBrowser();
        this.keyword = URLDecoder.decode(this.keyword, "UTF-8");
        if (this.pageInfo.getPageNo() > 0) {
            this.pageInfo.setPageNo(this.pageInfo.getPageNo());
        }

        RecordSet var1 = new RecordSet();
        String var2 = " select count(0) as count from meeting_remind_type t1  where isuse=1 ";
        if (StringUtils.isNotEmpty(this.keyword)) {
            var2 = var2 + " and name like '%" + this.keyword + "%'";
        }

        var1.executeSql(var2);
        int var3 = 0;
        if (var1.next()) {
            var3 = Util.getIntValue(var1.getString("count"), 0);
        }

        this.pageInfo.setTotalCount(var3);
        String var4 = " from  meeting_remind_type t1  ";
        String var5 = "t1.id as id,t1.name as name";
        String var6 = " where isuse=1 ";
        if (StringUtils.isNotEmpty(this.keyword)) {
            var6 = var6 + " and t1.name like '%" + this.keyword + "%' ";
        }

        String var7 = "t1.id";
        String var8 = "t1.id";
        log.info("select " + var5 + var4 + " where " + var6);
        this.pageInfo.setResult(this.getLimitPageData(var5, var4, var6, var7, var8, 1, this.pageInfo.getPageNo(), this.pageInfo.getPageSize(), 3));
    } catch (Exception var9) {
        var9.printStackTrace();
    }

}
```

看到这里就可以直接构造注入 payload 了，最终注入的效果如下所示。

![](/img/post/weaver-ecology9-browser-sqli/20230306225108035.png)

![](/img/post/weaver-ecology9-browser-sqli/20230306225238865.png)

## 认证绕过分析

在上面有埋下一个坑，就是在使用 BurpSuite Intruder 请求多个不同的目标的`/mobile/plugin/browser.jsp`时，发现有很多返回 404 状态码的站。

![](/img/post/weaver-ecology9-browser-sqli/20230305233525394.png)

我们需要再次对比 v10.54 和 v10.56 的补丁包。

![](/img/post/weaver-ecology9-browser-sqli/20230301153726944.png)

找到`SecurityRuleMobile29.class`这么一个补丁文件，再次对比，首先可以发现在 v10.54 补丁包中的 `SecurityRuleMobile29.class`文件的最后修改时间是 2020 年 9 月 10 日，那么如果 ecology 没有打过这个补丁则无需考虑绕过的情况，`/mobile/plugin/browser.jsp`路径可以被直接访问，这也就是为什么有一些站直接访问该路径不会出现 404 的原因。

```shell
# stat -f %SB Ecology_security_20220805_v10.54/WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleMobile29.class Ecology_security_20230213_v10.56/WEB-INF/myclasses/weaver/security/rules/ruleImp/SecurityRuleMobile29.class
Sep 10 09:51:04 2020
Dec  7 20:26:20 2022
```

![](/img/post/weaver-ecology9-browser-sqli/20230301153906834.png)

未更新该补丁之前的`validate`方法内容如下图。

![](/img/post/weaver-ecology9-browser-sqli/20230305223551904.png)

我们先从 153 行的 else 语句开始看起。毫无疑问，当请求路径中存在`/mobilemode/`或`/mobile/`或`/cpt/`其中一个，并且请求路径的结尾是`.jsp`时，顺利进入到 154 行 if 分支。

```java
else {
    if (path.indexOf("/mobilemode/") != -1 || path.indexOf("/mobile/") != -1 || path.indexOf("/cpt/") != -1 && path.endsWith(".jsp")) {
        List<String> mobileNoLoginUrlList = (List)sc.getRule().get("mobile-no-login-urls");
        if (mobileNoLoginUrlList != null && !mobileNoLoginUrlList.isEmpty()) {
            Iterator var7 = mobileNoLoginUrlList.iterator();

            while(var7.hasNext()) {
                String url = (String)var7.next();
                if (path.indexOf(url) != -1) {
                    return true;
                }
            }
        }

        List<String> mobileNeedLoginUrlList = (List)sc.getRule().get("mobile-need-login-urls");
        if (mobileNeedLoginUrlList != null && !mobileNeedLoginUrlList.isEmpty()) {
            Iterator var8 = mobileNeedLoginUrlList.iterator();

            while(var8.hasNext()) {
                String url = (String)var8.next();
                if (path.indexOf(url) != -1) {
                    boolean isLogin = this.isLogin(req, res);
                    if (!isLogin) {
                        sc.writeLog(">>>>Xss(Validate failed[Not Login]) validateClass=weaver.security.rules.SecurityRuleMobile29  path=" + req.getRequestURI() + " security validate failed!  source ip:" + ThreadVarManager.getIp());
                        return false;
                    }
                }
            }
        }
    }

    return true;
}
```

接下来，`mobileNoLoginUrlList`这个列表中的路径意味着无需登录即可直接访问，如果请求的路径在该列表中，则会返回 true。

<img src="/img/post/weaver-ecology9-browser-sqli/20230305225434361.png" style="zoom:50%;" />

`mobileNeedLoginUrlList`列表顾名思义，当请求的路径在该列表中，则是需要登录才能访问的，否则就会返回 false。而`/mobile/plugin/browser.jsp`恰巧在其之中。

<img src="/img/post/weaver-ecology9-browser-sqli/20230305225627024.png" style="zoom:50%;" />

当请求的路径既不属于`mobileNoLoginUrlList`，也不属于`mobileNeedLoginUrlList`，也是会返回 true 的。

那么我们继续看`validate`方法中新增的补丁代码片段：

![](/img/post/weaver-ecology9-browser-sqli/20230305230609467.png)

```java
else if (StringUtil.matches(path, "\\s") && StringUtil.matches(path, "/\\s+/")) {
  sc.writeLog(">>>>Xss(Validate failed[invalidate url]) validateClass=weaver.security.rules.SecurityRuleMobile29  path=" + req.getRequestURL() + " security validate failed!  source ip:" + ThreadVarManager.getIp());
  return false;
}
```

这里做了一个正则匹配，如果请求路径中存在空白字符，就会触发安全补丁的警告，并返回 false。

在此之前，需要留意`super.path`方法，这个是父类`ParentRule`中的方法。

![](/img/post/weaver-ecology9-browser-sqli/20230305231648026.png)

`path`方法会对请求路径中出现的一些特殊字符如`;`、`//`，那么则会做正则 replace。并且最后还会去除路径中出现的空白字符。最后返回`path.toLowerCase()`。

但是这里过滤的并不全，不然就不会出现补丁代码中又一遍的检查了。

```java
StringUtil.matches(path, "\\s") && StringUtil.matches(path, "/\\s+/")
```

所以这里必定是存在绕过的，别忘了`path`方法中开始会使用`uriDecode`方法对请求路径做 URL 解码的哦。

最后梳理下，原始的请求路径需要先经过 URL 解码，解码后其中不要存在有`;`、`//`字符，然后还要经过一遍去空白字符操作，再然后就会返回最终的路径，最后的路径能到达`/mobile/plugin/browser.jsp`。

![](/img/post/weaver-ecology9-browser-sqli/20230305233405114.png)

成功绕过后，就能对未打这次最新补丁的 ecology 进行 SQL 注入了。

![](/img/post/weaver-ecology9-browser-sqli/20230305235041800.png)
