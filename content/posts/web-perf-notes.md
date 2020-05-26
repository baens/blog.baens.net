So first thing first, who are your users? What are they on? Are they mobile, are they PC, what are they, and why do you care about performance in the first place?

You need to make sure you just aren’t doing performance for performance sake, otherwise you’ll never reach a goal and just spin wheels

once you’ve identified what your performance goals are, then we start looking at where we are currently, using tools like lighthouse (which is far better then what we had when we started this)

You have a few gates you have to pay attention to:
Time to first byte
Time to first paint
Time to first interaction

theres many mini gates between those, but these are the big 3 that you can focus and usually get an understanding of what your web page is doing


time to first byte:
How fast are you getting your HTML to the browser. In my experience, this is usually where most of the work has to be done, but that has changed over my career. If your first byte doesn’t get to you till 700-900ms, and you want to be under 500ms….huh, that ain’t going to happen in this physical plane. So make sure your first byte is getting to you fast enough. Usually, this the wins here are caches, put the right caches in the right places, and all of a sudden things are lightning fast (oh, and welcome to shooting yourself in the foot. There are only two hard problems in computer science: naming things, and invalidating caches)

Time to first paint:
This is where black freaking magic comes in. There are so many ways to screw up the browser painting process, structure of HTML, blocking content. This requires you to understand how and when a browser decides to paint, and like I said. Its black magic. The chrome and firefox tools are soo good now though, they can really help you dive in and watch how the browser paints things. Inspect those, look at how things look per frame. Its massively interesting, and you may find things that are blocking that you didnt realize
Turn off javascript and see what painting does, sometime javascript slows it down, make sure you have the best content first before you are adding scripts. Hell, get rid of javascript entirely if you can!

Time to first interaction
This is the hardest one to move the needle. This is all about javascript application, and what can you make come after the pages load, and what comes before. Learn how to do async loaded tags, find if you can load things AFTER that page is fully rendered and available. Lazy load is your friend, tweak things and see where it lands. I usually fight for trying to get rid of as much Javascript as possible, but its not always a winnable battle.
