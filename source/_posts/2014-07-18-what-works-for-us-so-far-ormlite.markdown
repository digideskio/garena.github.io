---
layout: post
title: "What works for us so far - Ormlite"
date: 2014-07-18 13:33:30 +0800
comments: true
categories: [Android,BeeTalk]
author: Zhao Cong
author_about: zhao-cong
published: true
---

###[Ormlite](http://ormlite.com/sqlite_java_android_orm.shtml)

Ormlite is the our choice of database ORM. Although we have decided to stick to it for new Android applications, there are quite a few thing missing in the manual.

###Name the Database

Normally you name the databas with user ID like this.

```java

String databaseName = String.format("user_%d.db", mUserId);

```

Then you are screwed because you don't speak Persian/Arabian. In some languages, you cannot expect one as `1` and this [wiki post](http://en.wikipedia.org/wiki/Eastern_Arabic_numerals) will show you a new way to write numbers. You can reproduce this issue by switching the locale to Arabic on the phone and you must stay clam if the app cannot find the database. The fix is simple, just specify the locale when formatting the string.

```java

String databaseName = String.format(Locale.ENGLISH,"user_%d.db", mUserId);

```
<!-- more -->

###Name the Column

Well, you have the database now and go ahead to create tables. Take the example from the website:

```java

@DatabaseTable(tableName = "accounts")
public class Account {
    @DatabaseField(id = true)
    private String name;

    @DatabaseField(canBeNull = false)
    private String password;
    ...
    Account() {
    	// all persisted classes must define a no-arg constructor with at least package visibility
    }
    ...
}
```

If you are going to run proguard, then you may be screwed because the column changes. There are two solutions to this, either naming the columb explicitly or make all database object class excluded from proguard, as shown below.

```java

APPROACH ONE - Name the column explicitly

@DatabaseTable(tableName = "accounts")
public class Account {
    @DatabaseField(id = true,columnName = "name")
    private String name;

    @DatabaseField(canBeNull = false, columnName = "password")
    private String password;
    ...
    Account() {
    	// all persisted classes must define a no-arg constructor with at least package visibility
    }
    ...
}

APPROACH TWO - Modify the proguard file

-keep class com.XXXXXX.bean.**{*;}

```
###Are those results cached?

Now you have the table and it is time to query the data out from the database. ORM generally makes your life easier but it does not mean that the app can get the data with ease. Ormlite claims that the result is cached but you have to do the homework yourself by following [this guide](http://ormlite.com/javadoc/ormlite-core/doc-files/ormlite_5.html#Object-Caches).

However it only works for 4.48+, in 4.46, you may write code like this
```java
 mUserDao = DaoManager.createDao(getConnectionSource(), UserInfo.class);
 mUserDao.setObjectCache(true);
```

And we guarantee that it does NOT work at all.

### Update the table

Database scheme always changes because PM always changes their mind.

OrmLite provides callback for you to decide how to upgrade.

```java
@Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, ConnectionSource connectionSource, final int oldVersion, final int newVersion) {
        if (newVersion > oldVersion) {
    	   //run upgrade
        }
    }
```

We follow this guide [here](http://ormlite.com/javadoc/ormlite-core/doc-files/ormlite_4.html#Upgrading-Schema) and we were happy until PM changes his mind on how to contact the customers.

```java
 if (oldVersion < 8 && newVersion >= 8) {
 	TableUtils.createTableIfNotExists(connectionSource, Customer.class);
 }

 ...//A lot of thing happens here

 if (oldVersion < 12 && newVersion >= 12) {
      dao.executeRaw("ALTER TABLE `customer` ADD COLUMN contact_number VARCHAR DEFAULT ``;");
 }
```

We were screwed because the app could not complete the upgrade after upgrade. The issue here is that you should not use `createTableIfNotExists` to update the database. Let's assume that the user wants to upgrade his database from version 7 to version 8. He gonna hit the 1st statement to create the database and before the 2nd statement runs, he already has the column named <em>customer</em>. The definition of <em>Customer</em> changes over time! You should always write raw query to update the database becuase the query won't change.

```java
 if (oldVersion < 8 && newVersion >= 8) {
    dao.executeRaw("CREATE TABLE `customer` ......");
 }

 ...//A lot of thing happens here

 if (oldVersion < 12 && newVersion >= 12) {
 	dao.executeRaw("ALTER TABLE `customer` ADD COLUMN contact_number VARCHAR DEFAULT ``;");
 }
```

Besides OrmLite, there are plenty of options out there and we have not got a chance to get hands on them. We are interested to explore [SQLCipher](http://sqlcipher.net/) in the near future.