---
title: "Adding tests to Kotlin & JAX-RS (Jersey) project"
date: 2018-05-27
---

I have a [blog post](https://blog.baens.net/posts/step-by-step-kotlin-jaxrs-hello-world/) about setting up a JAX-RS with Kotlin and one thing I neglected to show was how to test. So, let's go back and fix this. For this, I will be using [JUnit 5](https://junit.org/junit5/) for the test runner, [AssertJ](http://joel-costigliola.github.io/assertj/) as the assertion library, and [Mockito](http://site.mockito.org/) as the mocking framework.

Let's go over quickly what tests I've found valuable as I've created JAX-RS services. Well, in reality, there's only two: one that exercises the HTTP interface (so, "integration test") and one that tests the class and methods in isolation (so, "unit tests"). I find it more valuable to tell which parts of the system are being exercised in the tests rather than trying to identify what type of common test level they are at. 

# Setting up the dependencies in gradle
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

# Setting up the structure
First, let's do a little bit of work to setup the inversion of control system that is used the JAX-RS version I used (Jersey), which happens to be H2. We are going to create a place we can inject bindings and determine if these are test bindings, or if they are production bindings. This gives us a layer of control of which systems to use and makes it easier for testing.  

A basic binding implementation for H2 looks like this:

```kotlin
class Bindings : AbstractBinder() {
    override fun configure() {

    }
}
```

This is where we will house all of our bindings inside of that `configure` method. We will get into more of what that looks like a little bit later.

Now, let's actually get something useful. Let's create an interface that will hand back a list of data. So, something like this:

```kotlin
data class HelloJson(val prop1: Int, val prop2: String)

interface DataService {
    fun all() : List<HelloJson>
}
```

Nothing fancy, just a method that would give us back a list of data objects. 

# Notes on how to make IntelliJ work with this

So, IntelliJ doesn't like the source along side the tests files. And honestly, I've found that to be a much better project structure in every language I'ved used. So, to make this work (though very kludgy) here is the special stuff you need to add to the `build.gradle` file:

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

This is intended only for IntelliJ. I will try and test Eclipse here soon and report back but I've honestly haven't touched that in years. 

# HTTP Test: Verify end point returns 200

One of the very first tests I write when creating end points is that I can get a 200 back from the end point and get JSON back. This test will be my happy path test, so it should always pass. As requirements grow for this end point I will go back and add things that may be required to make this test pass by default. This is what these tests may look like:

```kotlin
class `HTTP HelloWorldResource should` : JerseyTest(Application()) {
    @Test
    fun `returns 200`() {
        val statusCode = target("helloWorld").request().get().status
        assertThat(statusCode).isEqualTo(200)
    }
    
    @Test
    fun `returns JSON type`() {
        val mediaType = target("helloWorld").request().get().mediaType
        assertThat(mediaType).isEqualTo(MediaType.APPLICATION_JSON_TYPE)
    }
}
```

_**Note**_: that `@Test` annotation is from the JUnit 4 namepsace `import org.junit.Test`. If you don't have this and instead use the newer JUnit 5 namespace, the test will not run. That is because the `JerseyTest` base class assumes that certain things run that JUnit 5 doesn't do anymore (think static class setup process and such). You can be really neat and rename the JUnit 4 import like this: `import org.junit.Test as TestV4` to avoid confusion if you put these types of tests in the same file. 

As you can see, this test isn't that horrible. It is fairly straight forward and the boilerplate around the test isn't bad. Let's go into a more complicated example with bindings.

# Setting up a null binding as your first implementation

When I start adding layers to my application, I usually start with drilling down from my upper layers into the inner layers. Because of that I usually don't have an implementation per say to use for testing or for anything much else. I kind of like this approach because I can test out the API to see if I indeed want this abstraction and what kind of behavior I expect. With all of that in mind, usually when I create an interface I will always have a _null object_ right along side it. This will make it easier to start off with some default behavior and makes it easier to test the system down the road. 

For example, a the null object for the `DataService` defined above may look like this:

```kotlin
class NullDataService : DataService {
    override fun all(): List<HelloJson> = emptyList()
}
```

# Method behavior test: Testing that I interact with the DataService correctly

Now that we have this null object, let's say we need to add functionality to our `helloWorld` method so that it returns the data from this service. With a TDD approach, the first test maybe will just check that the method returns the same number of items. The next test may ensure that we actually have an item contained in the returning list. The tests are fairly straight forward so let's look at the code for it:

```kotlin
@ExtendWith(MockitoExtension::class)
class `HelloWorldResource Should` {

    fun createHelloJson(prop1: Int = 1, prop2: String = "test") = HelloJson(prop1, prop2)

    @Mock
    lateinit var mockDataService: DataService

    @InjectMocks
    lateinit var helloWorldResource: HelloWorldResource


    @Test
    fun `returns same number of elements as DataService`() {
        whenever(mockDataService.all()).thenReturn(listOf(createHelloJson(), createHelloJson()))

        assertThat(helloWorldResource.helloWorld()).hasSize(2)
    }

    @Test
    fun `contains item from returned list`() {
        val expectedObject = createHelloJson(prop1 = 2)

        whenever(mockDataService.all()).thenReturn(listOf(expectedObject))

        assertThat(helloWorldResource.helloWorld()).contains(expectedObject)
    }
}
```

Few things to point out here. I am using the `MockitoExtension` that allows us to reduce on the setup parts of the tests. This allows us to create properties with the `@Mock` and `@InjectMocks` annotations. This makes reuse in the test class VERY easy. The `lateinit` part is because Kotlin really wants classes to show when they will be initialized. This statement flags to the compiler that we will be doing it behind the scenes later and not to worry about things. 

I've also demonstrated here how to create a method that will allow easy recreation of data test. As you see at the top, we have a `createHelloJson` function that creates the data object. Then if we want to override different properties, we can easily (as demonstrated in the 2nd test). This allows the objects to have sane defaults, and if that object grows, makes it easy to add more fields without having to change 30 tests.

# Making the HTTP tests work after we add the DataService constructor dependency

Now, if you are following along you will notice that the above two tests work just fine, but if you run the HTTP test now, things break. That's because we now require the `DataService` to be injected at runtime from H2. To fix that we need to make sure we have our bindings correct, so here is what the test binding and injection will look like

```kotlin
// in our Application.kt file
class TestBindings : AbstractBinder() {
    override fun configure() {
        bind(NullDataService()).to(DataService::class.java)
    }
}

// in our HelloWorldResource.kt file
class HelloWorldResource
@Inject constructor(private val dataService: DataService)

// in our HelloWorldResource.tests.kt file
class `HTTP HelloWorldResource should` : JerseyTest(Application(TestBindings()))
```

First we needed to add the `@Inject` property to the constructor of our resource. This will signal to Jersey that this constructor will need outside dependencies and to use H2 to find them. Our dependency is defined in our `TestBindings` class, with the `NullDataService` as our default. Then we add that test binding to our `JerseyTest` creation making this all work.

# Wrapping up

Now, if any of this didn't make sense and you want to see it all put together, go over to the [github repository](https://github.com/baens/kotlin-jax-rs-helloworld-with-testing) where this is housed and you should be able to take a look at the full thing working. I've created a static list implementation of that data service for one to look at what an actual implementation may look like. I've also thrown in a exception handler so you can see exceptions pop up if you actually have any for easier debugging. 
