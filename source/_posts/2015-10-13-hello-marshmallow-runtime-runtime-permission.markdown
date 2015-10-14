---
layout: post
title: "Hello Marshmallow - Runtime Permission"
date: 2015-10-13 21:02:49 +0800
comments: true
categories: 
author: Zhao Cong
author_about: zhao-cong
categories: [Android]
published: true
---

We never care about new release of Android operating system. Three reasons. Firstly, there were hardly any broken changes and the new operating system promised to be compatible with the legacy. Secondly, it took months for the non-Nexus user to get the OTA and the adaption for new OS happens in a slow pace, i.e. Lollipop only hits around 20% in the global Android distribution. Thirdly, we don't have enough manpower to bother with new OS and engineers is occupied by one feature after another. However, the new Android 6.0 really grab our attention since it broke two barriers: the new operating system introudces breaking changes with the new permission system and the slighly bigger team want to deal with it. 

## Timeline 

OTA for Nexus devices kicks off at the first week of October 2015 and many more manufactuers pledged their support by the end of 2015. So roughly we have *three months* to deal with the issues. 

## What's happening?

Quote from [offical dashboard](https://developer.android.com/about/dashboards/index.html). 

![Offical Dashboard](/images/android-dashbord-2015-oct.png)

Our initial thoughts are

- Southeast Asia is in an older OS compared to the rest of the world.
- The coverage of the latest OS is not significant in the countries we do business with. 

It turns out our assumptions do not hold at all. We choose one app which is being distributed in Thailand and get the distribution data from Google Play developer console. Apparently people in Thailand are not stick to old fashioned operating system and we should worry about what gonna happen in three months' time. 

![Global VS Thailand](/images/th-android-app-comparison.png)

## What's new in Marshmallow?

The full list can be found [here](https://developer.android.com/about/versions/marshmallow/android-6.0-changes.html). Two things get our attention:

- Runtime permission
- Doze & app standby

This post will focus one Runtime Permission. 

There are tons of resources about runtime permission.

- [Video - Runtime permission](https://www.youtube.com/watch?v=C8lUdPVSzDk)
- [Video - Asking for Permission](https://www.youtube.com/watch?v=iZqDdvhTZj0)

Well, we care about what is hidden under the hood. 

## Permission or Permissions

The documentation never specifies the behavior when you request permissions in one Permission Group. For example, you want to access the location so that you check the permission `ACCESS_FINE_LOCATION` and `ACCESS_COARSE_LOCATION`. 
```java
 if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED ||
                ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED){
            // location permissions have not been granted.
            requestLocationPermission();
            return;
        }

```

Then you ask for permission. 

```java
	private static String[] PERMISSIONS_LOCATION = {Manifest.permission.ACCESS_FINE_LOCATION, Manifest.permission.ACCESS_COARSE_LOCATION};

	private void requestLocationPermission() {

        if (ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.ACCESS_FINE_LOCATION)){

            // Provide an additional rationale to the user if the permission was not granted
            // and the user would benefit from additional context for the use of the permission.
            // For example, if the request has been denied previously.

            // Display a SnackBar with an explanation and a button to trigger the request.
            Snackbar.make(mainLayout, R.string.track_down_wherever_you_are, Snackbar.LENGTH_INDEFINITE)
                    .setAction(R.string.ok, new View.OnClickListener() {
                        @Override
                        public void onClick(View view) {
                            ActivityCompat.requestPermissions(HomeActivity.this, PERMISSIONS_LOCATION,
                                    ACCESS_LOCATION_PERMISSION_REQUEST_CODE);
                        }
                    })
                    .show();
        } else {
            // Contact permissions have not been granted yet. Request them directly.
            ActivityCompat.requestPermissions(this, PERMISSIONS_LOCATION, ACCESS_LOCATION_PERMISSION_REQUEST_CODE);
        }
    }
```

and we implement the callbacks

```java
   
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                                           @NonNull int[] grantResults) {
        if (requestCode == ACCESS_LOCATION_PERMISSION_REQUEST_CODE){
            if (verifyPermissions(grantResults)){
                //perform the action
                acquireLocation();
            }else{
                Snackbar.make(mainLayout, R.string.cannot_access_location, Snackbar.LENGTH_SHORT).show();
            }
        }
    }

    public static boolean verifyPermissions(int[] grantResults) {
        // At least one result must be checked.
        if (grantResults.length < 1) {
            return false;
        }

        // Verify that each required permission has been granted, otherwise return false.
        for (int result : grantResults) {
            if (result != PackageManager.PERMISSION_GRANTED) {
                return false;
            }
        }
        return true;
    }

```

This code won't work becuase the `verifyPermissions` returns false!

According to the definition of [permission group](https://developer.android.com/training/permissions/requesting.html)

> The user only needs to grant permission once for each permission group. If your app requests any other permissions in that group (that are listed in your app manifest), the system automatically grants them. 

What they miss is that once the group permission is granted, the grant results for other permission in one group must be `PackageManager.PERMISSION_DENIED`.

However, you cannot put one representative permission when you request access for the permission group, as Google clearly claims

> Your app still needs to explicitly request every permission it needs, even if the user has already granted another permission in the same group. In addition, the grouping of permissions into groups may change in future Android releases. Your code should not rely on the assumption that particular permissions are or are not in the same group.

In conclusion, you shall never trust the `grantResults` and always check the permission one by one.  

## Alternative

Apparently Google recommends app not to ask for permission if unnecessary. However if the app does not have the `CAMERA` perssimion, it cannot fire intent to get the default app running. We are not sure if this is a bug or not. 

## Threading

Our demo app shows that it is fine to check the permission on background thread as long as you have a context. However, you cannot ask for permission without an activity as the API available is 

```java
public static void requestPermissions (Activity activity, String[] permissions, int requestCode)
```

You need an **activity** to ask for permissions.

Additionally, it is possible to call `requestPermission()` on the background thread and the dialog will show up. However since the result will reach you via the callback `onRequestPermissionsResult` on main thread,  you shall avoid calling the `requestPermission` on background thread to avoid nasty threading issue. 

## Upgrade Strategy

Initially we believe Google will take the approach like other 3rd party ROM does: if you don't get the permission, you will get nothing back and the UI moves on. However it is not the case. If you don't have the access the location, query the location API will break the app. 

```java
java.lang.SecurityException: "network" location provider requires ACCESS_COARSE_LOCATION or ACCESS_FINE_LOCATION permission.
    at android.os.Parcel.readException(Parcel.java:1599)
    at android.os.Parcel.readException(Parcel.java:1552)
    at android.location.ILocationManager$Stub$Proxy.requestLocationUpdates(ILocationManager.java:606)
    at android.location.LocationManager.requestLocationUpdates(LocationManager.java:880)
    at android.location.LocationManager.requestLocationUpdates(LocationManager.java:464)
    at com.garena.hellomarshmallow.HomeActivity.acquireLocation(HomeActivity.java:63)
    at com.garena.hellomarshmallow.HomeActivity.access$000(HomeActivity.java:15)
    at com.garena.hellomarshmallow.HomeActivity$1.onClick(HomeActivity.java:35)
    at android.view.View.performClick(View.java:5198)
    at android.view.View$PerformClick.run(View.java:21147)
```

This forces us to rethink whether we should make app target to the latest SDK version. 

- Don't target to API level 23 if you are not ready
- Feel free to update the SDK but keep the `targetSdkVersion` to 22 or lower

```
android {
    compileSdkVersion 23
    buildToolsVersion "23.0.1"

    defaultConfig {
        applicationId "com.garena.hellomarshmallow"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
}
```
In this case, you will find all required permissions granted by default. 

![Permission Granted](/images/permission-api-22.png)

However, we shall have **NO** execuse keeping this configuration as user can turn off the permission if they determine to do so. 


## The Most Challenging Part 

Engineers need to convince the product managers that you need to invest a huge amount of time on something not a feature. 

This is not a feature, but a death sentence in the near future. 


