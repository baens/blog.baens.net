---
title: "Setting up a Docker container on a Container-Optimized OS running on the Google Cloud"
date: 2019-02-19
---

Wow, that title looks like a mouth full doesn't it? So why do this? Isn't there Kubernetes for running Docker containers? Well, sometimes I only need just one container running and I really don't want to add the overhead of having to maintain another Kubernetes clusters. Lame excuse, but this works well for a small installation where I just don't need the power of a full Kubernetes cluster.

What exactly is a Container-Optimized OS (COS)? Well, this OS is built by Google to be optimized for being the base operating system of a Kubernetes cluster or even just a simple Docker container. Its built upon the Chromium OS and has a lot of security features baked into it from the onset. For example, the root filesystem is mounted as read only. So the binaries that are on the system don't change, and can't be changed. Go ahead and [read all about it](https://cloud.google.com/container-optimized-os/docs/), the project is a great one.

Here is what I will show you in this blog post. Throughout this blog post, we will use an the base [Nginx Docker image](https://hub.docker.com/_/nginx) as the container we want to run. We will start off by using the `gcloud` command to run this container. Next, we will manually control the setup of how the docker container gets started and run. Then finally, I'll show a [Terraform](https://www.terraform.io/) script that instantiates the whole system.

# Perquisites
This post will assume you have a GCP account all setup and project already carved out. If you haven't maybe follow the [quickstart tutorial](https://cloud.google.com/compute/docs/quickstart-linux), then come back.

*Warning*: I will be showing commands that create resources inside of a Google Cloud environment. That means real resources that cost real money. Be careful if you don't want to be charged.

# Making gcloud do all the work
Let's do something very basic, we will start up a container using the `gcloud` commands to run a container on top of the COS VM. Here is what that `gcloud` command looks like:

```
gcloud compute instances create-with-container nginx-vm --container-image nginx:1.15.8 --tags http-traffic
```

Take note of the external IP address that is spit out after running that command. We will need that in the next step.

Next we will need to expose that instance with a firewall rule:

```
gcloud compute firewall-rules create allow-http-traffic --rules tcp:80 --target-tags http-traffic
```

Once that is complete, go ahead and try and hit the external IP address of that image. You should now see a "Welcome to nginx!" screen.

Before the next part, if you want to delete that image, go ahead and run `gcloud compute instances delete nginx-vm`.

Well that doesn't look hard does it? We can take images from docker hub and push them up, and bam, we can have a nice software platform. However, what happens if we want our own code? Well, I bet that is stored inside a private repository somewhere, so let's create a private docker image and show you how to authenticate and pull that image down.

# Setting up a "private" Docker image

Let's first establish a private docker image that we would like to be running on the COS image. Let's take the base nginx image, and modify it with our own index page to demonstrate we can build and deploy our own image.

Here are the two files you will need, first the new `index.html` file:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to My Custom nginx!</h1>
</body>
</html>
```

And here is what the `Dockerfile` would look like:

```Dockerfile
FROM nginx:1.15.8

COPY index.html /usr/share/nginx/html
```

If you would like to test that locally, run `docker build -t test . && docker run --rm -p 8080:80 test` and point the browser to [http://localhost:8080](http://localhost:8080), you should see the "Welcome to My Custom nginx!" banner.

Let's now take our image and push it up into a repository. This is a little beyond the scope of this blog post, but go ahead and pick somewhere you can upload a docker image that can be accessed over the public internet. I've used gitlab but any registry really would work. Just make sure its accessible to the public and you have to be authenticated. If you need help with that part, reach out to me and I'll attempt to assist.

# Authenticating with the registry

Now that we have our image up in a registry, we need some way to authenticate with that registry on our VM. If you were doing this locally, I bet you ran a `docker login` command at some point to get authenticated with that registry. Well, thats exactly what we are going to do with this VM as well.

So how do we going about running that command? Well, COS has a toolkit installed called [cloud-init](https://cloudinit.readthedocs.io/en/latest/). `Cloud-init` is a set of tools to manage cloud images. These help provide ways to create startup scripts and the like to get your cloud image running in exactly the way you want. They are used on most cloud providers and most distros have hooks or tools that can be used. For this particular case we will be interested tapping into providing user scripts that can run at startup. For this, we will be providing a `user-data` metadata variable that provides a `cloud-init` configuration. Sounds easy right? Well, let's go!

Let's take a look at what this `cloud-init` configuration file may look like here:

{{< highlight yml "linenos=table">}}
users:
- name: cloudservice
  uid: 2000

write_files:
- path: /etc/systemd/system/myservice.service
  permissions: 0644
  owner: root
  content: |
    [Unit]
    Description=My Service
    Requires=docker.service network-online.target
    After=docker.service network-online.target

    [Service]
    Environment="HOME=/home/cloudservice"
    ExecStartPre=/usr/bin/docker login %REGISTRY% -u %USERNAME% -p %PASSWORD%
    ExecStart=/usr/bin/docker run --network=host --rm --name=myservice %DOCKER_IMAGE%
    ExecStop=/usr/bin/docker stop myservice
    Restart=on-failure
    RestartSec=10

    [Install]
    WantedBy=multi-user.target

runcmd:
- systemctl daemon-reload
- systemctl enable --now --no-block myservice.service
{{< / highlight >}}

This configuration file is used as a template that will generate us a configuration file for our `cloud-init` process. Let's point out some of the more important lines then I'll explain what the variables are

**Line 1 - 3**: Sets up a user dedicated to running this service. This creates a security sandbox because this user will not have any permissions assigned to it except to run this container.

**Line 5 - 24**: This is the file contents of `/etc/systemd/system/myservice.service`. As the path suggests, this is a `systemd` service file that will run our docker container. This section has information about the file permissions as well as the actual content of the file embedded in this configuration.

**Line 10 - 13**: Sets the description of the service as well as any requirements the service needs to run. In this case, we want the network to be online and docker to be running before this service starts.

**Line 15 - 21**: Sets how the service starts and stops. This is some of the more important bits. First, we are setting up the home directory to be the user that we had created earlier. Kind of sandboxing and isolating where this is running. Next the `ExecStartPre` is where some of this magic starts to come into place. This is where the `docker login` command is executed. All of the parameters are currently variables and can be replaced but this is where the magic happens. Next the `ExecStart` actually runs the `docker run` command. One important thing to note is the `--network=host` flag. We want the docker container to be using the host's network interfaces instead of creating the normal isolated network stack. This way the container can act just like the host on the network and have all ports exposed without any further magic.

**Line 20 - 21**: This sets up the service to restart on failure and wait 10 seconds between each retry

**Line 26 - 28**: This section runs once the configuration has been read and all other parts are completed. This acts like a startup script. First we reload the `systemd` daemon to read in our configuration files, then we enable and queue up startup of our service. This is really important to note,the `--now --no-block` commands are important here because we want the service to start up now, but we also want it queued in the `systemd` process so that it waits for the Docker service and network connects to be online. Otherwise our container may not start right because Docker isn't ready or we can't reach out to the internet to get our container.
