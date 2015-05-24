---
layout: post
title: "Powered by Dagger 2"
date: 2015-05-19 17:15:10 +0800
comments: true
categories: [Android]
author: Zhao Cong
author_about: zhao-cong
published: true
---

There are a lots of buzz about Android architecture recently. We feel excited about the evolvement of Android development and decide to give a try for production. After a few trials and errors, we are settled with the combination of [Dagger 2](http://google.github.io/dagger/) and [Espresso](https://code.google.com/p/android-test-kit/wiki/Espresso). We feel happy with it and would like to share our experience with Dagger 2 in this post.

##Pros

We start with [Dagger 1](http://square.github.io/dagger/) from Square and loves the concept. It is a brand new way for us to get rid of ugly singleton pattern and structure the code in an elegant manner.  Additionally, it fundamentally resolves the issues of dependency and acquiring resource is not an problem any more. 


Later we learn about the even awesome `Dagger 2`, it immediately catches our attention for those features we actually cares. The code generation avoids the tricks of reflection and fixed class naming, thus 100% friendly to proguard. Nevertheless, developers still have to decide how to structure the code. The great flexibility is prone to over engineering and eventually you will find a deadlock when attempting to run unit tests. Dependency injection brings the advantage of data mocking and we start with simple use case. 

### Dagger for Development

Assume you want to initiate a Http request after the user clicks a button. The code below shows how it works without dependency injection. 

``` java
	    
    private LoginRequest mRequest;

    @onClick(R.id.login)
    void loginButtonOnClicked(){
     	 Task.callInBackGround {
	     Response rp = mRequest.post()
	 }. onSuccess {
	    //Update the UI
	 }
    }
```

It is very straight forward, but using the production server or testing server is slow and you may hit limited number of login per day etc. Dagger saves our lives by `Qualifier`. To start with, you define a qualifier to distinguish implementation for production and internal testing. 

```java
	
   @Qualifier
   @Retention(RUNTIME)
   public @interface MockMode {
       String value() default "";  //true/false
   }	
```

Then you can provide different implementation based on the qualifier.

```java
   
    public class MockLoginRequest extends LoginRequest {
        //skip the network call and fake the result
    }

    @Provides
    @MockMode("mock")
    LoginRequest getMockLoginRequest(OkHttpClient httpClient) {
        return new MockLoginRequest(httpClient);
    }

    @Provides
    @MockMode("prod")
    LoginRequest getPasscodeRequest(OkHttpClient httpClient) {
        return new LoginRequest(httpClient);
    }      

```

If you want to fire the fake `LoginRequest`
``` java
    
    @MockMode("mock")	    
    private LoginRequest mRequest;

```
or you want to connect to the real server

``` java
    
    @MockMode("prod")	    
    private LoginRequest mRequest;

```

In any case, you should not have debug code in production and Dagger is right there to offer the hand with gradle flavor.  You shall always have the definition `MockLoginRequest` in the debug flavor and define two modules in `release` and `debug` flavor. 

Only the production code will go into the `release` module.

```java   

    @Provides
    @MockMode("prod")
    LoginRequest getInitPasscodeRequest(OkHttpClient httpClient) {
        return new LoginRequest(httpClient);
    }      

```
Offer more choice in debug flavor only.

```java
   
@Provides
@MockMode("mock")
LoginRequest getMockLoginRequest(OkHttpClient httpClient) {
    return new MockLoginRequest(httpClient);
}

@Provides
@MockMode("prod")
LoginRequest getInitPasscodeRequest(OkHttpClient httpClient) {
    return new LoginRequest(httpClient);
}      
```

If you misuse the qualifier, the compilation will fail and you will immediately know that something goes wrong. 

### Dig Deeper

You need to structure the code around Dagger. Generally we have 

- `AppComponent`: base component for sub components
- `AppModule`: provides application context
- `UserModule`: provides user information, i.e. `LoginSession`
- `NetworkModule`:  provides implementation of network class, i.e `OkHttpClient`,`Gson`
- `UIModule`:  provides implementation for naviagtion
-  Activity specific modules: provides domain related resources, i.e. `LoginRequest`, `PurcahseRequest`

```java AppComponent.java
@ApplicationScope
@Component(
        modules = {AppModule.class, NetworkModule.class, AppUIModule.class, UserModule.class}
)
public interface AppComponent {

       Gson getGson();
       //other implementation
}

```

In sub components, utilize the domain-related module for injection

```java HomeModule.java
@Module
public class HomeModule {

    @Provides
    @MockMode("prod")
    GetUserInfoRequest getUserInfoRequest(OkHttpClient httpClient, Gson gson, LoginSession session) {
        return new GetUserInfoRequest(httpClient, gson, session);
    }

    //offer other modules
}
```
Quote the module in the component

```java HomeComponent.java
@HomeScope
@Component(
        dependencies = {AppComponent.class},
        modules = {HomeModule.class})
public interface HomeComponent {
    void inject(HomeActivity activity);
}
```

So when can we get hold of the component and perform the injection?

You definitely want everything ready before the resource is needed. The activity component should be created at the very beginning of the activity life cycle, i.e. `onCreate` for `Activity` and the activity performs the injection to views or UI presenters immediately after their new birth. As you may be aware, there are two ways of injection, either injection via constructor or injection via annotation. In general, we favor injection via annotation due to flexibility. 

### Even Better in Testing

Dagger brings much more flexibility to testing as you can easily swap the implementation via injection. For simple structure, as long as you can grab the reference to the object, you can perform injection. 

```java

        //input the sms
        onView(withId(R.id.cp_sms_code_input)).perform(typeText("111111"));

        //inject the fake sms authentication
        LoginSMSAuthActivity smsAuthActivity = (LoginSMSAuthActivity) getActivityInstance();
        mockModule.inject((LoginSMSAuthActivity.UIPresenter) smsAuthActivity.presenter());

        //click the button
        onView(withId(R.id.cp_sms_confirm_btn)).perform(click());

```
The combination of Dagger and Espresso is just awesome but it is never short of work around and fixes. We shall talk more about this topic in the next post.

## Cons

### More Methods
Dagger will increase the method count without grace. If you find the size of dex is growing out of control due to massive feature requirement, do think twice before triggering the magic of code generation. 

### No Second Chance
`Volley` is interchangeable with `OkHttp`, you can always switch implementation if one outperforms another. However,  once you design the app around `Dagger`, you have to stick to it since there is no alternative available to replace the code structure. Vice versa, it is going to take enormous effort to adapt Dagger half way. In other words, you have to make your mind when you start the project.  






 

 

