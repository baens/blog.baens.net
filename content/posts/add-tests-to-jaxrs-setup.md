---
title: "Adding tests to Kotlin & JAX-RS (Jersey) project"
date: 2018-03-26
---

I have a [blog post]() about setting up a JAX-RS with Kotlin and one thing I neglected to add was how to test. So, let's go back and fix this. For this, I will be using JUnit 5 for the test runner, AssertJ as the assertion library, and Mockito as the mocking framework.

First thing we will need is something to actually test. So let's create an end point that relies on some kind of data service class to give it data to return. We will be doing a test that will exercise the HTTP end point, as well as testing the resource class in isolation. Hopefully this will demonstrate enough pieces that one could use this as an example else where.

#Setting up the dependencies in gradle
First, we need to make sure we are on gradle 3.6 or greater. I personally use the wrapper version, but just verify that you have it somehow. The wrapper version is installed via `gradle wrapper --gradle-version 3.6` which is what I recommend. 

Here are the few dependencies you will need in your gradle file

```gradle
def junitVersion = "5.1.0"

dependencies {
    testCompile "org.junit.jupiter:junit-jupiter-api:${junitVersion}"
    testCompile "org.glassfish.jersey.test-framework.providers:jersey-test-framework-provider-grizzly2:${jerseyVersion}"
    testCompile 'org.assertj:assertj-core:3.9.0'
    testCompile 'org.mockito:mockito-core:2.17.2'
    testCompile 'com.nhaarman:mockito-kotlin:1.5.0'
    testCompile 'name.falgout.jeffrey.testing.junit5:mockito-extension:1.0.0'
    testRuntime "org.junit.vintage:junit-vintage-engine:${junitVersion}"
    testRuntime "org.junit.jupiter:junit-jupiter-engine:${junitVersion}"
}
```

Kind of messy isn't it? So a note on some of these. Any of the `org.junit` ones are the core JUnit libraries you include for JUnit 5. We've include the `junit-vintage-engine` because for our tests which exercise the HTTP tests are still based upon the older JUnit 4 tests from the `jersey-test-framework-provider`. We will get into that more later. 

I've included [AssertJ]() as the assertion library because I like the syntax better than the out of the box JUnit 5 style. 

Mockito is the mocking framework, along with a few nice to have from `mockito-kotlin` that turn the `any()` option into an actual instance instead of a `null` object. And `mockito-extension` which allows us to run Mockito with JUnit 5. 

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
apply plugin: 'idea'

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
