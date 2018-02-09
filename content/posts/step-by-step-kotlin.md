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

Our end goal for this step will be to make it print "Hello World" from the `./gradlew run` command.

Setting up the tool chain

We will need a `build.gradle` file that will look like this

```gradle
plugins {
    id 'org.jetbrains.kotlin.jvm' version '1.2.21'
}

apply plugin: 'application'

mainClassName = 'MainKt'

repositories {
    mavenCentral()
}

dependencies {
    compile 'org.jetbrains.kotlin:kotlin-stdlib-jre8'
}

sourceSets {
    main.kotlin.srcDirs += 'src'
}

```

Then we will need `src/Main.kt` file:

```kotlin
fun main(args: Array<String>) {
    println("Hello World")
}
```

You should now be able to run `./gradlew run` and see this:

```bash
>./gradlew run

> Task :run
Hello World


BUILD SUCCESSFUL in 5s
2 actionable tasks: 2 executed
```

## Step 3: Kotlin hello world inside docker

The next step will be to run this inside of a docker container. While this would be fairly straight forward, I want to demonstrate how to separate your build docker image from your running docker image. While this adds a little bit of complexity, this will actually be more like how you really use these kind of things.

Tweak the build to create a fat jar

```gradle
plugins {
    id 'com.github.johnrengelman.shadow' version '2.0.2'
}


jar {
    manifest {
        attributes 'Main-Class': 'MainKt'
    }
}
```


Add dockerfile

```dockerfile
ARG VERSION=8u111

FROM java:${VERSION}-jdk as BUILD

COPY . /src
WORKDIR /src
RUN ./gradlew --no-daemon -Dorg.gradle.parallel=false shadowJar

FROM java:${VERSION}-jre

COPY --from=BUILD /src/build/libs/kotlin-the-hard-way-all.jar /bin/runner/run.jar
WORKDIR /bin/runner

EXPOSE 8080

CMD ["java","-jar","run.jar"]
```

Run `docker build -t kotlin . && docker run kotlin`