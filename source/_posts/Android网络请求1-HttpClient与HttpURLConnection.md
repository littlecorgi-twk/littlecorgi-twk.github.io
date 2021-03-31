---
title: Android网络请求1--HttpClient与HttpURLConnection
categories:
  - Android
tags:
  - Android
  - kotlin
  - Volley
  - 源码
comments: true
copyright: true
abbrlink: eb898689
date: 2019-08-06 19:51:46
updated: 2019-08-06 19:51:46
---
在早期的时候，Android上还没有像`Volley`、`OkHttp`、`Retrofit`这些优秀的开源库，如果想要使用网络请求的话，就只能自己封装`HttpClient`和`HttpURLConnection`。现在我们就来看下`Apache`的这两个类。
<!-- More -->

# 1. HttpClient

## 2.1 导入HttpClient
由于从Android 6.0 开始，谷歌就将HttpClient从Android中删除了，所以若现在想使用他，还得导入依赖：
在项目的`build.gradle`的`Android`代码块下加入依赖，示例：
```java
android {
    useLibrary 'org.apache.http.legacy'
    ...
}
```
## 2.2 HttpClient的Get
首先通过`DefaultHttpClient`来实例化一个`HttpClient`，并配置好参数：
```java
 //创建HttpClient
private HttpClient createHttpClient() {
    HttpParams mDefaultHttpParams = new BasicHttpParams();
    //设置连接超时
    HttpConnectionParams.setConnectionTimeout(mDefaultHttpParams, 15000);
    //设置请求超时
    HttpConnectionParams.setSoTimeout(mDefaultHttpParams, 15000);
    HttpConnectionParams.setTcpNoDelay(mDefaultHttpParams, true);
    HttpProtocolParams.setVersion(mDefaultHttpParams, HttpVersion.HTTP_1_1);
    HttpProtocolParams.setContentCharset(mDefaultHttpParams, HTTP.UTF_8);
    //持续握手
    HttpProtocolParams.setUseExpectContinue(mDefaultHttpParams, true);
    HttpClient mHttpClient = new DefaultHttpClient(mDefaultHttpParams);
    return mHttpClient;
}
```
接着创建`HttpGet`和`HttpClient`，请求网络并得到`HttpResponse`，并对`HttpResponse`进行处理：
```java
private void useHttpClientGet(String url) {
    HttpGet mHttpGet = new HttpGet(url);
    mHttpGet.addHeader("Connection", "Keep-Alive");
    try {
        HttpClient mHttpClient = createHttpClient();
        HttpResponse mHttpResponse = mHttpClient.execute(mHttpGet);
        HttpEntity mHttpEntity = mHttpResponse.getEntity();
        int code = mHttpResponse.getStatusLine().getStatusCode();
        if (null != mHttpEntity) {
            InputStream mInputStream = mHttpEntity.getContent();
            String respose = converStreamToString(mInputStream);
            Log.i("wangshu", "请求状态码:" + code + "\n请求结果:\n" + respose);
            mInputStream.close();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
`converStreamToString()`方法将请求结果转换成`String`类型：
```java
private String converStreamToString(InputStream is) throws IOException {
    BufferedReader reader = new BufferedReader(new InputStreamReader(is));
    StringBuffer sb = new StringBuffer();
    String line = null;
    while ((line = reader.readLine()) != null) {
        sb.append(line + "\n");
    }
    String respose = sb.toString();
    return respose;
}
```
最后开启线程访问:
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        useHttpClientGet("http://www.baidu.com");
    }
}).start();
```
## 1.3 HttpClient的POST
和GET差不多，只需要修改传递的参数：
```java
private void useHttpClientPost(String url) {
    HttpPost mHttpPost = new HttpPost(url);
    mHttpPost.addHeader("Connection", "Keep-Alive");
    try {
        HttpClient mHttpClient = createHttpClient();
        List<NameValuePair> postParams = new ArrayList<>();
        //要传递的参数
        postParams.add(new BasicNameValuePair("username", "moon"));
        postParams.add(new BasicNameValuePair("password", "123"));
        mHttpPost.setEntity(new UrlEncodedFormEntity(postParams));
        HttpResponse mHttpResponse = mHttpClient.execute(mHttpPost);
        HttpEntity mHttpEntity = mHttpResponse.getEntity();
        int code = mHttpResponse.getStatusLine().getStatusCode();
        if (null != mHttpEntity) {
            InputStream mInputStream = mHttpEntity.getContent();
            String respose = converStreamToString(mInputStream);
            Log.i("wangshu", "请求状态码:" + code + "\n请求结果:\n" + respose);
            mInputStream.close();
        }
    } catch (IOException e) {
            e.printStackTrace();
    }
}
```

