---
title: "Talking to a HTTP server via Telnet"
date: 2018-01-07
---

# Have you ever tried to talk to a HTTP server via Telnet?

As web developers, either front end or backend, we should strive to at least strive to have a understanding of the technologies we build our success on. And I've found through out the years that a surprising number of web developers don't know how it works. So I thought it was fitting to start by looking at the basics of the communication mechanism we build upon and use every day, HTTP.

## Small word about the world of HTTP RFCs

Standards: that beautiful thing we all strive for as programmers. We want to know that we are doing the right thing at the right time, and people much brighter then we have paved that path already. There is a beautiful process that started very early on how exactly certain things were going to be set in stone and work over the internet. I highly recommend reading [the wikipedia page on the RFCs](https://en.wikipedia.org/wiki/Request_for_Comments), or even read up on the [spec of how to make a RFC](https://tools.ietf.org/html/rfc2026)! 

If you've never looked at the what makes up the HTTP standard, I encourage you to start with [RFC 7231](https://tools.ietf.org/html/rfc7231) or it's parent [RFC 7230](https://tools.ietf.org/html/rfc7230). Yes, its long and there are a lot of pieces to it. But you may learn something about HTTP and what it can actually do. 

## Let's get our hands dirty

I am going to be using [HTTPBin.org](https://httpbin.org) in these following examples. This was one of the better, simpler services out there that I found as a good playground to work with.

Now, lets make our first HTTP call with telnet:

```shell
> telnet httpbin.org 80
```

You should see something like this:

```plain
Trying 23.23.209.130...
Connected to httpbin.org.
Escape character is '^]'.
```

If you are at this point, excellent! We have established a TCP connection to a server that DNS has identified as _httpbin.org_ on port 80! Fairly neat right?

Now, lets make the actual HTTP request. Type the following then hit enter **twice** after:

```http
GET /ip HTTP/1.1
Host: httpbin.org
```

You will hopefully see something like this in return:

```http
HTTP/1.1 200 OK
Connection: keep-alive
Server: meinheld/0.6.1
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.000519037246704
Content-Length: 33
Via: 1.1 vegur

{
  "origin": "<ip>"
}
```

If you have the above, congratulations! You have now successfully done what browsers do all day, by hand! Hopefully that is exciting, and if you've never had a clue, at least eye openning.

So, what in the hell have we just done. Let's first go over the first part which is the request.

## The Request

{{< highlight http "linenos=table" >}}
GET /ip HTTP/1.1
Host: httpbin.org
{{< / highlight >}}

This is a HTTP request. After we've established a connection to port 80 on a server, we can send plain text of our request to a server. That very first line is called the `start-line` of the HTTP request and is defined in [RFC 7230 Section 3.1.1](https://tools.ietf.org/html/rfc7230#section-3.1.1). What it basically says that you get the following: 

`<HTTP Method (i.e. GET, POST, PUT)> <URI> HTTP/1.1`

So in our example `GET` is the method. Our URI is `/ip`. The HTTP version is `1.1`. Fairly straight forward. 

The next lines are defined as the request header. Which can be 0 or more depending on what you are doing. The header we sent in the above example is only 1 line which is `Host: httpbin.org`. To put it simply, this says please send to host named `httpbin.org`. This is because multiple servers could be hosted at that IP and this is what routes us to the right server. Again, that is glossing over the details greatly, but that is why we included it in our example. If we had not included the host header, this particular server returns with an error. 

## The Response

{{< highlight http "linenos=table" >}}
HTTP/1.1 200 OK
Connection: keep-alive
Server: meinheld/0.6.1
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.000519037246704
Content-Length: 33
Via: 1.1 vegur

{
  "origin": "<ip>"
}
{{< /highlight >}}

There are 3 major parts to a response (and if you want to follow the spec [Section 3 of RFC 7230](https://tools.ietf.org/html/rfc7230#section-3)) The *start-line*, the *headers* then the *body*. If you've been following along this may seem highly similar to a request, and you would be right. A request and a response are nearly the same thing.

In a response, the *start-line* is actually the *status-line*. This has information about what exactly the server has done with the request in a numerical and a textual representation. So in the example above `Line 1` shows `200 OK`. The **200** is the numerical status and **OK** is the text repesentation. A table of the commonly used ones are in [RFC 7231 Section 6](https://tools.ietf.org/html/rfc7231#section-6). 

The *headers* are lines 2 - 10, and are used as meta information about the response. All kinds of information can be passed in through here. We have an example of [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) information on line 5 - 6, [Media Type](https://en.wikipedia.org/wiki/Media_type) information on line 4, and something we will explain here in a minute line 2 has information about the connection.

The *body* follows the header. One important thing to note is that you should see a blank line like on line 11. This separates the header from the body. In this case the body is a JSON object that represents information about the IP we were requesting from.


## Connection: Keep-alive

Now, you may have noticed if you ran the previous example that your telnet session didn't close and just stayed openned to a blank line. This is because by default a HTTP/1.1 connection trys to stay open so a client could make a request again to the same serer. I'm not going to go into too much detail on exactly what this is important, but wanted to point it out. A way to make that NOT happen is to add the following after the `Host` line:

`Connection: close`

This instructs the server that after the response is sent, to close the connection.  If you add that line, you should receive a message saying something like `Connection closed by foreign host.`

## A POST 

Now that we've established the basics, lets try to another HTTP method, POST. A POST message is basically a GET message, but with a body in the request. As we established, a request and a response are made up of the same parts, they just begin in a different way. A POST demonstrates that because we are going to have a body now in the request. So, start up the telnet session again, connect to the server and type the following:

```http
POST /post HTTP/1.1
Host: httpbin.org
Connection: close
Content-type: application/json
Content-length: 14

{"test":true}
```

Now if all goes well hopefully you got something like this back in return:

{{< highlight http "linenos=table" >}}
HTTP/1.1 200 OK
Connection: close
Server: meinheld/0.6.1
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0.000699043273926
Content-Length: 326
Via: 1.1 vegur

{
  "args": {},
  "data": "{\"test\":true}\r",
  "files": {},
  "form": {},
  "headers": {
    "Connection": "close",
    "Content-Length": "14",
    "Content-Type": "application/json",
    "Host": "httpbin.org"
  },
  "json": {
    "test": true
  },
  "url": "http://httpbin.org/post"
}
{{< /highlight >}}

If it didn't go well lets try and explain why that may have not gone well. First, as you may have noticed there are a fair bit more pieces to a POST. And with telnet, there are a lot of ways to screw this up. Telnet sends the content to the server as soon as you hit enter, so the server will try and process the information as soon as you do that. For this reason you have to be more careful about how you send it to the server over what code would have to worry about. The **Content-length** header is very, very important. If you even have extra whitespace or newline characters, it won't work and you may get 5xx errors or other problems. If you notice on line 14 of the output, you will see an extra return character (I ran this example on a mac). So, if you have issues, try again and make sure to type EXACTLY the same thing. Or fiddle with content-length. If all else fails, try Postman or another HTTP tool and see what it sends over. 

Some other highlights from this are to note that on line 14 you can see exactly what was sent to the server, extra character and all. Along with all of the headers and other information. It even parsed the JSON information we passed to it because we set the `Content-type` to `application/json`.

And this is it, a POST request.

## Conclusion

Alright, I hope you enjoyed this post. Hopefully you learned a little bit, and even tried the examples. Please drop me a line or such if you have any questions or comments. Thanks for reading!

