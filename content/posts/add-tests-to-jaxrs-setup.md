---
title: "Adding tests to Kotlin & JAX-RS (Jersey) project"
date: 2018-03-26
---

I have a [blog post]() about setting up a JAX-RS with Kotlin and one thing I neglected to add was how to test. So, let's go back and fix this. For this, I will be using JUnit 5 for the test runner, AssertJ as the assertion library, and Mockito as the mocking framework.

Let's go over quickly what tests I've found valuable as I created JAX-RS services. Well, in reality, there's only two: one that exercises the HTTP interface (so, "integration test") and one that tests the class and methods in isolation (so, "unit tests"). While I am avoiding the common words here, I have found it more valuable to tell what parts of the system I am trying to put under test. For my JAX-RS services, have a test that simulates a HTTP test is very valuable, as well as the expected testing at the method level with everything else in isolation. 

Using the example app I have created, we will set up all the necessary tooling and build chain to execute these tests. Then we will explore the two types of testing and the different components to each test. 

#Setting up the dependencies in gradle
First, we need to make sure we are on gradle 4.6 or greater. I personally use the wrapper version, but just verify that you have it somehow. To get that specific wrapper version, you can run `gradle wrapper --gradle-version 4.6` and then all things should be good there.

For the actual code dependencies, here is what you will need:

```gradle
def junitVersion = "5.1.0"

dependencies {
    // junit dependencies
    testCompile "org.junit.jupiter:junit-jupiter-api:${junitVersion}"
    testRuntime "org.junit.vintage:junit-vintage-engine:${junitVersion}"
    testRuntime "org.junit.jupiter:junit-jupiter-engine:${junitVersion}"
    
    // jersey test dependencies for HTTP testing
    testCompile "org.glassfish.jersey.test-framework.providers:jersey-test-framework-provider-grizzly2:${jerseyVersion}"
    
    // assertJ
    testCompile 'org.assertj:assertj-core:3.10.0'
    
    // mockito
    testCompile 'org.mockito:mockito-core:2.18.3'
    testCompile 'org.mockito:mockito-junit-jupiter:2.18.3'
    testCompile 'com.nhaarman:mockito-kotlin:1.5.0'
    
}
```

Kind of messy isn't it? So a note on some of these. Any of the `org.junit` ones are the core JUnit libraries you include for JUnit 5. We've include the `junit-vintage-engine` because for our tests which exercise the HTTP tests are still based upon the older JUnit 4 tests from the `jersey-test-framework-provider`. We will get into that more later. 

I've included [AssertJ](http://joel-costigliola.github.io/assertj/) as the assertion library because I like the syntax better than the out of the box JUnit 5 style. 

[Mockito](http://site.mockito.org/) is the mocking framework, along with a few nice to have from `mockito-kotlin` that turn the `any()` option into an actual instance instead of a `null` object. And `mockito-junit-jupiter` which allows us to run Mockito with JUnit 5. 

#Setting up the structure
First, let's do a little bit of work to setup the inversion of control system that is used in Jersey, which happens to be H2. So, let's create a class that will house the bindings we will use. For the sample project, we need to add an implementation of a `AbstractBinding` that will have the different parts of it. To do that, first you create a binding class like this

```kotlin
class Bindings : AbstractBinder() {
    override fun configure() {

    }
}
```

This is where we will house all of our bindings inside of that `configure` method. Now, let's actually get something useful. Let's create an interface that will hand back a list of data. So, something like this:

*src/Dataservice.kt*
```kotlin
data class HelloJson(val prop1: Int, val prop2: String)

interface DataService {
    fun all() : List<HelloJson>
}
```

Nothing fancy, just a method that would give us back a list of data objects. 

#Notes on how to make IntelliJ work with this

So, IntelliJ doesn't like the source along side the tests files. And honestly, I've found that to be a much better project strcuture in every language I'ved used. So, to make this work (though clonky) here is the special stuff you need to add to the `build.gradle` file:

```gradle
apply plugin: 'idea'd

// This is here to fix the intellij's issues with
// the complicated source mappings. Use the 'idea' job
// to generate a intellij project
// Ticket to make this just work out of the box: https://youtrack.jetbrains.com/issue/IDEA-188436
idea {
    module {
        scopes.COMPILE.plus += [ configurations.testCompile ]
    }
}
```