# 2. HttpURLConnection
`HttpURLConnection`较`HttpClient`来说更轻量，而且他`API`也比`HttpClient`简单。特别是`Android 6.0`将`HttpClient`移除之后，现在只能使用`HttpURLConnection`。
## 2.1 HttpURLConnection的POST请求
首先我们创建一个UrlConnManager类，然后里面提供getHttpURLConnection()方法用于配置默认的参数并返回HttpURLConnection：
```java
public static HttpURLConnection getHttpURLConnection(String url){
    HttpURLConnection mHttpURLConnection=null;
    try {
        URL mUrl=new URL(url);
        mHttpURLConnection=(HttpURLConnection)mUrl.openConnection();
        //设置链接超时时间
        mHttpURLConnection.setConnectTimeout(15000);
        //设置读取超时时间
        mHttpURLConnection.setReadTimeout(15000);
        //设置请求参数
        mHttpURLConnection.setRequestMethod("POST");
        //添加Header
        mHttpURLConnection.setRequestProperty("Connection","Keep-Alive");
        //接收输入流
        mHttpURLConnection.setDoInput(true);
        //传递参数时需要开启
        mHttpURLConnection.setDoOutput(true);
    } catch (IOException e) {
        e.printStackTrace();
    }
    return mHttpURLConnection ;
}
```
因为我们要发送POST请求，所以在UrlConnManager类中再写一个postParams()方法用来组织一下请求参数并将请求参数写入到输出流中：
```java
public static void postParams(OutputStream output,List<NameValuePair>paramsList) throws IOException{
    StringBuilder mStringBuilder=new StringBuilder();
    for (NameValuePair pair:paramsList){
        if(!TextUtils.isEmpty(mStringBuilder)){
            mStringBuilder.append("&");
        }
        mStringBuilder.append(URLEncoder.encode(pair.getName(),"UTF-8"));
        mStringBuilder.append("=");
        mStringBuilder.append(URLEncoder.encode(pair.getValue(),"UTF-8"));
    }
    BufferedWriter writer=new BufferedWriter(new OutputStreamWriter(output,"UTF-8"));
    writer.write(mStringBuilder.toString());
    writer.flush();
    writer.close();
}
```
接下来我们添加请求参数，调用postParams()方法将请求的参数组织好传给HttpURLConnection的输出流，请求连接并处理返回的结果：
```java
private void useHttpUrlConnectionPost(String url) {
    InputStream mInputStream = null;
    HttpURLConnection mHttpURLConnection = UrlConnManager.getHttpURLConnection(url);
    try {
        List<NameValuePair> postParams = new ArrayList<>();
        //要传递的参数
        postParams.add(new BasicNameValuePair("username", "moon"));
        postParams.add(new BasicNameValuePair("password", "123"));
        UrlConnManager.postParams(mHttpURLConnection.getOutputStream(), postParams);
        mHttpURLConnection.connect();
        mInputStream = mHttpURLConnection.getInputStream();
        int code = mHttpURLConnection.getResponseCode();
        String respose = converStreamToString(mInputStream);
        Log.i("wangshu", "请求状态码:" + code + "\n请求结果:\n" + respose);
        mInputStream.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
最后开启线程请求网络：
```java
private void useHttpUrlConnectionGetThread() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            useHttpUrlConnectionPost("http://www.baidu.com");
        }
    }).start();
}
```