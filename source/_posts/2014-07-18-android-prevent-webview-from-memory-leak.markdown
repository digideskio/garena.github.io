---
layout: post
title: "Android prevent webview from memory leak"
date: 2014-07-18 14:40:33 +0800
comments: true
published: true
categories: [Android,Beetalk]
author: Xiang Yongzhou
categories: 
---

In Android app development, webview is widely used.  However, if you are using webview without caution, it will easily consume a lot of memory.  And your app may even crash due to insufficient memory.

The memory issue of webview is well-known, here are two great links about the issue. 

[Post in English](http://stackoverflow.com/questions/3130654/memory-leak-in-webview)

[Post in Chinese](http://my.oschina.net/zhibuji/blog/100580)

For android 4.4 and above, android is using chromium as the core of webview. For android 3.1 to 4.3, android is using webkit as the core of webview. Most of the memory leak happens on webkit. 

And the blog will briefly summarize the tips to prevent webview memory leak 

Tip 1:
When clearing the webview reference. Simply set it to null is not enough. webview.destroy() must be called.
Example code:
```java
    @Override
    public void onDestroy() {
        super.onDestroy();
        if(webView != null) {
            webView.removeAllViews();
            webView.destroy();
        }
        webView = null;
    }
```

Tip 2:
Using application context when initializing webview. This means webview should not be initialized from xml. You need to create it programmatically.
Example Code:
```java
    WebView webView = new WebView(getApplicationgContext()); 
    LinearLayout linearLayout = findViewById(R.id.xxx); 
    linearLayout.addView(webView);
```
By doing this, 90% of the time, webview memory consumption is no longer an issue. However, if you want to play a full screen video in your webview, the onShowCustomerView of WebChromeClient will be triggered. And before onShowCustomerView is triggered, the Webkit will do some preprocessing. Due to a bug in Webkit's implementation, a static variable in html5videoviewproxy will hold the webview's reference which will caused a leak and you cannot do much about it. 

Another problem using application context is that when your webview wants to popup a window. In webview's default behaviour, there is a cast from ApplicationContext to ActivityContext which will cause your app crash. Of course, you can implement your own method of handling popup to walk around this problem, but it takes a lot of unnecessary effort.

P.S. we can inject javascript into webview to do many tricks. I will later post another blog about using javascript in webview.

Tip 3:
I personally did not test this method, but I think the method is creative and instructive so I am noticing it down here.

The main idea is webkit does not provide us a method to release the reference, so let us do it ourself by reflection. The reason why I did not use it is because it highly depends on webkit's implementation and you have to write different code for different android versions.

```java
public void setConfigCallback(WindowManager windowManager) {
    try {
        Field field = WebView.class.getDeclaredField("mWebViewCore");
        field = field.getType().getDeclaredField("mBrowserFrame");
        field = field.getType().getDeclaredField("sConfigCallback");
        field.setAccessible(true);
        Object configCallback = field.get(null);
 
        if (null == configCallback) {
            return;
        }
 
        field = field.getType().getDeclaredField("mWindowManager");
        field.setAccessible(true);
        field.set(configCallback, windowManager);
    } catch(Exception e) {
    }
}
```
Call the method in onCreate and onDestroy
```java
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setConfigCallback((WindowManager)getApplicationContext().getSystemService(Context.WINDOW_SERVICE));
}
 
public void onDestroy() {
    setConfigCallback(null);
    super.onDestroy();
}
```

Ultimate Tip:
This method is the ultimate way to solve the problem and many well-known app such as QQ and Wechat are using it as well. 

The idea is to start a separate process for webview and kill the process when existing the webview. The drawback of doing this is that you need to handle interprocess communication carefully. My suggestion is you can create a preProcessProxy and postProcessProxy to handle IPC and passing information around using intent.

Example code:
define webview activity in a separate process

```xml
<activity
 android:name=".webview.WebViewActivity"
 android:launchMode="singleTop"
 android:process=":remote"
 android:screenOrientation="unspecified" />
``` 

Kill the process when destroying the activity

```java
    @Override
    protected void onDestroy() {
        super.onDestroy();
        System.exit(0);
    }
```

To sum up:
If you are using webview for simple use, use application context to start webview is enough.
If your webview needs to do some complex actions such as playing video or dealing popups, start it in a different process.

