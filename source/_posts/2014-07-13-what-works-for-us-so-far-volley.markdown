---
layout: post
title: "What works for us so far - Volley"
date: 2014-07-13 17:15:15 +0800
comments: true
categories: [Android,BeeTalk]
author: Zhao Cong
author_about: zhao-cong
published: true
---

There are quite a few open source projects which aims to accelerate Android application development. For example,
[this series](http://java.dzone.com/articles/how-become-lazy-productive) is a great summary on how to be "lazy productive". All of those frameworks or tools look great at the first sight but they may screw up you when you drive deeper and deeper.

To help you guys to make better decision, we want to share our experience and today let's starts with [Volley](https://android.googlesource.com/platform/frameworks/volley/) from Google.

### [Volley](https://android.googlesource.com/platform/frameworks/volley/)

Volley debuts in Google IO last year and we fall in love with it quickly. It provides a complete set of toolkit to download anything via HTTP protocol. In addition to the general file download framework, it shows us a great example on how to download and process the images. With Volley, you don't need to worry about

- Establish the connection
- How to handle HTTP files
- How to handle the images
- Can extend to download anything

<!-- more -->

By default, Volley can help you download JSON and images, but it can help you to download anything.

Build a simple byte data wrapper

``` java
public class ByteWrapper {

    private byte[] data;

    public ByteWrapper(byte[] data) {
        this.data = data;
    }

    public byte[] getData(){
        return this.data;
    }
}
```

Then implement your own download request object

``` java
public class BinaryHttpRequest extends Request<ByteWrapper> {

    private Response.Listener<ByteWrapper> mProcessor;

    public BinaryHttpRequest(int method, String url,Response.Listener<ByteWrapper> processor,Response.ErrorListener listener) {
        super(method, url, listener);
        mProcessor = processor;
        //retry once
        setRetryPolicy(new DefaultRetryPolicy(DefaultRetryPolicy.DEFAULT_TIMEOUT_MS, 2, DefaultRetryPolicy.DEFAULT_BACKOFF_MULT));
    }

    @Override
    protected Response<ByteWrapper> parseNetworkResponse(NetworkResponse response) {
        byte[] data = response.data;
        return Response.success(new ByteWrapper(data),this.getCacheEntry());
    }

    @Override
    protected void deliverResponse(ByteWrapper response) {
        mProcessor.onResponse(response);
    }
}
```

If you want to parse the bytes into JSON, you can easily write a callback similar to this

```
public abstract class BBJsonDownloadEvent implements DownloadEvent {

	//OVERRIDE this class to process the JSON object
    protected abstract void onJSonObject(JSONObject obj) throws JSONException;

	@Override
    public void onError(int errorCode) {

    }

    @Override
    public void onFinish(byte[] data, int nLen) {
        try {
            String JSONString = new String(data, "UTF-8");
            JSONObject response= new JSONObject(JSONString);
            onJSonObject(response);
        } catch (UnsupportedEncodingException |JSONException e) {
            BBAppLogger.e(e);
        }
    }

    @Override
    public void onDownloading(int nAllLen, int nPos) {

    }

    @Override
    public void onDestroy() {

    }
}
```

Then you just need to read the bytes and get what you want, happy eh :-)

Soon after we drive Volley to do everything for us, we found ourselves in deep trouble. Apparently the memory usage is high and 10+ threads starts with the application. We dig the source and eventually realize we misuse the download queue. Normally we start the Volley download queue by
```
  Volley.newRequestQueue(); //now you get 6 idle threads
```
It is simple and elegant. Read the source code and we are shocked at the fact that it always starts 6 threads right away. We have a few request queue in different modules so Volley starts tons of useless threads when the applicaiton starts. The fix is simple - we make sure that all different modules share the same request queue (in another Singleton nightmare ><).

Then after upgrade to latest Volley, it screws up again because it always scales the full size images to fit the size for display :-(

Well, sorry, Volley,we don't want you to be our default image loader anymore because user shall not download the images twice just because it gonna display in different sizes. We have built our own image loading solution which we pretty much copy the idea from Volley but backended by file downloading via in house protocol.  Essentially we follows those general principals to design our own image loading solution.

- Share the LRU cache
- engage synchorous loading if the bitmap is in cache
- engage asynchorous loading if we don't have the images ready (not in cache or need to download from the Internet)

As conclusion, Volley inspires us to write a decent image processing module and it shows us a excellent lightweight design for network requests.