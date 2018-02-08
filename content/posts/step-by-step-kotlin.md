---
title: "Step-by-step: Kotlin and Docker"
date: 2018-02-05
---

So, I know there are a lot of tools out there that just get things rocking for you. But sometimes, you just need to know how all the bits are put together. 

So with that in mind, let's get a HTTP REST service, written in Kotlin and Jersey. And can be used inside of a docker container. 


Repo: https://bitbucket.org/baens/kotlin-the-hard-way

## Step 0: Prerequisites

* Java (JDK 1.8.0_144)
* Gradle (to create the initial wrapper)
* Docker (17.12.0-ce-mac49 (21995))

## Step 1: Gradle

For this step we will get a gradle wrapper setup and working inside of the repository



Run the following:

`> gradle wrapper`

Verify it works by running:

```shell
> ./gradlew -v

------------------------------------------------------------
Gradle 4.5
------------------------------------------------------------

Build time:   2018-01-24 17:04:52 UTC
Revision:     77d0ec90636f43669dc794ca17ef80dd65457bec

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.9 compiled on February 2 2017
JVM:          1.8.0_144 (Oracle Corporation 25.144-b01)
```

## Step 2: Kotlin Hello World locally

Our end goal for this step will be to make it print "Hello World" from the gradle run command.

Setting up the tool chain

We will need a build.gradle file that will look like this

```gradle

```

## Step 3: Kotlin hello world locally

## Step 4: Kotlin hello world inside docker