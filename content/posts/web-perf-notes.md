I was asked recently by a friend on what advice to look at when looking at web performance. I was flattered. I did some work on [Overstock.com](https://www.overstock.com) on helping improve the performance and this these are my notes to him.

# First, who are the users

When starting out with any adventure, you need to explain the why. For web performance, we need to figure out why we are trying to improve the performance to begin with. We need to understand what our current base line metrics are, and what we are even setting out to do.

This requires first to understand our users. Our users are using the product, so what are they experiencing. Are they even experiencing what you think they are? Do your best to get metrics around the problem. Find ways to report back how the actual user sees your site. This isn't exactly easy, but without metrics, we will be flying blind in the next steps. We can't just rely on what we see, we need to see what the user sees.

How do you go about that? There are several decent 3rd party vendors out there to help collect these kind of stats from users that I would recommend starting there. Don't reinvent that wheel, find a 3rd party you can drop in and today start collecting metrics from everyone who visits your application.

# Established base line

Now that we have the baseline of what the users are seeing, we can now start putting down milestones of what we want to improve. Every time you do any kind of performance improvement, you need to know where you start. So you know if you are either helping or hindering the system. You need to make sure you just aren’t doing performance for performance sake, otherwise you’ll never reach a goal and just spin wheels.

You have a few gates you have to pay attention to:
* Time to first byte
* Time to first paint
* Time to first interaction

There's many mini gates between those, but these are the big 3 that you can focus and usually get an understanding of what your web page is doing.

# Time to first byte

This means how fast are you getting content to the browser. In my experience, this is usually where most of the work has to be done, but that has changed over my career. You need to understand what your floor is, so to speak. If you want your site to load under 500ms with everything. Yet, the content of your site isn't getting to the user till 800ms. Tweaking HTML or CSS all day isn't going to make your site faster. You need to deliver the content itself faster to your user.

Now, there isn't a magic bullet here. This will require that your backend systems perform decent and that you are delivering content within a decent time as well. If your content is really getting deliver as fast as possible, and the backend is doing everything it can. Yet, your speed still isn't there. You may want to consider caching. Caching is one of the techniques to get things faster. It is also the fastest way to shoot yourself in the foot. Caching things so you don't have to do the work over and over will definitly speed things up. But be careful, [caches are part of the two only hard problems in Compute Science](https://martinfowler.com/bliki/TwoHardThings.html)

If caching can't be done, or isn't helping. Find ways to offload the work somewhere else. May even after the page is loaded. The whole point is to get the content faster to your user, so they see something is working. If they see most of the page there, and have to wait a second for some little partial data. Do it. Sometimes cutting down the first bite is what you need.

# Time to first paint

This is where black freaking magic comes in. There are so many ways to screw up the browser painting process, structure of HTML, blocking content. This requires you to understand how and when a browser decides to paint, and like I said. Its black magic. Chrome and Firefox tools are soo good now though, they can really help you dive in and watch how the browser paints things.

Inspect your timeline view, and show the screenshots of each rendered frame. It's massively interesting, and you may find things that are blocking that you didn't even realize! There is so much that goes into rendering a frame, I'm sure you will find something out as you watch your site render.

Turn off Javascript and see what painting does. Javascript can block the rendering pipeline in unique ways, so first understand just how the content is being rendered. And yes, that would mean you are delivering a full page to the user, not a bootstrap program that then does the rendering. If you care about speed, you need to return a full page of content, not just the application. Server side rendering is a thing, it may be time to pull that out.


# Time to first interaction

This is the hardest one to move the needle. This is all about your javascript application, and what can you make come after the pages load, and what comes before.

Learn how to do async loaded tags, find if you can load things AFTER that page is fully rendered and available.

Lazy load is your friend, tweak things and see where it lands. I usually fight for trying to get rid of as much Javascript as possible, but its not always a winnable battle.
