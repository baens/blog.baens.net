---
title: "Step by Step: A simple Node.js, Docker, and Kubernetes setup"
date: 2019-11-16
---

I've now been playing with Node.js, Docker, and Kubernetes for quite some time. And it just so happen that recently someone needed a good introduction to Node.js, Docker, and Kubernetes. However, after searching online I couldn't find one that just had a few simple things to walk through. So, here this is. Hopefully this blog post will demonstrate how to create a simple Node.js, create a Docker container, demonstrate it running, then deploy that Docker container to a local Kubernetes setup. There will be light touches on what exactly all of those parts are and hopefully give you a starting point to start exploring these technology stacks.

# Step 0: Prerequisites

I am going to assume a few things in this blog post. First, you have [Node.js](https://nodejs.org/) installed. I prefer to use [nvm](https://github.com/nvm-sh/nvm) as my manager of my node instance, but there are several out there that can do the trick. For this blog post, I will be using the latest LTS Dubnium release 10.16.3. I will also be using [yarn](https://yarnpkg.com/) as the Node.js package manager.

Next, we will need [Docker](https://www.docker.com/) installed. If you are using Mac or Windows, go ahead and get the wonderful [Docker for Mac/Windows](https://www.docker.com/products/docker-desktop) tools. This will give you a wonderful set of tools to use Docker on those platforms. For Linux, go aheads and get a [Docker CE](https://docs.docker.com/v17.12/install/#server) from what ever distro package you have. For this blog post, I will be running Docker for Mac 2.1.3.0. I will also verify that it works on Linux, but sadly don't have a way to verify Windows at this time. There isn't anything too complicated here so I have some confidence that it should work across platforms fairly easily.

Next, we will need a Kubernetes instance running locally. For Mac and Windows, that is built into the [Docker for Desktop](https://docs.docker.com/docker-for-mac/#kubernetes) tool. For Linux, I recommend [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).

That should be all of the base tools you will need. Hopefully those are all fairly easy to install, but if you run into issues, please reach out to me and I'll attempt to help and add notes to this blog post for future visitors.

# Step 1: A basic node server running

First thing first, let's setup our environment with a very basic Node.js [Express](https://expressjs.com) server and get it running. Get to a blank directory and run the following command:

```bash
> yarn init -y
```

Next, let's get our `Express` library. We do that by running the following command:

```bash
> yarn add express@4.17.1
```

***Rant***: Now, if you are familiar with the Node.js ecosystem you may find it very odd that I added a specific version of the express library. First, you should definitely try and lock your packages down to as specific version as you can. Personally, I've been bitten far too many times by drifting dependencies. Yes, the lock files help this, but it still happens from time to time. So try and lock things down to as specific as possible. I hope you will thank me later, and I'm sad that the Node community uses fuzzy versions far too often in my opinion.

This should install the `Express` library and create a `yarn.lock` file and a `node_modules` folder with all the files needed for that library. Now that we have `Express`, let's create a very simple server. Here is what you want in the file `index.js`:

```javascript
const express = require('express');

const app = express();

app.get('/', (request, response) => response.send('Hello World'));

app.listen(8080, () => console.log('Running server'));
```

Let's go ahead and run this file by running the following in a command prompt: `node index.js`. You should get the `Running server` output on the console and then you can visit [http://localhost:8080](http://localhost:8080) and see the `Hello World` text in the web browser. If you do, congratulations! We have a very simple web server up and running. If not, double check that you have the package installed correctly, and that your `index.js` is in the same folder as the `package.json` and `node_modules` folder. Please reach out if you need help getting past this step so I can help troubleshooting steps.

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

*Line 4:* First we are copying over everything into our Docker container with a neat little trick of `COPY . .`. Now, this may look funny so let me explain what kind of magic this is doing. Remember that we are asking the Docker system to copy things into the Docker environment. So the first parameter in `COPY` is referencing from the filesystem relative to the `Dockerfile`. The second parameter is referencing in relation to where in the Docker container it should put those files. For us, we are asking to copy everything from our project, into the Docker container. It's a neat trick I employ instead of trying to copy different folders. If I need to exclude things, you will use the `[.dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file)` file.

*Line 5-6:* Now, this looks VERY odd, but just hang in there with me. First we use `yarn install` to get all of the dependencies. While, yes, the very next line we do `yarn install --production`, I do this for a good reason. More likely then not, you will want a build step to do something. Either packing, compiling, transpiling, take your pick. You can add any step in between those two `yarn install` commands to get the right build system setup that you need.

Now that we have a docker image, let's just go through and test this docker image and make sure things work just like they did in the last step. First, let's build the docker image by running `docker build . -t myimage`. The `-t myimage` tags the image with a name we can easily use.

To run the image you just built run `docker run --rm -it -p 8080:8080 myimage`. You should be able to hit [http://localhost:8080](http://localhost8080) and get the same `Hello World` text like you did in the last time. hit `ctrl+c` to stop the image.

# Step 3: Pushing a docker image and prep work for kubernetes

In this tutorial, I am going to assume you have a kubernetes instance up and running somewhere. If you don't, you can either use [Docker for Desktop](https://www.docker.com/blog/kubernetes-is-now-available-in-docker-desktop-stable-channel/) which has Kubernetes built in for both Mac and Windows. Or, you can use [minikube](https://github.com/kubernetes/minikube).

No matter where you have it running. This tutorial will assume you have `kubectl` pointed to a running Kubernetes instance and that you also have a registry you can upload your docker image.

Let me actually go into detail a little bit about that last thing. We need to push the Docker image to a registry for your Kubernetes instance to pull down. Now, there a wide range of place you can do that. And that require a wide variety of different methods to do it. I am going to assume that you can `docker push` some kind of image somewhere and that is accessible to your Kubernetes cluster. If you and running the Docker for Desktop tool, a `docker build` will suffice. If you are running Minikube, you will need to [reuse the Docker daemon](https://minikube.sigs.k8s.io/docs/tasks/docker_daemon/). If you are running a cluster in the cloud somewhere, you will have to make sure that [Kubernetes is setup to pull from that registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

# Step 4: Deploying that image to Kubernetes

With your image now ready to deploy, lets go through what that would require. For this tutorial we are going to create a [deployment](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) and a [service](https://kubernetes.io/docs/concepts/services-networking/service/).

A deployment is a Kubernetes object that defines how to create "[pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)". A pod is a single (but can be multiple) runner Docker instance. A deployment controls how many pods are currently running and has all of the logic built into making sure there are enough pods to satisfy your requirements. It also helps control roll outs as you update your image. This means that as you roll out a new image, it will bring a new pod up, make sure the pod is running, and then kill off old pods in a controlled manner. Deployments are usually your bread and butter, but they aren't the only objects that control pods. There are a [few different types](https://kubernetes.io/docs/concepts/architecture/controller/) of controllers out there but this tutorial will only be focused on the deployment variety.

So, if a deployment controls whats running inside of Kubernetes, how do we expose that pod to network traffic? Like maybe public internet traffic? That is where services come in. A service is a Kubernetes object that controls how network connections are made to the pods. A service defines which ports are open and are connected, and whether the pods should be exposed internally to the Kubernetes instance or externally. Services can also do [load balancing](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) if you desire.

Now, while this glossed over a lot of details, I think this should make you dangerous enough to start with. Let's look at how a deployment and service object are created and deployed to Kubernetes now. Let's take a look at this file:

{{< highlight yml "linenos=table" >}}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: myimage
        imagePullPolicy: Never
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
{{< / highlight >}}

Holy crap batman thats alot! Let's walk through what all of this means.

*Line 1 & 24*: For this example I put both objects inside of one file. Not always a normal thing todo, but its an option. The `---` is a YAML file separator for multiple YAML objects inside of a file. Just want to point this out first if you see these files separated in the wild. Thats fine, I just wanted to give you one file to play with instead of multiple.

*Line 2, 3, 25 & 26*: This describe the type of Kubernetes object. There are two parts to this. The `apiVersion`, and the `kind` of object. These set of properties let's Kubernetes define a whole host of options and let's them version out behavior for certain objects. You can find which objects are support by running `kubectl api-resources` and the versions of those with `kubectl api-versions`. The resources list which API group is used, which you cross-reference to which version you should use. If the resource is listed blank, its part of "core" which is usually just `v1`. You usually don't fiddle with this much, and just copy from project to project. But its better to be aware of why this is here then just blindly copying it.

*Line 4 - 7*: This section describes the metadata for the deployment. Metadata is just that, information about the object. For a deployment there are two main parts, a `name` which is exactly that, and is required. Then some kind of `label`. The label is important because this gives you the ability to "select" this deployment depending on what kind of values you give the object. This will become important later on in our service.

*Line 8*: This starts the meat of the deployment object, the `spec` or specification of what you want to deploy.

*Line 9*: The `replicas` is the number of instances you want running.

*Line 10 - 12*: This section describes what pods the deployment controls. Usually this means you create a selector that has the same matching labels as your `template` section. I personally haven't come across a case where this didn't match up with what I had in the `template` section, but I'm sure there are cases out there.

*Line 13*: This is the start of the `template` section. The template section will describe what each pod will have. This includes the image of the container, along with any environment variables, files, etc that are needed to run that pod.

*Line 14 - 16*: This section contains the `metadata` for each pod that is run. Again, usually this just contains the a label that has information for your selector in the above section.

*Line 17*: This defines the `spec` for a pod. In this example we will have only 1 container, but this is the section we would add information for an [`initContainer`](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) or [side car containers](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#how-pods-manage-multiple-containers).

*Line 18 - 23*: This is the meat of the pod. We define a `name`, a `image`, and the `ports` that are exposed. The name can be whatever you want, it doesn't necessarily have to match the deployment name, but usually does for making life easier later. The `image` is the location of the docker image. In this example I am assuming that you are using the Docker for Desktop tool, which means we can give it the same name as the last step (`myimage`). I also added a `imagePullPolicy` because the Kubernetes instance inside of that tool should not try and reach out to the internet for this image. I would recommend [reading up on which image pull policy](https://kubernetes.io/docs/concepts/containers/images/#updating-images) is right for your situation. We list the ports that are exposed next. This isn't completely necessarily but usually added for documentation proposes.

*Line 29*: This section defines our service and how it operates. Let's dig into this section now.

*Line 30 - 31*: This defines what pods should be exposed through this service. This usually matches very closely to what the deployment had in its selector as well.

*Line 32*: Since we want to expose this service we want to put a `type` on it. There are a [couple of types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types), and the one we are interested in is the `LoadBalancer`. This is because we want to expose this service outside of Kubernetes, and that requires a load balancer for that.

*Line 33 - 36*: This defines the ports that are going to be exposed from this service. For our example, we are going to take the pods port 8080 (`targetPort`) and expose it to the outside world on that same port 8080 (`port`). We could have exposed it on port 80 if we wanted too. But for this instance, we just went for the easy route of aligning those numbers up.

Phew, that is a lot. So what should I do with all of this now? Well, let's deploy it. To do that we would run `kubectl apply -f deploy.yaml`. This of courses assumes that all of the above is in a file called `deploy.yaml`. Kubectl would then submit that file to Kubernetes and the magic starts to happen on creating the pods. To see your pods up and running we would run `kubectl get pods` and _hopefully_ you would see something like this:

```bash
> kubectl get pods
NAME                    READY   STATUS        RESTARTS   AGE
my-app-bb697dc4-q6vl7   1/1     Running       0          14s
my-app-bb697dc4-qpjgf   1/1     Running       0          14s
my-app-bb697dc4-vsxcv   1/1     Running       0          14s
```

As you can see, you see the `name` attribute come through. Along with a deployment number (`bb697dc4` in this example) and a pod number (`q6vl7`, `qpjgf`, and `vsxcv` in this example).

If everything is running, we should then be able to hit the service. To view the status of the service we would run `kubectl get service` and see something like this:

```shell
> kubectl get service
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
my-service   LoadBalancer   10.106.118.92   localhost     8080:32361/TCP   44m
```

If we hit that `External-IP` with the port, we should see the same `Hello World` we saw in the above 2 examples.

# Conclusion

Well, we made it! I know there is a lot in here, and there is definitely a lot more, but hopefully this gives you enough pieces that you can start putting your own software together that can run on Kubernetes. Always feel free to reach out to me if you have question or comments.