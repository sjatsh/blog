--
layout: post
title: "七牛文件上传"
description: ""
tag: "qiniu"
date: "2017-07-10 00:00:00 +0800"
categories: "net"
--

#### 基本的环境搭建 

  这里使用的是七牛的Java sdk，框架用的是spring boot，至于spring boot的基本框架搭建就不说了。
maven引入依赖
```
<dependency>
    <groupId>com.qiniu</groupId>
    <artifactId>qiniu-java-sdk</artifactId>
    <version>[7.0.0, 7.2.99]</version>
</dependency>
```

<!--more-->

#### 上传token的获取
```
public String getUpToken(String appId){
	
    StringMap map       = new StringMap();
    StringMap putPolicy = new StringMap();

    map.put("key", "${key}");
    map.put("hash", "${hash}");
    map.put("bucket", "${bucket}");
    map.put("fsize", "${fsize}");
    map.put("fname", "${fname}");
    map.put("appId", appId);

    Auth auth = Auth.create(Constants.ACCESSS_KEY, Constants.SECRET_KEY);
    //上传文件不能重复(0:相同文件覆盖;1:相同文件不会覆盖) 只有在设置了key时才会生效,默认相同文件相同key不会覆盖正确返回
    putPolicy.put("insertOnly", 0);
    //是否启用上传模式(1:启用;0:关闭)启用后 由回调的返回参数的key字段指定七牛中的资源名称
    putPolicy.put("callbackFetchKey", 1);
    //回调Content-Type
    putPolicy.put("callbackBodyType", "application/json");
    //回调URL
    putPolicy.put("callbackUrl", Constants.CALLBACK_URL);
    //回调post请求体
    putPolicy.put("callbackBody", Json.encode(map));

    String uploadToken = auth.uploadToken(Constants.BUCKET, null, Constants.EXPIRES_1H, putPolicy);
    return uploadToken;
}
```
代码中的`callbackBody`定义上传返回的变量和用户自定义的魔法变量。其中：    
key：七牛存储空间中的资源名称，可以用户定义或者七牛自动生成一个随机的UUID  
hash：文件的hash值
bucket：存储空间名称
fsize：文件大小
fname：文件名
appId：自定义变量  
`callbackFetchKey`：七牛通过回调的返回参数中的key来确定资源文件名称，用于自定义前缀

![](https://olef5l6y5.qnssl.com/blog/xQ0AAONdzlhN5s8U-ff4b40ac-5181-4644-9521-80727d177349)  
这里的appId用于确定哪个业务线，实际项目中应该定义固定的appId，拿到上传Token就可以进行上传操作了。

#### 文件上传

```
public Object callback(HttpServletRequest request) throws IOException{

		String         line = "";
		StringBuilder  sb   = new StringBuilder();
		BufferedReader br   = new BufferedReader(new InputStreamReader(request.getInputStream()));
		while ((line = br.readLine()) != null){
			sb.append(line);
		}

		String authorization = request.getHeader("Authorization");
		String contentType   = request.getContentType();

		Auth auth = Auth.create(Constants.ACCESSS_KEY, Constants.SECRET_KEY);
		//七牛回调鉴权
		boolean             isQiniuCallback = auth.isValidCallback(authorization, Constants.CALLBACK_URL, sb.toString().getBytes(), contentType);
		Map<String, Object> returnMap       = Maps.newHashMap();

		if (isQiniuCallback){

			CallBackParam callBackParam = Json.decode(sb.toString(), CallBackParam.class);

			String appId = callBackParam.getAppId();
			String key   = callBackParam.getKey();

			callBackParam.setKey(appId + "/" + key);
			returnMap.put("key", appId + "/" + key);
			returnMap.put("payload", callBackParam);

		} else{

			returnMap.put("success", false);

		}
		return returnMap;
	}
```
回调中获得appId以及七牛生成的key，我们将其组装起来以自定义前缀。  我们会看到之前设置的自定义变量`appId:blog`是在生成token的时候设置的，
上传完成后会返回给我们。可能有人会问，这里直接上传到七牛不就好了么，为什么还要写一个方法。其实原因很简单，从上面的生成token的方法中已经可
以看出我们设置了`callbackUrl`以及`callbackBody`来让七牛在上传完成的时候本来回调我们自己的接口，只有当我们自己的接口回调成功整个上传才
会成功，反之失败。而且通过回调我们可以知道我们什么时候上传完成，同时可以进行一系列的log、插表等操作来记录上传信息。
![](https://olef5l6y5.qnssl.com/blog/7B0AAFxPM5lH5s8U-b432ca33-f78b-4648-b54b-56ca658c9656)
#### 设置callbackFetchKey情况
![](https://olef5l6y5.qnssl.com/blog/qi0AAOjAXrzC6M8U-c0dd96ac-5bf5-4063-afab-444443605274)
#### 未设置callbackFetchKey情况
![](https://olef5l6y5.qnssl.com/blog/ZFsAAKtAPyKg6c8U-5378915c-6904-44b5-9e29-91539e734464)
![](https://olef5l6y5.qnssl.com/blog/ZFsAAKXEBeuk6c8U-da6e85c9-ee98-49bf-977c-4935af027d52)
可以看到回参中有前缀但是七牛存储空间中是没有前缀的

因为七牛回调需要调用我们本地的接口，可是我们没有公网ip所以使用的ngrok内网穿透来实现的，ngrok的安装我的博客没有写，可以参考我
[同事的博客](https://wangjun.me/2017/02/24/ngrok/)。
