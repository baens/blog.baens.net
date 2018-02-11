---
title: "Step-by-step: Kotlin and Docker"
date: 2018-02-11
weight: 2
---

This is part of a series of blog posts that I've always wanted todo. [Read about them here](https://blog.baens.net/posts/step-by-step-series/).

For this step-by-step guide let's do something basic and get kotlin compiling and running inside a docker container.

[The companion repository is here]( https://github.com/baens/blog-step-by-step-kotlin).

## Step 0: Prerequisites

I am going to assume you have the following installed and already running. We will be using the local gradle wrapper instance after the initial setup, but you do need it for the initial creation.

The other one I am going to assume is having a running docker instance. 

* Java (JDK 1.8.0_144)
* Gradle (to create the initial wrapper)
* Docker (17.12.0-ce-mac49 (21995))

## [Step 1: Gradle](https://github.com/baens/blog-step-by-step-kotlin/commit/7572d1f5f186a3d3345e70f0935bd4be8d3badb1)

The goal of this step will be to get a gradle wrapper up and running so that we can lock in the gradle version for everyone that will be using this repository. 

Run the following:

`> gradle wrapper --gradle-version 4.5`

Verify it works by running:

```
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

That's it! You can double check that commit that I have associated with this step to see what I add to a git repository for this but this really is an easy step. 

## [Step 2: Kotlin Hello World locally](https://github.com/baens/blog-step-by-step-kotlin/commit/0071055905e749427dc610e8f34000cd746458de)

Our end goal for this step will be to make it print "Hello World" from the `./gradlew run` command.

Let's first setup our tool chain. This will be just a gradle file with all of the plugins and dependencies that we will need. 

**build.gradle**:

{{< highlight gradle "linenos=table" >}}
/**
 * This section is for all of the plugins we need to make this work. 
 * Kotlin, the fat jar builder, and flag it 
 * as an application so `./gradlw run` will work
 */
plugins {
    id 'org.jetbrains.kotlin.jvm' version '1.2.21'
    id 'com.github.johnrengelman.shadow' version '2.0.2'
}

apply plugin: 'application'


/**
 * Standard dependency section for gradle. 
 * Define the kotlin standard libray for
 * Java 8.
 */
repositories {
    mavenCentral()
}

dependencies {
    compile 'org.jetbrains.kotlin:kotlin-stdlib-jre8'
}

/**
 * Personal preference. I hate having src/main/kotlin
 * be the root, so I change it that 'src'
 * is the root of my source directory.
 */ 
sourceSets {
    main.kotlin.srcDirs += 'src'
}

// Define the main startup class and jar name
mainClassName = 'MainKt'
archivesBaseName = 'step-by-step-kotlin'

// tell the jar which class to startup in.
jar {
    manifest {
        attributes 'Main-Class': 'MainKt'
    }
}   
{{< / highlight >}}

Let's go through some of those lines again and draw out whats going on.

**Lines 6 - 11**: These are just plugin lines. Gradle has a couple of ways to define plugins. One is called the `plugin DSL` the other is called `script plugin`. I'm not going to try and explain the different here, please go read [the plugins doc](https://docs.gradle.org/current/userguide/plugins.html) to get a better idea. One plugin to note is the `shadow` plugin. This one is to create Fat Jars, or in other words, jars that contain every dependency that is needed to run. Instead of having to make sure things are on your classpath or some other nonsense, I can just give you this jar file, and it will run. This will make it running inside of docker easier.

**Line 24**: Here we are defining the kotlin standard for Java 8. This is newer in Kotlin 1.1 and you can read about it [here](https://kotlinlang.org/docs/reference/using-gradle.html#configuring-dependencies). There are a few ways to do this, but this was specific to what was needed for Java 8. I've run into issues using the backwards compatible one and try and use the Java 8 one so that the newer features actually work 

**Lines 32 - 34**: This is entirely optional, but this is how I prefer to setup my projects. I hate the default of `src/main/java` or something silly like that. That is 3 folders I have to start under just to get to my source. PLUS then I have to navigate down through the folders of my namespace. Rarely do projects have to have all of these different types of languages so you can eliminate the `java` folder. And 2nd, having to maintain a `test` source folder and a `main` source folder is the PITA. I much prefer to have my tests and source side by side. It just makes it easier to find my tests that are associated with each file.  Again, this is all personal choice but I've been doing this long enough to feel strongly about this one now.

**Line 37**: This is a simple line but I did want to call out something. `MainKt` is special because it will be the name of the file with `kt` at the end of it. I will explain a little bit more when we get to that source. But this is a convention Kotlin has when there is just a method inside of the file. It will automatically enclose that method in a class for the JVM to use. So be careful about that `kt`. 

 
Now for the actual, very simple, hello world code.

**src/Main.kt:**
```kotlin
fun main(args: Array<String>) {
    println("Hello World")
}
```

I want to point out and congratulate the Kotlin team on this. If you have ever done any Java programming, you know that you always need a class per file. Kotlin has a few conventions to make certain things easier and I think this is brilliant. I love things like this that just remove the pomp and circumstance. This is especially helpful when trying to get new developers into the language. Every time I've had to teach someone Java, the `public class Whatever` was foreign and we had to go down a rabbit hole even before we had anything running.  

You should now be able to run `./gradlew run` and see this:

```
>./gradlew run

> Task :run
Hello World


BUILD SUCCESSFUL in 5s
2 actionable tasks: 2 executed
```

## [Step 3: Kotlin hello world inside docker](https://github.com/baens/blog-step-by-step-kotlin/commit/77f3f0add270a086ac1fd46f673db74d1dbd7012)

The next step will be to run this inside of a docker container. While this would be fairly straight forward, I want to demonstrate how to separate your build docker image from your running docker image. While this adds a little bit of complexity, this will actually be more like how you really use these kind of things.

Since we have everything working, we just need to get a Dockerfile created and running.

**Dockerfile**
{{< highlight docker "linenos=table" >}}
ARG VERSION=8u111

FROM java:${VERSION}-jdk as BUILD

COPY . /src
WORKDIR /src
RUN ./gradlew --no-daemon shadowJar

FROM java:${VERSION}-jre

COPY --from=BUILD /src/build/libs/step-by-step-kotlin-all.jar /bin/runner/run.jar
WORKDIR /bin/runner

EXPOSE 8080

CMD ["java","-jar","run.jar"]
{{</ highlight >}}

**Line 3 - 9**: This is the "builder" section. This is one docker instance.

**Line 9 - 14**: This it the section that will be actually running.

This is an awesome way to wrap up in one docker file how to build and create the image inside of itself, as well as run it. What I try and do is make it so you can create this docker image from nearly anywhere. This builder image includes all of the tools needed to build things out. And since we have this gradle wrapper, the toolchain is also there. 

The 2nd part is what is actually running. The `COPY --from=BUILD` is the magic that we can pull from the last image (note: `BUILD` is from line 3 `as BUILD`). 

I'm not going to go too much of what exactly docker is doing but do want to point out a couple of quick things. We basically do a copy, setup what is the working directory (think `cd` if you were using command line), then do the work or run.

Run this and the image will build and run:

```shell
docker build -t kotlin . && docker run kotlin
```