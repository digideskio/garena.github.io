---
layout: post
title: "Memory Leaks in Android"
date: 2014-09-10 04:35:45 -0400
comments: true
categories: 
---
##What is a Memory Leak?
Memory leak is existence of objects in the memory, that are not needed anymore according to the application logic, but still retain memory and cannot be collected because they are referenced from other live objects, due to a bug in application itself. When the program runs for an extended time and consumes additional memory over time, it can diminish the performance of the system by reducing the amount of available memory. Eventually, in the worst case, too much of the available memory may become allocated and the system may run "[out of memory (OOM)](https://developer.android.com/reference/java/lang/OutOfMemoryError.html)".

## Understanding Memory Leaks
To understand memory leaks in Android and how they may happen, let's first start with a simple example:

```java
public class HomeActivity extends BBBaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        HomeView view = new HomeView(this);
        setContentView(view);

        new SampleTask().execute();
    }

    private class SampleTask extends AsyncTask<Void, Void, Void> {
            
        @Override
        protected Void doInBackground(Void... params) {
            // do nothing
            return null;
        }
    }
}
```

This code simply starts an `AsyncTask` when the activity is created. You could have similar code which may start the task when a button is clicked. What if I told you this code can cause potential memory leak! It is not obvious now (because we haven't really done anything in the background).

What if we add some code to the `doInBackground()` method:

```java
public class HomeActivity extends BBBaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        HomeView view = new HomeView(this);
        setContentView(view);

        new SampleTask().execute();
    }

    private class SampleTask extends AsyncTask<Void, Void, Void> {

        private long mStartTime;

        @Override
        protected void onPreExecute() {
            mStartTime = System.currentTimeMillis();
        }

        @Override
        protected Void doInBackground(Void... params) {
            doBackgroundWork();
            return null;
        }

        private void doBackgroundWork() {
            long timeNow;
            do {
                timeNow = System.currentTimeMillis();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } while (timeNow - mStartTime < (1 * 60 * 1000));
        }
    }
}
```

The above code performs some background work. For demonstration, it simply runs in a while loop for `1 minute`.  This is a fair sample because some tasks may be used for image downloading, file reading etc, which could take some time. Let's run this code and analyze the memory for leaks using [Memory Analyzer Toolkit (MAT)](http://android-developers.blogspot.sg/2011/03/memory-analysis-for-android.html).

![Memory Analysis using MAT](/images/leak.png)
 

As we can see in the picture above, there are multiple instances of the `HomeActivity` in the memory which are retained and not garbage collected. After running the above code, try rotating the screen multiple times, or close and launch the activity again a few times. Once the activity is destroyed and recreated, the old instance is still in the memory and is not cleared (even though the activity has been destroyed and is not needed by the application or code logic anymore). Recall, this is what we call **memory leak**.

###Why are the old instances of home activity not cleared?

At the fundamental level, old instances are not cleared even after the activity is destroyed because *"someone"* is keeping a reference to it. This *"someone"* is our `AsyncTask`.  Because we define our `AsyncTask` as a non-static inner class, it keeps an explicit reference to the parent class (`HomeActivity`). The `AsyncTask` will be in the memory (and not be garbage collected) until it has finished execution. So even though the activity has been destroyed, the `AsyncTask` keeps a reference to it, which prevents the activity instance from getting cleared. 

###When `AsyncTask` completes, the references will be cleared, so why is it a problem?

This is a problem because it can cause your app to run *"out of memory"* and crash. Android system allocates a certain maximum amount of memory to each app. If the app's memory usage exceeds the maximum allocated *heap size*, [the application will thow OOM and crash](https://developer.android.com/training/articles/memory.html). Having unwanted objects in the memory that are not timely cleared can cause the application to run out of memory. Imagine, if your activity keeps references to bitmaps, large buffers or other objects, there will be multiple copies of them in the memory. This will increase the RAM usage substantially. Before there is a chance for all references to get cleared, the app may run out of memory and crash.

###How can we resolve this issue?

To answer this, lets see what is the fundamental problem.:

> The `AsycTask` is keeping reference to the `HomeActivity` which prevents it from being cleared.

We can ask ourselves:

 1.  Does the `AsyncTask` need to keep a reference to the `HomeActivity`? 
 2. If it does, how can we make sure that this reference is automatically cleared when we are sure we don't need it anymore?

In the above case, the `AsyncTask` does not need to keep a reference to the activity as it is not accessing any parent members. We could resolve the issue by simply declaring the `AsyncTask` as a **`static nested`** class or in a different class file altogether. This ensures that they not keep a reference to the parent class. (In case you are wondering, [here is the difference between inner class and nested class](http://www.programmerinterview.com/index.php/java-questions/inner-vs-nested-classes/)?) 

```java
public class HomeActivity extends BBBaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        HomeView view = new HomeView(this);
        setContentView(view);

        new SampleTask().execute();
    }

    private static class SampleTask extends AsyncTask<Void, Void, Void> {

        private long mStartTime;

        @Override
        protected void onPreExecute() {
            mStartTime = System.currentTimeMillis();
        }

        @Override
        protected Void doInBackground(Void... params) {
            doBackgroundWork();
            return null;
        }

        private void doBackgroundWork() {
            long timeNow;
            do {
                timeNow = System.currentTimeMillis();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } while (timeNow - mStartTime < (5 * 60 * 1000));
            Log.d("TASK", this.toString());
        }
    }
}
```
We run this code, perform the same operations as before, rotate screen few times, close re-open activity etc. This time we observer that there are multiple instances of the `AsyncTask` in the memory but not multiple instances of the `HomeActivity`.

![Memory Analysis using MAT](/images/leak-fix.png)

This method does not work always. In some cases, we need to perform certain UI operation after the `AsycTask` completes, in which case we are required to keep a reference to the `Activity` or `View`. Now we simply need to make sure that this reference is `weak` in nature so that it is automatically cleared by the system once the `Activity` or `View` are destroyed. For this we can make use of Java [`WeakReference<T>`](https://weblogs.java.net/blog/2006/05/04/understanding-weak-references) class as follows:

```java
public class HomeActivity extends BBBaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        HomeView view = new HomeView(this);
        setContentView(view);

        new SampleTask(this).execute();
    }

    private static class SampleTask extends AsyncTask<Void, Void, Void> {

        private long mStartTime;
        private WeakReference<HomeActivity> mRef;

        public SampleTask(HomeActivity activity) {
            mRef = new WeakReference<HomeActivity>(activity);
        }

        @Override
        protected void onPreExecute() {
            mStartTime = System.currentTimeMillis();
        }

        @Override
        protected Void doInBackground(Void... params) {
            doBackgroundWork();
            return null;
        }

        private void doBackgroundWork() {
            long timeNow;
            do {
                timeNow = System.currentTimeMillis();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } while (timeNow - mStartTime < (5 * 60 * 1000));
        }

        @Override
        protected void onPostExecute(Void aVoid) {
            HomeActivity activity = mRef.get();
            if (activity == null) {
                // the activity reference was cleared, 
                // lets forget about it
            }
            else {
                // lets update the activity with the results
                // of the task
            }
        }
    }
}
```
And hence we are able to write code which is memory efficient and less likely to make the system run out of memory.

# Other Leak Scenarios
Now that we understand what memory leak is, let us explore some other ways we can end up in similar leak situations:

### 1) Anonymous Inner Classes

Java provides syntax for creating `inner` classes on the fly which do not have a name.  Lets take a look at an example:

```java
/*Pay attention to the opening curly braces
  and the fact that there's a semicolon
  at the very end, once the anonymous class is created:
*/
DownloadCallback mCallback = new DownloadCallback() {
	//code here...
};
```

These [`anonymous inner`](http://www.programmerinterview.com/index.php/java-questions/java-anonymous-class-example/) classes have the same problem as we discussed above. Each keeps a reference to the instance of the `parent` class which initialized it. This means that if you pass around the `mCallback` object (in the above case), you may end up with a memory leak. Let's see that with an example:

We have a `FileManager` which downloads files. It is implemented as follows:

```java
public class FileManager {

    private static FileManager mInstance;

    public static synchronized FileManager getInstance() {
        if (mInstance == null) {
            mInstance = new FileManager();
        }
        return mInstance;
    }

    public FileManager() {
        mCallbackQueue = new ArrayDeque<DownloadCallback>();
    }

    private ArrayDeque<DownloadCallback> mCallbackQueue;

    public void download(String fileId, DownloadCallback callback) {
        mCallbackQueue.add(callback);
        // ..
        // do the download
        // ..
    }
}
```

If we perform file download from our `Activity` or `View` as follows:

```java
public class HomeActivity extends BBBaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        HomeView view = new HomeView(this);
        setContentView(view);

        FileManager.getInstance().download("file1", new DownloadCallback() {
            @Override
            public void onDownloadComplete() {
                // download has been completed
            }
        });
    }
}
```

We have a **"memory leak"**. Since the `FileManager` is a `singleton` it will be in the memory for the lifetime of the application. The download callback is passed to this `singleton` and stored within `mCallbackQueue`. This means that there is a reference to the `mCallback` object from a `static` object in the memory. 

> This means that for every callback object in the `mCallbackQueue`, its associated `parent` class will not get cleared from the memory.

If we remove the callback from the `mCallbackQueue` after the download completes, there are still problems as the download may take a while to finish. In the mean-time the user may have scrolled away or navigated to a new screen. Other downloads may have been put into the queue. Before all downloads complete, the system may go out of memory and crash.

> Lesson: Avoid passing `anonymous inner` class objects to static / singleton classes which keep a strong reference to it.

To resolve this issue:

1. We can follow the same logic we used in the case of the `AsyncTask` 
2. Alternatively, we could make our singleton hold `WeakReference` to the callbacks instead:

```java
public class FileManager {

    private static FileManager mInstance;

    public static synchronized FileManager getInstance() {
        if (mInstance == null) {
            mInstance = new FileManager();
        }
        return mInstance;
    }

    public FileManager() {
        mCallbackQueue = new ArrayDeque<WeakReference<DownloadCallback>>();
    }

    private ArrayDeque<WeakReference<DownloadCallback>> mCallbackQueue;

    public void download(String fileId, DownloadCallback callback) {
        mCallbackQueue.add(new WeakReference<DownloadCallback>(callback));
        // ..
        // do the download
        // ..
    }
}
```

In Java`inner` classes are a handy construct. The key is to understand how they work and ensure that they are not used (or rather "overused") in situations that can critically impact the memory.

For more references:

- http://www.curious-creature.org/2008/12/18/avoid-memory-leaks-on-android/
- http://evendanan.net/2013/02/Android-Memory-Leaks-OR-Different-Ways-to-Leak/
- https://developer.android.com/training/articles/memory.html