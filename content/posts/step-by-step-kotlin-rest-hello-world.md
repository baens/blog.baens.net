---
title: "Step-by-step: Kotlin, JAX-RS (Jersey), and Docker"
date: 2018-02-13
weight: 2
series: ["step-by-step"]
tags: ["kotlin"]
---

Hello Again and welcome to another blog post in my ongoing series on step-by-step tutorials. This one will actually build atop the last Kotlin one and make a JAX-RS web service.

## Step 0: Prerequisites
For this one, I am going to start off with my Kotlin Hello World repository. This already has a gradle wrapper setup, along with some basic things. I've changed `mainClassName`  to `Server.kt` and `archivesBaseName` to match this projects name.

The `Dockerfile` has also been changed to make it so `docker build -t myserver . && docker run myserver` still works.

## Step 1: Web Server
First step will be to get a working web server up and running. This project will be using the grizzly web server, which I recommend, as the embedded web server. It has a decent enough history of performance and from my usage it does a decent job.

So let's take a look at how a quick server can be setup so we can verify we have the HTTP pipeline correct.

**src/Server.kt**
{{< highlight kotlin "linenos=table" >}}
import org.glassfish.jersey.grizzly2.httpserver.GrizzlyHttpServerFactory
import javax.ws.rs.core.UriBuilder

fun main(args: Array<String>) {
    val url = UriBuilder.fromUri("http://0.0.0.0/")
            .port(8080)
            .build()

    val httpServer = GrizzlyHttpServerFactory.createHttpServer(
            url,
            true
    )

    if (System.getenv().get("SHUTDOWN_TYPE").equals("INPUT")) {
        println("Press any key to shutdown")
        readLine()
        println("Shutting down from input")
        httpServer.shutdownNow()
    } else {
        Runtime.getRuntime().addShutdownHook(Thread {
            println("Shutting down from shutdown hook")
            httpServer.shutdownNow()
        })

        println("Press Ctrl+C to shutdown")
        Thread.currentThread().join()
    }
}
{{< /highlight >}}

**Line 6**: This creates a URI for the server to bind to. This particular one makes it so `http://localhost:8080` will work. Probably overkill but that is what the Grizzly Server factory wants.

**Line 9**: This creates the Grizzly Server and the 2nd parameter (`true`) makes the server start immediately.

**Lines 14 - 27**: Now, let me explain this because this is sort of magic. When running under gradle (`./gradlew run`), it captures all of the input and passes it along to the application. So for example, on the command line you usually stop applications by hitting `Ctrl+C`. This however doesn't work when you run it under Gradle because it captures it and stops the daemon that is running the application. This can be a gotcha because sometimes the Gradle daemon doesn't shutdown cleanly.

So my back to fix all of this was to read an environment variable that flags which method I should use. When running under Gradle (hold on one sec and I'll show you how) we will just wait for input. When running under everything else (i.e. docker, `java -jar`) we will wait for Ctrl+C or a kill signal from somewhere else. 

Now for the toolchain bits, we need to add the dependencies and setup the run bits to work with out shutdown hooks.

**build.gradle (only parts)**
{{< highlight gradle "linenos=table" >}}
...
dependencies {
    compile 'org.jetbrains.kotlin:kotlin-stdlib-jre8'
    compile "org.glassfish.jersey.containers:jersey-container-grizzly2-http:2.26"
}
...
// Properties for `./gradlew run`
run {
    standardInput = System.in
    environment = ["SHUTDOWN_TYPE" : "INPUT"]
}
{{< /highlight >}}

**Line 4**: We add the grizzly http server and Jersey container.

**Line 8 - 11**: This is how we make `./gradlew run` work. The `standardInput` we set to the normal `System.in` (it defaults to an empty set which breaks things). Then we setup the environment variables with the `environment` configuration.

So verify the two ways to run the application and view `http://localhost:8080`.

First run `./gradlew run` and if you hit any key, it will shutdown.

Second run the docker command `docker build -t myserver . && docker run -p 8080:8080 myserver`. This you can run and then hit *Ctrl+C* to shut it down. 

Quick aside on the docker command: there are two parts to it, first you build the image you want to run, then you run it. The *-t* you pass in the build command is a tag that makes it easier to look up later. In the run part the *-p* is the ports we will bind to the host. For this example we are taking the containers's 8080 port mapping it to the hosts 8080 port. 

## Step 2: Add JAX-RS
Add Jersey to the mix