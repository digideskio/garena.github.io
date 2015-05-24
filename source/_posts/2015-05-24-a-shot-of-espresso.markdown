---
layout: post
title: "A Shot of Espresso"
date: 2015-05-24 11:37:22 +0800
comments: true
categories: [Android]
author: Zhao Cong
author_about: zhao-cong
published: true
---

The days without testing in Garena is long gone.  Our QA team has done a great job to provide continuous integration for BeeTalk and now it is the time for the engineering team to catch up.  So we set time aside to explore the continuous integration for our new Project.  Apparently Google has promoted [Espresso](https://code.google.com/p/android-test-kit/wiki/Espresso) as the official framework for Android integration testing and it becomes our choice without second thought. 

## Configuration 

We found Espresso easy to set up and the integration is just smooth. Add the dependency to `build.gradle`
``` java
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.1'){
        exclude group: 'javax.inject' //support 2.3
    }
    androidTestCompile 'com.android.support.test.espresso:espresso-intents:2.1'
    androidTestCompile 'com.android.support.test.espresso:espresso-contrib:2.1'
    androidTestCompile 'com.android.support.test:runner:0.2'
```

For Android studio, simply create a new test configuration and it is all set. 

![Espresso Configuration](/images/espresso.png)

Of course, you shall put all your test cases in `/src/androidTest`. 

## Write test cases

We are interested to write test cases to cover basic user operation, including login, registration and purchase, in which many network requests are involved.  We aim to achieve three things:

- Robust:  test cases should always work 
- Lightweight:  zero impact or trivial modification to production code
- Efficient:  lower down the cost to run the test cases

#### Robustness
 
Our test cases are designed to cover the workflow with expected execution in mind.  Since we cannot run the logic without networking interaction, we decide to mock every single network request via our request wrapper.  For example, `TokenLoginRequest` is our class wrapper to fire a login request. 

```java
public class TokenLoginRequest extends EndpointHttpRequest {

    private String mMobileNumber, mToken;

    public TokenLoginRequest(OkHttpClient httpClient) {
        super(httpClient);
    }

    public void setLoginInfo(String mobile, String token) {
        mMobileNumber = mobile;
        mToken = token;
    }

    @Override
    protected RequestBody body() {
        return new FormEncodingBuilder().
                add("token", mToken).
                add("mobile", mMobileNumber).
                build();
    }

    @Override
    public String getServiceName() {
        return "services2/login/token/";
    }
}
```

To mock it, simply override the implementation 

```java
public class MockTokenLoginRequest extends TokenLoginRequest {

    private Gson gson = new Gson();

    public MockTokenLoginRequest(OkHttpClient httpClient) {
        super(httpClient);
    }

    @Override
    public Response execute() throws Exception {
        CommonResponse response = new CommonResponse();
        response.errorCode = NetworkConst.OK;
        response.errorMessage = "ok";
        return ResponseFactory.getJsonResponseBuilder(gson.toJson(response), postRequest());
    }
}
```

With mocked API request, there is no way to fail a case due to unstable WIFI network. How to put the mock class in action? There are two ways, either via a factory class or perform injection. We utilize Dagger and define test modules to replace the default implementation

```java
    @Test
    public void loginViaOtp() throws Exception {
        Intent intent = new Intent();
        intent.putExtra("number_ui", MockOTPValidationRequest.STANDARD_LOGIN);
        intent.putExtra("number_e164", MockOTPValidationRequest.STANDARD_LOGIN);

        mActivityRule.launchActivity(intent);

        //input the sms
        onView(withId(R.id.cp_sms_code_input)).perform(typeText("111111"));

        //inject the fake sms authentication
        LoginSMSAuthActivity smsAuthActivity = (LoginSMSAuthActivity) getActivityInstance();
        mockModule.inject(smsAuthActivity.presenter());

        //click the button
        onView(withId(R.id.cp_sms_confirm_btn)).perform(click());
        //....
    } 
```

You may not have the chance to perform the injection so easily due to complicated hierarchy but it is essential to keep the footprint small. In test cases, I really don't mind playing dirty tricks like reflection. 

### Lightweight

Test cases cannot pose any issue to the production code.  Gradle has offered great flexibility on how to separate debug code and production code, so does the case of testing. 

#### Code Strucure

`/src/androidTest`

- hold all test cases
- supporting library 

`/src/debug`

- mock classes

`/src/release`

- no mocking classes

#### Injection 
As stated above, we provide mock implementation via injection but sometimes it is not that straight forward. In those case in which you have to inject the fake implementation at the early stage of the activity life cycle, we simply override the activity and inject as early as possible. In other cases, you may not want the navigation happen because you cannot inject an activity which does not exist.  The code snippet below shows how we handle both cases. 

```java
public class TestLoginActivity extends LoginActivity {

    ActivityScreenSwitcher mScreenSwitcher;

    @Override
    protected void onCreateComponent(AppComponent appComponent) {
        //use the real app module
        AppModule module = new AppModule(CyberpayAdminApp.getApp());
        DebugAppUIModule uiModule = new DebugAppUIModule();
        //get the real login session
        UserModule userModule = new UserModule(CyberpayAdminApp.getApp().component().getLoginSession());
        //customized app component
        DebugAppComponent debugAppComponent = DaggerDebugAppComponent.builder().appModule(module).userModule(userModule)
                .debugAppUIModule(uiModule).build();

        mScreenSwitcher = debugAppComponent.activityScreenSwitcher();
        mActivityComponent = DaggerMockRoutineComponent.builder().appComponent(debugAppComponent).
                        mockRoutineModule(new MockRoutineModule()).build();
        //perform the injection with mock implementation
        mActivityComponent.inject(this);
    }
    //......
}
```
`mActivitiyComponent` will be used to inject everything inside the activity, i.e. views, presenters and we provide `DebugAppUIModule` to intercept the intent for navigation and save for verification. In the test cases, you should invoke `TestLoginActivity` instead of `LoginActivity`.  At the first sight it looks scary, because even new **hacked** activities are introduced. Well, there is no chance to have them in the production because you shall have everything under the flavor of `debug`. 

#### Async Trick

People always wonder how to test something asynchronous from the user perspective, i.e fire a network request and the activity moves to the next step. Espresso supports `AsyncTask` in its core implementation and offers flexibility to other async/task management framework. [Bolts]() is our choice of task management and our code is structured like this 
```java
mContinueBtn.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    getView().showLoading();
                    mLoginRequest.setNumber(input);
                    Task.callInBackground(new Callable<LoginInfo>() {
                          @Override
                          public LoginInfo call() throws Exception {
                                //make the request
                                return mLoginRequest.getResponse();
                          }
                    }).continueWith(new Continuation<LoginInfo, Void>() {
                          @Override
                          public Void then(Task<LoginInfo> task) throws Exception {
                                 getView().loadingView.setRefreshing(false);
                                 if (task.isFaulted()) {
                                     activity.resolveError(task.getError());
								 }
                                 //perform action 
                                 //......
                                return null;
                    }, Task.UI_THREAD_EXECUTOR));
			
``` 

We follow the [recommendation from Google](https://code.google.com/p/android-test-kit/wiki/EspressoSamples#Using_registerIdlingResource_to_synchronize_with_custom_resource) but later we found that [IdlingResource](http://developer.android.com/reference/android/support/test/espresso/IdlingResource.html) is poorly documented. 

> `isIdleNow()` returns true if resource is currently idle. Espresso will always call this method from the main thread, therefore it should be non-blocking and return immediately.

You may ask why I should care about this. Well, here is the core idea behind this helper class. The system will check on it time by time after registration. Assume you have a state variable `mIsBusy` and `isIdleNow` will return the value of `mIsBusy`, you should always modify the value `mIsBusy` from the main thread, otherwise there is no guarantee that the test runner will stop before the asynchronous call completed.  Ideally, you shall have test case like this 

```java
    @Test
    public void loginTest() {
       
      
        //attempt login 
        onView(withId(R.id.login_button)).perform(click());
        
        //FIRE NETWORK REQUEST......
        //WAIT UNTIL THE SERVER RETURNS THE RESULT
      
        //verify which activity it is now
        assertTrue(getActivityInstance() instanceof HomeActivity);
    }
```

You cannot change the `mIsBusy` before you click the login button, otherwise nobody will trigger the login action. You cannot change the value after the login button because you don't know when the network response will modify the value.  It seems to be a deadlock and the solution is to change the value right after the user clicks the button. 

```java

mContinueBtn.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    
                    //MODIFY THE mIsBusy FROM MAIN THREAD
                    mIsBusy = true;
                    
                   //fire network request
                });
```

A static variable cannot serves well for multiple network requests in sequence (they cannot decide when to flip the value). Additionally the production code should be unaware of `IdlingResource` presence.  Thus we comes with a helper class in `main` sources. 

```java 
public class NetworkStatus {

    private static long mStartTime;

    private static int mCount = 0;

    /**
     * Mark a new checkpoint of background requests, must call via UI thread.
     */
    public static void onNetworkRequestStart() {
        if (!BuildConfig.DEBUG) {
            return;
        }
        synchronized (NetworkStatus.class) {
            mCount++;
            mStartTime = BBTimeHelper.millNow();
        }
    }

    /**
     * Mark a new checkpoint of background requests, recommend to call via UI thread but not necessary to do so
     */
    public static void onNetworkRequestEnd() {
        if (!BuildConfig.DEBUG) {
            return;
        }
        synchronized (NetworkStatus.class) {
            mCount--;
            BBAppLogger.i("execution time %d", BBTimeHelper.millNow() - mStartTime);
        }
    }

    /**
     * Accessed via UI thread only
     *
     * @return true if no network task is ongoing
     */
    public static boolean isIdle() {
        return mCount == 0;
    }
}
```

and subclass the `IdlingResource` in `androidTest`

```java
public class NetworkIdlingResource implements IdlingResource {

    @Override
    public String getName() {
        return "network_mock";
    }

    @Override
    public boolean isIdleNow() {
        boolean idle = NetworkStatus.isIdle();
        if (idle && mCallback != null) {
            mCallback.onTransitionToIdle();
        }
        return idle;
    }

    private ResourceCallback mCallback;

    @Override
    public void registerIdleTransitionCallback(ResourceCallback resourceCallback) {
        mCallback = resourceCallback;
    }
}
```

Thus the production code should call `NetworkStatus.onNetworkRequestStart` from main thread before firing a network request and call `NetworkStatus.onNetworkRequestEnd` from main thread after a network requests completes.  Trivial modification is made and goal accomplished. 

### Efficient 
Android fragmentation scares people and many developers only test their application on physical devices. Ironically, most of bugs are caused by logic errors rather than device variance.  We choose Genymotion for continuous integration with [Jenkins](https://jenkins-ci.org/). Here is the setup

- Ubuntu 14.04 LTS on a lower end Dell laptop
- 3 Genymotion instances (Android 4.3, 4.4, 5.1)
- Connect to Jenkins machine with direct cable 

Genymotion is much faster than physical devices and the three instance can fire concurrently. 63 tests can finish with 3 minutes. 

![Espresso with Jenkins](/images/jenkins_espresso.png)

You can add as many VM or physical devices as you want and trigger the testing via 
```groovy
./gradlew connectedAndroidTestDebug

```

 ## Conclusion
Our journey with Espresso or continuous integration does not end here and we are working on a couple of things like 

- write test cases against our server API
- apply espresso to test shared libraries
- test upon different flavors 