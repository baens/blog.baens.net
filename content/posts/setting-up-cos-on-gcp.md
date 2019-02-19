---
title: "Setting up a Docker container on a Container-Optimized OS running on the Google Cloud"
date: 2019-02-19
---

Wow, that title looks like a mouth full doesn't it? So why do this? Isn't there Kubernetes for running Docker containers? Well, sometimes I only need just one container running and I really don't want to add the overhead of having to maintain another Kubernetes clusters. Lame excuse, but this works well for a small installation where I just don't need that 100% High Availability.

What exactly is a Container-Optimized OS (COS)? Well, this OS is built by Google to be optimized for being the base image of a Kubernetes cluster or even just a simple Docker container. Its built upon the Chromium OS and has a lot of security baked into it from the onset. For example, the root filesystem is mounted as read only. So the binaries that are on the system don't change, and can't be changed. Go ahead and [read all about it](https://cloud.google.com/container-optimized-os/docs/), the project is a great one.

Here is what I will show you in this blog post. Throughout this blog post, we will use an the base [Nginx Docker image]() as the container we want to run. We will start off by using the `gcloud` command to run this container. Next, we will manually control the setup of how the docker container gets started and run. Then finally, I'll show a [Terraform] script that instantiates the whole system.

This post will asumme you have a GCP account all setup and project already carved out. If you haven't maybe follow the [quickstart tutorial](https://cloud.google.com/compute/docs/quickstart-linux), then come back.

Let's do something very basic, we will start up a container using the `gcloud` commands to run a container on top of the COS VM. Here is what that `gcloud` command looks like:

```
gcloud compute instances create-with-container nginx-vm --container-image gcr.io/cloud-marketplace/google/nginx1:1.12
```