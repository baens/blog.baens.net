---
title: "Adding tests to Kotlin & JAX-RS (Jersey) project"
date: 2018-03-26
---

I have a [blog post]() about setting up a JAX-RS with Kotlin and one thing I neglected to add was how to test. So, let's go back and fix this. For this, I will be using JUnit 5 for the test runner, AssertJ as the assertion library, and Mockito as the mocking framework.

First thing we will need is something to actually test. So let's create an end point that relies on some kind of data service class to give it data to return. We will be doing a test that will excerize the HTTP end point, as well as testing the resource class in isolation. Hopefully this will demonstrate enough pieces that one could use this as an example else where.

First, let's do a little bit of work to setup the inversion of control system that is used in Jersey, which happens to be h2. So, let's create a class that will house the bindings we will use. For the sample project, we need to add an implementation of a `AbstractBinding` that will have the different parts of it.
