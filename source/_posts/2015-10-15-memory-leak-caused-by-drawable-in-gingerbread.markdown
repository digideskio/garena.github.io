---
layout: post
title: "Memory Leak caused by Drawable in Gingerbread"
date: 2015-10-15 00:34:00 +0800
comments: true
published: true
---

In the previous [blog](/blog/2014/09/10/android-memory-leaks/), we have briefly introduced memory leak in Android. Memory leak is one of the most important issues, we need to pay attention to during Android development. However, sometimes, we may get memory leak when we are unaware of it. In this blog, we would like to share a scenario of memory leak, which happens in Gingerbread. 

##Scenario

Let's first take a look at a simple example:

``` java
public class SecondActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        Drawable drawable = DrawableLruCache.getInstance()
                .getDrawable(R.drawable.android_logo);
        ImageView imageView = (ImageView) findViewById(R.id.image_view);
        imageView.setImageDrawable(drawable);
    }
}
```

This piece of code is very simple. In an Activity, we would like to get a Drawable from a LRU cache, and then show the Drawable in a ImageView inside the Activity. Could you figure out anything wrong with this piece of code?

Actually, this piece of code would cause memory leak in Gingerbread devices. When the Activity is destroyed, the LRU cache would likely continue to keep a strong reference of the Drawable. The Drawable would still keep a strong reference of the View. Therefore, the whole Activity, as well as everything inside the Activity would be memory leaked. 

##Reason
The memory leak is caused by a small “bug” in the design of the Drawable object in Gingerbread.

In Android, all the Views implement the ```Drawable.Callback``` interface. This interface is required for the Views to support animated Drawable. Whenever we need to display the Drawable in the View, for instance when we calling ```setImageDrawable(Drawable drawable)``` in a ImageView, Android would set the view as the callback client of the Drawable, in order to allow the View to support animation for the Drawable. You may figure out the detail implementation of the method from the source code of Android [here](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.1_r1.2/android/widget/ImageView.java#ImageView.setImageDrawable%28android.graphics.drawable.Drawable%29).

However, according to the [source code](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/2.3.7_r1/android/graphics/drawable/Drawable.java#Drawable.setCallback%28android.graphics.drawable.Drawable.Callback%29) of the Drawable, the Callback of the Drawable is passed as a strong reference in Gingerbread shown below:
``` java
public final void  setCallback(Callback cb) {
        mCallback = cb;
}
```

That means, whenever we want to display the Drawable to a View, the Drawable would keep a strong reference of the View. Therefore, even after the Activity or View is destroyed, the Drawable would still keep a strong reference to the View, which may cause the memory leak.

You may think it is the LRU cache causes the memory leak. However, even if we do not use the LRU cache, this memory leak may still happen easily. The reason is that Android does have some kind of recycling manager with Drawable component. Therefore, when we get the Drawable with the same resource ID at different places, it is very common that Android would return the same instance of Drawable in order to optimize the usage of the memory. 

In other words, even there is no LRU cache, the same Drawable may still be used by other components such as another ImageView in another Activity, as long as the other ImageView needs to show the same Drawable with the same resource ID. If the other ImageView exists in the memory, it would keep a strong reference of the Drawable. Consequently, the Drawable would keep a strong reference of the View and will cause the same memory leak.

##Solution

In order to solve this kind of memory leak, what we need to do is to manually set the Callback to be null when ```onDestory()``` is called in the Activity, or when ```onDetachedFromWindow()``` is called for the specific View. The source code we used to unbind the Drawable from the root view of the Activity is shown below:


``` java
public static void unbindDrawables(View view) {
    if (view == null) {
        return;
    }
    if (view.getBackground() != null) {
        view.getBackground().setCallback(null);
    }
    if (view instanceof ImageView) {
        ImageView imgView = (ImageView) view;
        if (imgView.getDrawable() != null) {
            Drawable b = imgView.getDrawable();
            b.setCallback(null);
            imgView.setImageDrawable(null);
        }
    } else if (view instanceof ViewGroup) {
        for (int i = 0; i < ((ViewGroup) view).getChildCount(); i++) {
            unbindDrawables(((ViewGroup) view).getChildAt(i));
        }
        try {
            if ((view instanceof AdapterView<?>)) {
                AdapterView<?> adapterView = (AdapterView<?>) view;
                adapterView.setAdapter(null);
            } else {
                ((ViewGroup) view).removeAllViews();
            }
        } catch (Exception e) {
        }
    }
}

```

One more thing need to mention is that, this issue has already fixed in the Android Ice Cream Sandwich and above. In the 
newer version of Android, the Callback would be passed as a ```WeakReference``` according to the source 
code [here](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.2.1_r1.2/android/graphics/drawable/Drawable.java#Drawable.setCallback%28android.graphics.drawable.Drawable.Callback%29). Hence, if you are no longer supporting Gingerbread, you don’t need to care about this issue. 
