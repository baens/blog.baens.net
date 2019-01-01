---
title: "Dockerize the Kotlin toolchain"
date: 2018-12-31
---

Am I crazy? Yes, yes I am. For what ever reason, I love the idea of a project that I can literally move from computer to computer very few things installed and be able to be productive in that code base. By this, I mean I could install, one, maybe two tools and then I can be productive in a code base. Right now, [Docker](https://www.docker.com/) feels like maybe I can achieve this crazy dream.

With this crazy idea in mind, I decided if I could fully dockerize the [Kotlin](https://kotlinlang.org/) tool chain. In theory I can setup a workstation by installing an editor, git, docker, and bam. I'm now 100% ready to code on this machine. Alright, with this, off we go.

# Step 1: Getting the basic build working

Now, to make things easy, I am going to reuse my [Kotlin base](https://blog.baens.net/posts/step-by-step-kotlin/) and recreate this project with this tool chain. First step, let's replace the `gradlew` file with a docker executed gradle file. It will look something like this:

```bash
#!/usr/bin/env sh

docker run --rm -it -v $(pwd):/workspace -w /workspace gradle:5.0.0-jdk11 gradle $@
```

This is a fairly straight forward `docker` command, but just in case you have never used `docker` before let me touch on some of the basics:

* `docker run` - we are going to run a docker image
* `--rm -it` - remove (`rm`) once completed and add standard input pipes ([`it`](https://docs.docker.com/engine/reference/commandline/run/#assign-name-and-allocate-pseudo-tty---name--it))
* `-v` - mount our current workspace into the directory inside the container at `/workspace`
* `-w` - make the current working direction `/workspace` (i.e. `cd /workspace` at start)
* `gradle:5.0.0-jdk11` - use gradle 5.0.0 with the JDK 11 image (see a whole list of tags on the [gradle docker hub](https://hub.docker.com/_/gradle/))
* `gradle` - execute the gradle command
* `$@` - pass all arguments from the parent shell to this command

Alright, let's test this and see what happens!

<video width="640" autoplay playsinline loop muted>
    <source src="/videos/gradle-first-build.mov" type="video/mp4">
</video>

Now, don't be too alarmed if your output is not _*exactly*_ the same. I had already ran it so the docker download process isn't going, but it should be fairly close after that.

Looks like we have some jars, let's get them into a container and run them!

# Step 2: Put into jar into Docker

Now, I currently had a `Dockerfile` that built the jar, then created the docker image, I am going to continue that idea. Here is what the `Dockerfile` looks like:

{{< highlight dockerfile "linenos=table" >}}
FROM gradle:5.0.0-jdk11 as BUILD

RUN mkdir -p /home/gradle/src
WORKDIR /home/gradle/src
COPY . /home/gradle/src
RUN gradle --no-daemon shadowJar

FROM openjdk:11.0.1-jre

COPY --from=BUILD /home/gradle/src/build/libs/step-by-step-kotlin-all.jar /bin/runner/run.jar
WORKDIR /bin/runner

CMD ["java","-jar","run.jar"]
{{< / highlight >}}

I'm not going to go much into this file since I already explained it in the other blog post, but let me point out that line 3 was special and needed. The permissions were a little wonky inside of the container and I had to make sure the folder was already created correctly. There are many many work around for this, but I didn't want to demonstrate that.

Let's build and run this now!

<video width="640" autoplay playsinline loop muted>
    <source src="/videos/docker-build-and-run.mov" type="video/mp4">
</video>

Excellent! It worked as expected. But dang is it slow. Let's try really quick and fix that. 

# Step 3: build time improvements

Ok, obviously the build loop on this is a tad bit ridiculous. Let me show you one quick improvement for build times. If you watched closely to the video, the downloading of plugins and other artifacts is by the largest amount of time spent. This should be easily fixed if we just cached those artifacts and let gradle discover them each time. The gradle tool does this natively by storing in your home directly this folder, so why don't we just create a folder that can live between each gradle run? Well, here's what the `gradlew` turns into if we want to do exactly that:

```sh
#!/usr/bin/env sh

[ -d .build_cache ] || mkdir .build_cache
docker run --rm -it -v $(pwd):/workspace -v $(pwd)/build_cache:/home/gradle/.gradle -w /workspace gradle:5.0.0-jdk11 gradle --no-daemon $@
```

Nothing wild. Just create the `build_cache` folder and then mount that folder inside of the docker image where it would be expected. My quick tests show we go from 30 seconds to 8 second builds. Not bad.

# Step 4: .....Profit??.... Evaluating if this is practical

Well, now that we have this sort of working and are producing builds, is this really practical? Well, honestly, its pretty close. It works, you don't need many things installed. There are some further speed improvements you could get with some tweaks. I would use (and I sort of use a setup for some projects like this now) this setup as my own setup, but I may not push it on other devs to use it unless they wanted to. But I'm starting to like this kind of setup. Again, there are many hiccups with this approach, but it gets pretty close.

So, this has at least been a fun adventure of seeing if this was possible and I think it shows promise into where things may go. 

# Special Notes

If you checkout the code, you may notice a `.dockerignore` file. This file tells docker what not to include when it is moving files over to the docker context. This is pretty important because the `.gradle` folder doesn't share well across the machines and things start breaking. 
