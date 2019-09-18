---
title: "Step by Step: A simple Node.js, Docker, and Kubernetes setup"
date: 2019-09-21
---

I've now been playing with Node.js, Docker, and Kubernetes for quite some time. And it just so happen that today someone needed a good introduction to Node.js, Docker, and Kubernetes. However, after searching online I couldn't find one that just had a few simple things to walk through. So, here this is. Hopefully this blog post will demonsrate how to create a simple Node.js, create a Docker container, demonstrate it running, then deploy that Docker container to a local Kubernetes setup. There will be light touches on what exactly all of those parts are and hopefully give you a starting point to start exploring these technology stacks.

# Step 0: Prerequisites

I am going to assume a few things in this blog post. First, you have `[Node.js](https://nodejs.org/)` installed. I prefer to use `[nvm](https://github.com/nvm-sh/nvm)` as my manager of my node instance, but there are several out there that can do the trick. For this blog post, I will be using the latest LTS Dubnium release 10.16.3. I will also be using `[yarn](https://yarnpkg.com/)` as the Node.js package manager.

Next, we will need `[Docker](https://www.docker.com/)` installed. If you are using Mac or Windows, go ahead and get the wonderful [Docker for Mac/Windows](https://www.docker.com/products/docker-desktop) tools. This will give you a wonderful set of tools to use Docker on those platforms. For Linux, go aheads and get a [Docker CE](https://docs.docker.com/v17.12/install/#server) from what ever distro package you have. For this blog post, I will be running Docker for Mac 2.1.3.0. I will also verify that it works on Linux, but sadly don't have a way to verify Windows at this time. There isn't anything too complicated here so I have some confidence that it should work across platforms fairly easily.

Next, we will need a Kubernetes instance running locally. For Mac and Windows, that is built into the [Docker for Desktop](https://docs.docker.com/docker-for-mac/#kubernetes) tool. For Linux, I recommend `[Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)`.

That should be all of the base tools you will need. Hopefully those are all fairly easy to install, but if you run into issues, please reach out to me and I'll attempt to help and add notes to this blog post for future vistors.

# Step 1: A basic node server running

First thing first, let's setup our environment with a very basic Node.js [Express](https://expressjs.com) server and get it running. Get to a blank directory and run the following command:

```bash
> yarn init -y
```

This should give you a simple `[package.json](https://docs.npmjs.com/files/package.json)` with some nice defaults. I won't expunge too much here about this but if this is now confusing. I may stopping here and exploring a few more tutorials on Node.js and some fundementals.

Next, let's get our `Express` library. We do that by running the following command:

```bash
> yarn add express@4.17.1
```

_Rant_: Now, if you are familiar with the Node.js ecosystem you may find it very odd that I added a specific version of the express library. First, you should definitly try and lock your packages down to as specific version as you can. Personally, I've been bitten far too many times by drifting dependencies. Yes, the lock files help this, but it still happens from time to time. So try and lock things down to as specific as possible. I hope you will thank me later, and I'm sad that the Node community uses fuzzy versions far to often in my opinion.

This should install the `Express` library and create a `yarn.lock` file and a `node_modules` folder with all the files needed for that library. Now that we have `Express`, let's create a very simple server. Here is what you want in the file `index.js`:

```javascript
const express = require('express');

const app = express();

app.get('/', (request, response) => response.send('Hello World'));

app.listen(8080, () => console.log('Running server'));
```

Let's go ahead and run this file by executing `node index.js`. You should get the `Running server` output on the console and then you can visit [http://localhost:8080](http://localhost:8080) and see the `Hello World` text in the web browser. If you do, congradulations! We have a very simple web server up and running. If not, double check that you have the package installed correctly, and that your `index.js` is in the same folder as the `package.json` and `node_modules` folder. Please reach out if you need help getting past this step so I can help troubleshooting steps.

# Step 2: Dockerize

Now that we have some working code, let's go ahead and get this application stuffed into a Docker container. Create a file named `Dockerfile` and put this inside of it:

{{< highlight Dockerfile "linenos=table" >}}
FROM node:10.16.3 as builder

WORKDIR /build
COPY . .
RUN yarn install
RUN yarn install --production

FROM node:10.16.3-slim

WORKDIR /app

COPY --from=builder /build/node_modules ./node_modules/
COPY --from=builder /build/index.js .

CMD node index.js
{{< / highlight >}}

Let's go through this line by line to understand what we are doing:

*Line 1:* Very first thing you do in a Dockerfile is define where the starting point is. For us, we are going to use the Node with our locked in version. Now, something you may not be familiar with is the `as builder`. We are going to use what is called a [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/). This is slightly overkill for our example, but this is a framework for future work. We are going to use a builder that will build up our application. Then we will copy over the smallest amount of bits we absolutely need for a production system. This way we have the smallest image we need to ship into production. Also from a security perspective, we are shipping the smallest amount of thing so our foot print is as small as possible.

*Line 3:* The `[WORKDIR](https://docs.docker.com/engine/reference/builder/#workdir)` command sets our default working from and also sets where we are currently working from. We are going to use a folder at the root called `build` and work from there

*Line 4:* First we are copying over everything into our Docker container with a neat little trick of `COPY . .`. Now, this may look funny so let me explain what kind of magic this is doing. Remember that we are asking the Docker system to copy things into the Docker environment. So the first parameter in `COPY` is referencing from the filesystem relative to the `Dockerfile`. The second parameter is referencing in relation to where in the Docker container it should put those files. For us, we are asking to copy everything from our project, into the Docker container. It's a neat trick I employee instead of trying to copy different folders. If I need to exclude things (which, you will), you will use the `[.dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file)` file.

*Line 5-6:* Now, this looks VERY odd, but just hang in there with me. First we use `yarn install` to get all of the dependencies. While, yes, the very next line we do `yarn install --production`, I do this for a good reason. More likely then not, you will want a build step to do something. Either packing, compiling, transpiling. Take your pick. You can add any step in between those two `yarn install` commands to get the right build system setup that you need.