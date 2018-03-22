---
title: "Notes from experimenting with GraphQL and Kotlin"
date: 2018-03-22
---

# What is this GraphQL thing

I have finally got time to try out GraphQL. I've been reading about it more and more and wanted to get a sense of how to build a data API with it. Now, if you don't know what GraphQL is, here is my take on it. It is a way to create data APIs that allow clients to tell you what data they actually need. Instead of you trying to design up front all of your use cases and force clients to use that. You let them tell you what data they need, when they need it. This makes it so you as the data designer just expose the data that you have and expose ways to filter the data and the relationships that are inside the data. You don't care how the client uses it, you just expose the data as you have it. I believe this is how a data API should be designed. This decouples the why from the how which usually empowers users of the API and makes it more useful longer term.

# Getting started with GraphQL on Kotlin

My first approach was to create a Jersey web application that exposed the correct end points for GraphQL. Apparently, this is the wrong approach. The library I've found ([graphql-java](https://github.com/graphql-java/graphql-java)) doesn't work that way. It works by adding a servlet directly to the server. Sorry if that is totally greek. Let's just say I was taking the approach of trying to build out a REST end point that just happens to have some GraphQL functionality. This was the wrong approach and what I needed to do was actually just add an HTTP route to a bit of logic and the libraries handle everything for me.

The ["How to GraphQL with Java"](https://www.howtographql.com/graphql-java/0-introduction/) tutorials were extremely helpful. This showed me how to wire up the library correctly and it gave me enough snipplets to figure out what needed to go where. So let's get started.

# Server Configuration

So for my approach I wanted to try and use the [embeddded Grizzly Java server](https://javaee.github.io/grizzly/) because that is what I've been using lately as a server inside of my JVM projects. This made things a little bit more complicated because it was intended to be used as a JAX-RS server. Which meant that all the information currently out there has information on how to set that up. But for this particular case I actually needed a servlet. This was definitely going off the well beaten path. A few posts later and some tinkering it does work. 

The first part I figured out was getting the GraphQL schema parsed. This file, which is embedded inside of the application, needs to be parsed out to figure out what data you have. Along with that, you then provide a [root resolver](https://github.com/graphql-java/graphql-java-tools#root-resolvers) that will contain the logic of how a request is intercepted and executed. This looks like this:

```kotlin
// Setting up graphql schema, servlet, and bindings to server
val graphqlSchema = SchemaParser.newParser()
        .file("schema.graphqls")
        .resolvers(Query()) // we can add any number of resolvers here
        .build()
        .makeExecutableSchema()
```

OK, that is fairly straight forward. Now we are going to take that schema and create a servlet that we can add to our server. This looks like this:

```kotlin
val graphqlServlet = SimpleGraphQLServlet
            .builder(graphqlSchema)
            .build()
```

That isn't rocket science either. However, the next part is what took me the longest. For Grizzly, there is a not well documented class that helps you setup servlets. The [WebAppContext](http://atetric.com/atetric/javadoc/org.glassfish.grizzly/grizzly-http-servlet-server/2.4.3/org/glassfish/grizzly/servlet/WebappContext.html) was the magical find and it really isn't well documented out there unless you look through Java source code (...yuck!...). This class sets up the servlet with all of the things it needs to run (context and bindings if you know servlets). I was hoping for something that just required a few annotations here and there but Grizzly doesn't support that (or at least I couldn't get it working). If you follow along the GraphQL how to, they setup a servlet using that annotation magic. But here, we have to do a little bit more work ourselves. This is how it ended up looking:

```kotlin
val webappContext = WebappContext("Graphql Context", "/")

webappContext.addServlet("GraphQL Endpoint", graphqlServlet)
             .addMapping("/graphql")

val server = HttpServer.createSimpleServer()

webappContext.deploy(server)
```

I've included the server portion as well because you have to "deploy" the webapp context to a server. For these few lines it took a surprising amount of work to get it there. But in the end I like it. It is a straight forward setup that anyone could follow along if they need too.

Now the next thing I wanted is to provide an interface that will allow users to play with the API. This is like the [swagger](https://swagger.io) interface people are familiar with for REST end points. For GraphQL the interface usually used is [GraphiQL](https://github.com/graphql/graphiql). To setup GraphiQL is relatively easy. You copy the `index.html` file from their github repository, place it in your code base. Tweak the file to point to a CDN, and bam. Done.  But, to get the Grizzly server to serve up that file took a little bit of coxing. Again, you need to add a servlet that will serve up that static HTML file correctly. This actually took a [StackOverflow question](https://stackoverflow.com/questions/20924739/grizzly-server-with-static-content-and-rest-resource) to get me on the right path. From that it looks like I needed the `CLStaticHttpHandler` class which looks like this:

```kotlin
server.serverConfiguration
          .addHttpHandler(CLStaticHttpHandler(Thread.currentThread().contextClassLoader), "/")
```

This really isn't ground breaking but there it is. The hard part is getting the pieces together in the right order. But that's why there are blog posts. To help spread this kind of information.

And that's it! We now have our server configured and ready to rock and roll.

# Embedding the schema and html files in the JAR

So, we have our Java code working fine, now we need to embeded those HTML files and GraphQL schema files inside our jar for the services to use. This is really straight forward but I wanted to point out that I personally don't like the default folder structure of Java projects. So I of course have to make life hard and tweak everything. Here is what I tweaked in gradle to make HTML files be in the folder `src/html` and graphql schema files be at `src/graphql`

```gradle
sourceSets {
    main.resources.srcDirs += ['src/html', 'src/graphql']
}
```

This sets the resources directories manually (in gradle, a resource is anything that isn't code) and places them correctly in the jar file.

# The whole shabang

So here is the final repository if you want to see everything working: [https://github.com/baens/blog-experimental-graphql-servlet](https://github.com/baens/blog-experimental-graphql-servlet)

You can download and fire up the server through `./gradlew run` or even use the Docker image and use that. In the end all this does is do the same thing the how to GraphQL site does, but the pieces above are where the work was at. The resolver and repository are nothing magical at this point and follow the examples laid out everywhere else.

# Note: magic is currently needed to make a few things work

So, hopefully this section won't be needed much longer but I do want to point out that there is a hack to make things work. I stumbled [upon a bug](https://github.com/graphql-java/graphql-java-servlet/issues/61) that would crash the GraphQL processing with this particular server. The details are a little messy, but until this issue has been resolved we need to get a custom build of the servlet library. Now this is where things are really neat. There is a service out there (how the hell do people do this for free?!) that will take any github repository and build a library for you. [Jitpack.io](https://jitpack.io/) is just absolutely amazing and I want to shout out that this service is just incredible. 

So the magic little bits I had to sprinkle inside of gradle file were

```gradle
repositories {
    maven { url 'https://jitpack.io' }
}

dependencies {
    compile 'com.github.briancullen:graphql-java-servlet:master-SNAPSHOT'
}    
```

Fairly straight forward. Point to the jitpack repository, and point to the repository that has the fix in it. 
