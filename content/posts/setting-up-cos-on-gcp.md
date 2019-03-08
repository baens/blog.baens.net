---
title: "Setting up a Docker container on a Container-Optimized OS running on the Google Cloud"
date: 2019-02-19
---

Wow, that title looks like a mouth full doesn't it? So why do this? Isn't there Kubernetes for running Docker containers? Well, sometimes I only need just one container running and I really don't want to add the overhead of having to maintain another Kubernetes clusters. Lame excuse, but this works well for a small installation where I just don't need the power of a full Kubernetes cluster.

What exactly is a Container-Optimized OS (COS)? Well, this OS is built by Google to be optimized for being the base operating system of a Kubernetes cluster or even just a simple Docker container. Its built upon the Chromium OS and has a lot of security features baked into it from the onset. For example, the root filesystem is mounted as read only. So the binaries that are on the system don't change, and can't be changed. Go ahead and [read all about it](https://cloud.google.com/container-optimized-os/docs/), the project is a great one.

Here is what I will show you in this blog post. Throughout this blog post, we will use a base [Nginx Docker image](https://hub.docker.com/_/nginx) as the container we want to run. We will first use some short cuts the `gcloud` command provides to run a container on this image. We will then go through what would be required to get a image from a private repository up and running. Let's get started!

All source code is [hosted on github]() as well so you don't have to retype if you don't want too.

# Perquisites
This post will assume you have a Google cloud account all setup and a Google cloud project already carved out. If you haven't maybe follow the [quickstart tutorial](https://cloud.google.com/compute/docs/quickstart-linux), then come back.

*Warning*: I will be showing commands that create resources inside of a Google Cloud environment. That means real resources that cost real money. Be careful if you don't want to be charged.

# Making gcloud do all the work
Let's do something very basic, we will start up a container using the `gcloud` commands to run a container on top of the COS VM. Here is what that `gcloud` command looks like:

```
gcloud compute instances create-with-container nginx-vm --container-image nginx:1.15.8 --tags http-traffic
```

Take note of the external IP address that is spit out after running that command. We will need that in the next step.

Next we will need to expose that instance with a firewall rule:

```
gcloud compute firewall-rules create allow-http-traffic --target-tags http-traffic --allow tcp:80
```

Once that is complete, go ahead and try and hit the external IP address of that image. You should now see a "Welcome to nginx!" screen.

Before the next part, if you want to delete that image, go ahead and run `gcloud compute instances delete nginx-vm`.

Well that doesn't look hard does it? We can take images from docker hub and push them up, and bam, we have a running docker container running in the cloud. However, what happens if we want our own code, or code not hosted on the Docker hub? Let's create a private docker image and show you how to authenticate and pull that image down.

# Setting up a custom Docker image

Let's establish a private docker image that we would like to be running on the COS image. We will take the base nginx image, and modify it with a custom index page to demonstrate we can build and deploy the custom image.

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

Let's now take our image and push it up into the private repository. This is a little beyond the scope of this blog post, but go ahead and pick somewhere you can upload a docker image that can be accessed over the public internet. I've used gitlab but any registry really would work. Just make sure its accessible to the public. If you need help with that part, reach out to me and I'll attempt to assist.

# Authenticating with the registry

Now that we have our image up in a registry, we need some way to authenticate with that registry on our VM. If you were doing this locally, I bet you ran a `docker login` command at some point to get authenticated with that registry. Well, thats exactly what we are going to do with this VM as well.

So how do we going about running that command? Well, COS has a toolkit installed called [cloud-init](https://cloudinit.readthedocs.io/en/latest/). `Cloud-init` is a set of tools to manage cloud images. These help provide ways to create startup scripts to get your cloud image running in exactly the way you want. They are used on most cloud providers and most distros have hooks or tools that can be used. For this particular case we will be interested in providing user scripts that can run at startup. For this, we will be providing a `user-data` metadata variable that provides a `cloud-init` configuration. Sounds easy right? Well, let's go!

Let's take a look at what this `cloud-init` configuration file may look like here:

{{< highlight yml "linenos=table">}}
#cloud-config

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
- iptables -A INPUT -p tcp -j ACCEPT
- systemctl daemon-reload
- systemctl enable --now --no-block myservice.service
{{< / highlight >}}

This configuration file is used as a template that will generate us a configuration file for our `cloud-init` process. Let's point out some of the more important lines then I'll explain what the variables are:

**Line 1**: This flags this file as a cloud config file. If you don't have this, the system won't pick it up and process it.

**Line 3 - 5**: Sets up a user dedicated to running this service. This creates a security sandbox because this user will not have any permissions assigned to it except to run this container.

**Line 7 - 26**: This is the file contents of `/etc/systemd/system/myservice.service`. As the path suggests, this is a `systemd` service file that will run our docker container. This section has information about the file permissions as well as the actual content of the file embedded in this configuration.

**Line 12 - 15**: Sets the description of the service as well as any requirements the service needs to run. In this case, we want the network to be online and docker to be running before this service starts.

**Line 17 - 23**: Sets how the service starts and stops. This is some of the more important bits. First, we are setting up the home directory to be the user that we had created earlier. Kind of sandboxing and isolating where this is running. Next the `ExecStartPre` is where some of this magic starts to come into place. This is where the `docker login` command is executed. All of the parameters are currently variables and can be replaced but this is where the magic happens. Next the `ExecStart` actually runs the `docker run` command. One important thing to note is the `--network=host` flag. We want the docker container to be using the host's network interfaces instead of creating the normal isolated network stack. This way the container can act just like the host on the network and have all ports exposed without any further magic.

**Line 22 - 23**: This sets up the service to restart on failure and wait 10 seconds between each retry

**Line 28 - 31**: This section runs once the configuration has been read and all other parts are completed. This acts like a startup script. First we setup the internal firewall to accept TCP connections using `iptables`. Then we reload the `systemd` daemon to read in our configuration files, then we enable and queue up startup of our service. This is really important to note,the `--now --no-block` commands are important here because we want the service to start up now, but we also want it queued in the `systemd` process so that it waits for the Docker service and network connects to be online. Otherwise our container may not start right because Docker isn't ready or we can't reach out to the internet to get our container.

Now, what are all these variables for:

**%REGISTRY%**: This defines the root domain of where the registry is. For example, in testing this, I used gitlab so the registry would have been `registry.gitlab.com`.

**%USERNAME%**: This is the username to be logged in as. Keep with the gitlab theme, I used a deploy token and the username was given to me for the deploy token.

**%PASSWORDD%**: This is the password or token that can be used to log in.

**%DOCKER_IMAGE%**: This is the docker image path. I.e. `registry.gitlab.com/myimage/test:1`.

Now, how do I run this? Well, replace all of the variables with the right values and then run this command:

```
gcloud compute instances create test-nginx --image cos-stable-72-11316-136-0 --image-project cos-cloud --tags http-traffic --metadata-from-file user-data=cloud-init-config.yml
```

If you take that IP address that is spit out by that command, you should see the "Welcome to My Custom nginx!" banner. If not, maybe read through the troubleshooting section. If not, go ahead and skip down to the Terraform section.

# Notes on troubleshooting

Alright, the site didn't come up. Now what? Well, let's first SSH into the box so we can start investigating what happened. To do that, let's make sure we have the firewall open by running this command: `gcloud compute firewall-rules create allow-ssh-traffic --allow tcp:22`. This should open up the firewall to allow you to SSH into the box by executing the following command `gcloud compute ssh <box name, i.e. test-nginx>`.

Now that you are on the box, what exactly are you looking for? Well, for our docker example we want to make sure we have the docker container up and running so we would issue a `docker container ls` command to verify it is up and running. If it is up and running, then you may have a firewall issue. Visit the [console](console.cloud.google.com) and inspect the network interface and see if the ingress is setup correctly. What we would like to see is that port 80 is open and working right. If it is, maybe the `iptables` didn't run as expected so verify on the VM instance itself you can do something like `curl localhost`. If that works, verify `iptables` is correctly setup.

But what if the container isn't even running? Where do I go from there? Well, we need to investigate the logs and start sifting through what may have happened. For this the `sudo journalctl` command is what you start poking around in. This gives you all of the logs of the box as it starts. Another command for our example is the `sudo systemctl status myservice` this shows you the raw status of the service, if its up or down, and what may be happening right now with the service.

Another thing to check that I ran into a lot is first very that the cloud init script is even running correctly. A great way to check for that is to verify that the files we were expecting are even there. In this example, we would expect to see a file at `/etc/systemd/system/myservice.service`. If that isn't there, usually the problem is that the very first line doesn't read `#cloud-config` EXACTLY. Double check your config and consult the logs for `user-data` parsing and retrieving.

# Conclusion

Hopefully I've given you a taste of how to setup a single Docker container instance using the Container-Optimized OS on the Google Cloud. I've found this a useful feature when I don't really need all the horse power of a kubernetes instance. This lightered weight infrastructure for a few containers provides to be useful and cost effective in managing some light architecture. But also be warned, these instances are obliviously not replicated in anyway, so if one goes down, it is all down. Be careful in the cloud because every stack should be built on the assumption that cloud resources could come and go at will.
