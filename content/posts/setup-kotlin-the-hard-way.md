---
title: "Setting up Kotlin the hard way"
date: 2018-02-05
---

So, I know there are a lot of tools out there that just get things rocking for you. But sometimes, you just need to know how all the bits are put together. 

So with that in mind, let's get a HTTP REST service, written in Kotlin and Jersey. And can be used inside of a docker container. 


Repo: https://bitbucket.org/baens/kotlin-the-hard-way

## Step 1: Gradle

The very first thing we are going to need is Gradle. This is what we are going to use as our build toolchain. I'm not going to elierabte much more on what gradle can and can't do so go visit the links if you want that.

The end goal will be to have a gradle wrapper installed and working.

Download a gradle distrubtion from here: https://services.gradle.org/distributions/ (pick on that says `-bin`)

Or use your favorite package manager to download it.

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
OS:           Mac OS X 10.13.3 x86_64
```

## Step 2: Kotlin Hello World locally

Our end goal for this step will be to make it print "Hello World" from the gradle run command.

Setting up the tool chain

We will need a build.gradle file that will look like this

```gradle

```

## Step 3: Kotlin hello world locally

## Step 4: Kotlin hello world inside docker