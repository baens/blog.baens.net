---
title: "Notes from experimenting with JAX-RS and Streams"
date: 2018-02-24
---

So, this idea is a slight continuation on from my notes of how to get a stream out of JDBC using other libraries. 

The goal here is to make it so I can send a stream of objects out with JAX-RS (Jersey). This makes it possible to have lazy evaluation of a stream of data from a database that gets pushed up to the browser as soon as it comes off the database.

I am going to use my starter JAX-RS project as my start point and I'll show where I've changed things here. There is also the [full repository]() to look at to see what everything looks like in the end.

So without further ado, let's first create the resource that will have the end point of something rather easy. 

```kotlin
@Path("stream")
class StreamResource {
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    fun stream() = Stream.of(1, 2, 3, 4 ,5)
}
```

If we run this, we get this kind of output

```json
{"parallel":false}
```

Well that isn't right! It looks like the Stream is just getting passed back and parsed as the object. The data itself isn't being iterated on.

To fix this, we need to create a [new body writer](https://jersey.github.io/documentation/latest/message-body-workers.html#d0e6550). This is how JAX-RS knows how to translate things into what ever media output it needs. For our purposes we are going to make a Stream to JSON body writer. 

Here is what the looks like

```kotlin
@Provider
class StreamMessageBodyWriter : MessageBodyWriter<Stream<Any>> {
    override fun isWriteable(type: Class<*>,
                             genericType: Type,
                             annotations: Array<out Annotation>,
                             mediaType: MediaType): Boolean {
        return mediaType == MediaType.APPLICATION_JSON_TYPE
    }

    override fun writeTo(stream: Stream<Any>,
                         type: Class<*>,
                         genericType: Type,
                         annotations: Array<out Annotation>,
                         mediaType: MediaType,
                         httpHeaders: MultivaluedMap<String, Any>,
                         entityStream: OutputStream) {
        val factory = JsonFactory(ObjectMapper())

        val generator = factory.createGenerator(entityStream, JsonEncoding.UTF8)

        generator.writeStartArray()

        stream.forEach {
            generator.writeObject(it)
            entityStream.flush()
        }

        generator.writeEndArray()
        generator.close()
    }
}
```

This body writer matches on when the media type is JSON. Then for the writing part, we are using the Jackson object mapper to product a stream of JSON formatted data. We iterate through the stream, writing out the JSON representation to the output stream, and that's it!

This should give you this

```json
[1,2,3,4,5]
```

There we go, much better.

Now, while this looks good, we also have to make sure that we are doing the right thing. And by right thing, I mean taking care of the trash. As I mentiod and demonstrated in my Stream JDBC post, we need to clean up a lot of things from a JDBC operation. So, how would this look like in this context. Well, as it turns out rather easy. Here look:

```kotlin
override fun writeTo(stream: Stream<Any>,
                         type: Class<*>,
                         genericType: Type,
                         annotations: Array<out Annotation>,
                         mediaType: MediaType,
                         httpHeaders: MultivaluedMap<String, Any>,
                         entityStream: OutputStream) {
    val factory = JsonFactory(ObjectMapper())

    val generator = factory.createGenerator(entityStream, JsonEncoding.UTF8)

    generator.writeStartArray()

    stream.use {
        it.forEach {
            generator.writeObject(it)
            entityStream.flush()
        }
    }

    generator.writeEndArray()
    generator.close()
    }
}
```

That `use` statement is our signal to the stream of where we want the end to be considered. It will execute that block, and once it is exited, it will run the clean up operations. And when I say clean up operations this is how it would look like on the other side of the stream:

```kotlin
fun stream() = Stream.of(1, 2, 3, 4 ,5).onClose { println("Closing") }
```

Easy! We can run what ever cleanup operations from there that we need and life will be awesome. 

Now, this has me thinking. This is definitely the happy path, what happens if there are exceptions or such? No idea what kind of exceptions could occur, but there are probably some in the JDBC pipeline that we could easily think up of. So lets try and solve this by saying, if there is an exception, we will try and "recover" in that we will send what we have, and that is it. So if we had intended to send 10 items, but item 4 bombs out, we should hopefully have the first 3 already sent along.

So let's try and reconstruct that and see what happens. First, lets try and make that same stream of 10 but controlling the loops this time. A method for that would look like this in Kotlin:

```kotlin
fun controlledSequence() = buildSequence {
    for (i in 1..5) {
        yield(i)
    }

    throw Exception("test")
}
```

Now this is some black freaking Kotlin magic. Here is what a [`buildSequence`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/build-sequence.html) from a documentation perspective. But in lay terms, its a way to make a lazy evaluating list of items. As you can see the `yield` statement, which should be familiar too you if you are from C# or python. Silly that you have to do a special block, but hey its the JVM. It is know for how verbose it can be.

Ok let's wire this up to that stream method and double check our output.

```kotlin
fun stream() = controlledSequence().asStream()
``` 

Easy, neat. That will give us the same output as before.
 